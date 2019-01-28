---
title: "First Pull Request in Open Source"
description: "The history of the first pull request to BenchmarkDotNet"
date: 2018-12-11T11:16:00-04:00
year: "2018"
authors:
  - Wojciech Nagórski
authorurl: "https://github.com/wojtpl2"
categories:
  - OpenSource
  - BenchmarkDotNet
tags:
  - BenchmarkDotNet
  - FirstPullRequest
---

Not so long ago, I saw a video [Adam Sitnik - My awesome journey with Open Source](https://www.youtube.com/watch?v=2HSPKyAyuik) and it opened my eyes. 

I am a senior developer with 11 years of experience in programming. In my career, I did a lot of great things connected to programming, but only my colleagues from work knew about it. If I would like to change my job, I would have to prove my qualifications. Another sad thing about both my personal and business projects is that most of my code is NOT used anymore. It really makes me sad. Than that video came up. Adam showed me that I can create code that will be used all over the world for many, many years. Solution is Open Source.

But, how to start? 

1. Find the repo of project that interests you.
2. Go to Issue tab and filter issue by labels: "good first issue", "help wanted" or "up-for-grabs".
3. Optional: Ask if selected issue is up to date and is not blocked. 
5. Try to implement missing functionality.
6. Make a pull request. If you don't know how to do it, please open Google or Youtube and type: "how to create pull request on github". It takes only few minutes. 

So, I tried. 

1. I've chosen [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) repo. Why? I've used this tool and learned many things when using it. 
2. I've found [Implement `--list`](https://github.com/dotnet/BenchmarkDotNet/issues/905) issue. The task had all the labels that I mentioned above.
3. I've reserved task for myself.
4. I will describe implementation below.
5. I've done [pull request](https://github.com/dotnet/BenchmarkDotNet/pull/914) with my changes. 

## Implementation

I didn't do anything special in this task. Remember that it was the first pull request to Open Source.

In this issue I had to implement console line `--list` option which will print all of available benchmark names. This option should have two modes:

##### 1. Flat list

This mode is simple list of all benchmark names. 

```ini
BenchmarkDotNet.Samples.exe --list flat
```

```
BenchmarkDotNet.Samples.Algo_Md5VsSha256.Md5
BenchmarkDotNet.Samples.Algo_Md5VsSha256.Sha256
BenchmarkDotNet.Samples.IntroArguments.Benchmark
BenchmarkDotNet.Samples.IntroArgumentsSource.SingleArgument
BenchmarkDotNet.Samples.IntroArgumentsSource.ManyArguments
BenchmarkDotNet.Samples.IntroArrayParam.ArrayIndexOf
BenchmarkDotNet.Samples.IntroArrayParam.ManualIndexOf
BenchmarkDotNet.Samples.IntroBasic.Sleep
BenchmarkDotNet.Samples.IntroBasic.Thread.Sleep(10)
[...]
```
##### 2. Tree list
In this mode BenchmarkDotNet should print all benchmark names as a tree list. 

```ini
BenchmarkDotNet.Samples.exe --list tree
```

```
BenchmarkDotNet
 └─Samples
    ├─Algo_Md5VsSha256
    │  ├─Md5
    │  └─Sha256
    ├─IntroArguments
    │  └─Benchmark
    ├─IntroArgumentsSource
    │  ├─SingleArgument
    │  └─ManyArguments
    ├─IntroArrayParam
    │  ├─ArrayIndexOf
    │  └─ManualIndexOf
    ├─IntroBasic
    │  ├─Sleep
    │  └─Thread
    │     └─Sleep(10)
[...]
```

At first I added new console parameter. [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) uses [CommandLineParser](https://github.com/commandlineparser/commandline) to parse command line attributes. CommandLineParser is useful tool that I implemented in many of projects. If you don't know it, here you can find [documentation](https://github.com/commandlineparser/commandline). 

This parameter should have three values: Disable, Flat and Tree. I just added enum:

```c#
    public enum ListBenchmarkCaseMode
    {
        /// <summary>
        /// Do not print any of the available full benchmark names.
        /// </summary>
        Disable,

        /// <summary>
        /// Prints flat list of the available benchmark names.
        /// </summary>
        Flat,

        /// <summary>
        /// Prints tree of the available full benchmark names.
        /// </summary>
        Tree
    }
```

Then I added the appropriate option to the class that is used for parsing command line attributes.

```c#
[Option("list", Required = false, Default = ListBenchmarkCaseMode.Disable, HelpText = "Prints all of the available benchmark names. Flat/Tree")]
public ListBenchmarkCaseMode ListBenchmarkCaseMode { get; set; }
```

Displaying the tree list was the most difficult part. Fortunately, I read [Andrew Lock](https://andrewlock.net/about/) blog who wrote post [Creating an ASCII-art tree in C#](https://andrewlock.net/creating-an-ascii-art-tree-in-csharp/). All I had to do was to check the source code license. Luckily for me, it was a [MIT license](https://opensource.org/licenses/MIT).

I created interface:

```c#
    internal interface IBenchmarkCasesPrinter
    {
        void Print(IEnumerable<string> testNames);
    }
```

And two implementation of it. One for each mode. 

```c#
    internal class FlatBenchmarkCasesPrinter : IBenchmarkCasesPrinter
    {
        public void Print(IEnumerable<string> testNames)
        {
            foreach (string test in testNames)
            {
                Console.WriteLine(test);
            }
        }
    }
```

```c#
    internal class TreeBenchmarkCasesPrinter : IBenchmarkCasesPrinter
    {
        public void Print(IEnumerable<string> testNames)
        {
            //Here is the algorithm from the Andrew Lock's blog
        }
    }
```

In next step I used [fasade pattern](https://en.wikipedia.org/wiki/Facade_pattern) because I wanted to mask interaction with more complex components behind a single API.

```C#
    internal class BenchmarkCasesPrinter : IBenchmarkCasesPrinter
    {
        private readonly IBenchmarkCasesPrinter printer;
        
        public BenchmarkCasesPrinter(ListBenchmarkCaseMode listBenchmarkCaseMode)
        {
            printer = listBenchmarkCaseMode == ListBenchmarkCaseMode.Tree
                ? (IBenchmarkCasesPrinter) new TreeBenchmarkCasesPrinter()
                : new FlatBenchmarkCasesPrinter();
        }
         public void Print(IEnumerable<string> testName) => printer.Print(testName);
    }
```

Next I searched for a place where BenchmarkDotNet gets all benchmarks. In the end, I created and run the fasade class, if the value of `--list` parameter was set. Bellow you can see how I changed the original code.

```diff
- var filteredBenchmarks = typeParser.Filter(effectiveConfig);
+ var filteredBenchmarks = typeParser.Filter(effectiveConfig, listBenchmarkCase);

  if (filteredBenchmarks.IsEmpty())
	return Array.Empty<Summary>();
	
+ var listBenchmarkCase = options.ListBenchmarkCaseMode != BistBenchmarkCaseMode.Disable;
+ if (listBenchmarkCase)
+ {
+ 	var testNames = filteredBenchmarks.SelectMany(p => p.BenchmarksCases)
+ 		.Select(p => p.Descriptor.GetFilterName()).Distinct();

+ 	var printer = new BenchmarkCasesPrinter(options.ListBenchmarkCaseMode);
+ 	printer.Print(testNames);

+ 	return Enumerable.Empty<Summary>();
+ }

  //Old beheviour
  summaries.AddRange(BenchmarkRunner.Run(filteredBenchmarks, effectiveConfig));
```

You can see all the details in [pull request](https://github.com/dotnet/BenchmarkDotNet/pull/914).

This task did not require knowledge of BenchmarkDotNet internals like running the tests. It was good lesson, you do not need to know the entire source code of the project. You should keep focus on the very specific problem which you want to solve. It saves a lot of time. 


## Summary

I did this task in no time but doing it made me really happy. The most rewarding thing is that this feature is useful both for me and also for people around the world. The funny thing is that Adam Sitnik showed this feature on Get.Net conference. I was absent but a colleague sent me a photo:

![YamlDotNet configurations](/images/GetNet.jpg#mid) 