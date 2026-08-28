# Data-Driven Wasserstein DRO for Inventory Optimization

A bounded-support single-item inventory project implementing an exact finite solution of a one-dimensional **1-Wasserstein distributionally robust** newsvendor model.

The project separates:

```text
historical demand
   ↓
data-driven Wasserstein radius
   ↓
ambiguity set around empirical distribution
   ↓
exact bounded-support W1 dual
   ↓
integer order decision
   ↓
nominal + shifted out-of-sample evaluation
```

## Inventory loss

The loss contains:

- procurement cost;
- linear holding cost;
- linear shortage cost;
- convex quadratic shortage escalation.

The quadratic term represents increasing expediting/lost-sales disruption for severe shortages and makes the ambiguity premium decision-dependent.

A useful development finding was that with a purely linear standard newsvendor loss, the 1-Wasserstein robustification on this fixture could add essentially a decision-independent Lipschitz premium and return the same order as SAA. Rather than hide that structural result, the final declared model uses bounded-support convex shortage escalation.

## Wasserstein radius

The code estimates a data-driven radius by repeatedly splitting the historical sample into two equal subsets and computing the exact empirical one-dimensional W1 distance:

```text
W1(F_a,F_b) = mean_i |sort(a)_i - sort(b)_i|
```

The selected upper quantile is a bootstrap-style calibration heuristic. It is **not** presented as a finite-sample ambiguity-set coverage theorem.

## Exact fixed-order W1 dual

For fixed integer order `q`:

```text
sup_{P: W1(P,P_emp) <= epsilon} E_P[loss(q,D)]
```

is solved through the Kantorovich dual:

```text
min_{lambda >= 0}
lambda * epsilon
+
mean_i sup_{z in [0,U]}
    [loss(q,z) - lambda |z-d_i|]
```

For the declared bounded-support convex piecewise-quadratic loss, on each region split by `{0, q, d_i, U}` the inner function is convex in `z`; hence its maximum is attained at a region endpoint.

The outer objective is a maximum of affine functions of `lambda` plus a linear term, hence convex piecewise-linear. The implementation enumerates all pairwise lambda breakpoints plus boundary values.

Finally every integer `q = 0,...,Qmax` is enumerated. Thus the returned DRO order is exact for the declared finite bounded-support model.

## Independent numerical oracle

A regression test compares the finite-breakpoint inner dual against a dense two-dimensional numerical check:

- 20,001 lambda values;
- 5,001 demand-support points per empirical sample.

The exact breakpoint result must agree to numerical tolerance.

## Development run

```text
historical samples      100
nominal test samples   2000
shifted test samples   2000
demand shift             0.8
calibrated W1 radius    6.4943
```

Result:

```text
method             order   nominal cost   shifted cost   shifted stockout
SAA                   48       204.46         248.79          62.40%
Box robust            84       292.84         286.83           6.05%
Wasserstein DRO       52       205.40         239.48          52.10%
```

The Wasserstein-DRO order was modestly more conservative than SAA, paying about one unit more nominal expected cost while reducing shifted expected cost in this fixture. The box-robust policy was much more conservative.

No universal DRO dominance claim is made.

## Tests

- exact empirical W1 hand check;
- positive data-driven radius;
- epsilon-zero W1 objective equals empirical SAA cost at fixed order;
- exact inner dual vs dense lambda/support grid;
- outer integer DRO enumeration;
- valid SAA/box/DRO orders.

## Run

```bash
pip install -r requirements.txt
python run_wasserstein_dro_inventory.py --self-test
python -m unittest discover -s tests -v
python run_wasserstein_dro_inventory.py
```

## Scope

Not claimed:

- radius calibration has formal ambiguity-set coverage;
- Wasserstein DRO always beats SAA under shift;
- one-product bounded-support results transfer directly to multi-echelon systems;
- box robust is an optimized state-of-the-art robust benchmark.
