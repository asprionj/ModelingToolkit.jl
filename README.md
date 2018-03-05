# SciCompDSL.jl

[![Build Status](https://travis-ci.org/JuliaDiffEq/SciCompDSL.jl.svg?branch=master)](https://travis-ci.org/JuliaDiffEq/SciCompDSL.jl)
[![Coverage Status](https://coveralls.io/repos/JuliaDiffEq/SciCompDSL.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/JuliaDiffEq/SciCompDSL.jl?branch=master)
[![codecov.io](http://codecov.io/github/JuliaDiffEq/SciCompDSL.jl/coverage.svg?branch=master)](http://codecov.io/github/JuliaDiffEq/SciCompDSL.jl?branch=master)

SciCompDSL.jl is a domain-specific language for scientific computing. It is used
to build computational graphs as an intermediate representation (IR) of
scientific computing problems. It uses a tagged variable IR in order to allow
specification of complex models and allow for transformations of models. Together,
this is an abstract form of a scientific model that is easy for humans to generate
but also easy for programs to manipulate.

#### Warning: This repository is a work-in-progress

## Introduction

### Example: ODE

Let's build an ODE. First we define some variables. In a differential equation
system, we need to differentiate between our dependent variables, independent
variables, and parameters. Therefore we label them as follows:

```julia
using SciCompDSL

# Define some variables
x = DependentVariable(:x)
y = DependentVariable(:y)
z = DependentVariable(:z)
t = IndependentVariable(:t)
D = Differential(t) # Default of first derivative, Derivative(t,1)
σ = Parameter(:σ)
ρ = Parameter(:ρ)
β = Parameter(:β)
```

Then we build the system:

```julia
eqs = [D*x == σ*(y-x),
       D*y == x*(ρ-z)-y,
       D*z == x*y - β*z]
```

Each operation builds an `Operation` type, and thus `eqs` is an array of
`Operation` and `Variable`s. This holds a tree of the full system that can be
analyzed by other programs. We can turn this into a `DiffEqSystem` via:

```julia
de = DiffEqSystem(eqs,[t],[x,y,z],Variable[],[σ,ρ,β])
de = DiffEqSystem(eqs)
```

where we tell it the variable types and ordering in the first version, or let it
automatically determine the variable types in the second version.
This can then generate the function. For example, we can see the
generated code via:

```julia
SciCompDSL.generate_ode_function(de)

## Which returns:
:((du, u, p, t)->begin
                x = u[1]
                y = u[2]
                z = u[3]
                σ = p[1]
                ρ = p[2]
                β = p[3]
                x_t = σ * (y - x)
                y_t = x * (ρ - z) - y
                z_t = x * y - β * z
                du[1] = x_t
                du[2] = y_t
                du[3] = z_t
            end
        end)
```

and get the generated function via:

```julia
f = DiffEqFunction(de)
```

### Example: Nonlinear System

We can also build nonlinear systems. For compatibility with the
```julia
@Var x y z
@Param σ ρ β

# Define a nonlinear system
eqs = [0 == σ*(y-x),
       0 == x*(ρ-z)-y,
       0 == x*y - β*z]
ns = NonlinearSystem(eqs)
nlsys_func = SciCompDSL.generate_nlsys_function(ns)
```

which generates:

```julia
(du, u, p)->begin  # C:\Users\Chris\.julia\v0.6\SciCompDSL\src\systems.jl, line 51:
        begin  # C:\Users\Chris\.julia\v0.6\SciCompDSL\src\utils.jl, line 2:
            y = u[1]
            x = u[2]
            z = u[3]
            σ = p[1]
            ρ = p[2]
            β = p[3]
            resid[1] = σ * (y - x)
            resid[2] = x * (ρ - z) - y
            resid[3] = x * y - β * z
        end
    end
```

We can use this to build a nonlinear function for use with NLsolve.jl:

```julia
f = @eval eval(nlsys_func)
# Make a closure over the parameters for for NLsolve.jl
f2 = (du,u) -> f(du,u,(10.0,26.0,2.33))
```
