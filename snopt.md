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
  
### ``Hessian full memory``
  This should always be used instead of ``Hessian limited memory``.
  
### ``Hessian frequency``
  This parameter is very much dependent on the nature of the optimization problem.
  If there is significant nonlinearity in the objective or constraints, for example due to KS aggregation, then a low value such as 10 could be used.
  For nicely-behaved problems such as aerodynamic optimization with twist, a sufficiently large number should be used such that no reset occurs during optimization.
  The default value for this parameter is 999999.
  
### ``LU complete pivoting``
  This refers to the LUSOL factorization within SNOPT.
  Through experimentation, it was determined that complete pivoting is the superior of the three options available, and should be always used.
  
### ``Function precision``
  This parameter describes the expected precision of the function evaluations, for both the objective and constraints.
  Line search is terminated when the difference between function values along the search direction becomes smaller than this value.



## How to interpret SNOPT results: the SNOPT_print.out file

### Exit Codes
SNOPT provides a large number of exit codes.
Here we list the ones commonly encountered in the lab, with helpful tips on debugging.
The exit codes come in a pair of numbers, the first one signifying the type of exit, and the second provides additional information.

#### 0/1
This is the dream.
Optimization finished and satisifed all prescribed tolerances.

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
