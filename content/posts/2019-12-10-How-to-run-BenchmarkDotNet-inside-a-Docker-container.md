---
title: "How to run BenchmarkDotNet in a Docker container"
description: "In this post I would like to show how to run BenchmarkDotNet in a Docker container."
date: 2019-12-10T02:13:54-04:00
year: "2019"
authors:
  - Wojciech Nagórski
authorurl: "https://github.com/wojtpl2"
categories:
  - OpenSource
  - BenchmarkDotNet
tags:
  - BenchmarkDotNet
  - Docker
---

The BenchmarkDotNet library is great for creating benchmarks that can be run on a local machine in a very simple way. But what if you wanted to run them in a Docker container with a different operating system or using a different version of .Net Core. In this post I would like to show you how to dockerize your benchmarks. 

Firstly, a new solution and a project need to be created. To do this, follow the instructions below.
I will be using .NET Core CLI, because it works on most systems. 

```ini
# 1. create new "BenchmarkDotNetInDocker.sln" inside the “BenchmarkDotNetInDocker” directory.  
dotnet new sln --name BenchmarkDotNetInDocker --output BenchmarkDotNetInDocker 

# 2. navigate to the previously created directory.  
cd BenchmarkDotNetInDocker 

# 3. create a new console project with the name of “BenchmarkDotNetInDocker”. 
dotnet new console --name BenchmarkDotNetInDocker 

# 4. add the previously created project to the sln file. 
dotnet sln add .\BenchmarkDotNetInDocker\BenchmarkDotNetInDocker.csproj 

# 5. add the BenchmarkDotNet nuget package to our project. 
dotnet add .\BenchmarkDotNetInDocker\BenchmarkDotNetInDocker.csproj package BenchmarkDotNet 

# 6. restore nuget packages 
dotnet restore 
```

Now our project is ready to add the first benchmark, but before we do it, we need to change the `Program.cs` file, as shown below. This code uses the `BenchmarkSwitcher` class which allows us to choose a benchmark to run during the execution. 
```c#
using System;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

namespace BenchmarkDotNetInDocker
{
    class Program
    {
        static void Main(string[] args)
        {
            BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args);
        }
    }
}
```
Then we need to create a new `.cs` file, e. g. `MyBenchmark.cs` that will contain the benchmark. As you can see below, the `MyBenchmark` class contains the `Sum` method which is marked with the `[Benchmark]` attribute. And that's all.

```c#
	public class MyBenchmark
    {
        [Benchmark]
        public int Sum()
        {
            int result = 0;
            for (int i = 0; i < 100; i++)
            {
                result += i;
            }

            return result;
        }
    }
```

Now we can run our project. If you are inside the project directory, just run:
```ini
dotnet run -c Release -- --filter *Sum*
```
If you are in the root directory, where the `sln` file is located, you should indicate which project you want to run:
```ini
dotnet run -c Release -p .\BenchmarkDotNetInDocker\BenchmarkDotNetInDocker.csproj -- --filter *Sum*
```
The `--filter *Sum*` parameter specifies which benchmarks should be run. In this particular case, we want to run all benchmarks that contain `Sum` inside their names. 

Having run this command, your benchmark will start. When it finishes, you will see the summary: 
```ini
[...]
// * Summary *

BenchmarkDotNet=v0.12.0, OS=Windows 10.0.16299.1387 (1709/FallCreatorsUpdate/Redstone3)
Intel Core i7-4770 CPU 3.40GHz (Haswell), 1 CPU, 8 logical and 4 physical cores
Frequency=3312641 Hz, Resolution=301.8739 ns, Timer=TSC
.NET Core SDK=3.0.100
  [Host]     : .NET Core 3.0.0 (CoreCLR 4.700.19.46205, CoreFX 4.700.19.46214), X64 RyuJIT
  DefaultJob : .NET Core 3.0.0 (CoreCLR 4.700.19.46205, CoreFX 4.700.19.46214), X64 RyuJIT


| Method |     Mean |    Error |   StdDev |
|------- |---------:|---------:|---------:|
|    Sum | 33.10 ns | 0.142 ns | 0.126 ns |
[...]
```

Please note that the summary includes information about the operating system, in my case, it was `Windows 10.0.16299.1387 (1709/FallCreatorsUpdate/Redstone3)`.

## Running BenchmarkDotNet in a Docker container

Now we have to add two additional files in the root directory, where the `sln` file is located. The first one will be a text document called `Dockerfile` and will contain all the commands to assemble a docker image. You can see my `Dockerfile` here: 

```ini
FROM mcr.microsoft.com/dotnet/core/sdk:3.1
WORKDIR /src
COPY . .
RUN dotnet restore

RUN dotnet build "BenchmarkDotNetInDocker/BenchmarkDotNetInDocker.csproj" -c Release -o /src/bin
RUN dotnet publish "BenchmarkDotNetInDocker/BenchmarkDotNetInDocker.csproj" -c Release -o /src/bin/publish

WORKDIR /src/bin/publish
ENTRYPOINT ["dotnet", "BenchmarkDotNetInDocker.dll"]
```
The following is a description of `Dockerfile`, line by line:

- `FROM mcr.microsoft.com/dotnet/core/sdk:3.1` - this line means that an image with .NET Core SDK 3.1 will be used. We should use the SDK version and not a Runtime one because BencharkDotNet generates, builds and runs the benchmarked projects. The full list of available images can be found on [Docker Hub](https://hub.docker.com/_/microsoft-dotnet-core-sdk/).
- `WORKDIR /src` - sets the working direcory to `/src`
- `COPY . .` - Copies all files except ignored ones into the container.
- `RUN dotnet restore` - restores the dependencies and tools of the project.
- `RUN dotnet build "BenchmarkDotNetInDocker/BenchmarkDotNetInDocker.csproj" -c Release -o /src/bin` - builds the project in the `Release` mode into the `/src/bin` directory.
- `RUN dotnet publish "BenchmarkDotNetInDocker/BenchmarkDotNetInDocker.csproj" -c Release -o /src/bin/publish` - publishs the project in the `Release` mode into the `/src/bin/publish` location.
- `WORKDIR /src/bin/publish` - sets the working direcory to `/src/bin/publish` where you can find the published application.
- `ENTRYPOINT ["dotnet", "BenchmarkDotNetInDocker.dll"]` - allows you to configure a container entrypoint. In this case it is the `dotnet BenchmarkDotNetInDocker.dll` command.
 
The second file will be the `.dockerignore` file that will allow you to exclude files from the docker image like a `.gitignore` file allow you to exclude files from your git repository. 

```ini
# .dockerignore

Dockerfile
[b|B]in
[O|o]bj
```

Now we can build our new docker image using the following command. We use the `-t` parameter to name the image, in this case `benchmarkdotnet`.

```ini
docker build -t benchmarkdotnet .
```
Having created the docker image with our project, we can run it using command: 

```ini
docker run -it benchmarkdotnet --filter *Sum* 
```
You should see the output log that contains information about the operating system, in this case it is `OS=debian 10`, and information about the .NET Core version, here - `3.1.0`:

```ini
// * Summary *

BenchmarkDotNet=v0.12.0, OS=debian 10
Intel Core i7-4770 CPU 3.40GHz (Haswell), 1 CPU, 2 logical and 2 physical cores
.NET Core SDK=3.1.100
  [Host]     : .NET Core 3.1.0 (CoreCLR 4.700.19.56402, CoreFX 4.700.19.56404), X64 RyuJIT
  DefaultJob : .NET Core 3.1.0 (CoreCLR 4.700.19.56402, CoreFX 4.700.19.56404), X64 RyuJIT


| Method |     Mean |    Error |   StdDev |
|------- |---------:|---------:|---------:|
|    Sum | 32.65 ns | 0.154 ns | 0.137 ns |
```

All parameters following the name of the docker image go directly to our application. In this case the `--filter *Sum*` parameter goes to our `Program.Main(args)` method and then to the `BenchmarkSwitcher` class which I mentioned earlier. Similarly, we can add additional parameters, such as `--memory` that enables `MemoryDiagnoser` and prints memory statistics.

```ini
docker run -it benchmarkdotnet --filter *Sum* --memory
```

The command above runs our benchmark inside a Docker container and adds additional columns: "Gen 0", "Gen 1", "Gen 2", and "Allocated". 

```ini
| Method |     Mean |    Error |   StdDev | Gen 0 | Gen 1 | Gen 2 | Allocated |
|------- |---------:|---------:|---------:|------:|------:|------:|----------:|
|    Sum | 34.35 ns | 0.334 ns | 0.296 ns |     - |     - |     - |         - |
```

You can also print all the available benchmarks, using either of the following parameters: `--list tree` and `--list flat`: 

```ini
docker run -it benchmarkdotnet --list tree
```

The following output will be printed when the above command has been finished: 

```ini
BenchmarkDotNetInDocker
 └─Benchmark
    └─Sum
```

## Where are the artifacts?


We are currently able to run any benchmark with any set of options. But where are the resulting files? The answer is simple, they are inside the docker container. There are methods of getting them out of the container, but a better approach is to generate them directly into a local directory. For this purpose, we will use the docker volumes. All we have to do is to add the `-v local-path:container-path` parameter, as shown below:

```ini
docker run -v c:\BenchmarkDotNet.ArtifactsFromDocker:/src/bin/publish/BenchmarkDotNet.Artifacts -it benchmarkdotnet --filter *Sum* -m
```

After excecution finishes, all artifacts should be in the `c:\BenchmarkDotNet.ArtifactsFromDocker` directory.
 
## Run benchmarks on different Linux distributions or different .NET Core versions

In Dockerfile above we used the `FROM mcr.microsoft.com/dotnet/core/sdk:3.1` instruction which gave us Debian 10 with .NET Core 3.1. But what if we wanted to change the operating system or the .NET Core version?

The `FROM` instruction has the syntax of: `FROM <image>[:<tag>]`. Therefore, in this particular case we used the `mcr.microsoft.com/dotnet/core/sdk` image and the `3.1` tag. The tag determined the .NET Core version. So if you want use .NET Core 2.2, just use the `2.2` tag instead. Using different tag suffixes, you can also change the operating system. The following table contains example tags for different operating systems. 

| Tag                | OS Version   | .NET Core Version |
| -----------------  | ------------ | ----------------- |
| 3.1 or 3.1-buster  | Debian 10    | .NET Core 3.1     |
| 3.1-alpine         | Alpine 3.10  | .NET Core 3.1     |
| 3.1-bionic         | Ubuntu 18.04 | .NET Core 3.1     |
| 2.2 or 2.2-stretch | Debian 9     | .NET Core 2.2     |
| 2.2-alpine         | Alpine 3.9   | .NET Core 2.2     |
| 2.2-bionic         | Ubuntu 18.04 | .NET Core 2.2     |
| ... |||

All possible tags can be found on [Docker Hub](https://hub.docker.com/_/microsoft-dotnet-core-sdk/).

Of course, .NET Core can be run on other Linux distributions for which Microsoft does not publish official docker images. But this topic is outside of this article’s scope. If you find it problematic, please let me know in the comments below, and I will consider writing a post with some additional information on the topic. 

## Additional information

When you run the benchmark project in Docker, you should get a message:

```ini
Failed to set up high priority. Make sure you have the right permissions. Message: Permission denied
```

It means that BenchmarkDotNet requires additional permissions. Fortunately, you can use a “privileged” container. 

```ini
docker run --privileged -it benchmarkdotnet --filter *Sum*
```

You can find the full description of this mode in the Docker [documentation](https://docs.docker.com/engine/reference/run/). It states that:

*Docker will enable access to all devices on the host as well as set some configuration in AppArmor or SELinux to allow the container nearly all the same access to the host as processes running outside containers on the host.*

## Limitations

This method should be used exclusively for development purposes, unless your production environment clearly requires the application to run within docker. You should always carry out final performance tests in the production environment. 

## Summary

In this article, I have shown how to run BenchmarkDotNet inside a Docker container. With this information you can benchmark your code for different systems and different versions of .NET Core easily. 

As I am going to write about profiling applications on Linux using BenchmarkDotNet as soon as I implement the `EventPipeProfiler` functionality, this post will come in handy in the nearest future... I hope. And you will be able to profile your benchmark inside a Docker container too. If you are already curious about how the profiling is going to function, you can follow PR [dotnet/BenchmarkDotNet#1321](PR.https://github.com/dotnet/BenchmarkDotNet/pull/1321/). 