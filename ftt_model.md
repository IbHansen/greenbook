# An FTT technology block

The energy technology block chooses between technologies for an energy service. This chapter describes DREAM's
block, why it is hard for Newton, and a block of our own built on FTT (Future Technology Transformations), where
the shares move gradually. The FTT block is an experiment. It is not DREAM's model.

Indices: technology $l$, energy service $es$, demander $d$ (an industry), year $t$.

## DREAM's block: a smooth merit order

| Equation | Text | Role |
| --- | --- | --- |
| Marginal capital input | `pESmarg = sum(e, uTE*pEpj_marg) + uTKmargNoBound*pTK` | the capital input per PJ a unit of technology $l$ can carry and still break even at the marginal price |
| Floor | `uTKmarg = max(0.0001, uTKmargNoBound)` | keeps the argument of the log positive |
| Share | `sqT = sqTPotential * Φ((log(uTKmarg/uTKexp) + eP²/2)/eP)` | the share; Φ is the standard normal distribution function |
| Clearing | `1 = sum(l, sqT)` | the shares sum to one; determines the marginal price `pESmarg` |

- Within a technology the capital input per PJ is log-normally distributed with mean `uTKexp` and spread `eP`.
  The units with capital cost below `uTKmarg` are in use.
- `sqTPotential` is the largest share a technology can take.
- The shares are a function of the prices of the year. When prices change, the shares move at once.

## The technologies in the data

The run output has 22 technologies in two units: `in_ETS` in industry 23001 and `heating` in industry 10030, 11
in each. The model has no names for them, so they are identified by their energy input. All have a life span of
5 years and an energy input of 1.1 PJ per PJ, and the potentials are 0.125 or 0.100. This looks like test data.

| Unit | Technologies |
| --- | --- |
| in_ETS, industry 23001 | diesel, gasoline, natural gas (2), electricity, district heat, waste, coal and coke (3), coal with CCS |
| heating, industry 10030 | diesel, gasoline, natural gas (2), electricity (3), district heat, waste, coal and coke, coal with CCS |

## Why the block is hard for Newton

The marginal price is found from the sum of shares. Its Newton step is the miss in the sum divided by the slope
of the sum in the price.

| Issue | What the data show | Consequence |
| --- | --- | --- |
| Most technologies are saturated | in heating 7 of 11 are at full potential and one is at zero | their share equations say nothing about the price |
| The price hangs on a few technologies | in in_ETS, 6.9 of a slope of 7.9 comes from coal with CCS alone | a small change in one technology changes the step a lot |
| The marginal technology changes with the shock | under the tax coal with CCS reaches its potential and its slope falls from 6.9 to 0.001; two coal technologies take over | the Jacobian of the starting point describes the wrong technologies |
| The floor is active | three technologies in in_ETS sit on the floor in the baseline, and gasoline crosses it under the tax | a kink between the start and the solution |
| The log has a pole at zero | `uTKmarg` goes down to 0.0001 | a step that overshoots lands where the slope is thousands of times larger |
| Very different spreads | `eP` from 0.05 to 7.1 | near-steps and nearly flat shares in the same equation |

## The FTT block

In FTT the share is a stock. Each year share flows between every pair of technologies, towards the cheaper one,
at a rate limited by how fast capacity is replaced.

FTT was developed by Mercure for the power sector [@mercure_ftt_2012; @mercure_dynamics_2014]. FTT-FLEX
[@chewpreecha_ftt-flex_2024] is a simplified version for countries with little data, which can be linked to a
macroeconomic model; its reference list has more of the FTT literature.

$$\begin{aligned}
sqT_{l,t} = sqT_{l,t-1} &+ \sum_{ll} (sqT_{l,t-1} + seed_l)\, sqT_{ll,t-1}\, \frac{F_{l,ll}\, room_l}{LifeSpan_{ll}} \\
 &- \sum_{ll} (sqT_{ll,t-1} + seed_{ll})\, sqT_{l,t-1}\, \frac{F_{ll,l}\, room_{ll}}{LifeSpan_l} \\
 &+ \text{calibrated baseline flows}
\end{aligned}$$

| Part | Definition | Meaning |
| --- | --- | --- |
| Cost `CT` | `sum(e, uTE*pEpj_marg) + uTKexp*pTK + JCT` | energy and levelised capital per PJ |
| Spread `SIGMAT` | `eP*uTKexp*pTK` | spread of the cost |
| Preference $F_{l,ll}$ | logistic in `(CT_ll - CT_l)/sqrt(SIGMAT_l² + SIGMAT_ll²)` | 0.5 when the costs are equal, towards 1 when $l$ is cheaper |
| Room | `1 - sqT(-1)/sqTPotential` | stops a technology at its potential |
| Seed `SQTSEED` | exogenous variable, 0.001 | lets a technology with share 0 enter |
| Capital use | `qESK = qES*JQESK*sum(l, sqT*uTKexp)` | the average capital input of the technologies in use |

The flow from $ll$ to $l$ is the same term in both share equations, so the shares keep their sum without a
clearing price. The marginal price, the floor and the log are gone.

### Calibration

The block is calibrated so that every equation holds on DREAM's baseline.

- `JCT` makes the costs of the technologies of a unit equal in the baseline, so all preferences are 0.5. A
  technology with a baseline share below its seed keeps its own cost: it is not competitive in the baseline.
- What is left of the share equation becomes calibrated flows. An outflow is a rate times the share
  (`JSQTOUT`), so it can not make a share negative. The pooled outflow goes to the technologies that gain in the
  baseline (`JSQTIN`).
- `JQESK` is DREAM's capital use relative to the plain average.

### The two blocks side by side

| | DREAM's block | FTT block |
| --- | --- | --- |
| The share is | a function of this year's prices | last year's share plus net flows |
| Sum to one | a clearing price is solved for | automatic |
| Speed of change | none: the share jumps | limited by the life span and by the shares |
| A new technology | enters as soon as the price reaches it | needs a seed |
| Time | static within the year | one lag |
| The CO2 tax from the baseline | Newton diverges; solved from a staged start | Newton converges in 6 iterations |

### Known weaknesses

1. A share can end slightly above its potential (largest case 0.17 percentage points). The calibrated inflow is
   not limited by the room.
2. Every technology gives share to the one that gains, also technologies the tax does not hit.
3. With equal costs in the baseline every cost change counts. In DREAM's block the saturated technologies are far
   from the margin and do not move.
4. The spread `SIGMAT` is our own mapping from `eP`.
5. The seed sets the start of the S-curve and has to be chosen.

The two notebook chapters show the dynamics: [the block alone](greu_ftt_block.ipynb) and
[the CO2 tax in the whole model](greu_ftt_co2tax.ipynb). The tool is `tools\ftt_block.py`.
