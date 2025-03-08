#fsharp #dotnet
```fsharp
open BenchmarkDotNet.Attributes  
open BenchmarkDotNet.Configs  
open BenchmarkDotNet.Environments  
open BenchmarkDotNet.Jobs  
open BenchmarkDotNet.Running  
  
[<MemoryDiagnoser(displayGenColumns=false)>]  
[<HideColumns("Job", "Error", "StdDev", "Median", "RatioSD")>]  
type Benchmarks () =  
   let x = Some 3  
  
   [<Benchmark>]  
   member _.DoSomeStuffWithOptions () =  
       x  
       |> Option.map ((+) 1)    // Used to allocate on the heap, now it doesn't.  
       |> Option.map ((+) -1)   // Used to allocate on the heap, now it doesn't.  
       |> Option.defaultValue 0  
  
ignore <| BenchmarkRunner.Run<Benchmarks>  
   (DefaultConfig.Instance  
       .AddJob(Job.Default.AsBaseline().WithRuntime CoreRuntime.Core80)  
       .AddJob(Job.Default.WithRuntime CoreRuntime.Core90))
```

output:
```
| Method                 | Runtime  | Mean      | Ratio | Allocated | Alloc Ratio |
|----------------------- |--------- |----------:|------:|----------:|------------:|
| DoSomeStuffWithOptions | .NET 8.0 | 3.8792 ns |  1.00 |      48 B |        1.00 |
| DoSomeStuffWithOptions | .NET 9.0 | 0.6274 ns |  0.16 |         - |        0.00 |
```
