---
layout: post 
title:  "Using Physics-informed Neural Networks to solve 2D heat equation"
date: 2022-09-15 00:00:00 
categories: "ai deeplearning physics"
comments: false 
use_math: true
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

<center>
$ \frac{\partial u}{\partial t} = \alpha \frac{\partial^2 u}{\partial x} \frac{\partial^2 u}{\partial y} $
</center>

Often simplified using notation:

<center>$u_t = \alpha \cdot u_{xx} \cdot u_{yy}$</center>

where $u$ is the function that associates $t, x, y$ with a temperature, $x$ and $y$ points on the 2D surface, $t$ the time 
variable and $\alpha$ is the coefficient of diffusion.

A PDE is also defined by its boundary and initial conditions. In present case we'll set left, bottom and right edges as 
cold surfaces at 0 unit of temperature, while the top is heated at 1 unit of temperature:

<center>$u(t, 0, y) = u(t, x_{max}, y) = u(t, x, 0) = 0$,  $0 \leq y < y_{max}$</center>
<center>$u(t, x, y_{max}) = 1$</center>

As an initial condition, we'll consider the whole surface at 0 unit of temperature, ie $u(0, x, y) = 0$.

## Solving with Finite-Difference Method

The Finite-Difference Method (FDM) offers a way to numerically solve PDEs using discrete space and time spaces 
to enable derivative functions approximation. 

First, $\frac{\partial u}{\partial t}$ can be approximated with 
$ \frac{u_{i,j}^{k+1} - u_{i,j}^{k}}{\Delta t} $. 

Then for second order derivatives, we can transform this way: 

<center>
$\frac{\partial^2 u}{\partial x} = \frac{\partial}{\partial x} \cdot (\frac{\partial u}{\partial x}) \approx \frac{u_{i+1,j}^{k} - 2u_{i,j}^{k} + u_{i-1,j}^{k}}{\Delta x^2} $ 
</center>

If we transform the whole original equation we get:

<center>
$ \frac{u_{i,j}^{k+1} - u_{i,j}^{k}}{\Delta t} = \alpha (\frac{u_{i+1,j}^{k} - 2u_{i,j}^{k} + u_{i-1,j}^{k}}{\Delta x^2} + \frac{u_{i,j+1}^{k} - 2u_{i,j}^{k} + u_{i,j-1}^{k}}{\Delta y^2}) $
</center>

Taking $\Delta x = \Delta y$ (a grid with steps of equal size in all directions), we can simplify the equation to:

<center>
$ u_{i,j}^{k+1} = \gamma(u_{i+1,j}^{k} + u_{i-1,j}^{k} + u_{i,j+1}^{k} + u_{i,j-1}^{k} -4u_{i,j}^{k}) + u_{i,j}^{k} $
</center>
with $ \gamma = \alpha \frac{\Delta t}{\Delta x^2} $
<br/>
This can be quickly implemented using a 3 nested for loops in python, or you can go for a more efficient method using 
Jax and its Just In Time (JIT) compiler:

```python
@jax.jit
def compute_next_k(u_k):
    """
    Finite-difference method, approximation of derivatives using discretization of space dimensions
    to compute u_k+1
    """
    next_u = jnp.array(u_k)

    u_next_k = gamma * (
            jnp.roll(u_k, 1, axis=0) +
            jnp.roll(u_k, -1, axis=0) +
            jnp.roll(u_k, 1, axis=1) +
            jnp.roll(u_k, -1, axis=1)
            - 4 * u_k
    ) + u_k

    return next_u.at[1:-1, 1:-1].set(u_next_k[1:-1, 1:-1])
```

## Solving with PINN

## References

<a id="1">[1]</a>
Raissi, Maziar and Perdikaris, Paris and Karniadakis, George E
<i>Physics-informed neural networks: A deep learning framework for solving forward and inverse problems involving nonlinear partial differential equations</i> [2019](https://www.sciencedirect.com/science/article/pii/S0021999118307125)

<a id="2">[2]</a>
Raissi, Maziar and Perdikaris, Paris and Karniadakis, George E
<i>Physics Informed Deep Learning (Part I): Data-driven Solutions of Nonlinear Partial Differential Equations</i> [2017](https://arxiv.org/abs/1711.10561)

<a id="3">[3]</a>
[Heat Equation](https://en.wikipedia.org/wiki/Heat_equation)