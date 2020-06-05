# SNOPT

Portions of this guide have been adapted by work from MDO Lab members, specifically Neil Wu.


## SNOPT overview

### What is SNOPT?

SNOPT is a proprietary SQP optimizer, which solves a sequence of quadratic programming (QP) problems, wrapped in [pyOptSparse](https://github.com/mdolab/pyoptsparse).
At each major iteration, several steps are taken.
First, SNOPT determines a search direction based on the function value and its gradients at that point.
Second, a line search is queries the merit function along the search direction to determine the best step size.
The optimizer then proceeds along the search direction by that step size, and begins a new major iteration.
This process is repeated until convergence.

Please also take some time to cross-reference the various parameters mentioned here with the [SNOPT manual](https://www.ccom.ucsd.edu/~peg/papers/sndoc7.pdf), which has very detailed explanations of all the parameters mentioned here.
For a theory guide, please read the [SNOPT paper](https://epubs.siam.org/doi/abs/10.1137/S0036144504446096).

### Some theory details

#### Solving the QP

Due to the QP solution process, the linear constraints are satisfied exactly at each major iteration.
The nonlinear constraints are linearized using the constraint Jacobian, and satisfied approximately.


#### Line Search
The line search is performed on the merit function, which is a sum of the objective and a measure of how violated the constraints are.
During each major iteration, the sufficient decrease condition on the line search ensures that the merit function should decrease each time.
However, there are scenarios where SNOPT may increase a penalty parameter, which results in the merit function value increasing.


## SNOPT Options

Here's a list of SNOPT options which are commonly used.
These options are set via an options dictionary via pyOptSparse or OpenMDAO.
One thing to note is that certain options do not take a corresponding value, i.e. they're more like a switch which turns on a feature when enabled.
For these options, the corresponding value in the dictionary must be supplied as the Python keyword ``None``.

### ``Major iterations limit``
  This sets the maximum number of major iterations.
  If you want to simply test your code without running the full optimization, you could set this value to 0 or 1.
  
### ``Major feasibility tolerance``
  This sets the feasibility tolerance of the optimization, which is a measure of the constraint violation. Specifically, it is computed as

![\max_i \frac{|g_i(x)|}{||x||}](https://render.githubusercontent.com/render/math?math=%5Cmax_i%20%5Cfrac%7B%7Cg_i(x)%7C%7D%7B%7C%7Cx%7C%7C%7D)

  Obviously, it does not make sense to prescribe a feasibility tolerance which is tighter than the function precision of the constraint functions.
  
### ``Major optimality tolerance``
  This is essentially a measure of the numerical tolerances on the KKT system.
  Please look at the SNOPT manual for a precise definition of this term.
  
### ``Verify level``
  This controls whether a derivative check is performed, where the supplied analytic gradients are compared against finite-difference computations.
  The default is 0, which performs a cheap check, and higher values result in more expensive and detailed checking.
  Setting the value to -1 disables the checking, which should be done on a production run if you have already verified the derivatives.
  
### ``Major step limit``
  This value :math:`r` is used to compute an upper bound on the step length:

![\beta = r\left(\frac{1 + ||x||}{||p||}\right)](https://render.githubusercontent.com/render/math?math=%5Cbeta%20%3D%20r%5Cleft(%5Cfrac%7B1%20%2B%20%7C%7Cx%7C%7C%7D%7B%7C%7Cp%7C%7C%7D%5Cright))

  and the first step length attempted is :math:`\alpha_1=\min(1,\beta)`.
  On typicaly problems, the default value of :math:`r=2` will cause :math:`\beta > 1`, so it will not affect the line search.
  However, if the line search repeatedly takes large steps, possibly into undefined regions in your optimization problem, then adjusting this number to 0.1 or even 0.01 may improve the optimization behavior.
  But small step sizes will significantly impede progress towards the end of the optimization.
  It may be worthwhile to look into improved optimization scaling to alleviate this problem instead.
  
### ``Derivative linesearch`` and ``Nonderivative linesearch``
  These options control whether or not to use gradients in the line search.
  If gradients are relatively cheap to compute, then they can help in the line search process.
  Given the initial point and a candidate point along the search direction, if both function values and gradients are available at those two points, then there exists a unique cubic polynomial which interpolates them.
  This can be used to reduce the number of function evaluations needed during line search, at the expense of gradient evaluations.
  However, in some cases it may be more effective to use non-derivative line search, especially if the function is highly nonlinear and the gradients are expensive.
  
### ``Penalty parameter``
  This sets the initial penalty parameter used with the merit function during line search.
  A higher value would force the line search to satisfy feasibility more quickly, but perhaps at the cost of slower convergence.
  Note that this only sets the initial value, and SNOPT internally will adaptively change the penalty parameter during optimization.
  Furthermore, a scalar value is applied here, but in reality the penalty parameter is a vector, with each entry corresponding to one constraint.
  It is not possible to set different initial values for different constraints.
  
### ``Function precision``
  This parameter describes the expected precision of the function evaluations, for both the objective and constraints.
  Line search is terminated when the difference between function values along the search direction becomes smaller than this value.



## How to interpret SNOPT results: the `SNOPT_print.out` file

When SNOPT runs it produces two files: `SNOPT_print.out` and `SNOPT_summary.out`.
The more useful file to examine is `SNOPT_print.out`, so we explain how to read it here.

### Example `SNOPT_print.out` file

```

         ==============================
         S N O P T  7.2-12.2 (Jul 2013)
         ==============================
      Derivative level               3
1
 
 SNMEMB EXIT 100 -- finished successfully
 SNMEMB INFO 104 -- memory requirements estimated

         ==============================
         S N O P T  7.2-12.2 (Jul 2013)
         ==============================
      Derivative level               3
1
 
 Parameters
 ==========

 Files
 -----
 Solution file..........         0       Old basis file ........         0       Standard input.........         5
 Insert file............         0       New basis file ........         0       (Printer)..............        18
 Punch file.............         0       Backup basis file......         0       (Specs file)...........         0
 Load file..............         0       Dump file..............         0       Standard output........         6

 Frequencies
 -----------
 Print frequency........       100       Check frequency........        60       Save new basis map.....       100
 Summary frequency......       100       Factorization frequency        50       Expand frequency.......     10000

 QP subproblems
 --------------
 QPsolver Cholesky......
 Scale tolerance........     0.900       Minor feasibility tol..  1.00E-06       Iteration limit........     10000
 Scale option...........         0       Minor optimality  tol..  1.00E-06       Minor print level......         1
 Crash tolerance........     0.100       Pivot tolerance........  3.25E-11       Partial price..........         1
 Crash option...........         3       Elastic weight.........  1.00E+04       Prtl price section ( A)         2
                                         New superbasics........        99       Prtl price section (-I)         2

 The SQP Method
 --------------
 Minimize...............                 Cold start.............                 Proximal Point method..         1
 Nonlinear objectiv vars         2       Major optimality tol...  2.00E-06       Function precision.....  3.00E-13
 Unbounded step size....  1.00E+20       Superbasics limit......         2       Difference interval....  5.48E-07
 Unbounded objective....  1.00E+15       Reduced Hessian dim....         2       Central difference int.  6.70E-05
 Major step limit.......  2.00E+00       Derivative linesearch..                 Derivative level.......         3
 Major iterations limit.      1000       Linesearch tolerance...   0.90000       Verify level...........         0
 Minor iterations limit.       500       Penalty parameter......  0.00E+00       Major Print Level......         1

 Hessian Approximation
 ---------------------
 Full-Memory Hessian....                 Hessian updates........  99999999       Hessian frequency......  99999999
                                                                                 Hessian flush..........  99999999

 Nonlinear constraints
 ---------------------
 Nonlinear constraints..         2       Major feasibility tol..  1.00E-06       Violation limit........  1.00E+06
 Nonlinear Jacobian vars         2

 Miscellaneous
 -------------
 LU factor tolerance....      3.99       LU singularity tol.....  3.25E-11       Timing level...........         3
 LU update tolerance....      3.99       LU swap tolerance......  1.22E-04       Debug level............         0
 LU partial  pivoting...                 eps (machine precision)  2.22E-16       System information.....        No
                                                                                 Sticky parameters......        No
1
 

 

 Matrix statistics
 -----------------
               Total      Normal        Free       Fixed     Bounded
 Rows              2           2           0           0           0
 Columns           2           0           0           0           2

 No. of matrix elements                    4     Density     100.000
 Biggest  constant element        0.0000E+00  (excluding fixed columns,
 Smallest constant element        0.0000E+00   free rows, and RHS)

 No. of objective coefficients             0

 Nonlinear constraints       2     Linear constraints       0
 Nonlinear variables         2     Linear variables         0
 Jacobian  variables         2     Objective variables      2
 Total constraints           2     Total variables          2
1
 

 
 The user has defined       4   out of       4   constraint gradients.
 The user has defined       2   out of       2   objective  gradients.

 Cheap test of user-supplied problem derivatives...

 The constraint gradients seem to be OK.

 -->  The largest discrepancy was    3.65E-07  in constraint     4
 

 The objective  gradients seem to be OK.

 Gradient projected in one direction  -9.03003000000E+02
 Difference approximation             -9.03002177940E+02
1
 
 

   Itns Major Minors    Step   nCon Feasible  Optimal  MeritFunction     L+U BSwap     nS  condHz Penalty
      1     0      1              1  9.6E-01  8.7E+00  9.0900000E+02       3                              _  r
      2     1      1 5.1E-01      2  7.6E-01  1.3E+00  1.9894663E+06       3                              _n rli
      3     2      1 7.3E-01      3  6.9E-01  1.7E-01  1.0136448E+06       3            1 2.0E+06         _s  li
      4     3      1 1.0E+00      5  4.0E-01  1.2E+03  8.6840019E+05       3                      2.0E+06 _sM
      4     4      0 4.2E-01      7  1.9E-01  7.2E+02  5.0308509E+05       3                      2.0E+06 _
      4     5      0 6.3E-01      9  5.9E-02  2.6E+02  1.8503308E+05       3                      2.0E+06 _
      4     6      0 1.0E+00     10 (0.0E+00)(0.0E+00) 3.0650000E+02       3                      2.0E+06 _
1
 
 SNOPTC EXIT   0 -- finished successfully
 SNOPTC INFO   1 -- optimality conditions satisfied

 Problem name                 H
 No. of iterations                   4   Objective value      3.0650000000E+02
 No. of major iterations             6   Linear objective     0.0000000000E+00
 Penalty parameter           1.954E+06   Nonlinear objective  3.0650000000E+02
 No. of calls to funobj             11   No. of calls to funcon             11
 No. of degenerate steps             0   Percentage                       0.00
 Max x                       2 2.0E+00   Max pi                      1 7.0E+02
 Max Primal infeas           0 0.0E+00   Max Dual infeas             0 0.0E+00
 Nonlinear constraint violn    0.0E+00
1
 
 Name           H                        Objective Value      3.0650000000E+02

 Status         Optimal Soln             Iteration      4    Superbasics     0

 Objective      
```

### Exit Codes
SNOPT provides a large number of exit codes.
Here we list the ones commonly encountered in the lab, with helpful tips on debugging.
The exit codes come in a pair of numbers, the first one signifying the type of exit, and the second provides additional information.

#### 0/1
This is the dream.
Optimization finished and satisfied all prescribed tolerances.

#### 0/3
Optimization finished, but could not achieve the prescribed optimality tolerance.
This occurs when SNOPT is less than two orders of magnitude away from the prescribed optimality tolerance, but cannot proceed further.
This is actually EXIT 40/41 wrapped in a deceptively benign-looking way.

#### 10/13
Problem is infeasible.
Check your constraints.

#### 30/32
Major iteration limit has reached.

#### 40/41
This is the dreaded "numerical difficulties".
It may be prudent to verify the gradient are correct.
Furthermore, it's possible that the function precision is too low, and the convergence tolerance of the analysis needs to be tightened.
SNOPT states that it is only reasonable to expect an optimality that is the square root of the function precision of the analysis.
That is, if you are getting 6 digits of accuracy on the analysis, then you cannot expect to get better than 1E-3 for optimality.
Therefore, it's really important to improve the convergence of the analyses.
Correspondingly, it's equally important to provide accurate gradients, meaning a tightly-converged adjoint.

As a last thing, poor problem scaling can easily cause this problem.

#### 60/61
The initial function evaluation failed.
The initial point did not result in a valid function value; there may be an analysis error.

#### 60/63
In subsequent cases, backtracking is used to avoid undefined regions.
However, if this fails, then SNOPT basically gives up.
It's important to examine why the evaluations failed, and try to develop a robust analysis so that SNOPT can continue.
