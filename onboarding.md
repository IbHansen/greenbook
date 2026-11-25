# Onboarding: from GAMS to ModelFlow

The input is DREAM's run output: for each run the model text after gamY has expanded it, and a GDX file with every
set, parameter, level and bound. There are five runs: the baseline without the energy technology block
(`main_CGE`), the baseline with it (`main_energy_technology`), and three shocks.

## The pipeline

```
python tools\dict_scan.py                  # the scalar model GAMS wrote, for checking
python tools\run_translate.py <variant>    # model text + GDX -> one translation per equation instance
python tools\write_model.py <variant>      # -> model\greu_<variant>.frm
python tools\build_data.py <variant>       # model + GDX -> the baseline DataFrame
python tools\check_baseline.py <variant>   # every equation evaluated once on DREAM's solution
```

A variant is one GDX file. A shock is translated as a variant of its own, because its GDX carries its own first
year (2021 for the CO2 tax) and its own closure.

## One equation per variable instance

The MAKRO translation wrote each GAMS equation once, as a line over lists. That does not work here. GREU's first
solve year, 2020, is also the calibration year, and many conditions change by year: first year, last year, data
years, and the life-span windows of the energy technology block.

So every equation is translated per instance: all set indices are fixed, conditions are evaluated, and sums are
written out term by term. The result is one equation per endogenous variable instance, for example
`PY_I_D__10010__C = ...`.

- An equation whose conditions do not depend on the year is written once.
- The others are written for every year. Texts that differ are merged with year dummies:
  `V = RECODE(YD_2050, text_2050, text_other_years)`.
- A variable that has no equation in some years gets `V = RECODE(V_D, V_X, ...)`, with `V_D` = 1 in those years
  and `V_X` its level.

## Which variable an equation determines

gamY writes the *exists dummy* of the endogenous variable into the condition of its equation. The equation

```
E_pW[t]$(... and pw_exists_dummy[t])..  qL[t] =E= qProductivity[t]*nL[t]
```

is the equation of the wage `pW`, which does not occur in it: it is the labour market clearing. The translator
pairs each equation with its variable from that dummy.

- 67 357 equations are normalized: the variable occurs once and can be isolated.
- 5 234 are written as residuals, `V___RES = right side - left side`.

## The closure

Some variables that an equation is paired with are fixed in the run, and another variable is free instead. For
energy goods the price `pY_CET` is fixed and the markup is free. In the shocks public consumption is fixed and
its share of GDP is free, and the lump sum transfer balances the public budget.

Free means lower bound below upper bound in the GDX. Each equation whose paired variable is fixed gets a new
variable by a search for the shortest alternating path through the free variables. In `main_CGE` this re-pairs
992 equation instances, and in the CO2 tax shock it finds GAMS's own closure.

## Outside the exists dummy is zero

In GAMS the model statement limits each variable to its exists dummy. An instance outside it is zero in the model,
whatever level the GDX holds. The translator drops such references, and the data builder sets those levels to 0
(4 223 values). Before this rule 53 equations failed on the baseline. After it all hold.

## Data

Every ModelFlow name `NAME__E1__E2` maps to `name[e1,e2,t]` in the GDX. The DataFrame has the years 2015-2050 as
rows, plus one row after 2050 for the leads of the last year. Parameters are columns, like variables. The levels
are DREAM's, adjusted for growth and inflation as in GAMS.

## Checks

| Check | Result |
| --- | --- |
| Equation instances against GAMS's scalar model | equal for every equation and year |
| Endogenous variables against the GDX bounds | exact: as many as equations, nothing fixed, nothing free left out |
| Every equation on DREAM's baseline, 2020-2050 | all 72 591 hold, largest error 4e-7 |
| The same with the energy technology block | all 73 186 hold |
| Every equation on DREAM's shock solutions | all hold |
| Exogenous variables times 1.01 | 30 214 equations fail, so the check is not empty |

One thing was not as expected: the scalar model DREAM sent is the calibration model, not the simulation model.
The simulation equations are a subset of it with the same conditions, so the comparison still covers 319 of the
320 equations.

## Three rewrites for Newton

The translated text is changed in three places. The solutions are the same; all variants still hold on DREAM's
solutions afterwards.

| Rewrite | What | Why |
| --- | --- | --- |
| Normal distribution arguments | the argument of each `NORMCDF` becomes a variable of its own (44 new variables) | the argument is differentiated as its own equation |
| The floor on the marginal capital input | `uTKmarg` is replaced by its definition `MAX(0.0001, uTKmargNoBound)` where it is used | as an unknown, `uTKmarg` could be moved below 0 and the log in the share equation crossed its pole |
| Values in place of prices | where a free price enters only as price times quantity, the equation determines the value, and the price follows (477 equations) | when the CO2 tax turns natural gas imports into exports, the quantity passes 0 and the price is undetermined there |
