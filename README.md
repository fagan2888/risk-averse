# Marietta

MATLAB toolbox for risk-averse optimization

This is still work in progress.

NOTE: Add `matlab/` to your path to be able to use this toolbox.

## Create scenario trees

We may generate a scenario tree from an IID process
 
```matlab
probDist = [0.6 0.4];
ops.horizonLength = 15;
ops.branchingHorizon = 4;
ops.dimValue = 1;
tree = marietta.ScenarioTreeFactory.generateTreeFromIid(probDist, ops);
disp(tree);
plot(tree);
```

or from a Markov chain with a given initial distribution and given probability transition matrix

```matlab
initialDistr = [0.2; 0.1; 0.0; 0.7];
probTransitionMatrix = [
    0.7  0.2  0.0  0.1;
    0.25 0.6  0.05 0.1;
    0.4  0.1  0.5  0.0;
    0.0  0.0  0.3  0.7];
ops.horizonLength = 4;
tree = marietta.ScenarioTreeFactory.generateTreeFromMarkovChain(...
    probTransitionMatrix, initialDistr, ops);
disp(tree);
```

or from data

```matlab
options.ni = [2*ones(1,6) ones(1,10)];
tree = marietta.ScenarioTreeFactory.generateTreeFromData(data, options);
```


## Define data on nodes

Here's some very convenient functionality: we may assign a structure at each node of the tree.

This is particularly useful when we need to specify MJLS dynamics where every node `i` has a value `w_i` which is associated with matrices `A_i` and `B_i`.

All we need to do is to define what values `A_i`, `B_i` correspond to each `w_i`.

The values `w_i` are the "values" at each node of the tree, that is, the values of the underlying stochastic process (see `ScenarioTree.getValueOfNode`).

Here is an example for a scenario tree generated by a Markov chain: 

```matlab
tree.setValueAtNode(1, 1);
w_map = [1 2 3 4];
A = {[1 .2;.3 0.4], [1 .25;.3 0.9],[1 .3;.3 0.3],[1 .05;.4 0.3]};
B = {[1;0.05], [1;0], [1;-0.04], [1;0]};
n = numel(w_map);
data_w_map = repmat(struct(), n, 1);
for i=1:n
    data_w_map(i).A = A{i};
    data_w_map(i).B = B{i};
end
tree.mapValuesToData(w_map, data_w_map);
```

## Iterators

Iterators allow us to traverse certain nodes of the tree easily and efficiently, 
without having to write lots of for loops. 

This is how we can traverse all nodes of a tree at stage `k=3`

```matlab
iter = tree.getIteratorNodesAtStage(3)
while iter.hasNext
  nodeId = iter.next;
end
```

Likewise, to iterate over all non-leaf nodes, we may use the iterator
`iter = tree.getIteratorNonleafNodes`.

An iterator can be restarted using `iter.restart()`. 

For details checkout `help marietta.util.Iterator`.


## Construct risk measures

We may use `marietta.ConicRiskFactory` to construct risk measures.

For example, to create a `ConicRiskMeasure` object which corresponds to the average value-at-risk over a probability space with a given probability vector `p` at level `alpha`, use:

```matlab
avar = marietta.ConicRiskFactory.createAvar(p, alpha);
```

We may then use this object to compute the risk of a random variable `Z` using `avar.risk(Z)`.

Here is a complete example in which we compute the average value-at-risk and entropic value-at-risk of a random variable `Z` at different levels `alpha`

```matlab
n = 10; Z = exp(linspace(0,5,n))';
p = exp(-0.5*(1:n))'; xd = [];
for alpha = [0.001:0.01:0.1 0.2:0.1:0.9 0.94:0.02:1.0]
    avar = marietta.ConicRiskFactory.createAvar(p, alpha);
    evar = marietta.ConicRiskFactory.createEvar(p, alpha);
    avarZ = avar.risk(Z); evarZ = evar.risk(Z);
    xd = [xd; alpha avarZ evarZ];
end

figure;
plot(xd(:,1), xd(:,2),'-o','linewidth', 2); hold on;
plot(xd(:,1), xd(:,3),'-x','linewidth', 2);
legend('AVaR', 'EVaR'); grid on;
xlabel('alpha'); ylabel('risk')
```

## Parametric risk measures

Risk measures are defined relative to a probability space

Often they are parametrized with additional parameters `alpha` (e.g., the average value-at-risk and the entropic value-at-risk have such a scalar parameter).

A parametric risk measure is one which depends on the underlying distribution `p` and a parameter `alpha`

In `marietta`, parametric risks are functions which follow the template `@(p, alpha) parametricRisk`

The factory class `ParametricRiskMeasure` can be used to generate such parametric risks

## Solve risk-averse optimal control problems

This is a step-by-step guide to constructing and solving a risk-averse 
optimal control problem.

First, let us construct a scenario tree using the example file `makeRiskExample`
and let us choose a risk measure from `marietta.ParametricRiskFactory`...

```matlab
alpha = 0.5; 
N = 5; 
[tree, umin, umax, QN] = makeRiskExample(N);
pAvar = marietta.ParametricRiskFactory.createParametricAvarAlpha(alpha);
```

Let us now construct a risk averse optimizer (using the Builder pattern)...

```matlab
rao = marietta.RiskAverseOptimalController();
rao.setInputBounds(umin,umax)...
    .setScenarioTree(tree)...
    .setParametricRiskCost(pAvar)...
    .setTerminalCostMatrix(QN);
rao.makeController();
```

We can now use the above controller/optimizer to solve a problem given an 
initial state `x0`...

```matlab
x0 = [-3;3.5];
solution = rao.control(x0);
```

We may now plot the solution using

```matlab
figure;
subplot(121); solution.plotStateCoordinate(1);
subplot(122); solution.plotStateCoordinate(2);
```

this produces...

![State vs time](./matlab/design/state.png)

## Documentation

In MATLAB, type

```matlab
doc marietta
```

to access the documentation of this toolbox.

## UML of marietta

![UML](./matlab/design/uml.png)