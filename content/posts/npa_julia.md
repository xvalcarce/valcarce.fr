+++
title = "Quantum bound on the CHSH inequality: NPA in Julia with NCTSSOS"
date = "2022-05-22"
type = ["posts","post"]
[ author ]
  name = "Xavier Valcarce"
+++

The CHSH inequality is used to grasp the non-local nature of bipartite games in which each party perform one out of two dichotomic measurements on a shared state.
The Tsilerson bound of the CHSH inequality is the maximum violation of this inequality under the best quantum strategy [1].
I.e. the highest CHSH score that can be obtained over all quantum states and measurements.  

Finding the best CHSH score can be seen as an optimization of a polynomial of non-commutative variables. Such optimization can be relaxed to a hierarchy of SDP [2], known as a NPA hierarchy [3]. We here use the Juia package `NCTSSOS` to perform such a relaxation. More on that package can be found here [4].

The optimization problem is

\\[\\begin{align}
	\max_{E,\rho} \quad & \sum_{i,j=0}^1 (-1)^{ij} Tr[E_i E_j \rho] \\\\
	\text{s.t.} \quad & Tr[\rho] = 1 \\\\
	& E_i^\dagger = E_i \\\\
	& E_i E_j = \delta_{ij} E_i, \quad E_i,E_j \in M_k, \forall M_k \\\\
	& \sum_i E_i = 1 \\\\
	& [E_i,E_j] = 0, \quad E_i\in [M_1,M_2],\quad E_j\in [M_3,M_4]
	\\end{align}
\\]

We start by defining 8 non-commutative variables (2 projectors per measurement) using the `DynamicPolynomials` package

```julia
using DynamicPolynomials

@ncpolyvar A1[1:2] A2[1:2] B1[1:2] B2[1:2]
A = [A1,A2]
B = [B1,B2]
```

We then define a function returning all the relevant constraints on the projectors composing the measurements
```julia
function constraints_projectors(A,B)
	cons = Vector{Polynomial{false,Float64}}() #empty vector of polynomials
	for M in [A,B]
		for m in M
			append!(cons,[m[1]*m[2]]) # orthogonality
			for e in m
				append!(cons,[e^2-e])
			end
		end
	end
	# commutativity
	for a in A
		for e in a
			for b in B
				for f in b
					append!(cons,[e*f-f*e])
				end
			end
		end
	end
	return cons
end

eq = constraints_projectors(A,B)
```

The objective function is a weighted sum of correlators. They are expressed using expectation value that we define with the function
```julia
expect(M_x;outcomes=[1,-1]) = outcomes[1]*M_x[1] + outcomes[2]*M_x[2] 
C = expect.([A1,A2,B1,B2])
```

Finally we express the objective function as the CHSH score 

```julia
obj = -(C[1]*C[3] + C[1]*C[4] + C[2]*C[3] - C[2]*C[4])
```
Note that the minus sign is due to `NCTSSOS` that will minimize and not maximize the objective function.  

Finally, we send everything to `NCTSSOS` that will construct and solve the problem.

```julia
using NCTSSOS

level = 1 #level of the hierarchy
n_eq = length(eq)
pop = [obj; eq]

opt, data = nctssos_first(pop, [A1...,A2...,B1...,B2...], level, numeq=n_eq, TS="block");
```

Here is the result for the first level of the hierarchy
```
***************************NCTSSOS***************************
NCTSSOS is launching...
Starting to compute the block structure...
------------------------------------------------------
The sizes of PSD blocks:
[9]
[1]
------------------------------------------------------
Obtained the block structure in 0.000528174 seconds. The maximal size of blocks is 9.
There are 45 affine constraints.
Assembling the SDP...
SDP assembling time: 0.000512516 seconds.
Solving the SDP...
Problem
  Name                   :                 
  Objective sense        : max             
  Type                   : CONIC (conic optimization problem)
  Constraints            : 45              
  Cones                  : 0               
  Scalar variables       : 29              
  Matrix variables       : 1               
  Integer variables      : 0               

Optimizer started.
Presolve started.
Linear dependency checker started.
Linear dependency checker terminated.
Eliminator started.
Freed constraints in eliminator : 0
Eliminator terminated.
Eliminator started.
Freed constraints in eliminator : 0
Eliminator terminated.
Eliminator - tries                  : 2                 time                   : 0.00            
Lin. dep.  - tries                  : 1                 time                   : 0.00            
Lin. dep.  - number                 : 0               
Presolve terminated. Time: 0.00    
Problem
  Name                   :                 
  Objective sense        : max             
  Type                   : CONIC (conic optimization problem)
  Constraints            : 45              
  Cones                  : 0               
  Scalar variables       : 29              
  Matrix variables       : 1               
  Integer variables      : 0               

Optimizer  - threads                : 4               
Optimizer  - solved problem         : the primal      
Optimizer  - Constraints            : 45
Optimizer  - Cones                  : 1
Optimizer  - Scalar variables       : 14                conic                  : 14              
Optimizer  - Semi-definite variables: 1                 scalarized             : 45              
Factor     - setup time             : 0.00              dense det. time        : 0.00            
Factor     - ML order time          : 0.00              GP order time          : 0.00            
Factor     - nonzeros before factor : 1035              after factor           : 1035            
Factor     - dense dim.             : 0                 flops                  : 3.98e+04        
ITE PFEAS    DFEAS    GFEAS    PRSTATUS   POBJ              DOBJ              MU       TIME  
0   1.0e+00  1.0e+00  1.0e+00  0.00e+00   0.000000000e+00   0.000000000e+00   1.0e+00  0.00  
1   2.9e-01  2.9e-01  4.1e-01  -6.84e-01  -1.325948088e+00  -2.628521667e+00  2.9e-01  0.00  
2   5.1e-02  5.1e-02  2.0e-02  4.00e-01   -3.126913608e+00  -3.133309605e+00  5.1e-02  0.01  
3   1.6e-03  1.6e-03  1.4e-04  1.25e+00   -2.818018898e+00  -2.821554903e+00  1.6e-03  0.01  
4   2.0e-05  2.0e-05  2.1e-07  9.90e-01   -2.828462112e+00  -2.828524906e+00  2.0e-05  0.01  
5   1.5e-06  1.5e-06  4.3e-09  9.96e-01   -2.828433218e+00  -2.828437977e+00  1.5e-06  0.01  
6   4.8e-08  4.8e-08  2.5e-11  9.99e-01   -2.828427384e+00  -2.828427541e+00  4.8e-08  0.01  
7   4.7e-10  4.7e-10  2.4e-14  1.00e+00   -2.828427128e+00  -2.828427129e+00  4.7e-10  0.01  
Optimizer terminated. Time: 0.01    

SDP solving time: 0.014336663 seconds.
optimum = -2.8284271275550936
```
We observe a maximum score of 2.8284271275550936 corresponding to 2√2, the expected results.

-----

[1] Clauser, J. F.; Horne, M. A.; Shimony, A. & Holt, R. A. [Proposed Experiment to Test Local Hidden-Variable Theories.](https://journals.aps.org/prl/abstract/10.1103/PhysRevLett.23.880) Physical Review Letters, 1969, 23, 880-884.   
[2] Pironio, S.; Navascués, M. & Acín, A. [Convergent Relaxations of Polynomial Optimization Problems with Noncommuting Variables.](https://epubs.siam.org/doi/10.1137/090760155) Siam Journal on Optimization, 2010, 20, 2157--2180.  
[3] Navascués, M.; Pironio, S. & Acín, A. [A convergent hierarchy of semidefinite programs characterizing the set of quantum correlations.](https://iopscience.iop.org/article/10.1088/1367-2630/10/7/073013) New Journal of Physics, 2008, 10, 073013.  
[4] Wang, J. & Magron, V. [Exploiting term sparsity in Noncommutative Polynomial Optimization.](https://arxiv.org/abs/2010.06956) Arxiv:2010.06956

