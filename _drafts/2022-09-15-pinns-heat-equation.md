---
layout: post 
title:  "Using Physics-informed Neural Networks to solve 2D heat equation"
date: 2022-09-15 00:00:00 
categories: "ai deeplearning physics"
comments: false 
author: Antoine Galataud
---
{% if page.noindex == true %}
  <meta name="robots" content="noindex">
{% endif %}

<style type="text/css">
.center {
    display:block;
    margin: 0 auto;
}
.double-left {
    width:49%;
    display:block;
    float:left;
    margin: 0 auto;
    margin-right: 10px;
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
}
.small-img {
  width: 45%;
}
ul {
  display: table;
}
</style>


In real world problems, data is often scarce and noisy, limiting how fast an ML/AI-based solutions can become effective 
in the field. Mathematical models of physical phenomenons exist, but they require intractable computation times and level of
details about the environment.

Physics-informed neural networks (PINNs) try to mary the best of both worlds:

- learn from the theoretical model represented by a partial differential equation (PDE), to ensure robust predictions and limit amount of field data to collect
- learn from field data to adapt the trained neural network to the target environment

## PINNs, the theory

Many physical aspects of our surrounding world can be described using PDEs. But computing the solution to these equations 
often require complex models and details about the environment that are often not available. Worse, solutions would need 
to be recomputed for every change in the PDE conditions, which will always happen. Depending on the variability of the 
environment, the number of computations would grow exponentially.

Physics-informed neural networks [[1]](#1)[[2]](#2) offer an elegant and flexible solution to solve PDEs in a more universal manner:

- thanks to the capacity of neural networks to approximate arbitrary functions, they can also approximate PDEs involving derivates of different orders
- they are not limited by discretization like classic approximators of PDEs, thus can infer any point in space
- they are not limited to specific boundary or initial conditions, given that they are appropriately trained (ie conditions given as input)
- they take advantage of modern deep learning frameworks auto differentiation methods to compute derivatives (ie `torch.autograd`)

In this article we'll see how we can solve the 2 dimensional heat equation. 

## The Partial Differential Equation of Heat

Heat diffusion equation [[3]](#3) describes the diffusion of heat over time and space. It's a PDE, involving time and space 
derivatives. The basic equation in a 2D space is:

<center>$$\frac{\partial u}{\partial t}$ = \alpha x $\frac{\partial^2 u}{\partial x}$ x $\frac{\partial^2 u}{\partial y}$$ (1.a)</center>
Often simplified using notation:

<center>$u_t = \alpha x u_xx * u_yy$ (1.b)</center>

where $u$ is the temperature function, $x$ and $y$ points on the 2D surface and $\alpha$ is the coefficient of diffusion.

A PDE is also defined by its boundary and initial conditions. In present case we'll set left, bottom and right edges as 
cold surfaces at 0 unit of temperature, while the top is heated at 1 unit of temperature:

<center>$u(t, 0, y) = u(t, x_max, y) = u(t, x, 0) = 0$ with $0 <= y < y_max$</center>
<center>$u(t, x, y_max) = 1$</center>

As an initial condition, we'll consider the whole surface at 0 unit of temperature, ie $u(0, x, y) = 0$.

## High dimensional action spaces

![sources]({{ site.baseurl }}/assets/ar_policy/ar_def.svg){: .center }
<center class="image-foot"><i>fig. 1: 3 ways of modeling decision processes with state / action transitions using independent actions distributions (a.) and AR models (b. and c.)</i></center>
<br/>

### State conditionally independent actions

This is the most common case. In many classic problems, each action of a multi-decision problem is modeled and its probability distribution is estimated as independent from other actions. This doesn't allow to account for relationships between actions that can be encountered in many problems.

### Introducing action dependence with autoregressive policies
Given a policy <img src="https://render.githubusercontent.com/render/math?math=\pi_{\theta}"> to optimize with regards to parameters <img src="https://render.githubusercontent.com/render/math?math=\theta">, state <img src="https://render.githubusercontent.com/render/math?math=s \in \mathcal{S}">, a set of n actions <img src="https://render.githubusercontent.com/render/math?math=a_{i} \in \mathcal{A}">, we can represent the policy as following:
<center><img src="https://render.githubusercontent.com/render/math?math=\pi_{\theta}(a, s) = \prod_{i=1}^{n} \pi_{\theta}(a_{i} \bigm\lvert a_{k<i}, s)"></center><br/>

This simple formulation is generic and describes simple relationships and order with action at rank i being dependent on all previous actions.

An example neural network architecture illustrating this formulation is given below

![sources]({{ site.baseurl }}/assets/ar_policy/ar_nn_arch.svg){: .center }
<center class="image-foot"><i>fig. 2: example neural network architecture to model conditional dependencies between actions</i></center>
<br/>

In this model, a policy with 2 actions is used. It should be noted that:

- `Action 1` and `Action 2` are actually parameters of their respective probability distributions (PD).
- `Action 1` scalar value (sampled from estimated PD) is passed as input for `Action 2` distribution estimation
- `Action 1` *logits* could be passed as input instead of scalar value. This can help preserve information on large discrete/categorical distributions.
- `Action 2` inputs could be solely made of `Action 1` output (scalar or logits). In such case policy can be formulated as <img src="https://render.githubusercontent.com/render/math?math=\pi_{\theta}(a, s) = \pi_{\theta}(a_1, s) \cdot \prod_{i=2}^{n} \pi_{\theta}(a_{i} \bigm\lvert a_{k<i})">

Note that this model wouldn't scale well for the estimation of a large number of action distributions. For large action spaces, one can refer to the *MADE* architecture [[1]](#1).

## Experiments

To validate benefits of such architecture, we compare performance of "classic" network models with the one described in previous section. To draw a fair comparison, the same total number of neural network cells is used in classic and AR policies networks. Furthermore, all other parameters of the Proximal Policy Optimization (PPO, Shulman et al. [[2]](#2)) algorithm are kept equal between the 2 models.

### A path-finding task

The first task is a toy problem where an agent must navigate through a 2D-grid and maximize its reward by moving to certain locations. The action space is designed with 2 actions (vertical and horizontal movements). 
To emphasize on the need to learn correlation between actions, the reward scheme is modeled using 2 gaussian distributions located on 2 corners of the grid, and a one-time bonus whenever the agent reaches a point close to the mean of each gaussian. The agent starts each episode at the center of the grid. Episode ends when all bonuses are found. We refer to this training environment as bi-modal environment (BME).

![sources]({{ site.baseurl }}/assets/ar_policy/bi_modal_env_grid.png){: .center }
<center class="image-foot"><i>fig. 3: rewards distribution over the BME grid</i></center>
<br/>

We run 5 experiments with different seeds for each method (classic and AR) on the BME and compare the mean return per episode below.

![sources]({{ site.baseurl }}/assets/ar_policy/bi_modal_avg_return.png){: .center }
<center class="image-foot"><i>fig. 4: average episode reward for non AR policy (classic, light green) and AR policy (dark green)</i></center>
<br/>

On this toy environment, AR policy achieves a much better mean episode reward than its counterpart.

Visual inspection of trajectories for each method can highlight non-efficiencies and differencies. 
Below animations capture sample trajectories after 50 training iterations.

![sources]({{ site.baseurl }}/assets/ar_policy/bi_modal_classic_traj.gif){: .double }
![sources]({{ site.baseurl }}/assets/ar_policy/bi_modal_ar_traj.gif){: .double }
<center class="image-foot"><i>fig. 5a and 5b: trajectory examples after 50 training iterations of an agent trained under a classic policy (left) and an AR policy (right)</i></center>
<br/>

### A simulated HVAC

We now use a simulated building and its HVAC. The building model, or digital twin, is created from an actual building and calibrated on Foobot platform using OpenStudio and EnergyPlus open source software. It is a 3 storey office building, with a dedicated outdoor air system, a single air handling unit (AHU) with eletric heating and chilled water coil served by a local plant, and fan coil units for zones comfort.

![sources]({{ site.baseurl }}/assets/ar_policy/dt_3d_geo.png){: .center .small-img }
<center class="image-foot"><i>fig. 6: office building used as a test environment</i></center>
<br/>

A policy is trained with and without autoregressive (AR) actions model. The observation and action spaces, reward function, and PPO parameters are identical between the 2 methods. The action space is composed of 2 actions that command the amount and temperature of air supplied to the zones by the AHU. Each experiment is run for 1e6 timesteps.

![sources]({{ site.baseurl }}/assets/ar_policy/hvac_env_rew.svg){: .double }
![sources]({{ site.baseurl }}/assets/ar_policy/hvac_env_power.svg){: .double }
<br/>
![sources]({{ site.baseurl }}/assets/ar_policy/hvac_env_tmp.svg){: .double }
![sources]({{ site.baseurl }}/assets/ar_policy/hvac_env_co2.svg){: .double }
<center class="image-foot"><i>fig. 7 average episode reward (a., upper left), power demand (b., upper right, in kW), indoor temperature (c., lower left, in Celsius degrees) and CO2 (d., in ppm) for classic and AR policy (blue line) models</i></center>
<br/>

AR policy model not only achieves a better mean episode reward (fig 7a.), but it significantly outperforms (by 8.5%) the energy consumption reduction of the classic method. Other reward components, like thermal comfort and indoor air quality, are also better in line with their respective constraints which are a maximum of 25°C indoor temperature and a maximum of 1000ppm for indoor CO2.

### Analysing impacts and relationships

To understand better our model, a short study is conducted to highlight how action 1 influence action 2, how each action influence the state space, and what part of the state space has the most influence on agent decisions.

To do that, we'll first map our system and learned policy with functions approximations that govern it. They can be seen as forward models that predict the behavior of our policy.

- *transitions* that associates an action and a state to a next state <img src="https://render.githubusercontent.com/render/math?math=f_{tr}(a_{t}, s_{t}) \Rightarrow s_{t'}">
- *rewards* that maps action and state with its immediate reward <img src="https://render.githubusercontent.com/render/math?math=f_{rew}(a_{t}, s_{t'}) \Rightarrow r_{t}">
- *policy* that maps a state and the next action <img src="https://render.githubusercontent.com/render/math?math=f_{p}(s_{t}) \Rightarrow a_{t'}">
- *autoregression* that looks for correlation between action 1 and action 2 <img src="https://render.githubusercontent.com/render/math?math=f_{ar}(a_{1, t}) \Rightarrow a_{2, t}">

We then compute the jacobian of each function in order to differentiate with respect to relevant parameters. For instance, to highlight what part of the state space is directly influenced by agent decisions, we'll use the jacobian of the transitions function <img src="https://render.githubusercontent.com/render/math?math=\mathbb{J}({f_{tr}})"/> and differentiate with respect to agent actions.

![sources]({{ site.baseurl }}/assets/ar_policy/jac_f_p1.png){: .double-unconst }
![sources]({{ site.baseurl }}/assets/ar_policy/jac_f_p2.png){: .double-unconst }
<center class="image-foot"><i>fig. 8: what part of the state space influences decisions the most, for each action a1 (left) and a2 (right). Actions a1 and a2 seem influenced by different parts of the state</i></center><br/>

![sources]({{ site.baseurl }}/assets/ar_policy/jac_f_tr.png){: .center }
<center class="image-foot"><i>fig. 9: what part of the state space is "controllable" by the policy, for each action (a1, a2) of the space. Here indoor conditions are the most influenced, from both actions</i></center>

Influence of a1 over a2 can also be estimated thanks to computing the jacobian of the forward model that approximates the predictor function that gives a2 knowing a1. This allows to highlight the much stronger influence score of a1 over a2, as shown in [figure 11](#fig11).

![sources]({{ site.baseurl }}/assets/ar_policy/jac_f_ar_bar.png){: .center }
<center class="image-foot"><i>fig. 10: a1 influence on a2, as a scalar score. Left bar for a classic model, right for the AR one</i></center>

We can also estimate how much each action influence the immediate reward. Interestingly, forward models learned on AR and classic policies lead different estimate: model learned on classic policy shows a somewhat equal influence of each action, while model learned on AR policy shows a much stronger influence of a1.

<a id="fig11"></a>![sources]({{ site.baseurl }}/assets/ar_policy/jac_f_rew.png){: .center }
<center class="image-foot"><i>fig. 11: a1 and a2 influence score on immediate reward, for each policy type</i></center>

## Conclusion

Through a series of examples we've demonstrated that autoregressive (AR) policies can bring significant performance and training speed improvements compared to more classic models.

## References

<a id="1">[1]</a>
Raissi, Maziar and Perdikaris, Paris and Karniadakis, George E
<i>Physics-informed neural networks: A deep learning framework for solving forward and inverse problems involving nonlinear partial differential equations</i> [2019](https://www.sciencedirect.com/science/article/pii/S0021999118307125)

<a id="2">[2]</a>
Raissi, Maziar and Perdikaris, Paris and Karniadakis, George E
<i>Physics Informed Deep Learning (Part I): Data-driven Solutions of Nonlinear Partial Differential Equations</i> [2017](https://arxiv.org/abs/1711.10561)

<a id="3">[3]</a>
[Heat Equation](https://en.wikipedia.org/wiki/Heat_equation)