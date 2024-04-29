

<center><h1>Abstract</h1></center>

Datalog has proven to be a powerful language for encoding program analyses, but its connection to program transformations remains underexplored. This paper investigates how ideas from database repair, data provenance, and symbolic execution can bridge the gap between using Datalog for program analysis and leveraging it for program transformations.
% 
We survey existing approaches and present an upcoming perspective on the symbolic execution of Datalog rules, which simultaneously enables the representation of a space of potential input database (program) changes and computing their effect on the analysis results.
% 
We posit that bridging Datalog's strengths for analysis with an ability to reason about and execute transformations would significantly expand its utility for program optimization, repair, and other transformations. 



# Introduction

The use of deductive databases and logic programming languages, such as Datalog [^datalog], for program analysis is far from new [^reps1995demand, ^whaley2004cloning, ^lam2005context]. The declarativeness of Datalog makes it attractive for specifying complex analyses [^souffle, ^madmax]. The ability to specify recursive definitions is particularly exciting, as program analysis is fundamentally a mixture of mutually recursive tasks (see Figure: analyses-deps).

A Datalog query is a set of Horn clauses such as `path(X,Y) :- edge(X,Z), path(Z,Y).` executed against a database of facts referred to as the extensional database (EDB). The result is a set of derived facts, referred to as the intensional database (IDB).

Naturally, a program to be analyzed is represented as an EDB; e.g., control-flow graph (CFG) as a set of facts. When a query is executed on this EDB, an optimized Datalog engine computes all facts that can be inferred using the query's rules. A big advantage of this setup is the multitude of optimizations from the classical database systems that come for free to make these analyses efficient [^doop].


While connecting program analysis to databases (via Datalog) has proved useful, an open question is whether *program transformations similarly translate to useful aspects of databases*.

Program transformations involve changes in a program to improve an objective. Reasoning about these changes is necessary in many practical applications. For example, while a program analysis can determine if a program's behavior violates a certain property, a valid repair (fix, exception handler, etc.) ensures a program that meets the desired property [^sadowski2018lessons]. Consequently, reflecting on what transformations lead to a desired property is of great interest.


# Existing Connections

Starting from a Datalog view of programs represented as an EDB, we naturally arrive at program transformations being transformations to the corresponding facts in the database. Consequently, we can view the challenges of reflecting on these database transformations modulo a desired property in two phases. First, is the *reflection* aspect that answers what is the origin or derivation of an inferred fact? Second, the *transformation* aspect answers how a change to the input affects the truthfulness of the desired property.

There have been various approaches to support both these aspects in traditional database and Datalog frameworks, some of which we explore next.

### Database Repair and Integrity Constraints

One well-studied desired property in traditional database applications is **Consistency**. Arenas et al. (1999) [^arenas1999consistent] define a database instance $r$ as *consistent* if $r$ satisfies a set of *integrity constraints* $IC$. They then logically characterize "consistent query answers in inconsistent databases". The intuitive interpretation here is that an answer to a query posed to a database that violates integrity constraints should be the same as that obtained from a minimally **repaired** version of the original database.

```markdown
| **Student** |
|-------------|
| S1 N1 D1    |
| S1 N2 D1    |

| **Course**  |
|-------------|
| S1 C1 G1    |
| S1 C2 G2    |
```

**Example:** Consider a student database. `Student(x, y, z)` means that `x` is the student number, `y` is the student's name, and `z` is the address. The following Integrity Constraints (ICs) state that the first argument is the relation's key:

$$
\forall(x,y,z,u,v)(Student(x,y,z) \land Student(x,u,v) \supset y = u),
$$
$$
\forall(x,y,z,u,v)(Student(x,y,z) \land Student(x,u,v) \supset z = v).
$$

The inconsistent database instance `r` (shown) has two repairs, each removing one of the tuples in `Student`. Considering all repairs, for a query $\exists{z} Course(S1, y, z)$, we obtain `C1` and `C2` as the consistent answers. However, for $\exists{(u,v)} (Student(u, N1, v) \land Course(u, x, y))$ we obtain no (consistent) answers.


This is interesting since a *repaired* database $r'$ was transformed to satisfy the desired integrity constraints $IC$. Of course, there could be many repairs for an inconsistent database.

Arenas et al. (1999) [^arenas1999consistent] hence propose a solution to retrieve consistent answers that use only the original database (despite inconsistency).

The idea is to syntactically transform a query $Q$, reinforcing residues of ICs locally, to a query $Q'$. Evaluating $Q'$ on the original database returns the set of consistent answers to the query $Q$. This avoids explicitly computing the repairs. Here's an example:

**Example:** Consider the integrity constraint $\forall{x} (\neg P(x) \lor Q(x))$. If $Q(x)$ is false, then $\neg P(x)$ must be true. So, when querying $\neg Q(x)$, we make sure to generate the query $$\neg Q(x) \land \text{\colorbox{gray}{$\neg P(x)$}}$$ where the highlighted part is the residue added.


**Challenges:** There are challenges in extending this directly to Datalog and program transformations. Mainly, queries and constraints here are limited to a fragment of first-order logic and rewritten into a new query in the same language. For instance, completeness is lost when it is applied to disjunctive or existential queries.

More work [^arenas2003answer, ^bertossi2011database] along these lines have attempted to extend writing repair strategies in logical languages like Datalog, which are more expressive than FO logic.
However, scalability issues exist when writing and applying these to large rulesets, as in static analysis.

A bigger challenge in (directly) connecting this to Datalog for program transformation is that these approaches circumvent computing repairs entirely. This is of little applicability when the goal is to find a transformation.


### Data Provenance and Debugging Datalog

While connections to database repair are apparent, another way to view transformations modulo a desired property is to look into **Data Provenance** for debugging why a property holds.

Despite the numerous advantages, the declarative semantics of Datalog poses a debugging challenge. Logic specifications lack the notion of state and state transitions. After evaluation, we can only view relations in their entirety without explaining the origin or derivation of data.

A standard solution for these explanations is a *proof tree* [^datalog]. A proof tree for a tuple describes the derivation of that tuple from input tuples and rules.

A valid proof tree (all nodes hold) can explain an unexpected tuple when debugging. A failed proof tree provides insight into why a tuple is not produced. Figure [here](#dummy-link) shows an example of a proof tree for an alias analysis.


**Challenges:** However, these state-of-the-art techniques do not scale to large program analysis problems. Firstly, unlike top-down evaluation, scalable bottom-up Datalog evaluation does not have a notion of proof trees. Consequently, techniques propose rewriting the Datalog specification with provenance information [^deutch2015selective, ^kohler2012declarative, ^lee2017efficiently]. Here, a common issue is the need for re-evaluation when debugging, which can be expensive for industrial-scale static analysis problems (think something as precise as **Doop** [^doop], which may take multiple days for medium-to-large Java programs). Another major challenge is infinitely many proof trees and their conciseness—brute force search is infeasible, and storing proof trees during evaluation is memory intensive.


Recently, Zhao et al. (2020) [^zhao2020debugging] proposed storing *proof annotations* alongside tuples that include the height of the minimal proof tree and the rule that generated the tuple. Further, they extend the standard subset lattice of bottom-up evaluation as follows:

1. Define a *provenance instance* $(I, h)$ as an instance of tuples $I$ along with a function $h$ that provides a height annotation for each tuple in the instance.
2. Define a *provenance lattice* as one that follows the ordering:
    $$
    (I_1,h_1) \sqsubseteq (I_2,h_2) \Longleftrightarrow I_1 \subseteq I_2 ~\text{and}~ \forall{t \in I_1}: h_1(t) \ge h_2(t).
    $$
3. Define the join of instances $(I, h)$ and $(I', h')$ as $(I \cup I', h'')$ where $h''$ is the minimum of $h$ and $h'$ for each tuple in the join.

Since bottom-up evaluation is equivalent to applying a monotone function to move "up" a lattice, this guarantees the minimality of these height annotations. During debugging, one can reconstruct one level of the minimal proof for a tuple $t$ using a top-down search. While we omit details here, the example below illustrates it:

**Example:** Say $alias(a,b)$ is the tuple of interest. From bottom-up evaluation, the height annotation is $h(alias(a,b))=4$, and the generating rule is $alias(X,Y) :- vpt(X,o), vpt(Y,o)$. Now, we can search for tuples for the body of this rule such that $X=a$, $Y=b$, $h(vpt(a,o))<4$, and $h(vpt(b,o))<4$. Finding these tuples forms a one-level proof tree.


Zhao et al. (2020) [^zhao2020debugging] find that this adds minimal overhead to Datalog evaluation while enabling debugging, even in **Doop**-like analysis generating millions of output tuples. This approach could take us one step further from program analysis to transformations with declarative approaches. Granularly reflecting on why we inferred a fact is particularly useful; it hints at potential transformation (repair) strategies.

Notably, this has been integrated into state-of-the-art Datalog engines like Soufflé [^souffleProvenance]. However, an aspect that remains unanswered is executing these transformations and viewing their effects.



### Incrementality and Datalog

The incrementality of Datalog engines allows for analysis on the fly, on-demand, and online analysis. Interactive applications using this benefit could provide insights into how to reason EDB transformations.

**Example:** As an example, program analysis frameworks like **Doop** [^doop] evaluate a set of rules defined over the abstract syntax tree of the program. Integrating this into an IDE requires re-evaluating the rules after every few keystrokes (note, an ongoing transformation). The evaluation must preserve intermediate results efficiently to achieve interactive performance.

Datalog dialects such as Differential Datalog (DDLog) [^ddlog] have been proposed to make writing such applications easier. A DDlog programmer writes traditional, non-incremental Datalog programs. However, DDlog's execution model is fully incremental: at runtime, DDlog programs receive streams of changes to the input relations and produce streams of corresponding changes to derived relations.

This capability significantly benefits tools like Language Server Protocols (LSP) servers and compilers, providing developers with real-time feedback and suggestions. However, this solves only part of the problem, where one can reason about the effect of changes. What remains is the part where we *reflect* on what change would provide the desired effect.

Elastic incrementalization [^elastic-incremental][^zhao2023automatic] extends this by incorporating provenance annotations for input adjustments in incremental Datalog. They switch between a low-overhead Bootstrap strategy that targets high-impact updates and an Update strategy that targets low-impact updates.


# References

[^datalog]: Citation for Datalog.
[^reps1995demand]: Citation for Reps et al., 1995.
[^whaley2004cloning]: Citation for Whaley et al., 2004.
[^lam2005context]: Citation for Lam et al., 2005.
[^souffle]: Citation for Souffle.
[^madmax]: Citation for MadMax.
[^doop]: Citation for DOOP.
[^sadowski2018lessons]: Citation for Sadowski et al., 2018.
[^arenas1999consistent]: Arenas, M., Bertossi, L., & Chomicki, J. (1999). Consistent Query Answers in Inconsistent Databases.
[^arenas2003answer]: Arenas, M., Bertossi, L., & Chomicki, J. (2003). Answer Sets for Consistent Query Answers. 
[^bertossi2011database]: Bertossi, L. (2011). Database Repairing and Consistent Query Answering.
[^deutch2015selective]: Deutch, D., et al. (2015). Selective Provenance for Datalog Programs.
[^kohler2012declarative]: Kohler, S., et al. (2012). Declarative Datalog Debugging for Mere Mortals.
[^lee2017efficiently]: Lee, J., et al. (2017). Efficiently Debugging Datalog with Provenance.
[^doop]: Bravenboer, M., and Smaragdakis, Y. (2009). Doop: Declarative Object-Oriented Programming.
[^zhao2020debugging]: Zhao, Y., et al. (2020). Debugging with Provenance Annotations.
[^souffleProvenance]: [Soufflé Provenance](https://souffle-lang.github.io/provenance)
[^ddlog]: Differential Datalog (DDLog). 
[^elastic-incremental]: Elastic Incrementalization.
[^zhao2023automatic]: Zhao, Y., et al. (2023). Automatic Elastic Incrementalization for Datalog.



