# vani-algebra — TODO

> Compiler builtins that already exist and must NOT be reimplemented:
> `sin` `cos` `sqrt` `abs` `acos` `exp` `f64_pi()` `f64_cbrt`
> `f64_quadratic_root` `push` `pop` `len` `set` `vec`
>
> Depends on vani-matrix (`mat_zeros`, `mat_eig_power`, `mat_solve`) and
> vani-calculus (`poly_eval`, `poly_deriv_coeffs`, `poly_mul`).

---

## v0.1.0 — Implemented ✓

### Polynomial construction / companion-matrix machinery (5 functions)
- [x] `algebra_poly_from_roots` -- product of `(x - root)` via vani-calculus's
      `poly_mul`, the inverse of root-finding
- [x] `algebra_poly_normalize_monic`, `algebra_poly_companion` -- build the
      companion matrix whose eigenvalues are the polynomial's roots
- [x] `algebra_poly_deflate` -- synthetic division (polynomial deflation,
      deliberately NOT matrix deflation -- see README)
- [x] `algebra_poly_refine_newton` -- Newton polish via vani-calculus's
      `poly_eval`/`poly_deriv_coeffs`

### Cubic, closed form (1 function)
- [x] `algebra_cubic_roots_real` -- Cardano's method (trigonometric formula
      for 3 real roots, two-cube-root formula for 1). Validated against a
      three-real-root case, a one-real-root case, the `p==0` special case,
      and a residual check (`a*x^3+b*x^2+c*x+d ~= 0` for every found root)

### General real-root finder, any degree (2 functions)
- [x] `algebra_poly_roots_real` -- dispatches to linear/quadratic direct
      formulas, the cubic closed form, or (degree >= 4) companion matrix +
      `mat_eig_power` + synthetic deflation + Newton polish, looping down
      to degree <= 3. Validated for degrees 1, 2, 4, 5, and a composed
      round-trip: build a degree-6 polynomial from 6 chosen roots via
      `algebra_poly_from_roots`, recover them with this function
- [x] `algebra_quartic_roots_real` -- thin wrapper over the same general
      path, not a separate Ferrari's-method implementation (see README)

### Nonlinear systems of equations (2 functions)
- [x] `algebra_newton_system` -- Newton-Raphson with an analytic Jacobian,
      `mat_solve` at every step. Validated against a 2-equation system
      (circle + line) with a residual check on the returned solution
- [x] `algebra_newton_system_fd` -- same, with a forward-finite-difference
      Jacobian instead of an analytic one (matching vani-optimize's
      precedent of offering both variants). Validated against the same
      system, and in `examples/nonlinear_system_demo.vani` against a
      circle/parabola intersection

### Tests and examples
- [x] `tests/test_cubic.vani` -- all three cubic cases plus a residual
      check against arbitrary coefficients
- [x] `tests/test_poly_roots.vani` -- every helper function individually,
      degrees 1/2/4 through the dispatcher, `algebra_quartic_roots_real`,
      and the from-roots/roots-real composed round trip at degree 6
- [x] `tests/test_newton_system.vani` -- analytic and FD system solvers,
      plus a residual check
- [x] `examples/root_finding_demo.vani` -- builds and solves a degree-6
      polynomial, printing the residual at every found root
- [x] `examples/nonlinear_system_demo.vani` -- circle/parabola intersection
      via the finite-difference solver

### Safety annotations
- [x] `#[bounded_stack(bytes=N)]` on every function, budgets set to `vanic
      check`'s exact reported worst-case (largest: `algebra_quartic_roots_real`
      at 1048 bytes, since it composes coefficient construction with the
      full `algebra_poly_roots_real` -> `mat_eig_power` -> `mat_vec_mul` chain)
- [x] No recursion anywhere in this library -- the degree-reduction loop in
      `algebra_poly_roots_real` is a `while` loop, not self-recursion

---

## Future

No v0.2.0 is currently planned. Candidates if a concrete need shows up:
complex root support (would need a vani-complex dependency and a genuinely
different companion-matrix eigenvalue method, since `mat_eig_power` cannot
find complex-conjugate pairs -- QR-algorithm-based eigenvalue extraction
would be the natural next step in vani-matrix first), a real Ferrari's-
method closed form for the quartic (only worth the risk if someone actually
needs the extra precision/speed over the numeric path), and Broyden's
method or a line-search globalization for `algebra_newton_system`/`_fd`
(plain Newton can diverge from a poor initial guess).
