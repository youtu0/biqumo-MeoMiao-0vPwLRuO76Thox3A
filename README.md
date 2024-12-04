[合集 \- Natasha(10\)](https://github.com)[1\.Natasha V5\.2\.2\.1 正式发布.2023\-04\-26](https://github.com/NMSLanX/p/17349425.html)[2\.动态编译库 Natasha 5\.0 兼容版本发布2022\-10\-10](https://github.com/NMSLanX/p/16769315.html)[3\.Natasha 4\.0 探索之路系列(四) 模板 API2022\-01\-23](https://github.com/NMSLanX/p/15814781.html)[4\.Natasha 4\.0 探索之路系列(三) 基本的动态编译2022\-01\-22](https://github.com/NMSLanX/p/15813760.html)[5\.Natasha 4\.0 探索之路系列(一) 概况2022\-01\-20](https://github.com/NMSLanX/p/15741371.html)[6\.Natasha 4\.0 探索之路系列(二) "域"与插件2022\-01\-21](https://github.com/NMSLanX/p/15799203.html):[slower加速器](https://chundaotian.com)[7\.基于roslyn的动态编译库Natasha2019\-08\-01](https://github.com/NMSLanX/p/11284573.html)[8\.轻量化动态编译库 Natasha v8\.0 正式发布！01\-10](https://github.com/NMSLanX/p/17901708.html)[9\..NET 创建动态方法方案及 Natasha V911\-14](https://github.com/NMSLanX/p/18299481)10\.Natasha v9\.0 为 .NET 开发者提供 \[热执行] 方案.12\-03收起
## 项目简介


自 Natasha v9\.0 发布起，我将基于 Natasha 的推出热执行方案，这项技术允许基于 控制台(Console) 和新版 Asp.net Core 架构的项目在运行中动态重编译，在不停止工程的情况下获取最新结果，以帮助技术初学者、项目初期开发人员等，进行快速实验以及试错。


为了更形象的说明 \[热执行] 请看下图:
![HE](https://img2024.cnblogs.com/blog/1119790/202406/1119790-20240622020357627-628105246.gif)


### 热执行


以下为了更加简洁，称热执行为 \[HE].
图中是 Asp.net Core 一个接口开发的案例，我更改了一个实体类的结构，并保存，可以看到接口返回了最新的实体类结构。借此简单阐述一些热执行的工作原理，文件发生变化会触发 \[HE] 对项目进行热编译，开发者无论是大改还是小改，只要你的项目文件(cs) 、依赖项目、csproj 发生了变化，\[HE] 就会代理整个项目并自动编译输出。对于有些老机器较慢，可能 \[HE] 热编译要比 \[按下F5\-程序跑起来] 要快的多。


### 热重载与热执行


也许有人会觉得这更像一个完全体的热重载，并不是，这是与热重载完全不同的技术，\[HE] 的核心技术是语法树重写与动态编译。而热重载是对 Runtime 的程序集进行热更新，热重载严重依赖 Debugger 组件，且目前从 [ENC 错误代码](https://github.com) 来看这项技术的限制还是很大的。
起初我也是闷头钻研热重载技术，但实验效果很不理想，热重载技术是一项前沿的，边界明确的技术，并不适合敞开手脚快刀阔斧的干，由于不是面对开发者，(截至2024年8月)资料也不是很多。与其死磕它，不如另辟蹊径，借助 Natasha 动态代理将项目管理起来。


## 指令简介


### 注释指令


HE 使用注释作为热代理指令，这些指令会影响语法树重建以及热编译选项，但不影响程序的发布和使用。目前具体如下：


* #### 优化级别


使用 //HE:Release 指令允许在 HE 重编译时，使用 Release 模式进行编译。


* #### 异步代理


当 Main 方法中有对象 A, A 需要延迟卸载，A 不干扰 new A (即全局可以不只有一个 A), 此时使用 //HE:Async 允许 HE代理 在上一次 A 对象未完全销毁时异步执行新 Main 方法。


* #### Using 排除


由于开发可能会开启隐式 using, 若开启，则 HE 在代理期间，会加载所有内存中存在的命名空间，因此有概率会出现 using 二义性引用问题，使用 //HE:CS0104 可以排除干扰 using，例如 //HE:CS0104 using1;using2\...


* #### 动态表达式


如果您需要在 HE 代理期间动态的调试输出一些结果，且不影响程序发布，您可以使用 //DS 或 //RS 指令输出其后的表达式。例如 `//DS 1+1` 在 Debug 模式下输出 2\. `//RS a.age+b.age` 在 Release 输出两个对象年龄相加。


* #### 参数传递


`void ProxyMainArguments()` 方法将在代理执行之前执行，该方法允许开发者在动态开发中，在 HE 代理期间模拟 控制台向 main 方法中传递参数。
伪代码类似于：



```
public static void ProxyMainArguments()
{
      HEProxy.AppendArgs("123");
      HEProxy.AppendArgs("参数2");
      HEProxy.AppendArgs("abc");
}
main("123","参数2","abc");

```

注意：HE 每次创建新的代理都是一次全新的 main 执行过程，因此将清空 Args, 避免上一次代理干扰本次执行。


* #### 仅在程序第一次运行


使用 //Once 命令允许程序仅在程序第一次开启时运行被其注释的代码，在后续的 HE 代理期间，被注释的语法节点将被剔除。


## 使用


目前该项目支持 .NET3\.0 即以上版本，且 .NET5\.0 版本以上有 Source Generator 技术加持。


### 无 SG 加持的版本


1. 引入热执行包：`DotNetCore.Natasha.CSharp.HotExecutor`



```
class Program
{

        public static void Main(string[] args)
        {

            //设置当前程序的类型 ，默认为 Console
            HEProxy.SetProjectKind(HEProjectKind.Console);

            //HE 代理周期日志(如果不需要 HE 写入日志，这句就不用写了)
            string debugFilePath = Path.Combine(VSCSProjectInfoHelper.HEOutputPath, "Debug.txt");
            HEFileLogger logger = new HEFileLogger(debugFilePath);

            //设置信息输出方式,该方法影响 DS/RS 指令的输出方式
            //默认是 Console.WriteLine 方式输出
            HEProxy.ShowMessage = async msg => {
                //一些项目可能禁用控制台，那就用日志输出 HE 信息
                await logger.WriteUtf8FileAsync(msg);
            };

            //编译初始化选项，主要是 Natasha 的初始化操作.
            //Once (热编译时使用 Once 剔除被注释的语句)
            HEProxy.SetCompileInitAction(() => {
                {
                    NatashaManagement.RegistDomainCreator();
                    NatashaManagement.Preheating((asmName, @namespace) =>

                                !string.IsNullOrWhiteSpace(@namespace) &&
                                (HEProxy.IsExcluded(@namespace)),
                                true,
                                true);
                }
            });

            //开始执行动态代理.
            //Once (热编译时使用 Once 剔除被注释的语句)
            HEProxy.Run();

            for (int i = 0; i < args.Length; i++)
            {
                Console.WriteLine(args[i]);
                //在 HE 代理期间输出 args 值
                //DS args[i]
            }
 
            //while 阻塞时需要指定 CancelToken ,热执行时 HE 将取消循环操作。
            CancellationTokenSource source = new CancellationTokenSource();

            //添加到 HE 中，以便下个编译时释放
            source.ToHotExecutor();

            while (!source.IsCancellationRequested)
            {
                Thread.Sleep(1000);
                //在 HE 代理期间输出 "In while loop!"
                //DS "In while loop!"
            }
  
            for (int i = 0; i < args.Length; i++)
            {
                 Console.WriteLine(args[i]);
            }
  
            //防止 while 退出后直接关闭主线程
            //Once (这句 `//Once` 可以不写，HE 有针对 “Console.Read” 的末尾阻塞检测)
            Console.ReadKey();

        }

        //方法体中的参数操作对应 Main(string[] args) 中的 args,  
        //热执行时，Main 将接收到 "参数11"，“参数2”，“参数23”
        //非必要，可以不写
        public static void ProxyMainArguments()
        {
            HEProxy.AppendArgs("参数11");
            HEProxy.AppendArgs("参数2");
            HEProxy.AppendArgs("参数23");
        }
}

```

这段代码是 HE 最原始的代码。


### SG 加持(.NET5\.0及以上版本)



> SG 主要是减少了 HE 初始化的一些操作。而一些需要手动传递的 cancel/dispose 实例仍然需要手动传递给 HE。


#### 简单案例


1. 引入 SG 包：`DotNetCore.Natasha.CSharp.HotExecutor.Wrapper`



```
internal class Program
{

    static void Main(string[] args)
    {
        for (int i = 0; i < args.Length; i++)
        {
            //DS args[i]
        }

        //这里仍然需要手动将 canceltoken 传递给 HE
        CancellationTokenSource source = new();
        source.ToHotExecutor();

        while (!source.IsCancellationRequested)
        {
            Thread.Sleep(1000);
            //DS "In while loop!"
        }
     
        //防止 while 退出后直接关闭主线程
        Console.ReadKey();
    }
    public static void ProxyMainArguments()
    {
        HEProxy.AppendArgs("参数1");
        HEProxy.AppendArgs("参数2");
        HEProxy.AppendArgs("参数3");
        HEProxy.AppendArgs("参数4");
    }
}

```

#### 代理新 Asp.net Core


HE 目前不能代理 MVC 项目和老版的 API 项目。



```
public class Program
{
    public static void Main(string[] args)
    {

        //HE:Async
        var builder = WebApplication.CreateBuilder(args);
        builder.Services.AddAuthorization();
        builder.Services.AddEndpointsApiExplorer();
        builder.Services.AddSwaggerGen();
        var app = builder.Build();
        if (app.Environment.IsDevelopment())
        {
            app.UseSwagger();
            app.UseSwaggerUI();
        }


        //将 APP 添加到 HE 中，以便在下一次编译中释放该对象。
        app.AsyncToHotExecutor();

        //更改以下的值，保存文件，会触发 HE 创建新的 WebApplicationBuilder
        var summaries = new[]
        {
            "Freezing441", "Bracing"
        };


        app.MapGet("/weatherforecast", (HttpContext httpContext) =>
        {
            var forecast = Enumerable.Range(1, 5).Select(index =>
                new WeatherForecast
                {
                    Date = DateTime.Now.AddDays(index),
                    TemperatureC = Random.Shared.Next(-20, 55),
                    Summary = summaries[Random.Shared.Next(summaries.Length)]
                })
                .ToArray();
            return forecast;
        })
        .WithName("GetWeatherForecast");

        app.Run();
    }
}

```

## 其他项目支持


截至目前而言， HE 对 Winform 的支持不是很好，WPF 的很难代理，时间和精力有限，不会深入去研究了。


## 鸣谢


感谢 [九哥](https://github.com) 的支持。


## 结尾


遇到问题可以到 [Natasha Issue 区](https://github.com) 提出反馈。


