
# Optimizing Fantasy Premier League Player Selection

## Introduction to Fantasy Premier League
Fantasy Premier League (FPL) is a popular online soccer fantasy game hosted by the English Premier League (EPL). In this game, users submit a team of 15 real-life players and collect points based on how they perform during the season. The goal of the game is to compete with others for the most points at the end of the season. 

Selecting players for FPL can be formulated into an optimization problem. The user is interested in maximizing total points under several constraints. The position constraints are as follows: exactly two goalkeepers selected, exactly five defenders selected, exactly five midfielders selected, and exactly three attackers selected. There is also a total budget constraint of 100 currency, and a constraint that no more than three players can be chosen from an existing team.

## Setting up the Problem
Through Kaggle Datasets, we are able to retrieve all the information we need. We use a naïve approach of determining each player's predicted points by copying their points from the previous season. This is one area of this project that can be improved on, possibly with the use of machine learning, to get a more holistic prediction. Regardless, we have the data, as shown by the example below.

| Name      | Cost | Role  | Team | PredictedPoints |
|-----------|------|-------|------|-----------------|
| John Doe  | 5.5  | FWD   | MCI  | 234             |
| Mary Jane | 7.0  | MID   | WOL  | 304             |

The problem is formulated as follows. We start by defining `x` as a binary vector such that `x_j` represents player at index `j`, where `x_j ∈ {0, 1}, ∀j ∈ [0, length(x))`. 

With this data, we can define the following problem:

$$max \sum_{j=0}^{m} \text{PredictedPoints}_j * x_j$$

Under the following constraints:


$$\sum_{j=0}^{m} \text{cost}_j * x_j \leq 100$$

$$\sum_{i \in \text{GKP}} x_i = 2$$

$$\sum_{i \in \text{DEF}} x_i = 5$$

$$\sum_{i \in \text{MID}} x_i = 5$$

$$\sum_{i \in \text{FWD}} x_i = 3$$

$$\sum_{i \in \text{T}_j} x_i \leq 3 \quad \forall \text{ teams } T_j$$

## Concerns
We would like to solve this problem with convex optimization techniques, but it is a binary linear program and thus non-convex. This is because we have constrained ourselves to optimizing a vector whose inputs can only be 0 or 1. We will proceed in two separate ways. The first is to use a library that supports the Coin-OR Branch and Cut solver to directly solve this problem. The other solution is going to be to relax the integer constraint into a continuous range so that each entry in the vector `x_j ∈ [0,1]`. This then means we can use typical linear programming solvers such as the simplex method to solve this problem.

## Branch and Cut Solution
The PuLP Library supports the Branch and Cut solver and is the library we will use to solve this problem. Before we set up the problem using this library, let us understand how the Branch and Cut solver works.

The algorithm starts, as we would expect, by relaxing the integer constraint into a continuous range. In our case, that would be relaxing the binary constraint to the range [0,1]. After this initialization, the solver uses a typical linear programming algorithm to solve the problem. This is ironically exactly what we planned on doing with the simplex method solution. Unlike our solution using the simplex method, we do not stop here. We perform a check if our found solution has integer values. If it does, the algorithm is finished. 

If this is not the case, we must have at least one entry in which our variable does not take on an integer value. We will choose exactly one of these variables and perform a branch. If we consider the example of an entry with value 3.8 while the entry is restricted between [0,5], we will create two distinct nodes to add to a search tree. One node will have this variable restricted to the range [0,3], a lower bound, while the other node will have the variable restricted to the range [4,5], an upper bound. In our specific case, we have an easy situation where, since every entry is in the range [0,1], if the variable does not take an integer value, we can simply branch in the case where the variable is 0 and in the case it is 1.

The solver continues by taking nodes off of this search tree and attempting to prune the tree. Pruning the tree is important in the time and space complexity of this algorithm. It is not important to understand how the solver prunes nodes off the tree, but simply that it does. The above steps are then repeated until the algorithm finds an optimal solution.

## Simplex Method Solution
The simplex method is an algorithm used for solving linear programming problems with linear equality and inequality constraints. At a high level, we can think of this algorithm as traveling along the edges of the feasible region to explore the vertices. By nature of the problem, the solution must lie on a vertex of the convex polytope that defines the feasible region. In 3 dimensions, we can imagine the feasible region as a ball-like shape made up of many flat sides with the optimal solution lying in one of the corners connecting these sides.

This algorithm works fantastically for our purposes as it considers both equality constraints and inequality constraints. The inequality constraints are converted into equality constraints by adding a slack variable, basically adding the extra amount needed to turn the inequality into an equality. This means if we have a constraint in the form $`a_1 x_1 + a_2 x_2 ≤ b`$, we can rewrite the inequality constraint as $`a_1 x_1 + a_2 x_2 + s = b`$, where `s` is the slack variable, turning it into an equality constraint.

Turning all our constraints into equality constraints allows us to conveniently write our constraints in the form `Ax = b`, where `A` is a coefficient matrix, `x` is a vector of variables, and `b` is a coefficient vector.

The algorithm is initialized on a single vertex, and then evaluates all adjacent vertices and finds the one that improves the objective function the most. Traveling to this vertex is called pivoting. The algorithm keeps pivoting until the objective function can no longer be improved, in which case the algorithm terminates with the current vertex being the solution.

This process can be done by hand for smaller-scale problems, but in our case with the optimized variable being 500+ entries long and adding all the slack variables, it would be extremely difficult to deal with the size of the long coefficient matrix. Libraries like PuLP are also capable of handling problems with the simplex method and are the library used in the code.

