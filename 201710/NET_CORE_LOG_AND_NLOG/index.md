## .NET Core 日志与 NLog使用

### 简介
.NET Core 抽象了日志框架的API,结束了各家Logger框架各自为政的局面，使用者只需学习统一的API即可，降低了学习负担。本文将总体分析.NET Core Logging框架的使用，并针对NLog与.NET Core Logging框架的使用进行分析。

### 日志模型
.NET Core 的日志框架主要有三个核心的接口ILogger,ILoggerFactory,ILoggerProvider,三者的关系如下图所示

ILoggerProvider的源码如下
```csharp (type)
    public interface ILoggerProvider : IDisposable
    {
        // 创建一个ILogger,categoryName一般由ILoggerFactory的CreateLogger传递
        ILogger CreateLogger(string categoryName);
    }
````
ILoggerProvider本身没有什么好说的，就是微软常玩的Provider模式，老把戏了。

ILoggerFactory的源码如下

```csharp (type)
    /// <summary>
    /// 依次调用AddProvider方法注册的ILoggerProvider的CreateLogger方法,并将创建出的多个ILogger使用Composite模式封装成一个ILogger
    /// </summary>
    public interface ILoggerFactory : IDisposable
    {
        /// <summary>
        /// 依次调用AddProvider方法注册的ILoggerProvider的CreateLogger方法,并将创建出的多个ILogger使用Composite模式封装成一个ILogger
        /// categoryName将会被传递给ILoggerProvider的CreateLogger方法
        /// </summary>
        ILogger CreateLogger(string categoryName);
        /// <summary>
        /// 向ILoggerFactory中注册一个provider
        /// </summary>
        void AddProvider(ILoggerProvider provider);
    }
```
ILoggerFactory的功能：
* 通过AddProvider方法向ILoggerFactory中注册provider
* 通过CreateLogger方法，依次调用AddProvider方法注册的ILoggerProvider的CreateLogger方法,并将创建出的多个ILogger使用Composite模式封装成一个ILogger

ILogger的源码如下
```csharp (type)

    public interface ILogger

    {

        /// <summary>
        /// 记一条日志
        /// </summary>
        /// <param name="logLevel">日志的LogLevel</param>
        /// <param name="eventId">Id of the event.</param>
        /// <param name="state">The entry to be written. Can be also an object.</param>
        /// <param name="exception">The exception related to this entry.</param>
        /// <param name="formatter">一个Func用来格式化eventId，默认state.ToString().</param>
        void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception exception, Func<TState, Exception, string> formatter);
        /// <summary>
        /// 检测当前ILogger是否支持LogLevel
        /// </summary>
        /// <param name="logLevel">level to be checked.</param>
        bool IsEnabled(LogLevel logLevel);
        /// <summary>
        /// 开启一个Scope.返回值是一个IDisposable，可通过using包装后结束Scope
        /// </summary>
        /// <param name="state">Scope的Id</param>
        IDisposable BeginScope<TState>(TState state);
    }
````
ILogger主要提供了几个方法
* IsEnabled(LogLevel logLevel): 是否记录指定logLevel的日志
* Log<TState> : 记录一条日志，并指定其logLevel,EventId,State等参数，我们通常使用的LogDebug,LogTrace,LogInformation等都是ILogger的扩展方法，并最终都会归结为对次方法的调用

总结：通过阅读代码，我们发现日志模型的大致使用方式如下
1. 通过ILoggerFactory的AddProvider方法注册provider，每个provider通常代表不同的日志渠道，比如说ConsoleLoggerProvider将日志记录到控制台输出
2. 通过ILoggerFactory的CreateLogger方法调用每个provider的CreateLogger方法,并将得到的多个Logger【我们称为Concrete Logger】使用Composite模式组合成一个Logger
3. 使用2中的Composite 的 ILogger的LogXXX方法记录日志，这些方法会最终轮询调用到每个Concrete Logger 的ILogger.Log<TState>方法，将日志分发到不同的provider上

### 咋用
控制台应用程序中可通过如下的方式
```csharp (class)
public static void Main(string[] args = null)
{
  ILoggerFactory loggerFactory = new LoggerFactory()
    .AddConsole() // 添加ConsoleProvider
    .AddDebug();  // 添加DebugProvider
  ILogger logger = loggerFactory.CreateLogger<Program>();
  logger.LogInformation("This is a test of the emergency broadcast system.");
}
```
