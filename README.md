# vani-algebra

Polynomial root-finding and nonlinear equation system library for the
[vāṇी compiler](https://github.com/enthusiasticgeek/vani-compiler).

Depends on [vani-matrix](https://github.com/enthusiasticgeek/vani-matrix)
(`mat_zeros`, `mat_eig_power`, `mat_solve`) and
[vani-calculus](https://github.com/enthusiasticgeek/vani-calculus)
(`poly_eval`, `poly_deriv_coeffs`, `poly_mul`).

## Add to your project

```toml
# vani.toml
[deps]
algebra = { registry = "kosh", version = "^0.1" }
```

```sh
vanic add algebra
vanic build
```

## What's included (v0.1.0 — complete; see TODO.md)

| Module | Functions |
|---|---|
| Polynomial construction / companion-matrix machinery | `algebra_poly_from_roots`, `algebra_poly_normalize_monic`, `algebra_poly_companion`, `algebra_poly_deflate`, `algebra_poly_refine_newton` |
| Cubic (closed form) | `algebra_cubic_roots_real` |
| General real-root finder (any degree) | `algebra_poly_roots_real`, `algebra_quartic_roots_real` |
| Nonlinear systems of equations | `algebra_newton_system`, `algebra_newton_system_fd` |

## Scope: real roots only, and quartic is not a separate closed form

Read this before using the library -- both are deliberate, documented
tradeoffs, not oversights:

- **Quadratic** isn't reimplemented: the compiler builtin
  `f64_quadratic_root` already covers it. This package derives the second
  root from it via Vieta's formulas (`r1 + r2 = -b/a`) instead of computing
  the discriminant twice.
- **Cubic** gets a genuine closed form: Cardano's method, using the
  trigonometric formula when the depressed cubic has 3 real roots and the
  two-cube-root formula when it has 1.
- **Quartic and higher degree do NOT get a hand-derived closed form.**
  Ferrari's method for the quartic has a delicate resolvent-cubic
  derivation that's easy to get subtly wrong without a symbolic-math
  reference to check against. Instead, degree >= 4 uses one general,
  well-tested path: build the companion matrix, find its dominant
  eigenvalue via vani-matrix's `mat_eig_power`, polish it with
  `algebra_poly_refine_newton`, deflate the polynomial by **synthetic
  division** (not matrix deflation -- `mat_eig_deflate`'s `A - λvvᵀ` trick
  assumes a symmetric matrix with orthonormal eigenvectors, which a
  companion matrix is not; using it here would silently corrupt every root
  after the first), and repeat until the remaining degree is <= 3.
  `algebra_quartic_roots_real` is a thin wrapper over this same path, not a
  separate implementation.
- **This whole package finds real roots only.** `mat_eig_power`'s power
  iteration converges to a real dominant eigenvalue and has no mechanism to
  detect complex-conjugate eigenvalue pairs (same caveat vani-matrix's own
  `mat_cond` documents). A polynomial whose remaining factor at some
  deflation step is dominated by a complex-conjugate pair will not have
  that pair detected.
- **No wrapper for plain linear systems**: call vani-matrix's `mat_solve`
  directly, there's nothing for this package to add on top. The nonlinear
  solvers here (`algebra_newton_system`/`_fd`) use `mat_solve` internally
  at every Newton step.

## Correctness

Every root-finding path is checked by substituting the found root(s) back
into the original polynomial/system (not just compared to an expected
value), plus a composed round-trip test: build a degree-6 polynomial from
six chosen roots with `algebra_poly_from_roots`, then recover them with
`algebra_poly_roots_real` -- exercising the companion matrix, eigenvalue
solve, synthetic deflation, and Newton polish together. See `tests/`.

## What this library does NOT provide

These are already vāṇी compiler builtins — call them directly, no import needed:

`sin` `cos` `sqrt` `abs` `acos` `exp` `f64_pi()` `f64_cbrt`
`f64_quadratic_root` `push` `pop` `len` `set` `vec`

## License

MIT
