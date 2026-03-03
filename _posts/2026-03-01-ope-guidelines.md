---
layout: post 
title:  "Off-policy evaluation: lessons learned from the field of HVAC control"
date: 2026-03-01 00:00:00 
categories: "ai hvac control"
comments: true
use_math: true
author: Antoine Galataud, Davide Coraci
---
{% if page.noindex == true %}
  <meta name="robots" content="noindex">
{% endif %}

<style type="text/css">
.center {
    display:block;
    margin: 0 auto;
    margin-top: 1.5rem;
    margin-bottom: 3rem;
}
.center-m {
    display:block;
    margin: 0 auto;
    width: 75%;
    margin-top: 1.5rem;
    margin-bottom: 3rem;
}
.center-sm {
    display:block;
    margin: 0 auto;
    width: 50%;
    margin-top: 1.5rem;
    margin-bottom: 1.5rem;
}
.double-left {
    width:49%;
    display:block;
    float:left;
    margin: 0 auto;
    margin-right: 2rem;
}
.double-left-sm {
    width:35%;
    display:block;
    float:left;
    margin: 0 auto;
    margin-right: 2rem;
}
.double-right {
    width:49%;
    display:block;
    float:right;
    margin: 0 auto;
    margin-right: 2rem;
    margin-left: 2rem;
}
.double-right-sm {
    width:35%;
    display:block;
    float:right;
    margin: 0 auto;
    margin-right: 2rem;
    margin-left: 2rem;
}
.add-margin-right {
    margin-right: 20px;
}
.double {
    width: 49%;
    margin: 0 auto;
}
.double-unconst {
    width: auto;
    margin: 0 auto;
    margin-left: 1.5rem;
}
.image-foot {
    font-size:10pt;
    max-width: 30rem;
    margin: 0 auto;
    margin-bottom: 2rem;
}
.small-img {
  width: 45%;
}
.small-txt {
  font-size: 0.8rem;
}
ul {
  display: table;
}
pre {
  white-space: pre-wrap;
  overflow-x: auto;
  margin: 1rem 0pt 1rem 0pt;
}
code {
  font-size: 0.9rem;
  margin: 1rem 0pt 1rem 0pt;
}
h1, h2, h3, h4 {
  margin: 1rem 0pt 1rem 0pt;
}
table {
    width:80%;
    margin: auto;
    border-collapse: collapse;
    text-align: center;
}
thead {
    font-weight: bold;
    background-color: #f2f2f2;
}
th, td {
    border:1px solid #ccc;
    padding:0.5rem;
}
tbody tr:nth-child(even) {
    background-color: #fafafa;
}
tbody tr:nth-child(odd) {
    background-color: #ffffff;
}
</style>

## Introduction

**Off-Policy Policy Evaluation (OPE)** is a technique for estimating the performance of a new reinforcement learning policy ($\pi_e$) using historical data collected from a different behavior policy ($\pi_b$). 
This approach enables policy comparison and selection without deploying new policies into production, reducing risk and cost.

OPE relies on the mathematical framework of **importance sampling** and **model-based estimation** to provide unbiased or low-bias estimates of policy performance. 

Under importance sampling, the value of the target policy $\pi_e$ can be estimated using trajectories collected under the behavior policy $\pi_b$ as follows:

<center>
$V_{\pi_e} = E_{\tau \sim \pi_e}[\sum_{t=0}^T r_t] 
 = E_{\tau \sim \pi_b}[\sum_{t=0}^T \frac{\pi_e(a_t|s_t)}{\pi_b(a_t|s_t)} r_t] 
 \approx \frac{1}{N} \sum_{i=1}^N \sum_{t=0}^T \frac{\pi_e(a_t^i|s_t^i)}{\pi_b(a_t^i|s_t^i)} r_t^i$
</center>
<br>

See [Hopes library documentation](https://hopes.foobot.io/overview/index.html) for more details on the mathematical background and different estimators.

Under model-based estimation, a model of the environment is learned from the data collected under $\pi_b$, 
and then used to simulate trajectories under $\pi_e$ to estimate its performance.

It is particularly valuable when:

- Deploying untested policies could have negative consequences (safety, costs, user experience)
- We want to evaluate multiple candidate policies before committing to one
- We have sufficient logged data from past deployments

This document explores when OPE is applicable, which estimators to use, how to interpret results, and what alternatives exist from our experience in the field of HVAC control.

## When does it make sense?

The applicability of OPE depends on the specific problem, the available data, and the policies being evaluated. Here are some key considerations.

### IPS estimators

If $\pi_b$ is deterministic, estimators of the Inverse Propensity Score (IPS) family will fail since $\pi_b(a|s)$ is 
used as denominator in most definitions. Adding a $\epsilon$ to avoid division by zero introduces bias, 
and dividing by a small number will make variance blow.

⇒ No IPS if $\pi_b$ is deterministic.

### Direct method

Direct Method estimator uses a learned model of cumulative rewards to estimate policy value. At its core, we use 
$V^{DM}(s_0) = \sum_{a} \pi_e(a|s_0)·Q(s_0,a)$ for a given episode to estimate future policy return. 
Since $Q$ is learned from data logged under $\pi_b$, the following assumptions hold:

- if $\pi_b$ and $\pi_e$ have very different responses (actions distribution), it’s very likely that $Q$ will be biased.
- if $\pi_b$ never explores region where $\pi_e$ operates, it’ll be biased too.
- if episodes horizon is long, there is more chance for error to happen and compound.

A possible way to decrease bias for the learned model is to split long episodes into smaller, regular ones, and learn a separate model for each time bucket. While not solving the coverage problem, it can decrease errors due to temporal extrapolation, and makes changes in action distribution more explicit. 

Another way to improve results, and this time coverage, is to form a dataset under multiple offline / online policies. For instance, using evaluation data generated during training (when model-based, the bias can then come from system dynamics that don’t match real world system), or by different versions of the behaviour policy.

## What estimators can be used, and prerequisites?

### IPS estimators

Most IPS estimators use $\frac {\pi_e(a|s)}{\pi_b(a|s)}·r$ as a manner of re-weighting rewards obtained under $\pi_b$. 
This assumes we need to log following information under $\pi_b$:

- states
- actions
- actions probabilities (propensity scores) → they can’t be reconstructed from logged actions alone by rolling out the policy.
- rewards

In practice, this information should be logged by the policy server (preferred), unless all these information is returned by the policy server and in this case the client side could log it.

We also need $\pi_e$ to be available so we can evaluate action probabilities under this policy. This should be in the shape of the policy model itself, or a probability function that expresses the policy correctly.

Since $\pi_b$ is the denominator, we also need support (coverage), expressed as $\pi_e(a|s) > 0 \implies \pi_b(a|s) > 0$. 
Which practically means: 

- all state - action pairs visited under $\pi_e$ should have been visited under $\pi_b$.
- for all state - action pairs visited under $\pi_e$, the probability of taking the action should never be 0 under $\pi_b$.
- $\pi_b$ shouldn’t be deterministic.

(see also “When does it make sense”).

### Direct method

DM assumes learning a model that can predict cumulative return from an initial state, which can perform well across both policies. This assumption usually doesn’t hold, especially if $\pi_b$ and $\pi_e$ have different origins (e.g. very different modeling approaches), or when a new policy is supposed to perform better by taking trajectories unseen under old policy. DM should then be used with caution, by first performing a thorough analysis of the states and actions distributions under the two policies (see also “When does it make sense”).

Compared to IPS, we don’t need logged action probabilities (propensity scores) under $\pi_b$. The rest holds.

### General requirements

- Positivity of the reward: this is necessary when confidence interval and lower bound estimates  should be computed. This may require to rescale the logged / computed reward, for instance using a *MinMaxScaler* to obtain a $[0, 1]$ range.
- Reward design consistency: make sure reward function doesn’t change over time, which is often the case in early experiments. OPE might more be adequate when training and deployment worfklows are stable.
- Data logging pipeline robustness: as indicated above, there are quite a few things to log, some of them are easier to get than others. When having episodes of length greater than 1, having a field to easily identify them can be interesting (or a *done* column).
- Finite variance assumption: with long horizons, or slight probability mismatch, variance may explode. In this case, prefer per-decision estimators and / or self-normalized ones.
- Stationarity: if environment is unstable (e.g. in HVAC control depends on weather, occupancy, etc) it can be interesting to segment the analysis by control context to better understand where a new policy works better than an other.

## How to make sense of the value?

OPE estimators are often biases or have high variance. Well, nothing new, there’s always a trade-off. Except here estimators can be largely biased, and variance can explode easily. Knowing that, how to interpret the output of our estimators? When to trust them?

### Good practices

At its core, OPE uses policies values as scores. Policies values are nothing but cumulative sum of rewards over a trajectory, obtained with different manners depending on the estimator.

As a consequence, if rewards, and more importantly their cumulative sum, are easily interpretable (e.g. cases where there is a single term, or equal weights for each term, no scale differences, etc), then we can give an interpretation about the scalar value of the OPE output.

Otherwise, the values should be interpreted by comparison, as in $V(\pi_e) >= V(\pi_b)$, which would mean the cumulative sum of rewards under first policy is higher than under second.

However, comparing values can be a weak estimator (single value comparison, suffering from estimators biases and variances). It’s important to rely on:

- confidence intervals (lower and upper bounds) for the estimators.
- multiple estimators outputs.

For instance, checking if $LB(V(\pi_e)) >= LB(V(\pi_b))$ with $LB$ the lower bound of the confidence interval. It offers stronger guarantees about first policy being “better” than second.

Note that, in all cases it’s possible to:

- output policy value estimates for both $\pi_e$ and $\pi_b$. The first is obtained with OPE estimators, and the second is simply the cumulative sum of rewards under the same trajectories. In practice, we can even run OPE estimators with $\pi_e = \pi_b$, some terms will cancel.
- estimate multiple policies values to take a decision, e.g. $\pi_e^i \in P$ where $P$ is an ensemble of available target policies.

### Beyond good practices, understanding data at hand

There are several practical things to consider about the dataset and the problem at hand which may lead to different interpretations of the results:

- Reward representativeness: does the reward truly capture the operational objective? In real-world problems, this isn’t always the case, because of missing information for reward design that is replaced by shaping. Sparse or delayed rewards are particularly difficult to handle, especially for trajectory-wise estimators: for instance Direct Method is only as good as the Q model it relies on, for which a new trajectory might be out of distribution. Consider careful inspection of reward distributions, using multiple estimators, splitting episodes into representative subsets, and introducing time or step information observations.
- Reward scale and variance: high reward variance increases estimator variance. It requires dealing with outliers steps or episodes (pruning), re-scaling rewards, and using robust estimators.
- Action space shape: special care needed when action imbalance exists. This leads to numeric instability of weights under IPS estimators, and estimators using a Q model may never see some actions taken under $\pi_b$.

## Going in production: How to handle overridden policy decisions?

In real-world systems policy decisions can be overridden by safety rules, manual decisions, etc. This comes as a 
challenge for OPE as the logged propensities under $\pi_b$ no longer match the actual action applied, resulting in 
state and reward not being the result of the policy decision.

A concrete example is the HVAC equipment optimal start problem we tackle with RL: once the policy has decided to switch 
on the system, an "action stickiness" mechanism applies to keep the equipment turned on (both during training and real-world inference). 
This comes as a challenge for many estimators, like IPS-style estimators: 

- $\pi_b(a|s)$ can still tell action 0 is more likely, but the state and reward will be the one obtained 
  by taking $a = 1$. 
- The estimator become biased as we will basically reweight the reward of taking $a = 1$ with the probability of not taking it.
- Cumulative sum of rewards must be computed until end of episode as states and rewards after the initial switch time 
  are important: the verification of the achievement is delayed and compounds as policy is penalized at every step for  
  taking an action early and penalized for not achieving comfort at specific time (end of episode). 
  So we can’t truncate episodes until first switch on time.

There are 2 practical solutions that differ in implementation but eventually end up to the same result:

1. Correct propensities by integrating the action stickiness information as an augmented state $s' = (s, z =\text {switch has happened})$. 
   For both $\pi_e$ and $\pi_b$, we should have $\pi(1|z=1) = 1$. This guarantees support and no division by zero.
2. Consider the episodes as 2 sub-episodes, the first part where propensities are applied until first switch time, 
   the second where only reward is kept as the importance ratio cancel with $\frac {\pi_e(1|s) = 1}{\pi_b(1|s) = 1} = 1$

The first option could be cleaner though, and could be used for all kind of estimators. This solution only holds if action stickiness applies to both policies.

## Post-deployment evaluation

As with any continuous deployment technique, continuous monitoring is necessary as a post-deployment evaluation step. This will allow to make sure deployed policy outcome is the desired one. The good thing here is that we won’t rely only on the reward, but on every data available from the target system. 

One potential technique, given enough data, is an evaluation based on control-context equivalence (CCE): gathering periods with “similar” context prior and posterior to new policy deployment will allow to make sure the new policy is actually doing better. 

Below example shows control-context equivalent evaluation on an optimal start problem:

![sources]({{ site.baseurl }}/assets/ope_guidlines/image.png){: .center }

Of course, it’s still possible to rely on OPE, and to compute reward estimates based on logged data after deployment, then compare $\hat{V}(\pi_e)$ with $V(\pi_e)$ with $\hat{V}$ being the value estimated on logged trajectories.

## What alternatives exist?

OPE requires to proceed with care and has many pitfalls. It’s easy to fall into traps or regions where the method can’t cover our needs.

Alternatives include:

- using a model of the environment (probabilistic, physics-based twin, etc) to evaluate both policies. 
  This would offer robustness and coverage (any type of policy can be used), but it’s sometimes not available and 
  it’s harder to put in practice (need to run the model with a simulation engine, hook into it to rollout the policy, etc). 
  It introduces a bias:
  - in assuming model accurately captures transition dynamics of the environment, 
    e.g. $\hat{p}(s'|a,s) \approx p(s'|a,s)$
  - in assuming trajectories can be replayed deterministically for the behavior policy $\pi_b$, which might not always be true (e.g. exploration is enabled under $\pi_b$).
- post-deployment policies comparison with control context equivalence (CCE): if both $\pi_b$ and $\pi_e$ have been deployed, 
  comparing their performance can be done if context equivalence can be ensured. As with previous point, we can not 
  only compare their value estimate, but also check their performance in details (per-goal). It also introduces a bias 
  (full equivalence is impossible), and is only possible in an online manner ($\pi_e$ has to be deployed) which reduces 
  its applicability (we often want to estimate performance whithout deploying).
- online A/B testing: online A/B testing consists in deploying the candidate policy $\pi_e$ to a fraction of traffic 
  while keeping the current policy $\pi_b$ on the remaining traffic, then comparing their outcomes under the same 
  reward definition and operational constraints. Online A/B testing is best used once offline screening (OPE or simulation) 
  suggests a candidate is safe, and when monitoring and rollback mechanisms are in place.

Comparison of the approaches:

| Method | No deployment?                                                    | No model? | Handles long horizon? | Main risk        |
| --- |-------------------------------------------------------------------|-----------|-----------------------|------------------|
| OPE | ✅                                                                 | ✅         | ⚠️                    | bias/variance    |
| Simulator | ✅                                                                 | ❌         | ✅                     | model error      |
| Online A/B | ❌                                                                 | ✅         | ✅                     | operations   |
| CCE | ❌ <br><span style="font-size: 0.8rem">(not simultaneously)</span> | ✅         | ✅                     | context mismatch |

## Conclusion

OPE is a powerful technique for evaluating policies without deployment, but it requires careful consideration of the 
problem, data, and estimators used. By following best practices and understanding the limitations, practitioners can 
make informed decisions about when and how to use OPE effectively. Always consider alternatives and complement OPE 
with post-deployment monitoring to ensure the best outcomes.

In a future article, we will share more on our experience with OPE in the field of HVAC control, including results obtained 
on real-world problems.

## References

[Eligibility traces for off-policy policy evaluation](http://www.incompleteideas.net/papers/PSS-00.pdf)<br>
[Intrinsically Efficient, Stable, and Bounded Off-Policy Evaluation for Reinforcement Learning](https://arxiv.org/abs/1906.03735)<br>
[The Self-Normalized Estimator for Counterfactual Learning](https://papers.nips.cc/paper_files/paper/2015/hash/39027dfad5138c9ca0c474d71db915c3-Abstract.html)<br>
[A Review of Off-Policy Evaluation in Reinforcement Learning](https://arxiv.org/abs/2212.06355)<br>
[Doubly Robust Policy Evaluation and Optimization](https://arxiv.org/pdf/1503.02834)<br>

## Tools

[HVAC optimisation with Off Policy Evaluation and Selection](https://hopes.foobot.io/)