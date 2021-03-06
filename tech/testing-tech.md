# 保证程序的可测试性

程序如果不可测，或者很难测试，那么就会像一个黑盒子，无法判断其正确性，或者只能被动的等待外部的反馈结果，结果就是开发变成了就火队员，到处救火。

## 一级水平：普通的测试

新手在测试一个程序是否正确时，通常只会在不同的地方，不断添加`print`或者`echo`之类的函数输出辅助信息。如果你也经常这样子测试，那么你就和一个新手其实差不了多少。这样的后果是：

- 系统留下一大堆被注释的测试代码。
- 程序今天运行也许正常，但是明天可能就不正常了，因为你的代码不是孤立的，输入可能发生变化等，但是这样测试的程序并没有适应性，别人也无法判断是否正确。
- 如果一个不小心，那么该注释掉的测试代码忘记了注释，则可能引起某些情况下的异常输出。

这样的例子很多，对于后端类的语言很多时候，就直接输出到控制台，而对于前端，例如js，很多就直接`alert`，显然这些都是经验不足的方式。有点经验的，会开始使用日志文件，前端的会开始使用控制台。但是这只是初级的方式，还不够，如果不加区分的记录信息，那就输出太多的测试信息，这对查找有用信息是一个浪费，而且最重要的是会影响性能，这对于生产环境几乎是不可接受的。

如果这时你想到的，又是在上线时，将不不要的测试输出注释掉，那你又是一个新手的做法，正确的做法应该是，对测试信息进行分级，例如`debug`, `info`, `warn`, `error`等，在测试阶段，我们可以输出所有的信息，但是到了生产环境，就通过配置变量的信息，只输出error级别的信息。这样方便简单，大大减少人为的因素，而且最最重要的是，当出问题时，只要调整配置参数，就能看到不同等级的日志，而不需要在业务逻辑中更改代码。

### 前后端分离开发的测试

对于前后端分离的开发模式，对于后端接口的测试还有些技巧性。除了单纯的写log，还有些更高明的测试技巧：

- `debug-bar`：laravel本身是有这个插件，对于前后端分离开发时，可以借鉴这种思想，例如：

```
// 接口原来的输出数据如下：
{
    data: {...}
}

// 可以增加一个输出字段，例如：
// 这里只是举例，具体实现可以参考debug-bar插件（应该是这个名字）
{
    data: {...},
    debug: {
        "sql": ["select ...", "select ..."],
        "msg": ["debug msg1", "debug msg2"],
        "runtime": ["..."]
    }
}
```

如果封装debug-bar比较麻烦，直接输出到浏览器的控制台也是一种方式。

laravel还封装了一个`dd`的函数，简单好用。

有了这些辅助的信息，测试就能淡定很多了。

## 二级水平：模块化与单元测试

单元测试是你程序是否可测的最基本的要求！

而单元测试的基础的模块化，如果没有良好的模块化划分，没有松耦合的设计，要进行高质量的单元测试几乎不可能。好的模块化设计的指导原则应该有：

- 明确的输入和输出
- 模块职责单一化：一个模块通常只实现一个功能或者特性
- 模块之间不能有紧耦合关系
- 模块内部包含许多函数，每个函数可以看成一个子模块，子模块也应该遵循以上原则
- 函数是单元测试最细的单元，最基本的要求就是代码行数

不同的语言有不同的单元测试工具，不过基本上都类似的。如果你还没有单元测试的经验，或者很少单元测试的经验，你是很难理解一个函数怎么样才算可测的，怎么才算松耦合。现实的确就是这样那个，单元测试是帮助你理解模块化设计的最简单有效的途径（另一个有效途径是不断的重构优化你的代码）。

单元测试中容易遇到的几种比较难处理的情况：

### 周期性执行任务的测试

这类任务不是直接执行函数就能看到结果，而是需要等待周期执行了才能看到结果，这时可以利用`time.Sleep()`这类的函数进行等待。作为例子可以看这里：https://github.com/ibbd-dev/go-log/blob/master/async-log/async_test.go

另一种方式如下：

```go
// 假设task是需要每分钟执行一次的任务
func task() {
    // do somethings...
}

// 周期性执行该函数
timer.AddFunc(task, time.Minute)
```

这个时候，再使用`time.Sleep()`来等待已经不行了，因为等待1分钟这个测试的效率太低了，于是可以将调用的地方修改为：

```go
// 定义一个全局变量
var taskFunc = task

// 调用的地方改为
timer.AddFunc(taskFunc, time.Minute)
```

这时不再时周期性执行task这个函数，而是taskFunc这个变量所定义的函数，这样在单元测试的时候，就可以重新对这个变量进行赋值，取消系统的自动调用，大概如下：

```go
// 在初始化函数将taskFunc赋值给一个空函数，这样周期性调用的就是该函数，什么都没做，这样就避免了对我们测试干扰
func init() {
    taskFunc = func() {}
}

func TestHello(t *testing.T) {
    // do somethings...

    task() // 直接执行任务函数
    // 对执行完之后的情况进行判断....
}
```

还有单元测试的一个小技巧：`var nowFunc = time.Now`，这样就可以用`nowFunc`来获取当前时间，同样在单元测试时可以改变这个函数，例如将当前时间调到5分钟之后。。。

总结：如果你想不清楚你的程序怎么进行单元测试，那么你的程序是不可测的，或者你的模块划分是失败的。

## 三级水平：业务测试

对于大型系统的复杂接口，例如dsp的投放接口，非常复杂，包含许多的条件组合，以至于不可能在整体的层面用单元测试去覆盖测试，如果要整体上覆盖，可能需要几百万甚至更多的测试用例。这种系统，首先需要完善刚才两个层次的测试，然后就要考虑业务测试（这是我起的名字，别人未必这样叫）。

业务测试的意思是指，以dsp广告投放系统为例，当广告主投放广告时，设置了一些条件，发现广告投放不出去，理论上应该是没问题的（实际有没有问题，比较难判断）。类似的来自业务的问题，在dsp系统中会经常碰到。怎么能够快速的定位问题，并解决，就能比较客观的反映了dsp系统的可测性水平了。

如果系统架构设计考虑不周，就只能到处救火了。还是以dsp系统为例，通常的业务需求为：

- 当广告投放出问题的时候，需要快速定位问题是出在哪个点上（账户余额，账户预算，状态，计划，项目，定向条件，创意等等，大大小小几十个可能出问题的点）
- 广告投放之后，发现数据不对了，这时怎么办，有没有相关日志来定位问题

这是两类最常见的问题，经常会遇到。

### 第一类问题：业务逻辑问题

架构上，应该能够通过配置（配置可以从文件获取，或者环境变量获取等，这样就不需要重启服务）来指定监控特定的广告，将该广告的投放过程的关键信息都记录下来，当然在记录这些信息的时候还是得考虑会不会写爆日志的可能，不能对正常运行产生过多的影响。

### 第二类问题：数据问题

对于数据问题，因为是滞后的，所以通常通过日志来定位问题。因此，日志系统的设计非常关键。日志通常包含如下部分：

- 守护进程或者nginx等日志：这类日志是独立于程序之外的，具体通常又分为访问日志和错误日志
- 数据日志：如订单日志，投放日志。
- 程序运行日志：程序运行过程中记录的各种日志，异常日志等。

如果日志系统设计良好，这些日志通常已经够辅助我们去做问题定位了。

