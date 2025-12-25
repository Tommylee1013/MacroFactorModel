## Macro Factor Model Research

Counterfactual Macro–Financial Simulation Framework

This repository implements a structural causal macro–financial model for simulating stock index dynamics under explicit counterfactual interventions.  The framework is designed for policy analysis, liquidity stress testing, and regime-based
scenario simulation. It is not a reduced-form forecasting model. All components are
organized as a Structural Causal Model (SCM) to guarantee valid do-operator–based
interventions.

The model explicitly separates policy expectations, realized policy rates, short-end
funding/liquidity premia, financial transmission channels, and stock price formation.

### 1. Core Principles

#### (1) Causality before correlation

All variables are arranged in a Directed Acyclic Graph (DAG). Stock indices are terminal nodes and are never conditioned upon during estimation or simulation.

#### (2) Conditioning is not intervention

The framework distinguishes strictly between:

$$
P(Y \mid X = x)
$$

and

$$
P(Y \mid \text{do}(X = x))
$$

Only the latter is considered a valid counterfactual experiment.

#### (3) Structure is deterministic, uncertainty is exogenous
Causal propagation is governed by structural equations. Randomness enters only
through exogenous shocks generated separately (e.g. via QuantGAN).

### 2. Variable Definitions

- $R^e$ : Expected policy rate extracted from 30-day Fed Funds Futures
- $\text{EFFR}$ : Effective Federal Funds Rate (realized policy rate)
- $R_{\text{3M}}$ : 3-month interest rate
- $\text{TP}$ : Short-end liquidity / funding premium - $\text{TP}_t = \text{R3M}_t - \text{EFFR}_t$
- $\text{OIL}$ : Oil price (energy-driven inflation pressure)
- $\text{MBS}$ : Mortgage-backed securities yield (housing and credit inflation proxy)
- $\text{CRB}$ : Commodity Index (financialized risk asset with real shock component)
- $\text{BONDS}$ : Bond Index (duration and discount-rate channel)
- $\text{FX}$ : Dollar Index (NYB)
- $\text{STOCK}$ : Stock Index return

### 3. Structural Causal Graph (SCM)

The causal ordering is fixed as:

$$
\begin{tikzpicture}[node distance=2cm, auto]

\node (OIL) {\text{OIL}};
\node (MBS) [below of=OIL] {\text{MBS}};
\node (Re) [right of=OIL, xshift=2cm] {\text{$R^e$}};

\node (EFFR) [right of=Re, xshift=2cm] {\text{EFFR}};
\node (TP) [below of=EFFR] {\text{TP}};
\node (R3M) [right of=EFFR, xshift=2cm] {\text{$R_{3M}$}};

\node (BONDS) [right of=R3M, yshift=1.5cm] {\text{BONDS}};
\node (FX) [right of=R3M] {\text{FX}};
\node (CRB) [right of=R3M, yshift=-1.5cm] {\text{CRB}};

\node (STOCK) [right of=FX, xshift=2cm] {\text{STOCK}};

\draw[->] (OIL) -- (Re);
\draw[->] (MBS) -- (Re);

\draw[->] (Re) -- (EFFR);
\draw[->] (EFFR) -- (R3M);
\draw[->] (TP) -- (R3M);

\draw[->] (R3M) -- (BONDS);
\draw[->] (R3M) -- (FX);
\draw[->] (R3M) -- (CRB);

\draw[->] (BONDS) -- (STOCK);
\draw[->] (CRB) -- (STOCK);

\end{tikzpicture}
$$


No causal arrows originate from STOCK.

### 4. Structural Equations

All variables are modeled in daily changes or returns.

Policy expectation:

$$ \Delta R^{e}_t  = a_{re} + \phi_{R^{e}}\Delta R^{e}_{t-1}+ \beta_o \Delta \text{OIL}_t+ \beta_m \Delta \text{MBS}_t+ u^{R^{e}}_t$$

Realized policy rate:

$$\Delta \text{EFFR}_t = a_{e} + \phi_{e}\Delta \text{EFFR}_{t-1}+ u^{e}_t$$

Short-end funding premium:

$$\Delta \text{TP}_t = a_{\text{TP}} + \phi_{\text{TP}}\Delta \text{TP}_{t-1}+ u^{\text{TP}}_t$$

Three-month rate:

$$\Delta R_{\text{3M}t} = \Delta \text{EFFR}_t + \Delta \text{TP}_t$$

Bond index:

$$r^{\text{bond}}_t = a_b + \phi_b r^{\text{bond}}_{t-1}+ \eta_b \Delta R_{\text{3M}t}+ u^{b}_t$$

Commodity index:

$$r^{\text{CRB}}_t = a_c + \phi_c r^{\text{CRB}}_{t-1}+ \eta_c \Delta R_{\text{3M}t}+ u^{c}_t$$

Exchange rate:

$$\Delta \text{FX}_t = a_{\text{FX}} + \phi_{\text{FX}}\Delta \text{FX}_{t-1}+ \eta_{\text{FX}} \Delta R_{\text{3M}t}+ u^{\text{FX}}_t$$

Stock index (terminal equation):

$$r^{\text{stock}}_t = a_s + \phi_s r^{\text{stock}}_{t-1}+ \lambda_b r^{\text{bond}}_t+ \lambda_c r^{\text{CRB}}_t+ u^{s}_t$$

### 5. Collider and Confounder Handling

STOCK is a collider:

$$\text{CRB}_t \rightarrow \text{STOCK}_t \leftarrow \text{BONDS}_t$$

The model never conditions on STOCK during estimation or shock generation.

R3M is a common cause of CRB and BONDS. For simulation purposes, R3M is handled
upstream via structural equations rather than regression conditioning.


### 6. Shock Generation

All stochastic components are exogenous shocks:

$$
u^{R^e}_t,\;
u^{e}_t,\;
u^{\text{TP}}_t,\;
u^{b}_t,\;
u^{c}_t,\;
u^{\text{FX}}_t,\;
u^{s}_t
$$

These shocks may be generated using Conditional QuantGAN or other heavy-tailed
generators. The GAN never generates observed variables directly.


### 7. Counterfactual Simulation Procedure

Counterfactual simulation follows the standard three-step SCM procedure.

Step 1: Abduction
Estimate and fix exogenous shocks from observed data.

Step 2: Action
Apply interventions using do-operators, for example:

$$
\text{do}(\text{EFFR}_t = x_t)
$$

or

$$
\text{do}(\text{TP}_t = y_t)
$$

Structural equations for intervened variables are removed.

Step 3: Prediction
Propagate the same exogenous shocks through the modified system to obtain
counterfactual paths for all downstream variables, including STOCK.

### 8. Supported Interventions

Policy intervention:
$$
\text{do}(\text{EFFR})
$$
Represents central bank policy actions.

Liquidity stress intervention:
$$
\text{do}(\text{TP})
$$
Represents short-end funding and money market stress.

Expectation management:
$$
\text{do}(R^e)
$$
Represents forward guidance or expectation shocks.


### 9. Intended Use Cases

- Policy counterfactual analysis
- Liquidity stress testing
- Regime-dependent stock index simulation
- Synthetic data generation for backtesting under alternative macro scenarios
