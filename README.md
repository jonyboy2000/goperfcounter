goperfcounter
==========

goperfcounter用于golang应用的业务监控。goperfcounter需要和开源监控系统[Open-Falcon](http://book.open-falcon.com/zh/index.html)一起使用。

概述
-----
使用goperfcounter进行golang应用的监控，大体如下: 

1. 用户在其golang应用代码中，调用goperfcounter提供的统计函数；统计函数被调用时，perfcounter会生成统计记录、并保存在内存中
2. goperfcounter会自动的、定期的将这些统计记录push给Open-Falcon的收集器([agent](https://github.com/open-falcon/agent)或[transfer](https://github.com/open-falcon/transfer))
3. 用户在Open-Falcon中，查看统计数据的绘图曲线、设置实时报警

另外，goperfcounter提供了golang应用的基础监控，包括runtime指标、debug指标等。默认情况下，基础监控是关闭的，用户可以通过[配置文件](#配置)来开启此功能。

安装
-----

在golang项目中使用goperfcounter时，需要进行安装，操作如下

```bash
go get github.com/niean/goperfcounter

```

使用
-----

用户自定义统计指标时，需要引入goperfcounter的包，需要在代码片段中调用goperfcounter的[API](#API)。比如，用户想要统计函数的出错次数，可以调用如下Meter方法。

```go
package xxx

import (
	pfc "github.com/niean/goperfcounter"
)

func foo() {
	if err := bar(); err != nil {
		pfc.Meter("bar.called.error", 1)
	}
}

func bar() error {
	// do sth ...
	return nil
}

```

这个调用主要会产生7个Open-Falcon统计指标，如下。其中，```endpoint```默认为服务器```Hostname()```，```tags```中至少包含一个```name=bar.called.error```的标签(```bar.called.error```为统计器的名称)。

```python
{
    "counterType": "COUNTER",
    "endpoint": "git",
    "metric": "meter.rate.falcon",
    "step": 20,
    "tags": "cop=xiaomi,owt=inf,pdl=falcon,module=perfcounter,name=bar.called.error",
    "timestamp": 1450681584,
    "value": 83
},
{
    "counterType": "GAUGE",
    "endpoint": "git",
    "metric": "meter.sum.all",
    "step": 20,
    "tags": "cop=xiaomi,owt=inf,pdl=falcon,module=perfcounter,name=bar.called.error",
    "timestamp": 1450681584,
    "value": 83
},
{
    "counterType": "GAUGE",
    "endpoint": "git",
    "metric": "meter.rate.1min",
    "step": 20,
    "tags": "cop=xiaomi,owt=inf,pdl=falcon,module=perfcounter,name=bar.called.error",
    "timestamp": 1450681584,
    "value": 0
},
{
    "counterType": "GAUGE",
    "endpoint": "git",
    "metric": "meter.rate.5min",
    "step": 20,
    "tags": "cop=xiaomi,owt=inf,pdl=falcon,module=perfcounter,name=bar.called.error",
    "timestamp": 1450681584,
    "value": 0
},
{
    "counterType": "GAUGE",
    "endpoint": "git",
    "metric": "meter.rate.15min",
    "step": 20,
    "tags": "cop=xiaomi,owt=inf,pdl=falcon,module=perfcounter,name=bar.called.error",
    "timestamp": 1450681584,
    "value": 0
},
{
    "counterType": "GAUGE",
    "endpoint": "git",
    "metric": "meter.rate.all",
    "step": 20,
    "tags": "cop=xiaomi,owt=inf,pdl=falcon,module=perfcounter,name=bar.called.error",
    "timestamp": 1450681584,
    "value": 0.02578528
},
{
    "counterType": "GAUGE",
    "endpoint": "git",
    "metric": "meter.rate.step",
    "step": 20,
    "tags": "cop=xiaomi,owt=inf,pdl=falcon,module=perfcounter,name=bar.called.error",
    "timestamp": 1450681584,
    "value": 0.5
}

```



另外，goperfcounter提供了一个完整的例子，代码见[这里](https://github.com/niean/goperfcounter/blob/master/example/main.go)。按照如下指令，运行这个例子。

```bash
# install
cd $GOPATH/src/github.com/niean
git clone https://github.com/niean/goperfcounter.git

# start
cd $GOPATH/src/github.com/niean/goperfcounter/example/scripts
./debug build && ./debug start

# proc
./debug proc metrics/falcon # list all metrics in falcon-model

```


配置
----
默认情况下，goperfcounter不需要进行配置。如果用户需要定制goperfcounter的行为，可以通过配置文件来进行。配置文件，需要满足以下条件:

+ 配置文件必须和golang的二进制文件在同一目录
+ 配置文件命名必须为```perfcounter.json```

配置文件内容格式、含义，如下

```go
{
    "debug": true, // 是否开启调制，默认为false
    "hostname": "", // 机器名，默认为本机名称
    "tags": "cop=xiaomi,module=perfcounter", // tag串，默认为空。多个tag用逗号分隔;name为保留字段，因此不允许设置形如"name=xxx"的tag
    "step": 30, // 上报周期，单位s，默认为60s
    "bases":[], // golang-vm信息采集项，可选值为"debug"、"runtime"，默认不采集
    "push": { // push数据到Open-Falcon
        "enabled":true, // 是否开启自动push，默认开启
        "api": "http:// 127.0.0.1:6060/api/push" // Open-Falcon接收器地址，默认为本地agent，即"http:// 127.0.0.1:1988/v1/push"
    },
    "http": { // http服务
        "enabled": true, // 是否开启http服务，默认不开启
        "listen": "0.0.0.0:2015" // http服务监听地址，默认为空
    }
}

```

数据上报
----

goperfcounter会将各种统计器的统计结果，发送到Open-Falcon。每种统计器，会被转化成不同的Open-Falcon指标，对应关系如下。每条数据，至少包含一个```name=XXX```的tag，```XXX```是统计器的名称。

<table border="1">
<tr>
  <th>统计器类型</th>
  <td>metric名称</td>
  <td>数据类型</td>
  <td>含义</td>
</tr>
<tr>
  <th rowspan="1">Counter</th>
  <td>counter.sum.all</td>
  <td>原始值(GAUGE)</td>
  <td>所有统计计数的累加和</td>
</tr>
<tr>
  <th rowspan="1">Gauge</th>
  <td>gauge.value</td>
  <td>原始值(GAUGE)</td>
  <td>最后一次的记录值(int64)</td>
</tr>
<tr>
  <th rowspan="1">GaugeFloat64</th>
  <td>gaugefloat64.value</td>
  <td>原始值(GAUGE)</td>
  <td>最后一次的记录值(float64)</td>
</tr>
<tr>
  <th rowspan="7">Meter</th>
  <td>meter.sum.all</td>
  <td>原始值(GAUGE)</td>
  <td>事件发生的总次数</td>
</tr>
<tr>
  <td>meter.rate.all</td>
  <td>原始值(GAUGE)</td>
  <td>事件发生的频率，单位CPS</td>
</tr>
<tr>
  <td>meter.rate.falcon</td>
  <td>计数器(COUNTER)</td>
  <td>最近一个上报周期内，事件发生的频率，单位CPS</td>
</tr>
<tr>
  <td>meter.rate.step</td>
  <td>原始值(GAUGE)</td>
  <td>最近一个MeterTick内，事件发生的频率，单位CPS</td>
</tr>
<tr>
  <td>meter.rate.1min</td>
  <td>原始值(GAUGE)</td>
  <td>事件发生频率的1min滑动平均，单位CPS</td>
</tr>
<tr>
  <td>meter.rate.5min</td>
  <td>原始值(GAUGE)</td>
  <td>事件发生频率的5min滑动平均，单位CPS</td>
</tr>
<tr>
  <td>meter.rate.15min</td>
  <td>原始值(GAUGE)</td>
  <td>事件发生频率的15min滑动平均，单位CPS</td>
</tr>
<tr>
  <th rowspan="9">Histogram</th>
  <td>histogram.50th</td>
  <td>原始值(GAUGE)</td>
  <td>所有采样数据中，处于中位50%处的数值</td>
</tr>
<tr>
  <td>histogram.75th</td>
  <td>原始值(GAUGE)</td>
  <td>所有采样数据中，处于75%处的数值</td>
</tr>
<tr>
  <td>histogram.95th</td>
  <td>原始值(GAUGE)</td>
  <td>所有采样数据中，处于95%处的数值</td>
</tr>
<tr>
  <td>histogram.99th</td>
  <td>原始值(GAUGE)</td>
  <td>所有采样数据中，处于99%处的数值</td>
</tr>
<tr>
  <td>histogram.999th</td>
  <td>原始值(GAUGE)</td>
  <td>所有采样数据中，处于99.9%处的数值</td>
</tr>
<tr>
  <td>histogram.max</td>
  <td>原始值(GAUGE)</td>
  <td>采样数据的最大值</td>
</tr>
<tr>
  <td>histogram.min</td>
  <td>原始值(GAUGE)</td>
  <td>采样数据的最小值</td>
</tr>
<tr>
  <td>histogram.mean</td>
  <td>原始值(GAUGE)</td>
  <td>采样数据的平均值</td>
</tr>
<tr>
  <td>histogram.stddev</td>
  <td>原始值(GAUGE)</td>
  <td>采样数据的标准差</td>
</tr>
<tr>
  <th rowspan="16">Timer</th>
  <td>timer.sum.all</td>
  <td>原始值(GAUGE)</td>
  <td>计时器被调用的总次数</td>
</tr>
<tr>
  <td>timer.rate.all</td>
  <td>原始值(GAUGE)</td>
  <td>计时器被调用的频率，单位CPS</td>
</tr>
<tr>
  <td>timer.rate.falcon</td>
  <td>计数器(COUNTER)</td>
  <td>最近一个上报周期内，计时器被调用的频率，单位CPS</td>
</tr>
<tr>
  <td>timer.rate.step</td>
  <td>原始值(GAUGE)</td>
  <td>最近一个TimerTick内，事件发生的频率，单位CPS</td>
</tr>
<tr>
  <td>timer.rate.1min</td>
  <td>原始值(GAUGE)</td>
  <td>计时器被调用频率的1min滑动平均，单位CPS</td>
</tr>
<tr>
  <td>timer.rate.5min</td>
  <td>原始值(GAUGE)</td>
  <td>计时器被调用频率的5min滑动平均，单位CPS</td>
</tr>
<tr>
  <td>timer.rate.15min</td>
  <td>原始值(GAUGE)</td>
  <td>计时器被调用频率的15min滑动平均，单位CPS</td>
</tr>
<tr>
  <td>timer.50th</td>
  <td>原始值(GAUGE)</td>
  <td>所有采样数据中，处于中位50%处的数值</td>
</tr>
<tr>
  <td>timer.75th</td>
  <td>原始值(GAUGE)</td>
  <td>所有耗时统计的采样数据中，处于75%处的数值</td>
</tr>
<tr>
  <td>timer.95th</td>
  <td>原始值(GAUGE)</td>
  <td>所有耗时统计的采样数据中，处于95%处的数值</td>
</tr>
<tr>
  <td>timer.99th</td>
  <td>原始值(GAUGE)</td>
  <td>所有耗时统计的采样数据中，处于99%处的数值</td>
</tr>
<tr>
  <td>timer.999th</td>
  <td>原始值(GAUGE)</td>
  <td>所有耗时统计的采样数据中，处于99.9%处的数值</td>
</tr>
<tr>
  <td>timer.max</td>
  <td>原始值(GAUGE)</td>
  <td>耗时统计的采样数据的最大值</td>
</tr>
<tr>
  <td>timer.min</td>
  <td>原始值(GAUGE)</td>
  <td>耗时统计的采样数据的最小值</td>
</tr>
<tr>
  <td>timer.mean</td>
  <td>原始值(GAUGE)</td>
  <td>耗时统计的采样数据的平均值</td>
</tr>
<tr>
  <td>timer.stddev</td>
  <td>原始值(GAUGE)</td>
  <td>耗时统计的采样数据的标准差</td>
</tr>
</table>


API
----

请移步到[这里](https://github.com/niean/goperfcounter/blob/master/doc/API.md)


Bench
----

请移步到[这里](https://github.com/niean/goperfcounter/blob/master/doc/BENCH.md)