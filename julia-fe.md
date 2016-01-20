---
layout: default
title: Solving Poisson's Equation with Julia via the finite element method
permalink: /projects/julia-fe/
---

DISCLAIMER: This is a living document.  Check back for more later.
==================================================================

Overview
========
* A gentle intro to the finite element method
* The method applied to Poisson's equation
* A practical implementation with Julia
* The issue of meshing
* Putting it all together
* Solving other problems & improving efficiency

Background
==========
The finite element method (FEM) is a group of techniques for approximating
the solution of boundary value problems (BVPs).  This writeup will assume that
you have some basic understanding of how FEM works, though I will give a quick
(and extremely informal) refresher on the basics in the next section.

We will try to solve Poisson's Equation with a Dirichlet boundary condition. 
First, let \\( \Omega \subset \mathbb{R}^{n} \\).  Then, Poisson's equation is 
given by: 

$$ \nabla^{2}u = f \text{ in } \Omega \tag{1}$$

with some boundary condition:

$$ u = g \text{ on }\partial\Omega \text{ where } g \in \mathbb{R}$$

Basic Principles
================
We begin with the weak formulation of the stated problem: Determine \\(u \in 
\mathbb{R}^{n}\\) such that:

$$ \int\nabla u \nabla v d\Omega = \int v f d\Omega \tag{2}$$

for all \\( v \in \mathbb{R}^{n} \\). Now, let \\(V^{2} \subset \mathbb{R}^{n} \\) 
be the set of continuous functions over some partition of \\(\Omega\\).  The 
finite element approximation of the original problem is then given by the following.

Determine \\( u_h \in V^{n}\\) such that:

$$  \int\nabla u_h \nabla v d\Omega = \int v f d\Omega \tag{3}$$

is satisfied for all \\( u_{n} in V^{n} \\). Since \\( u_h \in V^{n}\\) we can 
write:

$$ u_h = \sum_{i=0}^{N}c_i \phi_i \tag{4} $$

where {\\( \phi_j \\)} is the set of basis functions spanning \\(V^{n}\\).  Subsituting 
\\((4)\\) into \\((3)\\) and replacing \\( v \text{ with } \phi_j \\) we get:

$$ \sum_{i=0}^{N} c_i \int \nabla \phi_i \phi_j d\Omega = \int \phi_j f d\Omega 
\tag{5} $$

Which finally leads us to the matrix formulation of the problem:

$$ \hat{A} \vec{x} = \vec{b} $$

where: 

$$ A_{ij} = \int \nabla \phi_i \phi_j d\Omega $$
$$ x_i = c_i $$
$$ b_j = \int f \phi_j d\Omega $$  

Finding the approximate solution then is accomplished by solving for \\(\vec{x}\\).

Implementation
==============


Mesh Representation
===================


Improvements & Variations
=========================


References
==========
