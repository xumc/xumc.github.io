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
### targetScraper
### scrapeCache
### scrapeLoop
### target

