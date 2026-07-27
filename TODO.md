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

## v0.1.4 (2026-07-27)

- [x] `algebra_newton_system_damped` -- line-search-globalized Newton's
      method for `F(x) = 0`: the same full Newton step as
      `algebra_newton_system`, backtracked via an Armijo condition on the
      merit function `0.5*||F(x)||^2` (same technique as
      `vani-optimize`'s `armijo_line_search`) instead of always taking the
      full step. The Newton direction is provably always a descent
      direction for this merit (`d/dalpha` at `alpha=0` works out to
      exactly `-||F(x)||^2`, using `J(x).d = -F(x)` from the Newton step
      itself), so the same backtracking machinery applies directly.
      Falls back to the smallest step tried if backtracking exhausts
      `max_ls_iter` without satisfying Armijo, same convention as
      `armijo_line_search` itself. `#[bounded_stack(bytes = 906)]`,
      `vanic check`'s exact reported worst-case.
- [x] `tests/test_newton_system.vani` extended: converges to the same
      known root from a starting guess (8, 1) far enough that a full
      undamped step would badly overshoot, with a residual check (not
      just the expected value). Note: `x0` with `x = -y` exactly makes
      this system's Jacobian singular (`det J = -2(x+y)`) regardless of
      solver -- confirmed via scratch testing that even plain
      `algebra_newton_system` hangs from such a start (`mat_solve` on an
      exactly-singular matrix), so this is avoided in the test rather
      than presented as something the damped solver specifically fixes.

**Found but NOT fixed**: `algebra_newton_system_fd` fails to compile on
the `--backend=c` path -- `cc` reports `'v_xp' undeclared` inside the
emitted C for its Jacobian finite-difference loop (a `let xp: Vec<f64>`
declared inside a nested `while`, same shape as `vani-optimize`'s
`grad_fd`/`hessian_fd`, which compile fine on both backends). Confirmed
pre-existing via `git stash` against the unmodified file -- unrelated to
this version's changes, and blocks C-backend compilation of every test
file in this package (the C backend compiles every function in the
source file, not just the ones a given test calls). This is a
vani-compiler backend bug, not a vani-algebra code issue; needs
investigation in `backend_c.rs`, out of scope for this package's own
TODO. All testing in this version was done on the LLVM backend only.

## Future

No v0.2.0 is currently planned. Candidates if a concrete need shows up:
complex root support (would need a vani-complex dependency and a genuinely
different companion-matrix eigenvalue method, since `mat_eig_power` cannot
find complex-conjugate pairs -- QR-algorithm-based eigenvalue extraction
would be the natural next step in vani-matrix first), and a real Ferrari's-
method closed form for the quartic (only worth the risk if someone actually
needs the extra precision/speed over the numeric path).
