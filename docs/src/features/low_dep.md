# [Reduced Compile Time, Optimizing Runtime, and Low Dependency Usage](@id low_dep)

While DifferentialEquations.jl's defaults strike a balance between runtime
and compile time performance, users should be aware that there are many
options for controlling the dependency sizing and the compilation caching
behavior to further refine this trade-off. The following methods are
available for such controls.

## Controlling Function Specialization and Precompilation

By default, DifferentialEquations.jl solvers make use of function wrapping
techniques in order to fully precompile the solvers and thus decrease the
compile time. However, in some cases you may wish to control this behavior,
pushing more towards faster runtimes or faster compile times. This can be
done by using the `specialization` arguments of the `AbstractDEProblem` 
constructors.

For example, with the `ODEProblem` we have `ODEProblem{iip,specialize}(...)`.
The this second type parameter controls the specialization level with the
following choices:

- `SciMLBase.AutoSpecialize`: the default. Uses a late wrapping scheme to
  hit a balance between runtime and compile time.
- `SciMLBase.NoSpecialize`: this will never specialize on the constituant
  functions, having the least compile time but the highest runtime.
- `SciMLBase.FullSpecialize`: this will fully re-specialize the solver
  on the given ODE, achieving the fastest runtimes while increasing the
  compile times. This is what is recommended when benchmarking and when
  running long computations, such as in optimization loops.

For more information on the specialization levels, please see
[the SciMLBase documentation on specialization levels](https://scimlbase.sciml.ai/stable/interfaces/Problems/#Specialization-Levels)

## Decreasing Dependency Size by Direct Dependence on Specific Solvers

DifferentialEquations.jl is a large library containing the functionality of
many different solver and addon packages. However in many cases you may want
to cut down on the size of the dependency and only use the parts of the
the library which are essential to your application. This is possible
due to SciML's modular package structure.

### Common Example: Using only OrdinaryDiffEq.jl

One common example is using only the ODE solvers OrdinaryDiffEq.jl. The solvers all
reexport SciMLBase.jl (which holds the problem and solution types) and so
OrdinaryDiffEq.jl is all that's needed. Thus replacing

```julia
using DifferentialEquations
```

with

```julia
#Add the OrdinaryDiffEq Package first!
#using Pkg; Pkg.add("OrdinaryDiffEq")
using OrdinaryDiffEq
```

will work if these are the only features you are using.

### Generalizing the Idea

In general, you will always need SciMLBase.jl, since it defines all of the
fundamental types, but the solvers will automatically reexport it.
For solvers, you typically only need that solver package.
So SciMLBase+Sundials, SciMLBase+LSODA, etc. will get you the common interface
with that specific solver setup. SciMLBase.jl is a very lightweight dependency,
so there is no issue here! For PDEs, you normally need SciMLBase+DiffEqPDEBase
in addition to the solver package.

For the addon packages, you will normally need SciMLBase, the solver package
you choose, and the addon package. So for example, for parameter estimation you
would likely want SciMLBase+OrdinaryDiffEq+DiffEqParamEstim. If you aren't sure
which package a specific command is from, then use `@which`. For example, from
the parameter estimation docs we have:

```julia
using DifferentialEquations
function f(du,u,p,t)
  dx = p[1]*u[1] - u[1]*u[2]
  dy = -3*u[2] + u[1]*u[2]
end

u0 = [1.0;1.0]
tspan = (0.0,10.0)
p = [1.5]
prob = ODEProblem(f,u0,tspan,p)
sol = solve(prob,Tsit5())
t = collect(range(0, stop=10, length=200))
randomized = VectorOfArray([(sol(t[i]) + .01randn(2)) for i in 1:length(t)])
using RecursiveArrayTools
data = convert(Array,randomized)
cost_function = build_loss_objective(prob,t,data,Tsit5(),maxiters=10000)
```

If we wanted to know where `build_loss_objective` came from, we can do:

```julia
@which build_loss_objective(prob,t,data,Tsit5(),maxiters=10000)

(::DiffEqParamEstim.#kw##build_loss_objective)(::Array{Any,1}, ::DiffEqParamEstim.#build_loss_objective, prob::SciMLBase.DEProblem, t, data, alg)
```

This says it's in the DiffEqParamEstim.jl package. Thus in this case, we could have
done

```julia
using OrdinaryDiffEq, DiffEqParamEstim
```

instead of the full `using DifferentialEquations`. Note that due to the way
Julia dependencies work, any internal function in the package will work. The only
dependencies you need to explicitly `using` are the functions you are specifically
calling. Thus this method can be used to determine all of the DiffEq packages
you are using.
