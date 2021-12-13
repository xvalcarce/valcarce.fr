+++
title = "Python - LP/SDP for Quantum Information with PICOS"
date = "2021-12-13"
type = ["posts","post"]
[ author ]
  name = "Xavier Valcarce"
+++

## SDP interface and Solver

To write LP/SDP in Python I recommend the [PICOS](https://gitlab.com/picos-api/picos) interface. It is under the GPLv3 free-license, probably supports your favorite solver, has crucial features for quantum information and, most importantly, has an attentive and firendly developper community.  

Regarding SDP solvers, in my experience, nothing compares to [MOSEK](https://www.mosek.com). Sadly, this solver is proprietary (if price is of your concern, MOSEK offer a free one-year license for students). [CVXOPT](https://cvxopt.org) is a relatively good free-licensed alternative to MOSEK. However, in my experience, CXOPT have memory-leakage issue, bugs when solving complex SDP, and an important optimization time compare to MOSEK.  

## LP: Local correlations, a membership problem

For this section we need the following import
```python
import picos as pcs
import numpy as np
from itertools import product
```

Consider a scenario where a state is shared between `N` parties, each having `m` measurement choices (inputs) with `d` possible results (outputs). The correlations between the parties admit a local hidden-variable model iff the correlations belong to a [*local polytope*](https://valcarce.fr/physics/master_thesis.pdf#section.2.2) [1,2].  
To know if correlations of a given scenario are local, it is sufficient to check if these correlations belong, i.e. are "inside", the local polytope.  

Correlations are in a vector of probability \\[p = \{p(a_1 \dots a_N|x_1 \dots x_N)\}\\] for all inputs x_i and output a_i.
The vertices of the local polytope are the derministic strategies `p_d`, i.e. `p` is composed of 0s and 1s. 
The local polytope can be written as a the set of its vertices `L=p_d`.  
Here a simple function that construct the local polytope

```python
def local_polytope(N,m,d):
    """Local Polytope generation.

    Generate :math:`d^{m.N}` vertices of the Local Polytpe for a system with ``N`` partite, each having ``m`` measurement/input and ``d`` output.

    Parameters
    ----------
    N : int
        number of partite
    m : int
        number of input of one partite
    d : int
        number of output of one partite

    Returns
    -------
    vertices : array_like
        Polytope. Each row is a vertex (corresponding to a deterministic strategy), each column is the :math:`n^{th}` element to that vertice

    """
    D = np.zeros((d**m,m), dtype=int)
    for _ in range(d**m):
        D[_][:] = np.array(np.unravel_index(_,(d,)*m))
    vertices = np.zeros(((d**(m*N),)+(m,)*N+(d,)*N))
    c = 0
    for _ in product(range(d**m), repeat=N):
        for x in product(range(m), repeat=N):
            vertices[(c,)+x+tuple([D[_[i]][x[i]] for i in range(N)])]=1
        c += 1
    shape = np.prod(np.delete(vertices.shape[:],0,0))
    polytope = vertices.reshape(vertices.shape[0],shape)
	return polytope
```

A correlation vector `p` is described by a local model iff `p` can be written as a convex sum of the local-polytope vertices.
This is a linear programming problem: find a vector `v` such that `v*L=p`, with `sum(v)=1` (convexity) and `v>0` (each element of `v` positive). 
If `v` exists then `p` is compatible with a local model.  

Using `PICOS` we write this linear program

```python
def is_local(correlation,polytope):
	pb = pcs.Problem() # Create a picos Problem instance
	# The two constants of our problem
	L = pcs.Constant("L", polytope)     # local polytope
	p = pcs.Constant("p", correlation)  # correlation vector
	# The variable v (the coefficient of the convex sum)
	dim = L.shape[0]                    # number of vertices
	v = pcs.RealVariable("v", dim)      # v is a real vector with dim element
	# Constraints
	pb.add_constraint(v >= 0)           # all elements of v are positive
	pb.add_constraint(sum(v) == 1)      # sum of elements of v is 1
	pb.add_constraint(L.T*v == p)       # v is the coefficient of the convex sum of all L vertices
	# Objective
	pb.set_objective("min", 1 | v)      # dummy objective

	# Solving the LP
	pb.solve(solver="cvxopt",primals=False)
	if pb.status == "optimal":
		return True                     # if an optimal solution exists, p belongs to L
	else:
		return False                    # otherwise p is non-local
```

To test our LP implementation let's now create a function that returns a local correlation vector `p`, by construction.

```python
def local_correlation(polytope):
	nb_vertices = len(polytope) # Number of vertices composing the polytope
	coeff = np.random.random(nb_vertices) # Random coefficients, one per vertices
	coeff /= np.sum(coeff) # Normalised coefficients
	p = np.sum([c*v for (c,v) in zip(coeff,polytope)],axis=0) # p is a convex sum of the polytopes vertices
	return p
```

We verify our implementation with two examples, one local, one non-local
```python
L = local_polytope(2,2,2)
p_l = local_correlation(L)
p_nl = L[1]+1e-3           # A non local correlation -> we take a vertice and we add 0.01 to each of its elements
print(is_local(p_l,L))     # Output True
print(is_local(p_nl,L))    # Output False
```

## (Complex) SDP: Maximize a Bell inequality

For this section, import and constant definition needed
```python
import numpy as np
import picos as pcs

# Paulis
Z = np.array([[1.,0.],[0.,-1.]])
X = np.array([[0.,1.],[1.,0.]])
# Kroenecker product
K = np.kron

```

Consider a scenario where Alice and Bob share a two-qubit state `rho`. They each have two measurements `A0,A1` for Alice, `B0,B1` for Bob.
These measurements are of the form \\[M_i = \cos(\theta)\sigma_x + (-1)^i \sin(\theta)\sigma_z\\] with `i` the measurement choice, 0 or 1, and `θ` the angle between measurement 0 and 1. Alice set `θ=a` while Bob set `θ=b`.  
A bell operator can be seen as a linear combination of the four correlators `AxBy`. For example the well-known CHSH operator `S=A0(B0+B1)+A1(B0-B1)`. Here is a function to generate such operators (return CHSH by default)

```python
def bell_op(a=np.pi/4,b=np.pi/4,coeff=[1,1,1,-1]):
    """ Return Bell operator, for specific angle between measurements (for each partite resp.)

    Parameters
    ----------
    a : float
        Angle between observabe, Alice side.
    b : float
        Angle between observabe, Bob side.
    coeff : array
        Numpy array of four elements, for A0B0, A1B0, A0B1, A1B1.

    Return
    ------
    bell : ndarray
        Bell operator
    """

    O = lambda x,a : np.cos(a)*X + (-1)**x*np.sin(a)*Z
    if coeff == [1,1,1,-1]:
        #CHSH case
        bell = K(O(0,a),O(0,b)+O(1,b)) + K(O(1,a),O(0,b)-O(1,b))
    else:
        bell = coeff[0]*K(O(0,a),O(0,b))+coeff[1]*K(O(1,a),O(0,b))+coeff[2]*K(O(0,a),O(1,b))+coeff[3]*K(O(1,a),O(1,b))
    return bell
```

The CHSH value achieved by `rho` is given by the linear operation \\[\text{Tr}\[S\rho\]\\]. Since all two-qubits state are semi-positive definite and the CHSH value is linear (in `rho`) we can use semi-definite programming (SDP) to find the maximum CHSH value a two-qubit state can achieve. Such SDP reads
\\[ \max_\rho \qquad \text{Tr}\[S\rho\]\\]
\\[ \text{s.t.} \qquad \rho \succeq 0\\]
\\[\\qquad \quad \text{Tr}\[\rho\]=1\\]
with `rho` a 4x4 Hermitian matrix.  

Using `PICOS`, we can write this SDP as follow

```python
def max_bell(operator):
    """ Compute the max(Tr[operator*rho]) over all rho = two-qubits state

    Parameters
    ----------
    operator : np.ndarray
        Two-qubits operator
    """
    sdp = pcs.Problem()
    #SDP "constant", the bell operator
    bell_operator = pcs.Constant(operator)
    #SDP variable (quantum state)
    rho = pcs.HermitianVariable('rho',4)
    #Constraints of the SDP :
        #semi positive definite
    sdp.add_constraint(rho >> 0)
        #Trace 1
    sdp.add_constraint(pcs.trace(rho) == 1)
    # Objective of the SDP: Tr[Bell*rho]
    obj = pcs.trace(bell_operator*rho).real
    sdp.set_objective('max',obj)
    
    # Some print stuff
    print("Solving: ")
    print(sdp)
    sol = sdp.solve(solver='cvxopt',verbosity=0)
    print(f"Solved with {sol.claimedStatus} status")
    print(f"Optimal violation: {obj}")
    print(f"State reaching this violation:\n{rho}")
```

Let's test our implementation
```python
S = bell_op()
max_bell(S)
```

Running this output
```python
Solving max(Tr[CHSH*rho] over all two-qubit states
Solving:
Complex Semidefinite Program
  maximize Re(tr([4×4]·rho))
  over
    4×4 hermitian variable rho
  subject to
    rho ≽ 0
    tr(rho) = 1
Solved with optimal status
Optimal violation: 2.8284271145023476
State reaching this violation:
[ 7.32e-02-j0.00e+00  1.77e-01-j1.85e-17  1.77e-01+j5.72e-18 -7.32e-02+j2.73e-17]
[ 1.77e-01+j1.85e-17  4.27e-01-j0.00e+00  4.27e-01+j5.60e-17 -1.77e-01+j4.77e-17]
[ 1.77e-01-j5.72e-18  4.27e-01-j5.60e-17  4.27e-01-j0.00e+00 -1.77e-01+j7.19e-17]
[-7.32e-02-j2.73e-17 -1.77e-01-j4.77e-17 -1.77e-01-j7.19e-17  7.32e-02-j0.00e+00]
```

We obtain the expectedviolation of `2.8284...` with corresponds to 2√2 achieved by Bell states [2].


## References

[1] Pironio S., J. Phys. A: Math. theor. 47 ([424020](https://iopscience.iop.org/article/10.1088/1751-8113/47/42/424020))  
[2] Brunner N. et. al., Rev. Mod. Phys. 86, ([419](https://journals.aps.org/rmp/abstract/10.1103/RevModPhys.86.419)) 
