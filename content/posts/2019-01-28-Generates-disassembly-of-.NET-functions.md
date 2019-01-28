---
title: "Generates disassembly of .NET functions"
description: "How to generate the disassembly of .NET functions and how to diff many of them."
date: 2019-01-28T02:13:54-04:00
year: "2019"
authors:
  - Wojciech Nagórski
authorurl: "https://github.com/wojtpl2"
categories:
  - OpenSource
  - BenchmarkDotNet
tags:
  - BenchmarkDotNet
  - Disassembly
---

In this post you will learn how to generate the disassembly of .NET functions and how to diff many of them.

It was not so long ago when I added a new feature to [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) that allows you to generate a diff raport for many dissassembly of .NET function. You can see all the changes in following PRs: [#927](https://github.com/dotnet/BenchmarkDotNet/pull/927), [#937](https://github.com/dotnet/BenchmarkDotNet/pull/937) and [#1022](https://github.com/dotnet/BenchmarkDotNet/pull/1022). 

In this post I will be using the benchmark code from PR [dotnet/coreclr#13626](https://github.com/dotnet/coreclr/pull/13626) to [CoreCLR](https://github.com/dotnet/coreclr) repo. In metioned PR [@mikedn](https://github.com/mikedn) enables JIT to generate more efficient BT instruction in some situations. It is great example to show my exporter in action.

##### Generates disassembly 

In [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) you can easily generate disassembler for .NET function. All you need is add `[DisassemblyDiagnoser]` attribute to your benchmark, like below:

```c#
    [DisassemblyDiagnoser]
    public class IntroDisassembly
    {
        [Benchmark(OperationsPerInvoke = 10_000_000)]
        public int LoweringTESTtoBT()
        {
            int y = 0, x = 0;

            while (x++ < 10_000_000)
                if ((x & (1 << y)) == 0)
                    y++;

            return y;
        }
    }
```

You can also do this with the `-d` or `--disasm` command line arguments.

Running this benchmark will create lot of reports in directory`BenchmarkDotNet.Artifacts\results`. Among them is `ProjectName.IntroDisassembly-asm.pretty.md` that looks like:

**.NET Core 2.1.6 (CoreCLR 4.6.27019.06, CoreFX 4.6.27019.05), 64bit RyuJIT**

```assembly
; BenchmarkDotNet.Samples.IntroDisassemblyCode.LoweringTESTtoBT()
       xor     eax,eax
       xor     edx,edx
       jmp     M00_L01
M00_L00:
       bt      ecx,eax
       mov     edx,ecx
       jb      M00_L01
       inc     eax
M00_L01:
       lea     ecx,[rdx+1]
       cmp     edx,989680h
       jl      M00_L00
       ret
; Total bytes of code 27
```

You can use this feature to understand why one solution is better then other or you can learn about how C# code which you wrote works on your computer.

Nothing new so far ;)

##### Generates diff of two disassembly

Sometimes your code works fast on newer version of .NET Core and works slow on the older one. That is quite common case, because .NET Core is improving very fast. In other case you probably would see performance regression and create issue to [CoreCLR](https://github.com/dotnet/coreclr) repo. 

If you want to generate disassembly of function for multiple .NET versions, just configure your test, e. g.: 

```c#
[Config(typeof(MultipleJits))]
public class IntroDisassembly
{
    [Benchmark(OperationsPerInvoke = 10_000_000)]
    public int LoweringTESTtoBT()
    {
        int y = 0, x = 0;

        while (x++ < 10_000_000)
            if ((x & (1 << y)) == 0)
                y++;

        return y;
    }
}

public class MultipleJits : ManualConfig
{
    public MultipleJits()
    {
        // .NET core 2.0
        Add(Job.ShortRun.With(Platform.X64).With(CsProjCoreToolchain.NetCoreApp20));
        // .NET core 2.1
        Add(Job.ShortRun.With(Platform.X64).With(CsProjCoreToolchain.NetCoreApp21));
        
        // Add disassembly diagnoser with printDiff = true
        Add(DisassemblyDiagnoser.Create(new DisassemblyDiagnoserConfig(printAsm: true, printPrologAndEpilog: false, recursiveDepth: 3, printDiff: true)));
    }
}
```

In this example I added two jobs. One for .NET Core 2.0 and other for .NET Core 2.1. I added also `DisassemblyDiagnoser` with `printDiff` option set to `true`. 

After benchmark run, you can see that [@mikedn's](https://github.com/mikedn) optimization made this code much faster on .NET Core 2.1:

```ini
           Method |     Toolchain |      Mean |     Error |    StdDev |
----------------- |-------------- |----------:|----------:|----------:|
 LoweringTESTtoBT | .NET Core 2.0 | 2.3164 ns | 0.8837 ns | 0.0484 ns |
 LoweringTESTtoBT | .NET Core 2.1 | 0.6531 ns | 0.2882 ns | 0.0158 ns |
```

In directory `BenchmarkDotNet.Artifacts\results` you will see also `ProjectName.IntroDisassembly-asm.pretty.diff.md` that looks:

**BenchmarkDotNet.Samples.IntroDisassemblyCode**
**Diff for LoweringTESTtoBT method between:**
.NET Core 2.0.9 (CoreCLR 4.6.26614.01, CoreFX 4.6.26614.01), 64bit RyuJIT
.NET Core 2.1.6 (CoreCLR 4.6.27019.06, CoreFX 4.6.27019.05), 64bit RyuJIT

```diff
; BenchmarkDotNet.Samples.IntroDisassemblyCode.LoweringTESTtoBT()
        xor     eax,eax
        xor     edx,edx
        jmp     M00_L01
 M00_L00:
-       mov     ecx,eax
-       and     ecx,1Fh
-       mov     edx,1
-       shl     edx,cl
-       mov     ecx,dword ptr [rsp+4]
-       test    edx,ecx
+       bt      ecx,eax
        mov     edx,ecx
-       jne     M00_L01
+       jb      M00_L01
        inc     eax
 M00_L01:
        lea     ecx,[rdx+1]
-       mov     dword ptr [rsp+4],ecx
        cmp     edx,989680h
        jl      M00_L00
-       add     rsp,8
-; Total bytes of code 49
+       ret
+; Total bytes of code 27
```

#### Limitations

Please note that this exporter internally uses the `git diff` command, so it requires a [GIT](https://git-scm.com/) installed on the system. 

#### Summary	

Thanks to this report you can easily see what has been changed in the assembly code on various versions of .NET runtimes. You can also copy and paste it to GitHub. 

