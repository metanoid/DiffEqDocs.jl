# DDE Solvers

`solve(prob::AbstractDDEProblem,alg;kwargs)`

Solves the DDE defined by `prob` using the algorithm `alg`. If no algorithm is
given, a default algorithm will be chosen.

## Recommended Methods

The recommended method for DDE problems are the `MethodOfSteps` algorithms.
These are constructed from an OrdinaryDiffEq.jl algorithm as follows:

```julia
MethodOfSteps(alg;constrained=false,
             fixedpoint_abstol = nothing,
             fixedpoint_reltol = nothing,
             fixedpoint_norm   = nothing,
             max_fixedpoint_iters = 10)
```

where `alg` is an OrdinaryDiffEq.jl algorithm. Most algorithms should work.

### Nonstiff DDEs

The standard choice is `MethodOfSteps(Tsit5())`. This is a highly efficient
FSAL 5th order algorithm with free interpolants which should
handle most problems. For fast solving at where non-strict error control is
needed, choosing `BS3()` can do well. Using `BS3` is similar to the MATLAB
`dde23`. For algorithms where strict error control is needed, it is recommended
that one uses `Vern6()`. Benchmarks show that going to higher order methods like
`DP8()` may not be beneficial.

### Stiff DDEs and Differential-Algebraic Delay Equations (DADEs)

For stiff DDEs, the SDIRK and Rosenbrock methods are very efficient as they will
reuse the Jacobian in the unconstrained stepping iterations. One should choose
from the methods which have stiff-aware interpolants for better stability.
`MethodOfSteps(Rosenbrock23())` is a good low order method choice. Additionally,
the `Rodas` methods like `MethodOfSteps(Rodas4())` are good choices because of
their higher order stiff-aware interpolant.

Additionally, DADEs can be solved by specifying the problem in mass matrix form.
The Rosenbrock methods are good choices in these situations.

### Lag Handling

Lags are declared separately from their use. One can use any lag by simply using
the interpolant of `h` at that point. However, one should use caution in order
to achieve the best accuracy. When lags are declared, the solvers can more
efficiently be more accurate. Constant delays are propagated until the
order is higher than the order of the integrator. If state-dependent delays are
declared, the algorithm will detect discontinuities arising from these delays and
adjust the step size such that these discontinuities are included in the mesh.
This way, all discontinuities are treated exactly.

If there are undeclared lags, the discontinuities due to delays are not tracked.
In this case, one should only use residual control methods like `RK4()`,
which is the current best choice, as these will step more accurately.
Still, residual control is an error-prone method. We recommend setting the
tolerances lower in order to get accurate results, though this may be costly
since it will use a rejection-based approach to adapt to the delay discontinuities.

## Special Keyword Arguments

- `minimal_solution` - Allows the algorithm to delete past history when `dense`
  and `save_everystep` are false, and only constant lags are specified. Defaults
  to true. If lags can grow or some lags are undeclared this may need to be set
  to false since it might impact the quality of the solution otherwise.

- `discontinuity_interp_points` - Number of interpolation points used to track
  discontinuities arising from dependent delays. Defaults to 10. Only relevant
  if dependent delays are declared.

- `discontinuity_abstol` and `discontinuity_reltol` - These are absolute and
  relative tolerances used by the check whether the time point at the beginning
  of the current step is a discontinuity arising from dependent delays. Defaults
  to 1/10^12 and 0. Only relevant if dependent delays are declared.

### Note

If the method is having trouble, one may want to adjust the parameters of the
fixed-point iteration. Decreasing the absolute tolerance `fixedpoint_abstol` and the
relative tolerance `fixedpoint_reltol`, and increasing the maximal number of iterations
`max_fixedpoint_iters` can help ensure that the steps are correct. If the problem still
is not correctly converging, one should lower `dtmax`. In the worst case scenario, one
may need to set `constrained=true` which will constrain timesteps to at most the size
of the minimal lag and hence forces more stability at the cost of smaller timesteps.
