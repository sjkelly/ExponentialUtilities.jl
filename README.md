# ExponentialUtilities

[![Join the chat at https://gitter.im/JuliaDiffEq/Lobby](https://badges.gitter.im/JuliaDiffEq/Lobby.svg)](https://gitter.im/JuliaDiffEq/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://github.com/SciML/ExponentialUtilities.jl/workflows/CI/badge.svg)](https://github.com/SciML/ExponentialUtilities.jl/actions?query=workflow%3ACI)
[![Coverage Status](https://coveralls.io/repos/github/SciML/ExponentialUtilities.jl/badge.svg?branch=master)](https://coveralls.io/github/SciML/ExponentialUtilities.jl?branch=master)
[![codecov](https://codecov.io/gh/SciML/ExponentialUtilities.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/SciML/ExponentialUtilities.jl)

ExponentialUtilities is a package of utility functions for matrix functions of exponential type, including functionality for the matrix exponential and phi-functions. The tools are used by the exponential integrators in OrdinaryDiffEq. The package has no external dependencies, so it can also be used independently.

## Matrix-phi-vector product

The main functionality of ExponentialUtilities is the computation of matrix-phi-vector products. The phi functions are defined as

```
ϕ_0(z) = exp(z)
ϕ_(k+1)(z) = (ϕ_k(z) - 1) / z
```

In exponential algorithms, products in the form of `ϕ_m(tA)b` is frequently encountered. Instead of computing the matrix function first and then computing the matrix-vector product, the common alternative is to construct a [Krylov subspace](https://en.wikipedia.org/wiki/Krylov_subspace) `K_m(A,b)` and then approximate the matrix-phi-vector product.

### `expv` and `phiv`

```julia
expv(t,A,b;kwargs) -> exp(tA)b
phiv(t,A,b,k;kwargs) -> [ϕ_0(tA)b ϕ_1(tA)b ... ϕ_k(tA)b][, errest]
```

For `phiv`, *all* `ϕ_m(tA)b` products up to order `k` is returned as a matrix. This is because it's more economical to produce all the results at once than individually. A second output is returned if `errest=true` in `kwargs`. The error estimate is given for the second-to-last product, using the last product as an estimator. If `correct=true`, then `ϕ_0` through `ϕ_(k-1)` are updated using the last Arnoldi vector. The correction algorithm is described in [1].

You can adjust how the Krylov subspace is constructed by setting various keyword arguments. See the Arnoldi iteration section for more details.

### `expv_timestep` and `phiv_timestep`

Unlike `expv` and `phiv`, the timestepping methods divide `t` into smaller time steps and compute the product step-by-step. By doing this in smaller chunks, the methods allow for finer error control as well as adaptation. The timestepping algorithm is described in [1], which is based upon the numerical package Expokit [2].

```julia
exp_timestep(ts,A,b;kwargs) -> U
```

Evaluates the matrix exponentiation-vector product `u = exp(tA)b` using time stepping.

```julia
phiv_timestep(ts,A,[b_0 b_1 ... b_p];kwargs) -> U
```

Evaluates the linear combination of phi-vector products `u = ϕ_0(tA)b_0 + tϕ_1(tA)b_1 + ... + t^pϕ_p(tA)b_p` using time stepping.

In both cases, `ts` is an array of time snapshots for u, with `U[:,j] ≈ u(ts[j])`. `ts` can also be just one value, in which case only the end result is returned and `U` is a vector.

Apart from keyword arguments that affect the computation of Krylov subspaces (see the Arnoldi iteration section), you can also adjust the timestepping behavior using the arguments. By setting `adaptive=true`, the time step and Krylov subsapce size adaptation scheme of Niesen & Wright is used and the relative tolerance of which can be set using the keyword parameter `tol`. The `delta` and `gamma` parameter of the adaptation scheme can also be adjusted. The `tau` parameter adjusts the size of the timestep (and for `adaptive=true`, the initial timestep). By default, it is calculated using a heuristic formula by Niesen & Wright.

### Support for matrix-free operators

You can use any object as the "matrix" `A` as long as it implements the following linear operator interface:

* `Base.eltype(A)`
* `Base.size(A, dim)`
* `LinearAlgebra.mul!(y, A, x)` (for computing `y = A * x` in place).
* `LinearAlgebra.opnorm(A, p=Inf)`. If this is not implemented or the default implementation can be slow, you can manually pass in the operator norm (a rough estimate is fine) using the keyword argument `opnorm`.
* `LinearAlgebra.ishermitian(A)`. If this is not implemented or the default implementation can be slow, you can manually pass in the value using the keyword argument `ishermitian`.

## Matrix-phi function `phi`

```julia
phi(z,k[;cache]) -> [ϕ_0(z),ϕ_1(z),...,ϕ_k(z)]
```

Compute ϕ function directly. `z` can both be a scalar or a `AbstractMatrix` (note that unlike the previous functions, you *need* to use a concrete matrix). This is used by the caching versions of the ExpRK integrators to precompute the operators.

Instead of using the recurrence relation, which is numerically unstable, a formula given by Sidje is used [2].

## Arnoldi iteration `arnoldi`

```julia
arnoldi(A,b[;m,tol,opnorm,iop,cache]) -> Ks
```

Performs [Arnoldi iterations](https://en.wikipedia.org/wiki/Arnoldi_iteration) to obtain the Krylov subspace `K_m(A,b)`. The result is a `KrylovSubspace` that can be used by `phiv` via the alternative interface

```julia
phiv(t,Ks,k;kwargs) -> [ϕ_0(tA)b ϕ_1(tA)b ... ϕ_k(tA)b][, errest]
```

The reason for having this alternative interface is that we may want to compute `ϕ_m(tA)b` for different values of `t`. In this case, we can compute `Ks` just once (which is expensive) and follow up with several `phiv` calls using `Ks` (which is not as expensive).

For `arnoldi`, if `A` is hermitian, then the more efficient [Lanczos algorithm](https://en.wikipedia.org/wiki/Lanczos_algorithm) is used instead. For cases when `A` is almost hermitian or when accuracy is not important, the incomplete orthogonalization procedure (IOP) can be used by setting the IOP length `iop` in `kwargs`.

For the other keyword arguments, `m` determines the dimension of the Krylov subspace and `tol` is the relative tolerance used to determine the "happy-breakdown" condition. You can also set custom operator norm in `opnorm`, e.g. efficient norm estimation functions instead of the default `LinearAlgebra.opnorm`. Only `opnorm(A, Inf)` needs to be defined.

## Matrix exponential `exponential!`

```julia
exponential!(A[,method[,cache]]) -> E
```


Computes the matrix exponential of `A` with method specified in `method`. The contents of `A` is modified allowing for less allocations. The `method` parameter specifies the implementation and implementation parameters, e.g. objects of the type `ExpMethodNative`,
`ExpMethodDiagonalization`, `ExpMethodGeneric`, `ExpMethodHigham2005`. Memory
needed in the method can be preallocated and provided in parameter `cache` in order to recycle in subsequent calls. The preallocation is done with the command `alloc_mem`: `cache=alloc_mem(A,method)`.

Methods available:

* `ExpMethodHigham2005`: A pure Julia implementation of a non-allocating matrix exponential using the Destructive matrix exponential using algorithm
from Higham, 2008. Mostly generic, though the coefficients are geared towards 64-bit floating point calculations, and the
use of BLAS requires a `StridedMatrix`.
* `ExpMethodHigham2005Base`: Same as `ExpMethodHigham2005` but follows `Base.exp` closer
* `ExpMethodGeneric{Vk}`: A pure Julia generic implementation of the exponential function using the [scaling and squaring method](https://doi.org/10.1137/04061101X), working on any `x` for which the functions `LinearAlgebra.opnorm`, `+`, `*`, `^`, and `/` (including addition with UniformScaling objects) are defined. Use the argument `Vk` to adjust the number of terms used in the Pade approximants at compile time.
* `ExpMethodNative`. Calls `Base.exp`
* `ExpMethodDiagonalization`. Computes the exponential with explicit diagonalization via a call to `eigen`.


## Advanced features

"In-place" versions for `phi`, `arnoldi`, `expv`, `phiv`, `expv_timestep` and `phiv_timestep` are available as `phi!`, `arnoldi!`, `expv!`, `phiv!`, `expv_timestep!` and `phiv_timestep!`. You can refer to the docstrings for more information.

In addition, you may provide pre-allocated caches to the functions to further improve efficiency. In particular, dedicated cache types for `expv!` and `phiv!` are available as `ExpvCache` and `PhivCache`. Both of them can be dynamically resized, which is crucial for the adaptive `phiv_timestep!` method.

## References

[1] Niesen, J., & Wright, W. (2009). A Krylov subspace algorithm for evaluating the φ-functions in exponential integrators. arXiv preprint arXiv:0907.4631.

[2] Sidje, R. B. (1998). Expokit: a software package for computing matrix exponentials. ACM Transactions on Mathematical Software (TOMS), 24(1), 130-15.

[3] Koskela, A. (2015). Approximating the matrix exponential of an advection-diffusion operator using the incomplete orthogonalization method. In Numerical Mathematics and Advanced Applications-ENUMATH 2013 (pp. 345-353). Springer, Cham.

[4] Higham, N. J. (2005). "The scaling and squaring method for the matrix exponential revisited." SIAM J. Matrix Anal. Appl. Vol. 26, No. 4, pp. 1179–1193
