---
title: "How I improved the YamlDotNet performance by 370%"
description: "Thanks to using BenchmarkDotNet I improved the YamlDotNet performance by 370%."
date: 2018-12-18T02:11:01-01:00
year: "2018"
authors:
  - Wojciech Nagórski
authorurl: "https://github.com/wojtpl2"
categories:
  - OpenSource
  - YamlDotNet
  - BenchmarkDotNet
tags:
  - BenchmarkDotNet
  - YamlDotNet
---

The [YamlDotNet](https://github.com/aaubry/YamlDotNet) is the most famous library to manage YAML format on .Net platform. This library is really stable and good solution. Many of my programs use this library, so I thought that this is prefect open source project that I can improve. 

## What can I do?

I've opened [issues tab](https://github.com/aaubry/YamlDotNet/issues) looking for issues with labels: "good first issue", "help wanted" or "up-for-grabs". However I couldn't find them because this specific labels names are not used in this project. 

I thought that I could read the source code and maybe I find something interesting to improve. I have to admit that the source code of the library is really clean. I took a look at the performance tests and immediately realized that I want to improve them. 

The [YamlDotNet](https://github.com/aaubry/YamlDotNet) used to use `Stopwatch` to measure performance of benchmark tests. The main loop that run the tests looked like:

```c#
foreach(var test in tests)
{
    Console.Write("{0}\t{1}\t", adapterName, test.GetType().Name);

    var graph = test.Graph;

    // Warmup
    RunTest(serializer, graph);

    if(!Stopwatch.IsHighResolution)
    {
        Console.Error.WriteLine("Stopwatch is not high resolution!");
    }

    var timer = Stopwatch.StartNew();
    for (var i = 0; i < iterations; ++i)
    {
        RunTest(serializer, graph);
    }
    var duration = timer.Elapsed;
    Console.WriteLine("{0}", duration.TotalMilliseconds / iterations);
}
```

You can find original code [here](https://github.com/aaubry/YamlDotNet/blob/v5.2.1/PerformanceTests/YamlDotNet.PerformanceTests.Lib/PerformanceTestRunner.cs#L46-L67).

As you can see tests here are using `StopWatch` class to measure code performance. The better way to do it is to use [BenchmarkDotNet](https://benchmarkdotnet.org/) which is a powerful .NET library for benchmarking. 

So, let's see how to start.

At first I wrote code of the test:

```c#
[MemoryDiagnoser]
public class ReceiptTest
{
	private readonly Receipt _receipt = new Receipt();
	private readonly StringWriter _buffer = new StringWriter();

	private readonly ISerializer _serializer = new SerializerBuilder()
		.WithNamingConvention(new CamelCaseNamingConvention())
		.Build();

    [Benchmark(Description = "Serialize vlatest")]
	public void Serialize()
	{
		_serializer.Serialize(_buffer, _receipt.Graph);
	}
}
```

The `Serialize()` method marked with `[Benchmark]` attribute is my benchmark test. I've used `[MemoryDiagnoser]` because I wanted to know how much memory was used by the [YamlDotNet](https://github.com/aaubry/YamlDotNet). 

Then I modified the `Program` class:

```C#
public class Program
{
	public static void Main(string[] args)
	{
		var summary = BenchmarkRunner.Run<ReceiptTest>();
	}
}
```

I couldn't wait to see my great change in action, so I run the program from console:

```ini
dotnet run -c Release
```

Unfortunately, I got error message:

```
// Validating benchmarks:
Assembly YamlDotNet.PerformanceTests.vlatest which defines benchmarks references non-optimized YamlDotNet
        If you own this dependency, please, build it in RELEASE.
        If you don't, you can create custom config with DontFailOnError to disable our custom policy and allow this benchmark to run.
```

As a message say `If you own this dependency, please, build it in RELEASE` and I went to my Visual Studio and I see:

![YamlDotNet configurations](/images/YamlDotNetConfigurations.png#normal)

I had chosen `Release-PerformanceTests` but BenchmarkDotNet printed me a message that I had to build the library in `Release`. Something was wrong. 

For unknown reasons, `Release-PerformanceTests` doesn't work in the same way like `Release` configuration. After short research I find an [article](https://www.pedrolamas.com/2017/04/24/creating-custom-build-configurations-for-the-dotnet-core-project-format/) about this problem.

In turned out that the new .csproj format for .Net Core doesn't have the same behavior as the old one. The `Release-*` configurations doesn't inherit from `Release` configuration anymore. All I had to do was set several parameters in the project file:

```xml
<PropertyGroup Condition=" '$(Configuration)' == 'Release-Signed' Or '$(Configuration)' == 'Release-Unsigned' ">
	<DefineConstants>$(DefineConstants);RELEASE;TRACE</DefineConstants>
	<DebugSymbols>false</DebugSymbols>
	<DebugType>portable</DebugType>
	<Optimize>true</Optimize>
</PropertyGroup>
```

After that changes, [BenchmarkDotNet](https://benchmarkdotnet.org/) was working. Out of curiosity, I compared the results before and after my changes and I was shocked.

```ini
-----------------------------------------------------------------------------------
| Serialize                                                                       |
-----------------------------------------------------------------------------------
|         |      Mean |     Error |    StdDev |      Gen0 |      Gen1 | Allocated |
-----------------------------------------------------------------------------------
| v5.2.1  |  539.5 us |  5.710 us |  5.062 us |    8.7891 |    0.9766 |  30.82 KB |
| vlatest |  145.8 us |  1.671 us |  1.563 us |    8.3008 |    0.4883 |   30.7 KB |
-----------------------------------------------------------------------------------
```

**Performance increased about 370% !** 

That was great news! I've made pull request which you can see [here](https://github.com/aaubry/YamlDotNet/pull/356) . My changes have been approved to [YamlDotNet 5.3.0](https://www.nuget.org/packages/YamlDotNet/5.3.0) version.

## Summary

The biggest lesson from this post, is that we always have to measure the performance of our changes, even for small ones. A seemingly insignificant change can spoil our performance.
And last but not least, we should always use existing solutions. Their authors spent a lot of time, so that we could save our time. Instead of creating a new solution, simply use existing one, do not reinvent the wheel.