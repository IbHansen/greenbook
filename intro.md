# GREU in ModelFlow

GreenREFORM-EU 2.0 is DREAM's computable general equilibrium model for climate and energy policy, written in
GAMS (<https://github.com/DREAM-DK/GREU>). This book shows its Danish benchmark translated to ModelFlow. The
translation is called GREU (ModelFlow translation): DREAM asks that changed versions are not called
GreenREFORM-EU.

:::{note}
**Example calculations based on example data.** The results in this book show what the model and ModelFlow can
do. They are not forecasts or policy advice.
:::

## The model in numbers

| | |
| --- | --- |
| Generic GAMS equations in `main` | 320, in 23 modules |
| Years | 2020-2050, calibration year 2020 |
| Industries | 57 |
| ModelFlow equations per year | 72 591 without the energy technology block, 73 186 with it |
| Stacked system 2020-2050 | 1.54 million unknowns, 5.3 million non-zeros in the Jacobian |
| Leads and lags | one year forward and one year back |

## What is in the book

**The model in ModelFlow**

- [Onboarding](onboarding.md): how the GAMS model became a ModelFlow model, and how the translation is checked
  against DREAM's solutions.
- [Solving](solving.md): the stacked Newton, the linear solvers, and what it took to reproduce DREAM's CO2 tax
  shocks.
- [An FTT technology block](ftt_model.md): how DREAM models technology shares, why that block is hard for Newton,
  and an alternative block of our own where the shares move gradually.

**The CO2 tax**

- [Dose response](greu_co2tax_dose.ipynb): DREAM's CO2 tax shock in four sizes, with and without the energy
  technology block.
- [Energy flows](greu_co2tax_energy_flows.ipynb): energy and fossil CO2 from energy good to purpose to user, in
  the baseline and under the tax.
- [A tax plus a model switch](greu_s2_switch.ipynb): DREAM's CO2 tax shock without the technology block also
  switches an equation branch; the notebook separates the two.

**Technology dynamics with FTT**

- [The FTT block alone](greu_ftt_block.ipynb): speed, spread, seed and a temporary tax, run live in the notebook.
- [The CO2 tax with the FTT block](greu_ftt_co2tax.ipynb): the whole model with DREAM's block, with the FTT
  block, and with a tax that ends in 2034.

In the notebook chapters the code is folded. Click a cell's bar to see it.

## Status

| | Result |
| --- | --- |
| Equations on DREAM's baseline | all hold, 2020-2050, largest error 4e-7 |
| DREAM's CO2 tax shock without the technology block (S2) | reproduced, largest relative difference 1e-5 |
| DREAM's CO2 tax shock with the technology block (S3) | reproduced from DREAM's staged start, largest relative difference 6e-7 |
| DREAM's energy tax shock (S1) | a zero shock in DREAM's own output: the doubled tax is zero everywhere |
| FTT block | holds on DREAM's baseline; the CO2 tax solves from the baseline in 6 iterations |

The notebooks were run on a desktop computer. The book shows the stored results; the pages don't run anything.
