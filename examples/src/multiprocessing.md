# Multiprocessing
Tutorial by Jonas Wilfert, Tobias Thummerer

## License


```julia
# Copyright (c) 2021 Tobias Thummerer, Lars Mikelsons, Josef Kircher, Johannes Stoljar, Jonas Wilfert
# Licensed under the MIT license. 
# See LICENSE (https://github.com/thummeto/FMI.jl/blob/main/LICENSE) file in the project root for details.
```

## Motivation
This Julia Package *FMI.jl* is motivated by the use of simulation models in Julia. Here the FMI specification is implemented. FMI (*Functional Mock-up Interface*) is a free standard ([fmi-standard.org](http://fmi-standard.org/)) that defines a container and an interface to exchange dynamic models using a combination of XML files, binaries and C code zipped into a single file. The user can thus use simulation models in the form of an FMU (*Functional Mock-up Units*). Besides loading the FMU, the user can also set values for parameters and states and simulate the FMU both as co-simulation and model exchange simulation.

## Introduction to the example
This example shows how to parallelize the computation of an FMU in FMI.jl. We can compute a batch of FMU-evaluations in parallel with different initial settings.
Parallelization can be achieved using multithreading or using multiprocessing. This example shows **multiprocessing**, check `multithreading.ipynb` for multithreading.
Advantage of multithreading is a lower communication overhead as well as lower RAM usage.
However in some cases multiprocessing can be faster as the garbage collector is not shared.


The model used is a one-dimensional spring pendulum with friction. The object-orientated structure of the *SpringFrictionPendulum1D* can be seen in the following graphic.

![svg](https://github.com/thummeto/FMI.jl/blob/main/docs/src/examples/pics/SpringFrictionPendulum1D.svg?raw=true)  


## Target group
The example is primarily intended for users who work in the field of simulations. The example wants to show how simple it is to use FMUs in Julia.


## Other formats
Besides, this [Jupyter Notebook](https://github.com/thummeto/FMI.jl/blob/examples/examples/src/multiprocessing.ipynb) there is also a [Julia file](https://github.com/thummeto/FMI.jl/blob/examples/examples/src/multiprocessing.jl) with the same name, which contains only the code cells and for the documentation there is a [Markdown file](https://github.com/thummeto/FMI.jl/blob/examples/examples/src/multiprocessing.md) corresponding to the notebook.  


## Getting started

### Installation prerequisites
|     | Description                       | Command                   | Alternative                                    |   
|:----|:----------------------------------|:--------------------------|:-----------------------------------------------|
| 1.  | Enter Package Manager via         | ]                         |                                                |
| 2.  | Install FMI via                   | add FMI                   | add " https://github.com/ThummeTo/FMI.jl "     |
| 3.  | Install FMIZoo via                | add FMIZoo                | add " https://github.com/ThummeTo/FMIZoo.jl "  |
| 4.  | Install FMICore via               | add FMICore               | add " https://github.com/ThummeTo/FMICore.jl " |
| 5.  | Install BenchmarkTools via        | add BenchmarkTools        |                                                |

## Code section



Adding your desired amount of processes:


```julia
using Distributed
n_procs = 2
addprocs(n_procs; exeflags=`--project=$(Base.active_project()) --threads=auto`, restrict=false)
```




    2-element Vector{Int64}:
     2
     3



To run the example, the previously installed packages must be included. 


```julia
# imports
@everywhere using FMI
@everywhere using FMIZoo
@everywhere using BenchmarkTools
```

Checking that we workers have been correctly initialized:


```julia
workers()

@everywhere println("Hello World!")

# The following lines can be uncommented for more advanced informations about the subprocesses
# @everywhere println(pwd())
# @everywhere println(Base.active_project())
# @everywhere println(gethostname())
# @everywhere println(VERSION)
# @everywhere println(Threads.nthreads())
```

    Hello World!
          From worker 2:	Hello World!
          From worker 3:	Hello World!


### Simulation setup

Next, the batch size and input values are defined.


```julia

# Best if batchSize is a multiple of the threads/cores
batchSize = 16

# Define an array of arrays randomly
input_values = collect(collect.(eachrow(rand(batchSize,2))))
```




    16-element Vector{Vector{Float64}}:
     [0.7009592632073671, 0.14122291447485136]
     [0.8519292905409173, 0.7104494732070307]
     [0.7606686308505661, 0.49840360207929746]
     [0.565472181549699, 0.04185919969249141]
     [0.7788103334923618, 0.7887233988123873]
     [0.007288799443920135, 0.7143050138806146]
     [0.6148902464818164, 0.009614329303400448]
     [0.0115559933710212, 0.6372517905888284]
     [0.5925652462865798, 0.45044685351427993]
     [0.3435282290087226, 0.14632715398553608]
     [0.8854273284681342, 0.2100108915729616]
     [0.7987025257553151, 0.9882778011693342]
     [0.43449998729462125, 0.744833838733684]
     [0.8381116389870525, 0.38878269943700017]
     [0.45890039802697047, 0.602872539679466]
     [0.2451410335013151, 0.6865506033614049]



### Shared Module
For Distributed we need to embed the FMU into its own `module`. This prevents Distributed from trying to serialize and send the FMU over the network, as this can cause issues. This module needs to be made available on all processes using `@everywhere`.


```julia
@everywhere module SharedModule
    using FMIZoo
    using FMI

    t_start = 0.0
    t_step = 0.1
    t_stop = 10.0
    tspan = (t_start, t_stop)
    tData = collect(t_start:t_step:t_stop)

    model_fmu = FMIZoo.fmiLoad("SpringPendulum1D", "Dymola", "2022x")
end
```

We define a helper function to calculate the FMU and combine it into an Matrix.


```julia
@everywhere function runCalcFormatted(fmu, x0, recordValues=["mass.s", "mass.v"])
    data = fmiSimulateME(fmu, SharedModule.tspan; recordValues=recordValues, saveat=SharedModule.tData, x0=x0, showProgress=false, dtmax=1e-4)
    return reduce(hcat, data.states.u)
end
```

Running a single evaluation is pretty quick, therefore the speed can be better tested with BenchmarkTools.


```julia
@benchmark data = runCalcFormatted(SharedModule.model_fmu, rand(2))
```

    [33m[1m┌ [22m[39m[33m[1mWarning: [22m[39mUsing arrays or dicts to store parameters of different types can hurt performance.
    [33m[1m│ [22m[39mConsider using tuples instead.
    [33m[1m└ [22m[39m[90m@ SciMLBase ~/.julia/packages/SciMLBase/kTUaf/src/performance_warnings.jl:32[39m





    BenchmarkTools.Trial: 3 samples with 1 evaluation.
     Range [90m([39m[36m[1mmin[22m[39m … [35mmax[39m[90m):  [39m[36m[1m1.674 s[22m[39m … [35m  1.704 s[39m  [90m┊[39m GC [90m([39mmin … max[90m): [39m3.70% … 3.66%
     Time  [90m([39m[34m[1mmedian[22m[39m[90m):     [39m[34m[1m1.678 s              [22m[39m[90m┊[39m GC [90m([39mmedian[90m):    [39m3.70%
     Time  [90m([39m[32m[1mmean[22m[39m ± [32mσ[39m[90m):   [39m[32m[1m1.685 s[22m[39m ± [32m16.533 ms[39m  [90m┊[39m GC [90m([39mmean ± σ[90m):  [39m3.69% ± 0.02%
    
      [34m█[39m[39m [39m [39m [39m [39m [39m█[39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [32m [39m[39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m [39m█[39m [39m 
      [34m█[39m[39m▁[39m▁[39m▁[39m▁[39m▁[39m█[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[32m▁[39m[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m▁[39m█[39m [39m▁
      1.67 s[90m         Histogram: frequency by time[39m         1.7 s [0m[1m<[22m
    
     Memory estimate[90m: [39m[33m467.05 MiB[39m, allocs estimate[90m: [39m[33m12001282[39m.



### Single Threaded Batch Execution
To compute a batch we can collect multiple evaluations. In a single threaded context we can use the same FMU for every call.


```julia
println("Single Threaded")
@benchmark collect(runCalcFormatted(SharedModule.model_fmu, i) for i in input_values)
```

    Single Threaded





    BenchmarkTools.Trial: 1 sample with 1 evaluation.
     Single result which took [34m26.489 s[39m (3.99% GC) to evaluate,
     with a memory estimate of [33m7.30 GiB[39m, over [33m192020500[39m allocations.



### Multithreaded Batch Execution
In a multithreaded context we have to provide each thread it's own fmu, as they are not thread safe.
To spread the execution of a function to multiple processes, the function `pmap` can be used.


```julia
println("Multi Threaded")
@benchmark pmap(i -> runCalcFormatted(SharedModule.model_fmu, i), input_values)
```

    Multi Threaded
          From worker 2:	[33m[1m┌ [22m[39m[33m[1mWarning: [22m[39mUsing arrays or dicts to store parameters of different types can hurt performance.
          From worker 2:	[33m[1m│ [22m[39mConsider using tuples instead.
          From worker 2:	[33m[1m└ [22m[39m[90m@ SciMLBase ~/.julia/packages/SciMLBase/kTUaf/src/performance_warnings.jl:32[39m
          From worker 3:	[33m[1m┌ [22m[39m[33m[1mWarning: [22m[39mUsing arrays or dicts to store parameters of different types can hurt performance.
          From worker 3:	[33m[1m│ [22m[39mConsider using tuples instead.
          From worker 3:	[33m[1m└ [22m[39m[90m@ SciMLBase ~/.julia/packages/SciMLBase/kTUaf/src/performance_warnings.jl:32[39m





    BenchmarkTools.Trial: 1 sample with 1 evaluation.
     Single result which took [34m14.135 s[39m (0.00% GC) to evaluate,
     with a memory estimate of [33m86.27 KiB[39m, over [33m1457[39m allocations.



As you can see, there is a significant speed-up in the median execution time. But: The speed-up is often much smaller than `n_procs` (or the number of physical cores of your CPU), this has different reasons. For a rule of thumb, the speed-up should be around `n/2` on a `n`-core-processor with `n` Julia processes.

### Unload FMU

After calculating the data, the FMU is unloaded and all unpacked data on disc is removed.


```julia
@everywhere fmiUnload(SharedModule.model_fmu)
```

### Summary

In this tutorial it is shown how multi processing with `Distributed.jl` can be used to improve the performance for calculating a Batch of FMUs.
