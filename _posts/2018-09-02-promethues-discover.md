---
layout: post
title: "Promethues Service Discover 服务发现"
date: 2018-09-02
---

### 什么是 Service Discovery

Service Discovery（SD) 是当前分布式系统的一个重要组成部分。具体可参见http://dockone.io/article/509

### Prometheus SD目前支持的平台

1. azure
2. zookeeper
3. consul
4. dns
5. ec2
6. file
7. gce
8. kubernetes
9. marathon
10. openstack
11. triton

本文会选取其中的file, dns以及kubernetes这四种种典型的类型作为例子讲解相关源码。

### Common 逻辑
不论哪种平台，服务发现的逻辑都是类似的， 在prometheus中，首先定义了一个公共逻辑部分，我们在此称为Common逻辑。

首先，定义了Target Group这个数据结构，target group是一组common的label集合。各大平台会以targeting group的形式 发送信息给prometheus。
```go
// Group is a set of targets with a common label set(production , test, staging etc.).
type Group struct {
	// Targets is a list of targets identified by a label set. Each target is
	// uniquely identifiable in the group by its address label.
	Targets []model.LabelSet
	// Labels is a set of labels that is common across all targets in the group.
	Labels model.LabelSet

	// Source is an identifier that describes a group of targets.
	Source string
}
```

当服务发现（如zookeeper)系统当中的相关信息发生变化后，会发送相关信息给prometheus，对应的prometheus必须实现下面的run接口，之后相关的信息就会发送到 `up chan<- []*config.TargetGroup`这个channel了，最后prometheus service就会更新相关的配置。第一次服务发现系统需要发送全部的TargetGroup到这个channel，之后就可以仅仅发送增量部分的change了。 prometheus的服务发现的Manager组件会处理全量和增量这两种情况。
```go
type Discoverer interface {
  Run(ctx context.Context, up chan<- []*config.TargetGroup)
}
```
例如我们的服务发现系统有如下的group信息。

```go
[]config.TargetGroup{
  {
    Targets: []model.LabelSet{
       {
          "__instance__": "10.11.150.1:7870",
          "hostname": "demo-target-1",
          "test": "simple-test",
       },
       {
          "__instance__": "10.11.150.4:7870",
          "hostname": "demo-target-2",
          "test": "simple-test",
       },
    },
    Labels: map[LabelName][LabelValue] {
      "job": "mysql",
    },
    "Source": "file1",
  },
  {
    Targets: []model.LabelSet{
       {
          "__instance__": "10.11.122.11:6001",
          "hostname": "demo-postgres-1",
          "test": "simple-test",
       },
       {
          "__instance__": "10.11.122.15:6001",
          "hostname": "demo-postgres-2",
          "test": "simple-test",
       },
    },
    Labels: map[LabelName][LabelValue] {
      "job": "postgres",
    },
    "Source": "file2",
  },
}
```
在这里我们定义了两个group，一个是从file1得来的，另一个是从file2得来的。每一种实现，必须要保证每一个服务发现系统发送的group里面都应该有唯一的source，这个source应该是所有的group里面是惟一的。

在上里面的例子中， 两个group军事在第一次run的时候发送到prometheus的。对于更新信息来讲， 我们需要发送所有的改变到prometheus。 如果``hostname: demo-postgres-2`这台机器挂掉了，我们应该送一下信息给prometheus。
```go
&config.TargetGroup{
  Targets: []model.LabelSet{
     {
        "__instance__": "10.11.122.11:6001",
        "hostname": "demo-postgres-1",
        "test": "simple-test",
     },
  },
  Labels: map[LabelName][LabelValue] {
    "job": "postgres",
  },
  "Source": "file2",
}
```
如果所有的group都挂掉了， 我们应该发送空的`Targets`给prometheus。比如 `job: postgres`的所有targets都挂掉了， 我们应该发送
```go
&config.TargetGroup{
  Targets: nil,
  "Source": "file2",
}
```

prometheus discover的核心数据结构就是target group，其他都是围绕target group展开的。

西面我们讲讲prometheus中的discover manager。 Manager维护了一系列与服务发现系统的连接，这些服务发现系统发送的所有更新都会发送到syncCh这个chan上。注意syncCh是一个map，map的key是在scrape config中定义的job name. value则是target group数组。具体scrap config 请参见[scrape config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#%3Cscrape_config%3E). Manager struct 中targets负责存放target信息。 poolKey中setName对应配置文件的job name，provider是provider的类型（如dns, kubernetes等） + 在当前job里面的index组成。这样通过poolkey以及source就可以快速找到对应target group 数组。
```go
// Manager maintains a set of discovery providers and sends each update to a map channel.
// Targets are grouped by the target set name.
type Manager struct {
	logger         log.Logger
	mtx            sync.RWMutex
	ctx            context.Context
	discoverCancel []context.CancelFunc
	// Some Discoverers(eg. k8s) send only the updates for a given target group
	// so we use map[tg.Source]*targetgroup.Group to know which group to update.
	targets map[poolKey]map[string]*targetgroup.Group
	// The sync channels sends the updates in map[targetSetName] where targetSetName is the job value from the scrape config.
	syncCh chan map[string][]*targetgroup.Group
}

type poolKey struct {
	setName  string
	provider string
}
```

至此，我们就可以通过Manager struct里面的信息拿到所有的信息了，可是这些信息又是如何被prometheus的其他组件使用的呢？ 在这里我们以scrape组件为例讲解。 从名字就可以看出scrape的主要用来抓取prometheus相关的数据的。 抓取数据第一步肯定要首先知道抓取哪些机器的数据。 这就需要SD提供相应的信息了。 在`/prometheus/cmd/prometheus/main.go`我们可以看到启动scrape manager的时候需要把discover manager的syncCh channel作为参数出到scrape manager里面去，也就是吧SD中的信息传递给scrape manager。

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
scrape manager会监控syncCh channel 的数据变动，一旦channel有新message传入， scrape就会reload 变动的target group。
```go
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
```

那么，prometheus配置文件中SD相关的配置改动的时候，SD会做什么呢？ 比如新加一个job，如果这个job的target group信息的机器是由服务发现系统提供的， 我们可以简单思考一下，肯定是prometheus需要reload配置文件，然后配置文件中的SD相关的信息会传递给discover manager， 然后discover manager会和目标服务发现系统进行通信以获取SD信息。 我们看一下源码。在`/prometheus/cmd/prometheus/main.go`中， reload handler部分， 当通过命令行或者web界面产生reload配置文件命令的时候，prometheus会调用reloadConfig函数进行reload config操作，
```go
		// Reload handler.

		// Make sure that sighup handler is registered with a redirect to the channel before the potentially
		// long and synchronous tsdb init.
		hup := make(chan os.Signal)
		signal.Notify(hup, syscall.SIGHUP)
		cancel := make(chan struct{})
		g.Add(
			func() error {
				<-reloadReady.C

				for {
					select {
					case <-hup:
						if err := reloadConfig(cfg.configFile, logger, reloaders...); err != nil {
							level.Error(logger).Log("msg", "Error reloading config", "err", err)
						}
					case rc := <-webHandler.Reload():
						if err := reloadConfig(cfg.configFile, logger, reloaders...); err != nil {
							level.Error(logger).Log("msg", "Error reloading config", "err", err)
							rc <- err
						} else {
							rc <- nil
						}
					case <-cancel:
						return nil
					}
				}

			},
			func(err error) {
				// Wait for any in-progress reloads to complete to avoid
				// reloading things after they have been shutdown.
				cancel <- struct{}{}
			},
```

reloadConfig 函数会依次调用reloader数组里面的函数。 reloader数组里面的其中一个函数就是`discoveryManagerScrape.ApplyConfig(c)`
```go
func reloadConfig(filename string, logger log.Logger, rls ...func(*config.Config) error) (err error) {
	level.Info(logger).Log("msg", "Loading configuration file", "filename", filename)

	defer func() {
		if err == nil {
			configSuccess.Set(1)
			configSuccessTime.SetToCurrentTime()
		} else {
			configSuccess.Set(0)
		}
	}()

	conf, err := config.LoadFile(filename)
	if err != nil {
		return fmt.Errorf("couldn't load configuration (--config.file=%q): %v", filename, err)
	}

	failed := false
	for _, rl := range rls {
		if err := rl(conf); err != nil {
			level.Error(logger).Log("msg", "Failed to apply configuration", "err", err)
			failed = true
		}
	}
	if failed {
		return fmt.Errorf("one or more errors occurred while applying the new configuration (--config.file=%q)", filename)
	}
	level.Info(logger).Log("msg", "Completed loading of configuration file", "filename", filename)
	return nil
}
```


```go
	reloaders := []func(cfg *config.Config) error{
	...
		func(cfg *config.Config) error {
			c := make(map[string]sd_config.ServiceDiscoveryConfig)
			for _, v := range cfg.ScrapeConfigs {
				c[v.JobName] = v.ServiceDiscoveryConfig
			}
			return discoveryManagerScrape.ApplyConfig(c)
		},
	...
	}
```

下面我们讲一讲具体的例子。
### File

file是一种最简单最原始的服务发现形式。具体来说，就是把target 信息存储到yaml或者json文件中，然后把这个文件路径配置到prometheus 的配置文件中。当这个yaml或者json文件内容有改变的时候，prometheus 会通过watch file的形式感知到target内容的变动。 所以当前在prometheus没有对某些小众的服务发现系统进行集成的情况下， prometheus建议以file这种形式和这些小众的服务发现系统进行集成。

prometheus 会使用[fsnotify](https://gopkg.in/fsnotify/fsnotify.v1)监控在prometheus配置文件中定义的所有的文件。

```go
// Discovery provides service discovery functionality based
// on files that contain target groups in JSON or YAML format. Refreshing
// happens using file watches and periodic refreshes.
type Discovery struct {
	paths      []string
	watcher    *fsnotify.Watcher
	interval   time.Duration
	timestamps map[string]float64
	lock       sync.RWMutex

	// lastRefresh stores which files were found during the last refresh
	// and how many target groups they contained.
	// This is used to detect deleted target groups.
	lastRefresh map[string]int
	logger      log.Logger
}
```

当discover manager调用file.go中的Run方法时， 如果被监控的文件有任何的改动，就会重新读取这些文件，然后生成target group传递给syncCh Channel。
```go
// Run implements the Discoverer interface.
func (d *Discovery) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		level.Error(d.logger).Log("msg", "Error adding file watcher", "err", err)
		return
	}
	d.watcher = watcher
	defer d.stop()

	d.refresh(ctx, ch)

	ticker := time.NewTicker(d.interval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return

		case event := <-d.watcher.Events:
			// fsnotify sometimes sends a bunch of events without name or operation.
			// It's unclear what they are and why they are sent - filter them out.
			if len(event.Name) == 0 {
				break
			}
			// Everything but a chmod requires rereading.
			if event.Op^fsnotify.Chmod == 0 {
				break
			}
			// Changes to a file can spawn various sequences of events with
			// different combinations of operations. For all practical purposes
			// this is inaccurate.
			// The most reliable solution is to reload everything if anything happens.
			d.refresh(ctx, ch)

		case <-ticker.C:
			// Setting a new watch after an update might fail. Make sure we don't lose
			// those files forever.
			d.refresh(ctx, ch)

		case err := <-d.watcher.Errors:
			if err != nil {
				level.Error(d.logger).Log("msg", "Error watching file", "err", err)
			}
		}
	}
}
```

### DNS
TBD

### Kubernetes
TBD
