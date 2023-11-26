# Promethues源码解读

第一次解读一个非常庞大的的项目（这个文件实在是太多太大了，咱也是第一次看这么这么大的开源项目，只能一起加油吧）

[开源项目地址]([prometheus/prometheus: The Prometheus monitoring system and time series database. (github.com)](https://github.com/prometheus/prometheus))， [我的解读地址](暂无)



## 1. Promethues框架



![image-20231126173602851](C:\Users\王浩宇\Pictures\image-20231126173602851.png)

可以看出来这个系统从我们写的定时任务/job 以及自己本身的服务器上拿到日志数据后，在自己的的服务器/服务节点上存储。由于有不同的节点，就需要有相关查询的路由索引（Retrival)，比如k8s的注册中心，或者干脆由dns导航等等，然后存储在硬盘中。需要时可使用自定义的索引语句promQL检索，将结果输出给下游的UI （比如我们常用的监控大盘Grafana 做显示)   

综合来看 prometheus就是分布式节点的时序数据库一样的存在，我初步认为我要关注的有一下几点：

* 数据如何调用回传收集给pomethues？我们在我们的任务与接口里如何调用它
* retrival的结构是什么，如何分配与控制节点 ,如何配置相关规则
* PromQL的数据结构应该长什么样，怎么在ssd中存储，索引代价是多少，为什么不用内存存储
* PromQL的解析，类似SQL的解析么，说明这个项目里难道说有可能有语法解析与语义解析树啊，看起来感觉很有趣

接下来就可以顺着这些问题，顺藤摸瓜的去探究这些问题的答案：



## 2. 代码架构

我讨厌大型开源项目的原因之一就是项目文件超级多，架构比较复杂。不够由于Go语言本身的特性，至少不会出现循环调用这种恶心的事情发生。所以 lets start

<details>  <summary><font size="4" color="black">文件架构 点击展开</font></summary>  <pre><code class="language-cpp">├─cmd
│  ├─prometheus
│  │  └─testdata
│  │      ├─consoles
│  │      └─rules
│  └─promtool
│      └─testdata
├─config
│  └─testdata
│      └─scrape_configs
├─consoles
├─console_libraries
├─discovery
│  ├─aws
│  ├─azure
│  ├─consul
│  ├─digitalocean
│  ├─dns
│  ├─eureka
│  ├─file
│  │  └─fixtures
│  ├─gce
│  ├─hetzner
│  ├─http
│  │  └─fixtures
│  ├─install
│  ├─ionos
│  │  └─testdata
│  ├─kubernetes
│  ├─legacymanager
│  ├─linode
│  ├─marathon
│  ├─moby
│  │  └─testdata
│  │      ├─dockerprom
│  │      │  └─containers
│  │      └─swarmprom
│  ├─nomad
│  ├─openstack
│  ├─ovhcloud
│  │  └─testdata
│  │      ├─dedicated_server
│  │      └─vps
│  ├─puppetdb
│  │  └─fixtures
│  ├─refresh
│  ├─scaleway
│  │  └─testdata
│  ├─targetgroup
│  ├─triton
│  ├─uyuni
│  ├─vultr
│  ├─xds
│  └─zookeeper
├─docs
│  ├─command-line
│  ├─configuration
│  ├─images
│  └─querying
├─documentation
│  ├─examples
│  │  ├─custom-sd
│  │  │  ├─adapter
│  │  │  └─adapter-usage
│  │  ├─kubernetes-rabbitmq
│  │  └─remote_storage
│  │      ├─example_write_adapter
│  │      └─remote_storage_adapter
│  │          ├─graphite
│  │          ├─influxdb
│  │          └─opentsdb
│  ├─images
│  └─prometheus-mixin
├─model
│  ├─exemplar
│  ├─histogram
│  ├─labels
│  ├─metadata
│  ├─relabel
│  ├─rulefmt
│  │  └─testdata
│  ├─textparse
│  ├─timestamp
│  └─value
├─notifier
├─plugins
├─prompb
│  └─io
│      └─prometheus
│          └─client
├─promql
│  ├─fuzz-data
│  │  ├─ParseExpr
│  │  │  └─corpus
│  │  └─ParseMetric
│  │      └─corpus
│  ├─parser
│  │  └─posrange
│  └─testdata
├─rules
│  └─fixtures
├─scrape
│  └─testdata
├─scripts
├─storage
│  └─remote
│      ├─azuread
│      │  └─testdata
│      └─otlptranslator
│          ├─prometheus
│          └─prometheusremotewrite
├─template
├─tracing
│  └─testdata
├─tsdb
│  ├─agent
│  ├─chunkenc
│  ├─chunks
│  ├─docs
│  │  └─format
│  ├─encoding
│  ├─errors
│  ├─fileutil
│  ├─goversion
│  ├─index
│  ├─record
│  ├─testdata
│  │  ├─index_format_v1
│  │  │  └─chunks
│  │  └─repair_index_version
│  │      └─01BZJ9WJQPWHGNC2W4J9TA62KC
│  ├─tombstones
│  ├─tsdbutil
│  └─wlog
├─util
│  ├─annotations
│  ├─documentcli
│  ├─fmtutil
│  ├─gate
│  ├─httputil
│  ├─jsonutil
│  ├─logging
│  ├─osutil
│  ├─pool
│  ├─runtime
│  ├─stats
│  ├─strutil
│  ├─teststorage
│  ├─testutil
│  ├─treecache
│  └─zeropool
└─web
    ├─api
    │  └─v1
    └─ui
        ├─module
        │  ├─codemirror-promql
        │  │  └─src
        │  │      ├─client
        │  │      ├─complete
        │  │      ├─lint
        │  │      ├─parser
        │  │      ├─test
        │  │      └─types
        │  └─lezer-promql
        │      ├─src
        │      └─test
        ├─react-app
        │  ├─public
        │  └─src
        │      ├─components
        │      ├─constants
        │      ├─contexts
        │      ├─fonts
        │      ├─hooks
        │      ├─images
        │      ├─pages
        │      │  ├─agent
        │      │  ├─alerts
        │      │  │  └─__snapshots__
        │      │  ├─config
        │      │  ├─flags
        │      │  │  └─__snapshots__
        │      │  ├─graph
        │      │  ├─rules
        │      │  ├─serviceDiscovery
        │      │  ├─status
        │      │  │  └─__snapshots__
        │      │  ├─targets
        │      │  │  └─__testdata__
        │      │  └─tsdbStatus
        │      ├─themes
        │      ├─types
        │      ├─utils
        │      └─vendor
        │          └─flot
        └─static
            ├─css
            ├─js
            └─vendor
                ├─bootstrap-4.5.2
                │  ├─css
                │  └─js
                ├─bootstrap4-glyphicons
                │  ├─css
                │  ├─fonts
                │  │  ├─fontawesome
                │  │  └─glyphicons
                │  └─maps
                ├─js
                └─rickshaw
                    └─vendor</code> </pre> </details>

 看起来还是超复杂的，我们慢慢啃就完事啦



## 3. 模块精读 

俗话说的好，代码的阅读方法是从上层阅读到底层，或从底层阅读到上层的。二我们首当其中的能在代码中看到入口cmd. 于是我们直接从cmd开始阅读就vans了。

###  3.1 cmd 工具的入口

<details>  <summary><font size="4" color="black">文件架构 点击展开</font></summary>  <pre><code class="language-cpp">
├─prometheus
│  │  main.go
│  │  main_test.go
│  │  main_unix_test.go
│  │  query_log_test.go
│  │
│  └─testdata
│      ├─consoles
│      │      test.html
│      │
│      └─rules
│              test.yml
│
└─promtool
    │  archive.go
    │  backfill.go
    │  backfill_test.go
    │  debug.go
    │  main.go
    │  main_test.go
    │  metrics.go
    │  rules.go
    │  rules_test.go
    │  sd.go
    │  sd_test.go
    │  tsdb.go
    │  tsdb_test.go
    │  unittest.go
    │  unittest_test.go

可以看出程序的切入口在main.go里面， 而tool里面多半是封装的方法与命令行参数的补充

首先我们可以看到他定义一些全局变量变量

```go
var (
	appName = "prometheus"
	// 默认数据保留时间，这个model.Duration就是普通的time.duration，多封装重写类一些方法
    // ps.为什么很多项目要对这种已有的基础库重新封装。我的理解是为了将这段代码纳入自己维护的范围内随项目一起维护，而不依赖基础库
	defaultRetentionString   = "15d"
	defaultRetentionDuration model.Duration
    
    // agent 模式 一个为2021年的新模式，解决promethues集群的数据同步问题，后面我会讲
	agentMode                       bool
	agentOnlyFlags, serverOnlyFlags []string

    configSuccess = prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "prometheus_config_last_reload_successful",
		Help: "Whether the last configuration reload attempt was successful.",
	})
	configSuccessTime = prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "prometheus_config_last_reload_success_timestamp_seconds",
		Help: "Timestamp of the last successful configuration reload.",
	})
)
```

上面这段全局变量一共有两个重点：

1. promethues的agent mode是什么
2. promethues中自定义的一个存储数据结构 Gauge   由此，我们可以浅浅挖一下promethues数据的结构

#### a. promethues数据结构

我们可以先来看一下gauge的结构：

```go
type Gauge interface {
    Metric {
        Desc()
        Write()
    }
    Collector {
        Describe(chan<- *Desc)
        Collect(chan<- Metric)
    }

	// Set sets the Gauge to an arbitrary value.
	Set(float64)
	// Inc increments the Gauge by 1. Use Add to increment it by arbitrary
	// values.
	Inc()
	// Dec decrements the Gauge by 1. Use Sub to decrement it by arbitrary
	// values.
	Dec()
	// Add adds the given value to the Gauge. (The value can be negative,
	// resulting in a decrease of the Gauge.)
	Add(float64)
	// Sub subtracts the given value from the Gauge. (The value can be
	// negative, resulting in an increase of the Gauge.)
	Sub(float64)

	// SetToCurrentTime sets the Gauge to the current Unix time in seconds.
	SetToCurrentTime()
}


type gauge struct {
	// valBits contains the bits of the represented float64 value. It has
	// to go first in the struct to guarantee alignment for atomic
	// operations.  http://golang.org/pkg/sync/atomic/#pkg-note-BUG
	valBits uint64
	selfCollector
	desc       *Desc
	labelPairs []*dto.LabelPair
}

```

我们可以看到guge这个数据结构必然要是先几个接口，分别是Metric Collector 以及一些其他的数字接口。这些接口主要仓空着gauge中的valBits这个变量。这里引用的是client_golang这个库。想要理解为什么这么设计，我们首先要明白promethues中的数据结构与分层



**Metric** 指标，可以先简单理解为promethues监控的数据，最基础的数据类型。还记得我们最开始的那张图么，系统拉日志数据所用的就是pull metrics这个描述哦

基于metric ，衍生了四种promethues监控的基本数据类型：[Counter, Guage, Histogram, Summary]([搞懂 Prometheus 这四种指标类型，谁都可能成为监控老司机？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/656355135)),  我们上面所展示的就是Guage 一种。可以支持加一减一设定的变量



我们先不看些许复杂的Histogram 还是以Guage为例。因为作为Metric 要实现一个重要的方法Write.我们来看看他是如何实现的：

```go
func (g *gauge) Write(out *dto.Metric) error {
	val := math.Float64frombits(atomic.LoadUint64(&g.valBits))
	return populateMetric(GaugeValue, val, g.labelPairs, nil, out, nil)
}
// 这里的的dto就是下面的Metric. 它来自import dto "github.com/prometheus/client_model/go"
func populateMetric(
	t ValueType,
	v float64,
	labelPairs []*dto.LabelPair,
	e *dto.Exemplar,
	m *dto.Metric,
	ct *timestamppb.Timestamp,
) error {
	m.Label = labelPairs
	switch t {
	case CounterValue:
		m.Counter = &dto.Counter{Value: proto.Float64(v), Exemplar: e, CreatedTimestamp: ct}
	case GaugeValue:
		m.Gauge = &dto.Gauge{Value: proto.Float64(v)}
	case UntypedValue:
		m.Untyped = &dto.Untyped{Value: proto.Float64(v)}
	default:
		return fmt.Errorf("encountered unknown type %v", t)
	}
	return nil
}



dto:
type Metric struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	Label       []*LabelPair `protobuf:"bytes,1,rep,name=label" json:"label,omitempty"`
	Gauge       *Gauge       `protobuf:"bytes,2,opt,name=gauge" json:"gauge,omitempty"`
	Counter     *Counter     `protobuf:"bytes,3,opt,name=counter" json:"counter,omitempty"`
	Summary     *Summary     `protobuf:"bytes,4,opt,name=summary" json:"summary,omitempty"`
	Untyped     *Untyped     `protobuf:"bytes,5,opt,name=untyped" json:"untyped,omitempty"`
	Histogram   *Histogram   `protobuf:"bytes,7,opt,name=histogram" json:"histogram,omitempty"`
	TimestampMs *int64       `protobuf:"varint,6,opt,name=timestamp_ms,json=timestampMs" json:"timestamp_ms,omitempty"`
}

// 他用protobuff序列化消息存储与读取。在写之前需要有一个Metric的struct实例
func (x *Metric) ProtoReflect() protoreflect.Message {
	mi := &file_io_prometheus_client_metrics_proto_msgTypes[10]
	if protoimpl.UnsafeEnabled && x != nil {
		ms := protoimpl.X.MessageStateOf(protoimpl.Pointer(x))
		if ms.LoadMessageInfo() == nil {
			ms.StoreMessageInfo(mi)
		}
		return ms
	}
	return mi.MessageOf(x)
}
./prompb/io/promethues/metrics.proto
message Metric {
  repeated LabelPair label        = 1 [(gogoproto.nullable) = false];
  Gauge              gauge        = 2;
  Counter            counter      = 3;
  Summary            summary      = 4;
  Untyped            untyped      = 5;
  Histogram          histogram    = 7;
  int64              timestamp_ms = 6;
}
```

我们能看到这里的他的存储方式使用protobuff序列化存储。由于proto很明显是远程调用才持久化的，不过目前阶段我们还没办法挖到那么深。总之我们目前知道了模型的write方法以来Metric这个总struct， 会调用序列化存储我们的数据即可



我们回头来看程序的入口长这样(main函数)

```go
func main() {
    ......
    cfg := flagConfig{
            notifier: notifier.Options{
                Registerer: prometheus.DefaultRegisterer,
            },
            web: web.Options{
                Registerer: prometheus.DefaultRegisterer,
                Gatherer:   prometheus.DefaultGatherer,
            },
            promlogConfig: promlog.Config{},
        }
        
    ......
    a := kingpin.New(XXXX)
    a.Flag("config.file", "Prometheus configuration file path.").
		Default("prometheus.yml").StringVar(&cfg.configFile)
	......
	// 一堆配置项的判断
	if cfg.XXXX {do XXXX}
    // 借用一句很经典的话：全部启动
	var g run.Group
	{    
	    ....
	    g.add(XXXservice.Run())
	    ....
	}
	if err := g.Run(); err != nil {
		level.Error(logger).Log("err", err)
		os.Exit(1)
	}

```

main函数我们后面细节的看逻辑，粗略来说就干了这几件事：定义了一个运行参数cfg,  然后从命令提示符运行参数里面拿对应的参数（a.flag) 之后走一些具体初始化逻辑判断后，把各个重要的组件启动起来（g.add, g.run)  再看这些组件具体的逻辑之前，我们先来扫盲一下main文件中其他的一些重要函数（下周继续)