---
layout: post
title: "Promethues Scrape 部分源码解析"
date: 2018-09-12
---

在prometheus中，scrape组件负责拉取监控目标的数据，并且把拉取的数据交给storage组件。scrape需要抓取的监控目标由discover组件提供。示意图如下

<div class="mermaid">
graph LR
A[Service Discover] --> B[Scrape]
B --> C[Storage]
</div>

在scrape组件主要分成manager, scrapePool和target三部分。下面我们来看看这三部分的数据结构。
```go
// Manager maintains a set of scrape pools and manages start/stop cycles
// when receiving new target groups form the discovery manager.
type Manager struct {
	logger    log.Logger
	append    Appendable
	graceShut chan struct{}

	mtxTargets     sync.Mutex // Guards the fields below.
	targetsActive  []*Target
	targetsDropped []*Target
	targetsAll     map[string][]*Target

	mtxScrape     sync.Mutex // Guards the fields below.
	scrapeConfigs map[string]*config.ScrapeConfig
	scrapePools   map[string]*scrapePool
}

// scrapePool manages scrapes for sets of targets.
type scrapePool struct {
	appendable Appendable
	logger     log.Logger

	mtx    sync.RWMutex
	config *config.ScrapeConfig
	client *http.Client
	// Targets and loops must always be synchronized to have the same
	// set of hashes.
	targets        map[uint64]*Target
	droppedTargets []*Target
	loops          map[uint64]loop
	cancel         context.CancelFunc

	// Constructor for new scrape loops. This is settable for testing convenience.
	newLoop func(*Target, scraper, int, bool, []*config.RelabelConfig) loop
}


// Target refers to a singular HTTP or HTTPS endpoint.
type Target struct {
	// Labels before any processing.
	discoveredLabels labels.Labels
	// Any labels that are added to this target and its metrics.
	labels labels.Labels
	// Additional URL parmeters that are part of the target URL.
	params url.Values

	mtx        sync.RWMutex
	lastError  error
	lastScrape time.Time
	health     TargetHealth
	metadata   metricMetadataStore
}
```

### Manaager
Manager负责维护scrape pool,并且管理着scrape组件的生命周期。Manager 主要有以下函数。
- func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error
- func (m *Manager) Stop()
- func (m *Manager) ApplyConfig(cfg *config.Config) error

Run函数由main.go中启动的scrape manager goroutine来调用。Run函数需要传入discover组件中的SyncCh channel，以便scrape组件感知discover target的变动。
```go
{
	// Scrape manager.
	g.Add(
		func() error {
			// When the scrape manager receives a new targets list
			// it needs to read a valid config for each job.
			// It depends on the config being in sync with the discovery manager so
			// we wait until the config is fully loaded.
			<-reloadReady.C

			err := scrapeManager.Run(discoveryManagerScrape.SyncCh())
			level.Info(logger).Log("msg", "Scrape manager stopped")
			return err
		},
		func(err error) {
			// Scrape manager needs to be stopped before closing the local TSDB
			// so that it doesn't try to write samples to a closed storage.
			level.Info(logger).Log("msg", "Stopping scrape manager...")
			scrapeManager.Stop()
		},
	)
}
```
Run函数中当，scrape会监听syncCh channel，一旦syncCh channel有message，就会触发manager的reload函数。reload函数中，会遍历message的数据，根据jobName(tsetName)从scrapePools中找，如果找不到，则新建一个scrapePool,如果jobName在scrapeConfig里面找不到，那么就会打印一下错误信息。每一个job会创建一个对应的scrapePool实例。reload函数最后会调用`sp.Sync(tgroup)`来更新scrapePool的信息。通过sync函数，就可以得出哪些target仍然是active的， 哪些target已经失效了。Sync函数的具体实现，会在下一部分ScrapePool中讲到。
```
// Run starts background processing to handle target updates and reload the scraping loops.
func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
	for {
		select {
		case ts := <-tsets:
			m.reload(ts)
		case <-m.graceShut:
			return nil
		}
	}
}

func (m *Manager) reload(t map[string][]*targetgroup.Group) {
	m.mtxScrape.Lock()
	defer m.mtxScrape.Unlock()

	tDropped := make(map[string][]*Target)
	tActive := make(map[string][]*Target)

	for tsetName, tgroup := range t {
		var sp *scrapePool
		if existing, ok := m.scrapePools[tsetName]; !ok {
			scrapeConfig, ok := m.scrapeConfigs[tsetName]
			if !ok {
				level.Error(m.logger).Log("msg", "error reloading target set", "err", fmt.Sprintf("invalid config id:%v", tsetName))
				continue
			}
			sp = newScrapePool(scrapeConfig, m.append, log.With(m.logger, "scrape_pool", tsetName))
			m.scrapePools[tsetName] = sp
		} else {
			sp = existing
		}
		tActive[tsetName], tDropped[tsetName] = sp.Sync(tgroup)
	}
	m.targetsUpdate(tActive, tDropped)
}
```

ApplyConfig函数是prometheus启动时或者reload配置文件信息时用到的。会删除掉reload配置文件前有的但是reload配置文件后没有的job对应的scrapePool实例。并且如果job的配置的信息和reload前不一致的话，也会被reload 对应scrapePool实例的配置。
```
// ApplyConfig resets the manager's target providers and job configurations as defined by the new cfg.
func (m *Manager) ApplyConfig(cfg *config.Config) error {
	m.mtxScrape.Lock()
	defer m.mtxScrape.Unlock()

	c := make(map[string]*config.ScrapeConfig)
	for _, scfg := range cfg.ScrapeConfigs {
		c[scfg.JobName] = scfg
	}
	m.scrapeConfigs = c

	// Cleanup and reload pool if config has changed.
	for name, sp := range m.scrapePools {
		if cfg, ok := m.scrapeConfigs[name]; !ok {
			sp.stop()
			delete(m.scrapePools, name)
		} else if !reflect.DeepEqual(sp.config, cfg) {
			sp.reload(cfg)
		}
	}

	return nil
}
```
### scrapePool
在prometheus scrape组件中，一个job对应一个ScrapePool实例。scrapePool有一下函数。
- func (sp *scrapePool) stop()
- func (sp *scrapePool) reload(cfg *config.ScrapeConfig)
- func (sp *scrapePool) Sync(tgs []*targetgroup.Group) (tActive []*Target, tDropped []*Target)
- func (sp *scrapePool) sync(targets []*Target)

最重要的函数是sync函数。sync会根据输入参数targets列表与原有的targets列表比对，如果有新添加进来的target，则会创建新的targetScraper和loop,并且启动新的loop。sync根据输入参数targets列表与原有的targets列表对比时，也会发现已经失效的target，这部分target， prometheus会stop掉并从列表中删除。如何理解loop呢？ loop其实就是管理scraper的manage。因为每一个loop都是用一个goroutine来run的。所以在loop内可以控制何时进行scraper操作。我们知道prometheus是的拉模型，需要定时到监控目标上拉取相应的数据，loop就是管理何时进行拉取这个操作的。
```go
// sync takes a list of potentially duplicated targets, deduplicates them, starts
// scrape loops for new targets, and stops scrape loops for disappeared targets.
// It returns after all stopped scrape loops terminated.
func (sp *scrapePool) sync(targets []*Target) {
	sp.mtx.Lock()
	defer sp.mtx.Unlock()

	var (
		uniqueTargets = map[uint64]struct{}{}
		interval      = time.Duration(sp.config.ScrapeInterval)
		timeout       = time.Duration(sp.config.ScrapeTimeout)
		limit         = int(sp.config.SampleLimit)
		honor         = sp.config.HonorLabels
		mrc           = sp.config.MetricRelabelConfigs
	)

	for _, t := range targets {
		t := t
		hash := t.hash()
		uniqueTargets[hash] = struct{}{}

		if _, ok := sp.targets[hash]; !ok {
			s := &targetScraper{Target: t, client: sp.client, timeout: timeout}
			l := sp.newLoop(t, s, limit, honor, mrc)

			sp.targets[hash] = t
			sp.loops[hash] = l

			go l.run(interval, timeout, nil)
		} else {
			// Need to keep the most updated labels information
			// for displaying it in the Service Discovery web page.
			sp.targets[hash].SetDiscoveredLabels(t.DiscoveredLabels())
		}
	}

	var wg sync.WaitGroup

	// Stop and remove old targets and scraper loops.
	for hash := range sp.targets {
		if _, ok := uniqueTargets[hash]; !ok {
			wg.Add(1)
			go func(l loop) {
				l.stop()
				wg.Done()
			}(sp.loops[hash])

			delete(sp.loops, hash)
			delete(sp.targets, hash)
		}
	}

	// Wait for all potentially stopped scrapers to terminate.
	// This covers the case of flapping targets. If the server is under high load, a new scraper
	// may be active and tries to insert. The old scraper that didn't terminate yet could still
	// be inserting a previous sample set.
	wg.Wait()
}
```

下面的这个函数便是loop的核心run方法。可以看到，大致逻辑是在应该触发scrape操作的时候，触发`scraper.scrape`函数，进行数据抓取操作。并且将抓取到的数据交给append函数进行存储操作。在scrape前，为了尽量提高性能，prometheus运用了go library中的sync.Pool机制来复用对象。可以看到prometheus对sync.Pool进行了简单的封装,封装后的Pool在`pkg/pool` package中。我们可以把这个封装过得pool理解成重用byte slice的地方。在每一次scrape前，都会向Pool申请和上一次scrape结果一样大小的byte slice,并封装成byte buffer供`scraper.scrape`填写上抓取到的数据。

```go
func (sl *scrapeLoop) run(interval, timeout time.Duration, errc chan<- error) {
	select {
	case <-time.After(sl.scraper.offset(interval)):
		// Continue after a scraping offset.
	case <-sl.scrapeCtx.Done():
		close(sl.stopped)
		return
	}

	var last time.Time

	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	buf := bytes.NewBuffer(make([]byte, 0, 16000))

mainLoop:
	for {
		buf.Reset()
		select {
		case <-sl.ctx.Done():
			close(sl.stopped)
			return
		case <-sl.scrapeCtx.Done():
			break mainLoop
		default:
		}

		var (
			start             = time.Now()
			scrapeCtx, cancel = context.WithTimeout(sl.ctx, timeout)
		)

		// Only record after the first scrape.
		if !last.IsZero() {
			targetIntervalLength.WithLabelValues(interval.String()).Observe(
				time.Since(last).Seconds(),
			)
		}

		b := sl.buffers.Get(sl.lastScrapeSize).([]byte)
		buf := bytes.NewBuffer(b)

		scrapeErr := sl.scraper.scrape(scrapeCtx, buf)
		cancel()

		if scrapeErr == nil {
			b = buf.Bytes()
			// NOTE: There were issues with misbehaving clients in the past
			// that occasionally returned empty results. We don't want those
			// to falsely reset our buffer size.
			if len(b) > 0 {
				sl.lastScrapeSize = len(b)
			}
		} else {
			level.Debug(sl.l).Log("msg", "Scrape failed", "err", scrapeErr.Error())
			if errc != nil {
				errc <- scrapeErr
			}
		}

		// A failed scrape is the same as an empty scrape,
		// we still call sl.append to trigger stale markers.
		total, added, appErr := sl.append(b, start)
		if appErr != nil {
			level.Warn(sl.l).Log("msg", "append failed", "err", appErr)
			// The append failed, probably due to a parse error or sample limit.
			// Call sl.append again with an empty scrape to trigger stale markers.
			if _, _, err := sl.append([]byte{}, start); err != nil {
				level.Warn(sl.l).Log("msg", "append failed", "err", err)
			}
		}

		sl.buffers.Put(b)

		if scrapeErr == nil {
			scrapeErr = appErr
		}

		if err := sl.report(start, time.Since(start), total, added, scrapeErr); err != nil {
			level.Warn(sl.l).Log("msg", "appending scrape report failed", "err", err)
		}
		last = start

		select {
		case <-sl.ctx.Done():
			close(sl.stopped)
			return
		case <-sl.scrapeCtx.Done():
			break mainLoop
		case <-ticker.C:
		}
	}

	close(sl.stopped)

	sl.endOfRunStaleness(last, ticker, interval)
}

```
在`scrapeLoop.run`函数中，我们可以看到，scrapeLoop拿到数据后，会将数据交给append函数。append函数主要在数据交给storage engine前，做一些预处理的工作。 在预处理工作中，prometheus设计了scrapeCache struct,用来追踪从metric string到label sets storage references的对应mapping关系。其次，scrapeCache还会记录两次scrape操作之间重复的那部分数据，以便快速的判断重复的数据，对于重复的数据，就没有必要在此存储到storage engine中了。 append函数首先会把拿到的数据(b)利用textparse parser解析成met(metric的简写)变量，然后判断met是不是重复了，如果重复了就会丢弃这条数据。下面会根据met从scripeCache.series中查找，如果查找到对应的cacheEntity,则会调用appender的AddFast函数进行存储操作。至于存储的细节，超出了scrape的范围，我们会在下面的文章中分析。如果没有查找到对应cacheEntity，那么会调用appender的Add方法进行存储操作，并且把相关信息存储到cache中，以备下一个次scrape的时候进行对比。appender是一个接口，具体有好几种实现存储的方式。类似于mysql中的engine，可以把数据存储在innoDB engine，也可以存储在其他mysql engine中。
```go
// scrapeCache tracks mappings of exposed metric strings to label sets and
// storage references. Additionally, it tracks staleness of series between
// scrapes.
type scrapeCache struct {
	iter uint64 // Current scrape iteration.

	// Parsed string to an entry with information about the actual label set
	// and its storage reference.
	series map[string]*cacheEntry

	// Cache of dropped metric strings and their iteration. The iteration must
	// be a pointer so we can update it without setting a new entry with an unsafe
	// string in addDropped().
	droppedSeries map[string]*uint64

	// seriesCur and seriesPrev store the labels of series that were seen
	// in the current and previous scrape.
	// We hold two maps and swap them out to save allocations.
	seriesCur  map[uint64]labels.Labels
	seriesPrev map[uint64]labels.Labels

	metaMtx  sync.Mutex
	metadata map[string]*metaEntry
}
```
```go
func (sl *scrapeLoop) append(b []byte, ts time.Time) (total, added int, err error) {
	var (
		app            = sl.appender()
		p              = textparse.New(b)
		defTime        = timestamp.FromTime(ts)
		numOutOfOrder  = 0
		numDuplicates  = 0
		numOutOfBounds = 0
	)
	var sampleLimitErr error

loop:
	for {
		var et textparse.Entry
		if et, err = p.Next(); err != nil {
			if err == io.EOF {
				err = nil
			}
			break
		}
		switch et {
		case textparse.EntryType:
			sl.cache.setType(p.Type())
			continue
		case textparse.EntryHelp:
			sl.cache.setHelp(p.Help())
			continue
		case textparse.EntryComment:
			continue
		default:
		}
		total++

		t := defTime
		met, tp, v := p.Series()
		if tp != nil {
			t = *tp
		}

		if sl.cache.getDropped(yoloString(met)) {
			continue
		}
		ce, ok := sl.cache.get(yoloString(met))
		if ok {
			switch err = app.AddFast(ce.lset, ce.ref, t, v); err {
			case nil:
				if tp == nil {
					sl.cache.trackStaleness(ce.hash, ce.lset)
				}
			case storage.ErrNotFound:
				ok = false
			case storage.ErrOutOfOrderSample:
				numOutOfOrder++
				level.Debug(sl.l).Log("msg", "Out of order sample", "series", string(met))
				targetScrapeSampleOutOfOrder.Inc()
				continue
			case storage.ErrDuplicateSampleForTimestamp:
				numDuplicates++
				level.Debug(sl.l).Log("msg", "Duplicate sample for timestamp", "series", string(met))
				targetScrapeSampleDuplicate.Inc()
				continue
			case storage.ErrOutOfBounds:
				numOutOfBounds++
				level.Debug(sl.l).Log("msg", "Out of bounds metric", "series", string(met))
				targetScrapeSampleOutOfBounds.Inc()
				continue
			case errSampleLimit:
				// Keep on parsing output if we hit the limit, so we report the correct
				// total number of samples scraped.
				sampleLimitErr = err
				added++
				continue
			default:
				break loop
			}
		}
		if !ok {
			var lset labels.Labels

			mets := p.Metric(&lset)
			hash := lset.Hash()

			// Hash label set as it is seen local to the target. Then add target labels
			// and relabeling and store the final label set.
			lset = sl.sampleMutator(lset)

			// The label set may be set to nil to indicate dropping.
			if lset == nil {
				sl.cache.addDropped(mets)
				continue
			}

			var ref uint64
			ref, err = app.Add(lset, t, v)
			// TODO(fabxc): also add a dropped-cache?
			switch err {
			case nil:
			case storage.ErrOutOfOrderSample:
				err = nil
				numOutOfOrder++
				level.Debug(sl.l).Log("msg", "Out of order sample", "series", string(met))
				targetScrapeSampleOutOfOrder.Inc()
				continue
			case storage.ErrDuplicateSampleForTimestamp:
				err = nil
				numDuplicates++
				level.Debug(sl.l).Log("msg", "Duplicate sample for timestamp", "series", string(met))
				targetScrapeSampleDuplicate.Inc()
				continue
			case storage.ErrOutOfBounds:
				err = nil
				numOutOfBounds++
				level.Debug(sl.l).Log("msg", "Out of bounds metric", "series", string(met))
				targetScrapeSampleOutOfBounds.Inc()
				continue
			case errSampleLimit:
				sampleLimitErr = err
				added++
				continue
			default:
				level.Debug(sl.l).Log("msg", "unexpected error", "series", string(met), "err", err)
				break loop
			}
			if tp == nil {
				// Bypass staleness logic if there is an explicit timestamp.
				sl.cache.trackStaleness(hash, lset)
			}
			sl.cache.addRef(mets, ref, lset, hash)
		}
		added++
	}
	if sampleLimitErr != nil {
		if err == nil {
			err = sampleLimitErr
		}
		// We only want to increment this once per scrape, so this is Inc'd outside the loop.
		targetScrapeSampleLimit.Inc()
	}
	if numOutOfOrder > 0 {
		level.Warn(sl.l).Log("msg", "Error on ingesting out-of-order samples", "num_dropped", numOutOfOrder)
	}
	if numDuplicates > 0 {
		level.Warn(sl.l).Log("msg", "Error on ingesting samples with different value but same timestamp", "num_dropped", numDuplicates)
	}
	if numOutOfBounds > 0 {
		level.Warn(sl.l).Log("msg", "Error on ingesting samples that are too old or are too far into the future", "num_dropped", numOutOfBounds)
	}
	if err == nil {
		sl.cache.forEachStale(func(lset labels.Labels) bool {
			// Series no longer exposed, mark it stale.
			_, err = app.Add(lset, defTime, math.Float64frombits(value.StaleNaN))
			switch err {
			case storage.ErrOutOfOrderSample, storage.ErrDuplicateSampleForTimestamp:
				// Do not count these in logging, as this is expected if a target
				// goes away and comes back again with a new scrape loop.
				err = nil
			}
			return err == nil
		})
	}
	if err != nil {
		app.Rollback()
		return total, added, err
	}
	if err := app.Commit(); err != nil {
		return total, added, err
	}

	sl.cache.iterDone()

	return total, added, nil
}
```

在提交到storage package 的sotrage engine前，还有两个操作需要注意。
1. timeLimitAppender
2. limitAppender

timeLimitAppender是用来限制data的时效性的。如果某一条数据在提交给storage进行存储的时候，生成这条数据已经超过10分钟，那么prometheus就会抛错。目前10分钟是写死到代码里面的， 无法通过配置文件配置。limitAppender是用来限制存储的数据label个数，如果超过限制，该数据将被忽略，不入存储；默认值为0，表示没有限制,可以通过配置文件中的sample_limit来进行配置。

### target

target部分较为简单，主要封装了一些小方法供manage和scrape使用。没有复杂的逻辑。
```go
// Target refers to a singular HTTP or HTTPS endpoint.
type Target struct {
	// Labels before any processing.
	discoveredLabels labels.Labels
	// Any labels that are added to this target and its metrics.
	labels labels.Labels
	// Additional URL parmeters that are part of the target URL.
	params url.Values

	mtx        sync.RWMutex
	lastError  error
	lastScrape time.Time
	health     TargetHealth
	metadata   metricMetadataStore
}
```

