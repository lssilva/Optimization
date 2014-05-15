# OVERVIEW

This project aims at producing a user-friendly optimization algorithm, inspired from existing Search-Based Algorithms (SBA) like Genetic Algorithms (GA), Hill Climbing (HC) and others. The outcome is expected to be an algorithm which relies on common concepts in GA, like mutations, combinations, exploration vs. convergence, and so on, but behave like a HC, focusing on convergence and exploring only when it appears as more interesting to do so. A particular focus is to make it user-friendly, by avoiding parameters which relates more to the algorithm than the problem to optimize, and to make it efficient by exploiting what have been learned from current techniques.

# TECHNIQUE

## Exploration VS Mutation

When we look for a solution, we have basically two ways to do so:

- **exploring**, meaning finding an area in the solution space in which there could have good solutions ;
- **converging**, meaning improving a solution at hand.

Basically, we could interpret the exploration as a global-level search (high coverage but low precision) and the convergence as a local-level search (low coverage but high precision).

In GA, one has to consider both of these goals together when he designs the *mutation* and *cross-over* operators. It makes sense because both have to be considered in order to search efficiently: the convergence ensures that we find the best we can and the exploration ensures that we don't get stuck in a local optimum. However, it is an optimization problem by itself to tune these operators, which appears to us as a dead end. What is the point to design an optimization algorithm if you need an optimization algorithm to make it works?

In our case, we consider that two goals lead to two separated management. Thus we consider an operator for each of them, which are represented by the interfaces **`Explorator`** and **`Mutator`**, which are both extending the interface `Generator` which provides the `generates()` method to generate new solutions. None of these operators have to care about the quality of the solutions they generate (or only superficially), in the sense that there objective is to generate "another solution". The evaluation of the solutions is the job of the `Evaluator` operator described later. Concretely, an explorator receives the current set of solutions and should generate a new solution to explore a new area in the search space. A mutator receives a single solution and should generate a new solution which could be better. Both have to implement also a `isApplicableOn()` method to explicit the specific conditions in which they can be used (e.g. a minimal number of solutions for an explorator combining existing solutions, or a specific type of solution for a specialized mutator).

In order to remains generic, no specific encoding is assumed for the solutions, but the explorators and mutators have to be consistent with an operator implementing the **`Evaluator`** interface. This operator aims at evaluating the value of each solution, so that we can infer which one is better. In the case where you are not able to have a simple valuation function, you can use a `Comparator` to provide a relative value. Currently, the assumption is that we aim at minimizing the value, thus the evaluator should provide a lower value for a better solution or the comparator should tell that a better solution is inferior to a worse solution.

## The Optimization Process

Basically, a mutator aims at improving a solution by providing a variant for it, called *mutant*. On the opposite, an explorator aims at providing a solution which represents an area of the solution space that we have not explored yet. Going ahead with this perspective, one can consider that, if a mutator does not provide a better solution, it is due to a lack of luck and another try could do the trick. On the other hand, the explorator provides a basis to improve by successive mutations.

This idea is represented by an **`Optimizer`**, which aims at finding a single good solution. When a new solution is generated by an explorator, a new optimizer is created and initialized with this solution. Then, a mutator is used to generate a new solution. If the mutant is strictly better, it replaces the old solution, otherwise the old one is kept. In both cases, the worst solution is forgotten.

*Why we keep it only if it is strictly better?*

As the aim of an optimizer is too improve its solution, if the mutant is of equal quality, there is no reason to make the effort of changing it. Going further, it can be a burden to change it when it is equal, because one of the two is forgotten and thus we don't remember it when we see it another time, while keeping always the same allows to exploit interesting properties to evaluate the optimality of the current solution.

*Why the worst solution is forgotten?*

Because if all the solutions generated are kept in memory (in a way or another), this algorithm could not fit for huge data structures or long computation. Actually, a bit more than the current solution is stored for other purposes, but the information stored is kept minimal.

## Local Optimality Evaluation

A working optimization algorithm is able to find optimum (or solutions close to be optimum). At best global ones, but one cannot assess a global optimality without having a formal proof or an exhaustive check over the entire solution space. However, on can assess the local optimality of a solution by ensuring that no other solution close to this one is better.

Considering that we use the mutators to find better solutions, these operators are the ones which shape the solution space. More formally, each mutator defines a topology on the solution space such that, given the current solution, a mutator selects another solution from a set of neighbors in the solution space. By browsing all the neighbors, we can certify that the current solution is a local optimum for the given mutator if no better solution has been found.

However, each mutator is defined depending on the knowledge at hand about the problem to optimize, independently to this algorithm. Thus, we do not have control on the neighboring defined by a given mutator and cannot make any assumption on it. In particular, the size of the neighboring is completely unknown at the start of the algorithm and nothing says that a neighbor will not be given several times by the same mutator for the same current solution (e.g. if some random processes are involved to generate mutations).

Nevertheless, a solution is considered as locally optimal if the mutator is not able to provide a better solution. This inability to provide better solutions can be evaluated in two ways:

- the mutator cannot be applied to the current solution (`isApplicableOn()` returns `false`)
- the mutator ability to provide new solutions decreases (the amount of redundant solutions increases)

Thus, we evaluate the probability for a solution to be optimal by considering these two properties. If the mutator cannot be applied, the probability for the current solution to be optimal is 1 (100%). Otherwise, we count the loops made by the mutator, meaning the number of times the mutator generates the same mutants for the same solution. In order to minimize the amount of information stored and maximize the robustness of this evaluation, we have made this trade-off: we cache the next mutant of the current solution ; if this mutant is generated another time, we count it as a loop and forget it to consider another mutant for the next loop. The implemented process is the following:

```
mutant = mutator.generates(solution)
if cache == null
	cache = mutant
else if mutant == cache
	loop = loop + 1
	cache = null
endif
```

Finally, the probability `p` to be locally optimal after some loops have occurred (at least 1) is computed by:

> `p = 1 - 1 / loops`

*Why forget the mutant and not consider always the same?*

Because if the mutator have some kind of regularity, it could be that we get a mutant which is not representative of the size of the neigboring (e.g. if some mutants are generated more frequently than others). By considering a different mutant for each loop, we can be representative of the neighboring size in average.

*Is this information reused if the same solution is regenerated later?*

The short answer is no. The long answer needs to consider two cases. In the first case, the same optimizer regenerates the same solution. It can be with the same mutator or a different one. With the same mutator, if it is from the same current solution, we are in the situation described above, and it is probably not the target of the question (yes it is reused, because it is the purpose). If it is from a different solution, nothing ensure that the statistics are preserved as it is a different set of neighbors, thus a different analysis need to be done and the information is cleared. It is the same thing with a different mutator: the topology changes and, consequently, no assumption can be made on the similarity of the statistics. A second case is when another optimizer (multiple optimizers are described in the next section) generates the same mutant, from the same current solution and with the same mutator. Even in this case the information is not reused while the topology is *probably* exactly the same and we are browsing *probably* exactly the same set of neighbors. "probably" because nothing says that there is no hidden variable influencing the mutator's topology. This last remark could be done also in our case, corrupting the optimality evaluation, but we assume that it is a good evaluation in general.

## Parallel Optimization

While a single optimizer care about the improvement of a single solution, the convergence to a local optimum is ensured. However, by considering a single solution, one cannot deal with exploration. Moreover, if we generate a new solution via an explorator, it is prone to not be better than the optimized solution (which has been finely improved with successive mutations). Thus, an explorator outcome has no reason to be used in an improvement step.

The aim of the explorator is to allow the browsing of a new area of the solution space. To do such a thing, a new optimizer is intantiated with the explorator's solution as a starting point, and then improved by successive mutations. Finally, each time an explorator is used to generate a new solution, an optimizer is created. All these optimizers are managed by an **`OptimizerPool`**, which contains the optimizers, provide the competition operators to decide whether a solution is better or not (i.e. the optimizer should consider the new solution or not) based on the evaluator/comparator provided.

This pool act as a container and does not provide any policy regarding how to manage the optimizers. It is then used by an `Incubator` which decides when new optimizers need to be added, which optimizer to improve, etc. This is described in the next section.

## The Incubator

In order to manage the optimization process, an **`Incubator`** is used to store the set of solutions currently managed and improve them. concretely, the incubator store an optimizer pool and, by calling its `incubate()` method, decides which optimizer to use and which explorator/mutator to apply on it.

Concretely, an incubation round is divided in 2 phases:

1. list all the applicable configurations, which means all the explorators which can be applied on the current set of solutions + all the pairs (mutator, optimizer) such as the mutator is applicable on the current solution of the optimizer ;
2. select one configuration based on its probability to provide a better solution and apply it, for a (mutator, optimizer) pair it is the complement (1-p) of the probability to be a local optimum, while for an explorator it is the product of all the probabilities to be a local optimum, ensuring that we do not reach a significant value unless all the solutions appear as probable optima.

This process allows to focus first on the operations which have the best chance to improve the current situation, namely the mutators (unless they are applied on local optima). In other words, when a new, better solution is found for a given optimizer, it will have a high chance to be considered, and after a while loops will occur, increasing its probability to be considered as optimal and giving a chance to the others to be executed as well. Rather than considering only the highest probability, we choose to use these probabilities as a probability distribution to still give a little chance to other operators to be executed. This way, an operator which consumes a lot of resources before to reach a significant probability to be optimal does not block other optimizers which could still be improved easily.

The incubator also manage size constraints such that:

- if the minimal size is not reached, we focus on explorators to add solutions ;
- if the maximal size is reached, new explorators will be used after the least interesting opimizer has been removed.

The least interesting optimizer is selected by considering the optimizer providing the worst local optimum. Because we are dealing with probabilities, we cannot consider strict optimality, thus we have to find a trade-off between the highest probability to be optimal and the worst value. In particular, a recent optimizer is prone to be the worst while the best solution is prone to be the one we have spent the most time on (thus the most probable local optimum). To avoid removing these extreme cases, we consider for future implementations to use techniques such as Pareto fronts. However, the current implementation is a naive selection of the worst optimizer: we rely on the focus on convergence to consider that all the solutions are probable optima when we add a new optimizer (which is true in average, but not everytime and not with the same confidence), thus we consider them equally by removing simply the worst of them.

# HOW TO USE

## Import in Your Project

The main code is provided by the *core* subproject. If you use Maven, you can add this dependency:

```xml
<dependency>
	<groupId>fr.matthieu-vergne</groupId>
	<artifactId>optimization-core</artifactId>
	<version>1.0</version>
</dependency>
```

Please check the last [release tag](https://github.com/matthieu-vergne/Optimization/releases) for the last version.

## Implement Your Own Problem

You can see some examples in the *sample* subproject. If you want to make your own, you can follow these guidelines:

1. Choose an encoding for your solutions, paying attention to its correctness\* ;
2. Implement a `Comparator` or an `Evaluator` (as a minimization problem) depending on whether you are able to provide a precise value for each individual (e.g. cost function) or only to make a relative comparison (e.g. between A and B, A is better) ;
3. Suppose that someone gives you one solution, try to identify which kind of modifications could improve it and implement a `Mutator` for each of them ;
4. Suppose that you already have several locally optimal solutions, each corresponding to a specific area in the solution space, try to identify which kind of modifications could lead you to a new area not yet explored and implement an `Explorator` for each of them ;
5. If all your explorators need to have at least 1 solution in the population (this should be reflected in their `isApplicableOn()` method), implement an `Explorator` which generates random solutions (pay attention to its correctness\*) or don't forget to add one or several initial solutions in the incubator before to run it ;
6. Instantiate the `ExperimentalIncubator` (or extend it and instantiate your own version) ;
7. Add the explorators and mutators you have implemented ;
8. Add the initial solutions to start from (if any) ;
9. Run the `incubate()` method of the incubator until your stopping condition is met (e.g. timeout, number of iterations, threshold).

\* When you choose your encoding, it could be that specific configurations are technically possible but semantically impossible. For instance, for the typical travelling salesman problem we want a list of N locations to follow in the order. If your encoding is a list of locations, with each index corresponding to one of the N available locations, you could have a "technically correct" solution which has the same location at different indexes (and miss other locations). It is not a "semantically correct" solution because it does not fit the requirement "pass through each location exactly one time". If you choose an encoding where each index corresponds to one of the remaining locations after we have removed the one already used, you don't have this problem anymore (but you could have others). However, such "non robust" encoding is not necessarily a problem, in the sense that you only need to ensure that:

- your explorators provide only semantically correct solutions
- your mutators preserve this correctness (if one receives a correct solution, it returns a correct solution)

Having an encoding which ensure correctness is one way, but it could make it harder to generate solutions. Having a more flexible encoding could make the problem and solutions easier to implement. Also, you can always deal with the incorrect solutions by ensuring that, when you have one, its evaluation is always worse than for a correct solution (e.g. increase heavily the cost each time you find an inconsistency). Yet it can make the search slower by adding useless computation. This is a practical trade-off to consider.