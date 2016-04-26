---
layout: default
title: Solving Poisson's Equation with Julia via the finite element method
permalink: /projects/julia-fe/
---

DISCLAIMER: This is a living document.  Check back for more later.
==================================================================

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

$$ u = g \text{ on }\partial\Omega $$

where \\(g \text{ and } f\\) are a continuous functions.

Basic Principles
================
We begin with the weak formulation of the stated problem: Determine \\(u \in 
\mathbb{R}^{n}\\) such that:

$$ \int\nabla u \nabla v d\Omega = \int v f d\Omega \tag{2}$$

for all \\( v \in \mathbb{R}^{n} \\). Now, let \\(V^{n} \subset \mathbb{R}^{n} \\) 
be the set of continuous functions over some partition of \\(\Omega\\).  The 
finite element approximation of the original problem is then given by the following.

Determine \\( u_h \in V^{n}\\) such that:

$$  \int\nabla u_h \nabla v d\Omega = \int v f d\Omega \tag{3}$$

is satisfied for all \\( u_{h} \in V^{n} \\). Since \\( u_h \in V^{n}\\) we can 
write:

$$ u_h = \sum_{i=0}^{N} u_i \phi_i \tag{4} $$

where {\\( \phi_j \\)} is the set of basis functions spanning \\(V^{n}\\).  Splitting 
the terms up into non-boundary and boundary terms respectively we see:

$$\sum_{i\notin\partial\Omega}u_i\phi_i+\sum_{i\in\partial\Omega}g\_i\phi_i \tag{5}$$ 

Subsituting 
\\((5)\\) into \\((3)\\) and replacing \\( v \text{ with } \phi_j \\) we get:

$$ 
\sum_{i \notin \partial\Omega} u_i \int \nabla \phi_i \phi_j d\Omega = 
\int\phi_j f d\Omega-\sum_{i\in\partial\Omega}(\int\nabla\phi_i\phi_j d\Omega)g_i 
\tag{5} 
$$

Which finally leads us to the matrix formulation of the problem:

$$ \hat{A} \vec{x} = \vec{b} $$

where: 

$$ 
A_{ij} = \int \nabla \phi_i \phi_j d\Omega 
$$

$$ 
x_i = u_i  
$$

$$ 
b_j = \int f \phi_j d\Omega -\sum_{i\in\partial\Omega}(\int\nabla\phi_i\phi_j d\Omega)g_i 
$$  

Finding the approximate solution then is accomplished by solving for \\(\vec{x}\\).

Implementation
==============
The solution to the stated problem is relatively easy once all of the parts are 
assembled, although actually implementing the solution may more of a challenge. Several
of the issues that are not commonly discussed are detailed in the next
three subsections.  

Mesh Representation
-------------------
Many introductory FEM examples provide in-code mesh definitions, making them 
difficult to adapt to different geometries.  Given that the ability to handle 
complex geometries gracefully is one of the greatest strengths of the FEM, I 
wanted to create a code that allowed for more advanced meshing capabilities.

In order to avoid going down the rabbit hole of building my own meshing package 
(maybe an excursion for a later time, however) I relied on [GMSH](http://gmsh.info/) 
to construct my example meshes and built an interface between the mesh file and 
the FEM solver module.  The exact details of how the mesh is read into Julia is 
fairly mundane, so I will skip over it.  If you wish to see how this works the 
code can be seen [here](https://github.com/arbennett/Julia-FE/blob/master/src/gmsh.jl#L40).

Once the GMSH file is read in the relevant information is encapsulated in a 
`Mesh` datatype:

    type Mesh
        n_nodes::Int64
        n_elements::Int64
        nodes::Array{Float64, 2}
        internal_nodes::Array{Int64}
        boundary_nodes::Array{Int64}
        elements::Array{Int64, 2}
        Mesh(
             n_nodes::Int64,
             n_elements::Int64,
             nodes::Array{Float64,2},
             internal_nodes::Array{Int64,1},
             boundary_nodes::Array{Int64,1},
             elements::Array{Int64,2}
            ) = new(
                    n_nodes, 
                    n_elements, 
                    nodes, 
                    internal_nodes, 
                    boundary_nodes, 
                    elements
                   )
    end

With the data structure laid out, we now face the issue of how to accurately 
reflect the geometry of the problem.  This implementation will use linear 
triangle shaped elements.  This representation makes the shape functions of the 
reference element extremely simple:

    # Quadrature points & weight
    const xi = 1/3
    const eta = 1/3
    const weight = 1/2
   
    # Line 1 
    phi1(xi,eta) = 1 - xi - eta
    const dphi1_dxi = -1.0
    const dphi1_deta = -1.0
   
    # Line 2
    phi2(xi,eta) = xi
    const dphi2_dxi = 1.0
    const dphi2_deta = 0.0
   
    # Line 3
    phi3(xi,eta) = eta
    const dphi3_dxi = 0.0
    const dphi3_deta = 1.0

The combination of the shape function definitions and the mesh datatype allow us 
to represent the solution.  To actually find the solution we must assemble our 
matrix equation.

Assembling \\(\hat{A}\\) and \\(\vec{b}\\)
------------------------------------------
To build the set of equations in \\( \(6\) \\) we will have to calculate some 
quantities of interest on the mesh elements first.  For each element in the 
solution we will calculate the gradient on the element, which is used to build 
local equivalents to \\( \hat{A} \\) and \\( \vec{b} \\).  Then, using the 
element-wise versions we can iteratively construct the global versions.  The 
process looks like the following.

Before we get into assembling \\(\hat{A}\\) and \\(\vec{b}\\), some setup:

    size = mesh.n_nodes - length(mesh.boundary_nodes)
    stiff = zeros(size,size)
    global_stiff = sparse(temp_stiff)
    stiff = zeros(3,3)
    gc() # Throw away the full sized array and keep the sparse one
    global_load = zeros(size)
    load = zeros(3)

Then, the following sections are to be in a loop over each element in the mesh.
 
Gradient calculation:

    # Get element nodes, and their locations
    elem_nodes = mesh.elements[elemIdx,:]
    x1, y1 = mesh.nodes[elem_nodes[1],:]
    x2, y2 = mesh.nodes[elem_nodes[2],:]
    x3, y3 = mesh.nodes[elem_nodes[3],:]    
    
    # Find the barycentric coordinates  
    x = x1×phi1(xi,eta) + x2×phi2(xi,eta) + x3×phi3(xi,eta)
    y = y1×phi1(xi,eta) + y2×phi2(xi,eta) + y3×phi3(xi,eta)    
    
    # Area and some slopes
    area = (x1-x3)×(y2-y1) - (x1-x2)×(y3-y1)
    dxi_dx, dxi_dy = (y3-y1)/area, -(x3-x1)/area
    deta_dx, deta_dy = -(y2-y1)/area, (x2-x1)/area    
    
    # Compute Jacobian matrix
    j11 = x1×dphi1_dxi  + x2×dphi2_dxi  + x3×dphi3_dxi
    j12 = x1×dphi1_deta + x2×dphi2_deta + x3×dphi3_deta
    j21 = y1×dphi1_dxi  + y2×dphi2_dxi  + y3×dphi3_dxi
    j22 = y1×dphi1_deta + y2×dphi2_deta + y3×dphi3_deta      
    
    # And the value of the Jacobian
    jacobian = abs(j11*j22 - j12*j21)    
    Dphi[1,1] = dphi1_dxi×dxi_dx + dphi1_deta×deta_dx
    Dphi[1,2] = dphi1_dxi×dxi_dy + dphi1_deta×deta_dy
    Dphi[2,1] = dphi2_dxi×dxi_dx + dphi2_deta×deta_dx
    Dphi[2,2] = dphi2_dxi×dxi_dy + dphi2_deta×deta_dy
    Dphi[3,1] = dphi3_dxi×dxi_dx + dphi3_deta×deta_dx
    Dphi[3,2] = dphi3_dxi×dxi_dy + dphi3_deta×deta_dy


\\(\hat{A}\\) calculation:

    for i=1:3
        for j=1:3
            stiff[i,j] = weight * jacobian * (Dphi[i,1]*Dphi[j,1] + Dphi[i,2]*Dphi[j,2])
        end
    end

    for i=1:3
        pos_i = findin(mesh.internal_nodes, elem_nodes[i])
        if length(pos_i) > 0
            for j=1:3
                pos_j = findin(mesh.internal_nodes, elem_nodes[j])
                if length(pos_j) > 0
                    global_stiff[pos_i[1], pos_j[1]] += stiff[i,j]
                end
            end
        end
    end

\\(\vec{b}\\) calculation:

    for i=1:3
        load[i] = weight * jacobian * f_ext(x,y) * psi[i]
    end

    for i=1:3
        pos_i = findin(mesh.internal_nodes, elem_nodes[i])
        if length(pos_i) > 0
            global_load[pos_i[1]] += load[i]
            for j=1:3
                pos_j = findin(mesh.internal_nodes, elem_nodes[j])
                if length(pos_j) < 1
                    global_load[pos_i[1]] -= stiff[i,j] * field[elem_nodes[j]]
                end
            end
        end
    end

The Solving the Matrix Equation
-------------------------------
Now that we've got the load vector and stiffness matrix we can use Julia's built in capabilities:

    U = global_stiff\global_load

