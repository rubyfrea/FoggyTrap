# Spark metrics源码介绍

By *[Kivi](https://github.com/qifanj)*

Metric是Apache Spark里用于监测参数的一个package，路径是`spark/core/src/main/scala/org/apache/spark/metrics/`

大多数的公司都有自己的一套metrics监测系统，Spark，IBM都有，这些都是高度定制化的，别人没法用。像Spark这个其实也是引入了一个外部的包，出自codahale。这个包提供了一系列的“量具”来提供metrics监测的功能，简单来说，我们只需要找出要监测的变量或方法，然后注册到系统中，它就可以提供不同的度量方式如值（Gauge），计数器（Counters），直方图（Histograms），计时器（Timers）和Health check，并可以用不同的形式输出，像JMX，csv，console，graphite等。总之是个好工具，[官网](http://metrics.dropwizard.io/)  [Github](https://github.com/dropwizard/metrics)

下面来说Spark

- **source**

source其实就是数据来源，就是这些metrics是从哪里采集来的。在source文件夹下可以看见Source.scala，这是一个trait，有点类似于java中的接口
		
	private[spark] trait Source {
	  def sourceName: String
	  def metricRegistry: MetricRegistry
	}

它包括了source name 和 MetricRegistry，MetricRegistry是codahale的metrics包里来的，理解成用来“装”metrics的
虽然source下面只有JVMsource一个具体的实现类，实际上spark还有各种各样的source实现类来提供metrics。
![kivi](/Users/qf_jin/Documents/rubyfrea.github.io/image/sourceclass.png "Source Classes")

- **sink**

sink比较简单，就是metrics的输出形式，spark提供了ConsoleSink, CsvSink, GraphiteSink, JmxSink, Slf4jSink这几种方式来进行metrics的输出。

举ConsoleSink来讲，这是将metrics直接输出到执行页面的形式，用户可以在config里自己设置轮询时间，其中很重要的变量就是：

	val reporter: ConsoleReporter = ConsoleReporter.forRegistry(registry)
	    .convertDurationsTo(TimeUnit.MILLISECONDS)
	    .convertRatesTo(TimeUnit.SECONDS)
	    .build()

这也是codahale包里的类，就是将ConsoleSink里放着的metrics都注册到ConsoleReporter里。
其他Sink也都差不多，就不一一讲了。

- **MetricsSystem**

这里就像是metrics监测的一个总控制台，它有这么几个成员变量：

	private[this] val metricsConfig = new MetricsConfig(conf)
	private val sinks = new mutable.ArrayBuffer[Sink]
	private val sources = new mutable.ArrayBuffer[Source]
	private val registry = new MetricRegistry()

metricConfig就是读取了用户配置文件里的一些参数
sinks就是要输出的sink集合，sources自然就是source集合

来看`start()`方法：

	def start() {
	  require(!running, "Attempting to start a MetricsSystem that is already running")
	  running = true
	  registerSources()
	  registerSinks()
	  sinks.foreach(_.start)
	}

先做一个判断，然后注册所有source，注册所有sink，最后将sink一一启动。

`registerSources()`

	registerSources()
	private def registerSources() {
	  val instConfig = metricsConfig.getInstance(instance)
	  val sourceConfigs = metricsConfig.subProperties(instConfig, MetricsSystem.SOURCE_REGEX)

	  // Register all the sources related to instance
	  sourceConfigs.foreach { kv =>
	    val classPath = kv._2.getProperty("class")
	    try {
	      val source = Utils.classForName(classPath).newInstance()
	      registerSource(source.asInstanceOf[Source])
	    } catch {
	      case e: Exception => logError("Source class " + classPath + " cannot be instantiated", e)
	    }
	  }
	}

这里用了一个反射，通过source的类名启动了一个source对象，再将该对象source注册

`registerSource()`

	registerSource()
	def registerSource(source: Source) {
	  sources += source
	  try {
	    val regName = buildRegistryName(source)
	    registry.register(regName, source.metricRegistry)
	  } catch {
	    case e: IllegalArgumentException => logInfo("Metrics already registered", e)
	  }
	}

这个方法是先给source起了个名，然后把这个source注册进MetricsSystem的MetricRetrigy对象里。

再看下一步`registerSinks()`

	private def registerSinks() {
	    val instConfig = metricsConfig.getInstance(instance)
	    val sinkConfigs = metricsConfig.subProperties(instConfig, MetricsSystem.SINK_REGEX)

	    sinkConfigs.foreach { kv =>
	      val classPath = kv._2.getProperty("class")
	      if (null != classPath) {
	        try {
	          val sink = Utils.classForName(classPath)
	            .getConstructor(classOf[Properties], classOf[MetricRegistry], classOf[SecurityManager])
	            .newInstance(kv._2, registry, securityMgr)
	          if (kv._1 == "servlet") {
	            metricsServlet = Some(sink.asInstanceOf[MetricsServlet])
	          } else {
	            sinks += sink.asInstanceOf[Sink]
	          }
	        } catch {
	          case e: Exception =>
	            logError("Sink class " + classPath + " cannot be instantiated")
	            throw e
	        }
	      }
	    }
	  }
	}
		
这边也是通过反射新建了一个sink的对象，然后把这对象放到MetricsSystem的sinks集合里。