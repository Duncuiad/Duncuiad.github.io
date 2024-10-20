---
layout: post
title:  "Partitions of Unity"
date:   2024-10-13 07:00:00 +0100
categories: "coffee"
---

<p align="center">
  <img src="/Pictures/TileMesh2_800w.jpg" alt="Square Triangle Tiling"/>
</p>

<ul>
<li>Back to the <a href="/topics/coffee">Coffee</a> category index.</li>
</ul>

A common problem for a game developer is the need for smoothly transitioning between states of a system. Sometimes this is a camera transition, other times it's moving between selected points of interest on a map. Most often, these transitions happen over time: they start at some instant $t_0$ and we expect them to be over after some time has passed, at some other instant $t_1$. Today we will look at a different but related type of transition: the *spatial* kind. ~~This allows us to have different target states in some regions of a level and to mix them together as we move between pairs of them~~.

In this post, we'll build a set of functions that when multiplied with a set of states act as spatial weights for those states. We'll build them from the ground up, starting from the simplest of cases.

<h2>1. Interpolation</h2>
In the life of a game developer, few mathematical concepts are as pervasive as the concept of interpolation. Everybody has, at some point in their life, scribbled down the formula for linearly interpolating between two points $p_1$ and $p_2$ in space:

$$
p(t) = \left( 1-t \right) p_1 + t p_2
\text{, for }
0 \le t \le 1
$$

This formula tells us that at time $t=0$ we begin our journey at $p_1$, that for times between $0$ and $1$ we move along a segment between $p_1$ and $p_2$ and that we finally reach $p_2$ at precisely time $t=1$.

This simple expression can be looked at through the lenses of many different mathematical contexts. For the length of this post, the lens we're interested in is that of a <a href="https://en.wikipedia.org/wiki/Convex_combination">convex combination</a>. Let me explain. When we interpolate from $p_1$ to $p_2$, we're mixing the two points using two coefficients $\lambda_1$ and $\lambda_2$. These two numbers tell us what *portion* of each extreme ends up in the interpolated result. In order for them to take *slices* of the original positions and put them back in a *whole*, we will expect them to only take values between $0$ and $1$ and to always sum to $1$.

If we change $\lambda_2$'s name to $t=\lambda_2$, the only way to ensure that the relation $\lambda_1 + \lambda_2 = 1$ always holds is to require $\lambda_1$ to be equal to $1 - t$. We're back to the linear interpolation! What have we gained, then, from introducing the lambdas? The lucky accident is that this construction lets us interpolate simultaneously among *any number* of states.

Given $n$ vectors $v_1, \ldots, v_n$, we can combine them into a new vector $v$ using $n$ weights $\lambda_1, \ldots, \lambda_n$ such that $0 \le \lambda_i \le 1$ for all $i$ between $1$ and $n$, and $\lambda_1 + \ldots + \lambda_n = 1$:

$$
v \coloneqq \lambda_1 v_1 + \ldots + \lambda_n v_n
$$

We say that $v$ is a *convex combination* of $v_1, \ldots, v_n$ via the weights $\lambda_1, \ldots, \lambda_n$. Notice in particular how, if one weight happens to be equal to $1$, all other weight must necessarily be equal to $0$. In this case, $v$ matches exactly the state corresponding to the weight equal to $1$.

<h2>X. Transitions</h2>

Once you know how to interpolate, you can also do transitions: let the weights vary, depending on some external factor like time, and you're done. 

In the case of time-based transitions, it means that the weights $\lambda_i$ are going to be functions of time $\lambda_i(t)$. To transition from the vector $v_j$ to the vector $v_k$ in one second, we need to make sure that $\lambda_j$ is $1$ for $t \le 0$ (which forces all the weights except $\lambda_j$ to be $0$ during that time) and that $\lambda_k$ is $1$ for $t \ge 1$ (again, forcing all other weights to vanish).

Here is an example of a transition from $v_1$ to $v_3$ given three points $v_1$, $v_2$ and $v_3$ on the plane for a particular collection of time-dependent weights. These are their values for $t \in \left(0,1\right)$:

$$
\begin{array}{rcl}
\lambda_1(t) &=& t^2 - 2t + 1 \\
\lambda_2(t) &=& -2t^2 + 2t \\
\lambda_3(t) &=& t^2
\end{array}
$$

{% comment %} 
Add graphs for the three weights and an animation of the quadratic Bezier curve
{% endcomment %}

We usually want our transitions to be continuous, so that we don't suddenly teleport to a different position. Sometimes we want them to also be smooth, so that the transition happens without sudden changes in movement. The weights above generate an example of a transition that is continuous but not smooth.

That's all good for time-based transitions. But we wanted to do space-based transitions, right? Well, as you've already figured out, we just need to use weights that are functions of space rather than functions of time. Suppose that we have $n$ regions $R_1, \ldots, R_n$ that overlap over a shared region $R$ and are mutually disjoint otherwise. 

{% comment %} 
Add a diagram to show the different regions
{% endcomment %}

~~We have a property that has a different value on each region. Say for example that on the region $R_i$, the value of that property is the vector $v_i$~~. Within $R$, we want to mix the $v_i$'s together in a way that links smoothly to the fixed values they take in the parts of regions $R_i$ outside the overlap $R$. In other words, we want to create weights $\lambda_i$ such that:

$$
\begin{array}{rl}
\lambda_i(p) = 0   & \text{ for }  p \in R_j \setminus R \text{, for all } i \ne j \\
\lambda_i(p) = 1   & \text{ for }  p \in R_i \setminus R \\
0 < \lambda_i(p) < 1 & \text { for } p \in R \\
\lambda_1(p) + \ldots + \lambda_n(p) = 1 & \text{ everywhere } 
\end{array}
$$

{% comment %} 
Add example weights
{% endcomment %}

This post will focus on creating sets of smooth weights that transition over the interior of a convex polygon $P$. We'll build weights for a triangle, but the process generalises naturally to any number of edges and to three dimensions. We'll start by solving a simpler problem: finding $n$ functions $f_1, \ldots, f_n$ defined on every point of the polygon except its vertices, each of which satisfies the following properties:

$$
\begin{array}{rl}
f_i(p) = 0   & \text{ for }  p \in E_j \text{, for all } i \ne j \\
f_i(p) = 1   & \text{ for }  p \in E_i \\
0 < f_i(p) < 1 & \text { for } p \in P^\circ
\end{array}
$$

where $P^\circ$ denotes the interior of the convex polygon and $E_i$ denotes the $i$-th edge of the polygon, excluding the vertices. Notice how we're not requiring them to sum to $1$, nor to extend smoothly to a constant function across each edge. These are properties that we can extract from a collection of functions that satisfy the properties above, by processing them further.

<h2>X. Edge-pair Transitions</h2>

We build functions $f_i$ like above by combining sets of simpler functions. I'll denote as 

$$
\hat{P} \coloneqq P \setminus \left\{v_i\right\}_{i=1}^{n}
$$

the polygon $P$ without its vertices. Given any pair of different edges $E_i$, $E_j$ of the polygon (so $i \ne j$), let $h_{ij}$ be a continuous and smooth function over $\hat{P}$ such that $h_{ij}(p)=1$ along $E_i$, $h_{ij}(p)=0$ along $E_j$, and $0 < h_{ij}(p) < 1$ over $P^\circ$. The core idea is for $h_{ij}$ to work as a (descending) ladder from $E_i$ to $E_j$. The rest of the construction works regardless of the specific $h_{ij}$ we choose, the important part being to satisfy those properties. In practice, certain choices of $h_{ij}$ might work better than others depending on your specific combination of needs, including quality and performance.

A simple but effective choice for $h_{ij}$ is the ratio of sines of the angles (...)

<h2>X. Combining and Normalizing</h2>

<h2>X. Making it Smooth</h2>

<h2>X. Result</h2>


___

<h3>Notes and References:</h3>
