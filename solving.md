# Solving

GREU looks one year forward, so the whole horizon is solved as one system: ModelFlow's stacked Newton
(`newtonstack_ng`). The derivatives are symbolic. The equations before and after the simultaneous core are just
evaluated; Newton runs on the core.

| | |
| --- | --- |
| Core variables per year | 49 800 |
| Stacked unknowns 2020-2050 | 1.54 million |
| Non-zeros in the Jacobian | 5.3 million |
| Finding the derivative expressions | about 4 minutes, once per model |
| One iteration | about 30 seconds, mostly the factorization |

## Symbolic derivatives

ModelFlow finds the derivatives with sympy. A function sympy does not know is differentiated numerically with a
fixed step, and that is not precise enough here. `RECODE`, `MAX` and the normal distribution function got symbolic
derivatives in ModelFlow for this model, so all 125 601 derivative expressions are symbolic.

## The linear solver

| Run | Linear solver | Result |
| --- | --- | --- |
| Zero shock 2020-2050 | PARDISO on the normal equations | the baseline back in 0 iterations, 432 s, 16 GB |
| CO2 tax, Jacobian kept from the start | PARDISO on the normal equations | the linear solve is not accurate enough, fails |
| CO2 tax, Jacobian kept from the start | MUMPS | diverges, by a factor 64 per iteration |
| CO2 tax, new Jacobian every iteration | MUMPS | converges in 6 iterations, 454 s |

MUMPS with a new Jacobian in every iteration (`nonlin=1`) is the setting used for all the shocks.

**Why the kept Jacobian diverged.** The closure makes the import price of some energy goods the free variable of
a value balance, where its coefficient is the import quantity. For natural gas that quantity is 0.019 in the
baseline and -1.22 under the tax: imports turn into exports. With the Jacobian of the starting point every
iteration multiplies the error by 1 - (-1.22/0.019), about 64. A shock where a quantity changes sign needs the
Jacobian renewed.

## DREAM's CO2 tax without the technology block (S2)

CO2 tax +750, phased in 2025-2030. From the baseline the solve converges in 6 iterations, and all 67 354
endogenous variables equal DREAM's solution: the largest relative difference is 1e-5 and the median 2e-15.

## DREAM's CO2 tax with the technology block (S3)

From the baseline this does not converge, although the Jacobian is right: at DREAM's solution the largest residual
is 5e-10, and a check of the Jacobian against differences agrees in all 2.2 million rows. The divergence sits in
the technology block. Its share equation is almost a step function of the marginal price, and the tax moves some
shares by more than a factor 10. The chapter [An FTT technology block](ftt_model.md) looks at that block.

DREAM does not solve it in one go either. Its script solves in stages, and ModelFlow does the same:

| Stage | DREAM | ModelFlow |
| --- | --- | --- |
| Energy prices | a partial model for energy prices and CO2 taxes per PJ | the 26 price families as a submodel (22 587 equations), iterated until settled |
| Technology shares | a grid of 100 candidate marginal prices; the first where the shares sum to 1 | the technology block as a submodel (158 equations); the marginal price by bisection on the sum of shares |
| Whole model | `solve main` | stacked Newton from that point |

How close the start is to DREAM's solution (largest relative distance):

| | Baseline plus shock | Staged start |
| --- | --- | --- |
| Technology shares | 12.2 | 0.008 |
| Marginal capital input | 13.3 | 0.02 |
| Marginal price of the energy service | 0.10 | 0.0004 |

From the staged start the full Newton converges in 5 iterations and 312 seconds. The largest residual goes 80,
6.3, 0.76, 1.5e-4, 6.6e-7, 1.0e-8. All 67 967 endogenous variables equal DREAM's solution: the largest relative
difference is 5.7e-7.

## Two things found in DREAM's shocks

- **The energy tax shock (S1) is a zero shock.** The script doubles `tE_duty`, which is zero everywhere; the
  equations use `tEmarg_duty`. DREAM's own output for the shock equals the baseline.
- **The CO2 tax without the technology block also switches a model branch.** The script changes a switch that
  moves energy demand by purpose from fixed coefficients to a price-elastic form, calibrated for the other one.
  The switch alone gives about -1.6 % GDP. The chapter
  [A tax plus a model switch](greu_s2_switch.ipynb) separates the two.

## Own shocks

DREAM's CO2 tax in four sizes (+250, 500, 750, 1000) converges in 5 iterations for every size, 2 to 2.5 minutes
per size once the derivatives are made. The results are in [Dose response](greu_co2tax_dose.ipynb).
