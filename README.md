---
title: Model Checking Intermediate Language
author: Cesare Tinelli
date: 2022-06-27
---

# Model Checking Intermediate Language (Draft)

The Model Checking Intermediate Language (IL, for short) is an intermediate language
meant to be a common input and output standard for model checker
for finite- and infinite-state systems,
with an initial focus is on finite-state ones.

## General Design Philosophy

IL has been designed to be general enough to be an intermediate target language
for a variety of user-facing specification languages for model checking.
At the same time, IL is meant to be simple enough to be easily compilable
to lower level languages or be directly supported by model checking tools based
on SAT/SMT technology.

For being an intermediate language, models expressed in IL are meant
to be produced and processed by tools, hence it was designed to have

* simple, easily parsable syntax;
* a rich set of data types;
* minimal syntactic sugar, at least initially;
* well-understood formal semantics;
* a small but comprehensive set of commands;
* simple translations to lower level modeling languages such as [Btor2](https://link.springer.com/chapter/10.1007/978-3-319-96145-3_32).

Based on these principles, IL provides no direct support for many of the features
offered by current hardware modeling languages such as VHDL and Verilog
or more general purpose system modeling languages
such as SMV, TLA+, PROMELA, Simulink, SCADE, Lustre.
However, it strives to offer enough capability so that problems expressed
in those languages can be reduced to problems in IL.

IL is an extension the [SMT-LIB language](https://smtlib.cs.uiowa.edu/language.shtml)
with new commands to define and verify systems.
It allows the definition of multi-component synchronous or asynchronous reactive systems.
It also allows the specification and checking of reachability conditions
(or, indirectly state and transition invariants) and deadlocks,
possibly under fairness conditions on input values.

IL assumes a discrete and linear notion of time and hence has a standard trace-based semantics.

Each system definition:
* defines a _transition system_ via the use of SMT formulas,
  imposing minimal syntactic restrictions on those formulas;
* is parametrized by a _state signature_, a sequence of typed variables;
* partitions state variables into input, output and local variables;
* can be expressed as the synchronous or asynchronous composition of other systems
* is _hierarchical_, i.e., may include (instances of) previously defined systems as subsystems;
* can encode both synchronous and asynchronous system composition.

The current focus on _finite-state_ systems.
However, the language has been designed to support the specification
of infinite-state system as well.

## Technical Preliminaries

The base logic of IL is the same as that of SMT-LIB:
many-sorted first-order logic with equality and let binders.
We refer to this logic simply as FOL.
When we say _formula_, with no further qualifications, we refer to an arbitrary formula
of FOL (possibly with quantifiers and let binders).

We say that a formula is _quantifier-free_ if it contains no occurrences
of the quantifiers $\forall$ and
$\exists$.
We say that it is _binder-free_ if it is quantifier-free and also contains no occurrences
of the let binder.

The _scope_ of binders and the notion of _free_ and _bound_ (occurrences of) variables
in a formula are defined as usual.

#### Notation
If $F$ is a formula and
$\boldsymbol{x} = (x_1, \ldots, x_n)$ a tuple of distinct variables,
we write $F[\boldsymbol{x}]$ or $F[x_1, \ldots, x_n]$ to express the fact that every variable
in $\boldsymbol{x}$ is free in $F$ (although $F$ may have additional free variables).
We write $\boldsymbol{x} \circ \boldsymbol{y}$ to denote the concatenation of tuple
$\boldsymbol{x}$ with tuple $\boldsymbol{y}$.
When it is clear from the context, given a formula $F[\boldsymbol{x}]$ and
a tuple $\boldsymbol{t} = (t_1, \ldots, t_n)$ of terms of the same type
as $\boldsymbol{x} = (x_1, \ldots, x_n)$,
we write $F[\boldsymbol{t}]$ or $F[t_1, \ldots, t_n]$ to denote the formula obtained from $F$
by simultaneously replacing each occurrence of $x_i$ by $t_i$ for all $i=1,\ldots,n$.

A formula may contain _uninterpreted_ constant and function symbols,
that is, symbols with no constraints on their interpretation.
For most purposes, we treat uninterpreted constants as free variables and
treat uninterpreted function symbols as _second-order_ free variables.

### Transition systems

Formally, a transition system $S$ is a pair of predicates of the form

$$S = (
   I_S [\boldsymbol i, \boldsymbol o,  \boldsymbol s],~
   T_S [\boldsymbol i, \boldsymbol o, \boldsymbol s, \boldsymbol{i'}, \boldsymbol{o'}, \boldsymbol{s'}]
)$$

where

* $\boldsymbol i$ and $\boldsymbol{i'}$ are two tuples of _input variables_
  with the same length and type;
* $\boldsymbol o$ and $\boldsymbol{o'}$ are two tuples of _output variables_
  with the same length and type;
* $\boldsymbol s$ and $\boldsymbol{s'}$ are two tuples of _local variables_
  with the same length and type;
* $I_S$ is the system's _initial state condition_, expressed as a formula
  with no (free) variables from $\boldsymbol{i'},\boldsymbol{o'},\boldsymbol{s'}$;
* $T_S$ is the system's _transition condition_, expressed as a formula
  over the variables
  $\boldsymbol{i},\boldsymbol{o},\boldsymbol{s},\boldsymbol{i'},\boldsymbol{o'},\boldsymbol{s'}$.

>**Note:**
For convenience, but differently from other formalizations, a (full) state of system $S$
is expressed by a valuation of the variables $\boldsymbol{i},\boldsymbol{o},\boldsymbol{s}$.
In other words, input and output variables are automatically _stateful_ since the transition relation formula can access old values of inputs and outputs in addition to the old values
of the local state.
This means that, technically, $S$ is a closed system.
The designation of some state variables as input or output is, however, important
when combining systems together, to capture which of the state values are shared
between two systems being combined, and how.
>
> **Note:** Similarly to Mealy machines, the initial state condition is also meant to specify
the initial system's output, based on the initial state and input.
Correspondingly, the transition relation is also meant to specify the system's output
in every later state of the computation.
>
>**Note:** The input and output values corresponding to the transition to the _new_ state are
those in the variables $\boldsymbol{i'}$ and
$\boldsymbol{o'}$.
The values of $\boldsymbol{i}, \boldsymbol{o}$ are the _old_ input and output values.

### Trace Semantics

A transition system in the sense above implicitly defines a model
(i.e., a Kripke structure) of First-Order Linear Temporal Logic (FO-LTL).

The language of FO-LTL extends that of FOL with the same modal operators of time
as in standard (propositional) LTL:
$\mathbf{always},$
$\mathbf{eventually},$
$\mathbf{next},$
$\mathbf{until},$
$\mathbf{release}$.
For our purposes of defining the semantics of transition systems
it is enough to consider just the $\mathbf{always}$ and
$\mathbf{eventually}$ operators.

The set of non-temporal operators depends on the particular theory, in the sense of SMT,
considered (linear integer/real arithmetic, bit vectors, strings, and so on, and
their combinations).
The meaning of theory symbols (such as arithmetic operators) and theory sorts
(such as $\mathsf{Int}$,
$\mathsf{Real},$
$\mathsf{Array(Int, Real)},$
$\mathsf{BitVec(3)}$, $\ldots$)
is fixed by the theory $\mathcal T$ in question.
Once a theory $\mathcal T$ has been fixed then, the meaning of a FO-LTL formula $F$
is provided by an interpretation of the uninterpreted (constant and function) symbols
of $F$, if any,
as well as an infinite sequence of valuations for the free variables of $F$.

More precisely, let us fix a tuple $\boldsymbol x = (x_1,\ldots,x_n)$
of distinct _state_ variables, meant to represent the state of a computation system.
We will denote by $\boldsymbol{x'}$
the tuple $(x_1',\ldots,x_n')$
and write formulas of the form $F[\boldsymbol f, \boldsymbol x, \boldsymbol x']$
where $\boldsymbol f$ is a tuple of uninterpreted symbols.
If $F$ has free occurrences of variables from $\boldsymbol x$
but not from $\boldsymbol{x'}$ we call it a _one-state_ formula;
otherwise, we call it a _two-state_ formula.

A _valuation of_ $\boldsymbol x$,
or a _state over_ $\boldsymbol x$, is function mapping
each variable $x$ in $\boldsymbol x$
to a value of $x$'s type.
Let $\kappa$ be a positive ordinal smaller than
$\omega$, the cardinality of the natural numbers.
A _trace (of length_ $\kappa$_)_ is
any state sequence $\pi = (s_j  \mid 0 \leq j < \kappa)$.
Note that $\pi$ is the finite sequence $s_0, \ldots, s_{\kappa-1}$
when $\kappa < \omega$ and
is the infinite sequence $s_0, s_1, \ldots$ otherwise.
For all $i$ with $0 \leq i < \kappa$ ,
we denote by $\pi[i]$ the state $s_i$ and
by $\pi^i$ the subsequence
$(s_j \mid i \leq j < \kappa)$.

### Infinite-Trace Semantics

Let $F[\boldsymbol f, \boldsymbol x, \boldsymbol{x'}]$ be a formula as above.
If $\mathcal I$ is an interpretation
of $\boldsymbol{f}$
in the theory $\mathcal T$ and
$\pi$ is an infinite trace,
then $(\mathcal I, \pi)$ _satisfies_ $F$,
written $(\mathcal I, \pi) \models F$, iff

* $F$ is atomic and
  $\mathcal{I}[\boldsymbol x \mapsto \pi[0](\boldsymbol x),\boldsymbol{x'} \mapsto \pi[1](\boldsymbol x)]$
  satisfies $F$ as in FOL;
* $F = \lnot G~$ and
  $~(\mathcal I, \pi) \not\models G$;
* $F = G_1 \land G_2~$ and
  $~(\mathcal I, \pi) \models G_i$ for $i=1,2$;
* $F = \exists x\, G$ and
  $(\mathcal I[z \mapsto v], \pi) \models G$ for some value $v$ for $x$;
* $F = \mathbf{eventually}~G$ and
  $(\mathcal I, \pi^i) \models G$ for some $i \geq 0$;
* $F = \mathbf{always}~G$ and
  $(\mathcal I, \pi^i) \models G$ for all $i \geq 0$.
<!-- * $(\mathcal I, \pi^1) \models G$ when $F = \mathbf{next}~G$; -->

The semantics of the propositional connectives $\lor, \rightarrow, \leftrightarrow$
and the quantifier $\forall$
can be defined by reduction to the connectives above
(e.g., by defining $G_1 \lor G_2$ as
$\lnot(\lnot G_1 \land \lnot G_2)$ and so on).
Note that $\exists$ is a _static_, or _rigid_, quantifier:
the meaning of the variable it quantifies does not change over time,
that is, from state to state in $\pi$.
The same is true for uninterpreted symbols.
They are _rigid_ in the same sense: their meaning does not change over time.
Another way to understand the difference between rigid and non-rigid symbols is that
state variables are mutable over time
whereas quantified variables, theory symbols and uninterpreted symbols are all immutable.

Now let
$S = (
   I_S [\boldsymbol i, \boldsymbol o,  \boldsymbol s],~
   T_S [\boldsymbol i, \boldsymbol o, \boldsymbol s, \boldsymbol{i'}, \boldsymbol{o'}, \boldsymbol{s'}]
)$
be a transition system with state variables $\boldsymbol i, \boldsymbol o,  \boldsymbol s$.

The _infinite trace semantics_ of $S$ is the set of all pairs $(\mathcal I, \pi)$
of interpretations $\mathcal I$ in $\mathcal T$ and infinite traces $\pi$ such that

$$(\mathcal I, \pi) \models
  I_S \land\ \mathbf{always}~T_S
$$

We call any such pair an _execution_ of $S$.

> **Note:**
[We focus on reactive systems]


### Finite-Trace Semantics

Let $F[\boldsymbol f, \boldsymbol x, \boldsymbol{x'}]$, $\mathcal I$, $\mathcal T$ and $\pi$
be defined is in the subsection above.
For every $n \geq 0$, $(\mathcal I, \pi)$ _$n$-satisfies_ $F$,
written $(\mathcal I, \pi) \models_n F$, iff

* $F$ is atomic and
  $\mathcal I[\boldsymbol x \mapsto \pi[0](\boldsymbol x),
              \boldsymbol{x'} \mapsto \pi[1](\boldsymbol x)]$ satisfies $F$ as in FOL;

* $F = \lnot G$ and $(\mathcal I, \pi) \not\models_n G$

* $F = G_1 \land G_2$ and $(\mathcal I, \pi) \models_n G_i$ for $i=1,2$;

* $F = \exists x\, G$ and $(\mathcal I[z \mapsto v], \pi) \models_n G$ for some value $v$ for $x$;

* $F = \mathbf{eventually}~G$ and $(\mathcal I, \pi^i) \models_{n-i} G$ for some $i = 0, \ldots, n$;

* $F = \mathbf{always}~G$ and $(\mathcal I, \pi^i) \models_{n-i} G$ for all $i = 0, \ldots, n$;


The semantics of the propositional connectives $\lor, \rightarrow, \leftrightarrow$
and the quantifier $\forall$
is again defined by reduction to the connectives above.

Intuitively, $n$-satisfiability specifies when a formula is true over
the first $n$ states of a trace.
Note that this notion is well defined even when $n=0$ regardless of whether $F$
has free occurrences of variables from $\boldsymbol{x'}$ or not.
In the atomic case, this is true because $\pi$, for being an _infinite_ trace,
does contain the state $\pi[1]$.
In the general case, the claim can be shown by a simple inductive argument.

The notion of $n$-satisfiability is useful when one is interested, as we are,
in state reachability.
The reason is that a state satisfying a (non-temporal) state property $P$ is reachable
in a system $S$
if and only if the temporal formula $\mathbf{eventually}~P$ is $n$-satisfied
by an execution of $S$ for some $n$.

## Supported SMT-LIB commands

SMT-LIB is a command-based language with LISP-like syntax ([s-expressions](https://en.wikipedia.org/wiki/S-expression), in prefix notation) designed
to be a common input/output language for SMT solvers.

IL adopts the following SMT-LIB commands:

* <tt>(declare-sort $s$ $n$)</tt>

  Declares $s$ to be a sort symbol (i.e., type constructor) of arity $n$.
  Examples:

  ```scheme
  (declare-sort A 0)
  (declare-sort Set 1)
  ; possible sorts: A, (Set A), (Set (Set A)), (Array Int Real), ...
  ```

* <tt>(define-sort $s$ ($u_1 \cdots u_n$) $\tau$)</tt>

  Defines $S$ as synonym of a parametric type $\tau$ with parameters $u_1 \cdots u_n$.
  Examples:

  ```scheme
  (declare-sort NestedSet (X) (Set (Set X)))
  ; possible sorts: (NestedSet A), ...
  (declare-sort Array2 (X Y) (Array X (Array X Y)))
  ; possible sorts: (Array2 Int Bool), ...
  ```

* <tt>(declare-const $c$ $\sigma$)</tt>

  Declares a constant $c$ of sort $\sigma$.
  Examples:
  
  ```scheme
  (declare-const a A)
  (declare-const n Int)
  ```

* <tt>(define-fun $f$ (($x_1$ $\sigma_1$) $\cdots$ ($x_1$ $\sigma_1$)) $\sigma$ $t$)</tt>

  Defines a (non-recursive) function $f$ with inputs $x_1, \ldots, x_n$
  (of respective sort $\sigma_1, \ldots, \sigma_n$), output sort $\sigma$, and body $t$.
  Examples:

  ```scheme
  (declare-fun sq ((n Int)) Int (* n n))
  (declare-fun isSqRoot ((m Int) (n Int)) Bool (= n (sq m)))
  (declare-fun max ((m Int) (n Int)) Int (ite (> m n) m n))
  ```

* <tt>(set-logic $L$)</tt>

  Defines the model's _data logic_, that is, the background theories of relevant datatypes
  (e.g., integers, reals, bit vectors, and so on) as well as the language of allowed logical constraints
  (e.g., quantifier-free, linear, etc.).

  ```scheme
  (set-logic QF_BV)
  ```

One addition to SMT-LIB is the binary symbol <tt>!=</tt> for disequality.
For each term <tt>s</tt> and <tt>t</tt> of the same sort, 
<tt>(!= s t)</tt> has the same meaning as <tt>(not (= s t))</tt> 
or, equivalently, <tt>(distinct s t)</tt>.

## IL-specific commands

### Enumeration declaration

<tt>(declare-enum-sort $s$ ($c_1$ $\cdots$ $c_n$))</tt>

Declares $s$ to be an enumerative type with (distinct) values $c_1, \ldots, c_n$.

### System definition command

#### Atomic systems

An atomic transition system is defined by a command of the form:

<tt>
(define-system $S$ <br>
&nbsp; :input (($i_1$ $\sigma_1$) $\cdots$ ($i_m$ $\sigma_m$)) <br>
&nbsp; :output (($o_1$ $\tau_1$) $\cdots$ ($o_n$ $\tau_n$)) <br>
&nbsp; :local (($s_1$ $\sigma_1$) $\cdots$ ($s_p$ $\sigma_p$)) <br>
&nbsp; :init $I$<br>
&nbsp; :trans $T$<br>
&nbsp; :inv $P$<br>
)
</tt>

where

* $S$ is the system's identifier;
* each $i_j$ is an _input_ variable of sort $\delta_j$;
* each $o_j$ is an _output_ variable of sort $\tau_j$;
* each $s_j$ is a _local_ variable of sort $\sigma_j$;
* each $i_j$, $o_j$, $s_j$ denote _current-state_ values
* _next-state variables_ are not provided explicitly but are denoted
  by convention by appending $'$ to the names of the current-state variables $i_j$, $o_j$, and $s_j$;
* $I$ is a one-state formula over the unprimed system's variables (input, output and local state variables)
  that expresses a constraint on the initial states of $S$;
* $T$ is a two-state formula over all of the system's variables (primed and unprimed)
  that expresses a constraint on the state transitions of $S$;
* $P$ is a one state formula over all of the _unprimed_ system's variables
  that expresses a constraint on all reachable states of $S$;
* all attributes are optional and their order is immaterial except that
  <tt>:input</tt>, <tt>:output</tt>, and <tt>:local</tt> must occur before <tt>:init</tt>, <tt>:trans</tt>, and <tt>:inv</tt>;
* the default value for a missing attribute is the empty list <tt>()</tt> 
  for <tt>:input</tt>, <tt>:output</tt>, and <tt>:local</tt>, 
  and <tt>true</tt> for <tt>:init</tt>, <tt>:trans</tt>, and <tt>:inv</tt>.
  
  
Syntactically, the system identifier, the input, output and local variables are SMT-LIB symbols.
In contrast, the sorts $\delta_j$, $\tau_j$, $\sigma_j$ are SMT-LIB sorts,
while the formulas $I$, $T$ and $P$ are SMT-LIB terms of type <tt>Bool</tt>.

<!--
The various aspects of the system are provided as SMT-LIB attribute-value pairs.
The order of the attributes can be arbitrary but each attribute can occur at most once.
A missing attribute stands for a default value:
the empty list <tt>()</tt> for <tt>:input</tt>, <tt>:output</tt> and <tt>:local</tt>; and
<tt>true</tt> for <tt>:init</tt>, <tt>:trans</tt> and <tt>:inv</tt>.
-->

> **Discussion:**
> We could allow multiple occurrences of the :inv attribute with conjunctive semantics.
> Should we allow multiple occurrences of the :trans attribute but with _disjunctive_ semantics?
> The rationale would be to facilitate the recognition of systems defined by alternative sets of transitions.
>

#### Atomic System Semantics

Let
$\boldsymbol{i} = (i_1, \ldots, i_m)$,
$\boldsymbol{o} = (o_1, \ldots, o_n)$,
$\boldsymbol{s} = (s_1, \ldots, s_p)$,
and
$\boldsymbol{v}$ = $\boldsymbol{i},\boldsymbol{o},\boldsymbol{s}$.


Formally, an atomic system $S$ introduced by the <tt>define-system</tt> command above
is a transition system whose behavior consists of all the (infinite) executions $(\mathcal I, \pi)$
over $\boldsymbol{v}$ such that

$$(\mathcal I, \pi) \models
  \underbrace{I[\boldsymbol{v}] \land P[\boldsymbol{v}]}_{I_S} \land
  \mathbf{always}\ (\underbrace{T[\boldsymbol{v},\boldsymbol{v'}] \land P[\boldsymbol{v'}]}_{T_S})
$$

> **Note:**
The relation expressed by the formula $T$ is not required to be functional
over $\boldsymbol{i},\boldsymbol{o},\boldsymbol{s},\boldsymbol{i'}$,
thus allowing the modeling of non-deterministic systems.

> **Note:**
The <tt>:inv</tt> attribute is not strictly necessary since a system
with a  declaration of the form
> 
>  <tt>
>  (define-system $S$
>  :input (($i_1$ $\sigma_1$) $\cdots$ ($i_m$ $\sigma_m$))<br>
>  &nbsp; :output (($o_1$ $\tau_1$) $\cdots$ ($o_n$ $\tau_n$))<br>
>  &nbsp; :local (($s_1$ $\sigma_1$) $\cdots$ ($s_p$ $\sigma_p$))<br>
>  &nbsp; :init $I$
>   &nbsp; :trans $T$
>   &nbsp; :inv $P$<br>
>   )
>   </tt>
>
> can be equivalently expressed with a declaration of the form
>
>  <tt>
>  (define-system $S$
>  :input (($i_1$ $\sigma_1$) $\cdots$ ($i_m$ $\sigma_m$))<br>
>  &nbsp; :output (($o_1$ $\tau_1$) $\cdots$ ($o_n$ $\tau_n$))<br>
>  &nbsp; :local (($s_1$ $\sigma_1$) $\cdots$ ($s_p$ $\sigma_p$))<br>
>  &nbsp; :init (and $I$ $P$)
>  &nbsp; :trans (and $T$ $P'$)<br>
>  )
>  </tt>
>
>where $P'$ is the formula obtained from $P$ by priming all the system variables in $P$.

> **Note:**
Systems are meant to be progressive: every reachable state has a successor with respect $T_S$. However, they may not be because of the generality of $T$ and $P$.
In other words, it is possible to define deadlocking systems.
(See later for more details on deadlocked states.)

#### Examples

(Adapted from "Principles of Cyber-Physical Systems" by R. Alur, 2015)

When reading these examples, it is helpful to keep in mind that, intuitively, in the <tt>:init</tt> formulas the input values are given and the local and output values are to be defined with respect to them. In contrast, in the <tt>:trans</tt> formulas the new input values, and old input, output and local values are given, and the new local and output values are to be defined.

The output of system <tt>Delay</tt> below is initially <tt>0</tt> and
then is the previous input.
No local variables are needed.

```scheme
(define-system Delay :input ( (i Int) ) :output ( (o Int) )
 :init (= o 0)
 :trans (= o' i) ; the new output is the old input
)
````

A variant of <tt>Delay</tt> where the output is initially any number in [0,10].

```scheme
(define-system Delay :input ( (i Int) ) :output ( (o Int) )
 :init (<= 0 o 10) ; more than one possible initial output
 :trans (= o' i)
)
````

A clocked lossless channel, stuttering when the clock is not ticking.
The clock is represented by a Boolean input variable <tt>clock</tt>.

```scheme
(define-system StutteringClockedCopy
 :input ((clock Bool) (i Int))
 :output ((o Int))
 :init (=> clock (= o i)) ; o is arbitrary when clock is false 
 :trans (ite clock (= o’ i’) (= o’ o)) ; ite is if-then-else
)
```

Events carrying data can be modeled as instances
of the polymorphic algebraic datatype <tt>(Event X)</tt>
where <tt>X</tt> is the type of the data carried by the event.

```scheme
(declare-datatype Event (par (X)
  (absent)
  (present (val X))
))
````

An event-triggered channel that arbitrarily loses its input data.

```scheme
(define-system LossyIntChannel
 :input ((i (Event Int)))
 :output ((o (Event Int)))
 :inv (or (= o i) (= o absent))
)
````

Equivalent formulation using unconstrained local state.

```scheme
(define-system LossyIntChannel
 :input ((i (Event Int))) 
 :output ((o (Event Int)))
 :local ((s Bool))
 ; at all times, whether the input event is transmitted 
 ; or not depends on value of s
 :inv (= o (ite s i absent))
)
````

<tt>TimedSwitch</tt> models a timed light switch where the light stays
on for at most 10 steps unless it is switched off before.
The input Boolean is interpreted as an on/off toggle.
The transition predicate is formulated as a set of transitions.

```scheme
(define-enum-sort LightStatus (on off))

(define-system TimedSwitch1
 :input ( (press Bool) )
 :output ( (sig Bool) )
 :local ( (s LightStatus) (n Int) )
 :inv (= sig (= s on))
 :init (and
   (= n 0)
   (ite press (= s on) (= s off))
 )
 :trans (let
  (; transitions
   (stay-off (and (= s off) (not press') (= s' off) (= n' n)))
   (turn-on  (and (= s off) press' (= s' on) (= n' n)))
   (stay-on  (and (= s on) (not press') (< n 10) (= s' on) (= n' (+ n 1))))
   (turn-off (and (= s on) (or press' (>= n 10)) (= s' off) (= n' 0)))
  )
  (or stay-off turn-on turn-off stay-on)
 )
)
```

A variant of the system above where the transition predicate is formulated
guarded-transitions style.

```scheme
(define-system TimedSwitch2
 :input ( (press Bool) )
 :output ( (sig Bool) )
 :local ( (s LightStatus) (n Int) )
 :inv (= sig (= s on))
 :init (and
    (= n 0)
    (ite press (= s on) (s off))
  )
 :trans (and
   (=> (and (= s off) (not press'))
       (and (= s' off) (= n' n)))            ; off --> off
   (=> (and (= s off) press')
       (and (= s' on) (= n' n)))             ; off --> on
   (=> (and (= s on) (not press') (< 10 n))
        (and (= s' on) (= n' (+ n 1))))      ; on --> on
   (=> (and (= s on) (or press' (>= n 10)))
       (and (= s' off) (= n' 0)))            ; on --> off
  )
)
```

Another variant but in equational style.

```scheme
(define-fun flip ((s LightStatus)) LightStatus
  (ite (= s off) on off)
)

(define-system TimedSwitch3
 :input ( (press Bool) )
 :output ( (sig Bool) )
 :local ( (s LightStatus) (n Int) )
 :inv (= sig (= s on))
 :init (and
   (= n 0)
   (= s (ite press on off))
 )
 :trans (and
   (= s' (ite press' (flip s)
            (ite (or (= s off) (>= n 10)) off
              on)))
   (= n' (ite (or (= s off) (s' off)) 0
            (+ n 1)))
  )
)
```

The non-deterministic arbiter below grants input requests expressed
by the Boolean inputs <tt>r1</tt> and <tt>r2</tt>.
Initially, no requests are granted. Afterwards, a request is always granted,
expressed by the Boolean outputs <tt>g1</tt> or <tt>grant2</tt>,
if it is the only request.
When both inputs contain a request, one of the two request is granted,
with a non-deterministic  choice.

```scheme
(define-system NonDetArbiter
 :input ( (r1 Bool) (r2 Bool) )
 :output ( (g1 Bool) (g2 Bool) )
 :local ( (s Bool) )
 :init ( (not g1) (not g2) )  ; nothing is granted initially
 :trans (and
  (=> (and (not r1') (not r2'))
      (and (not g1') (not g2')))
  (=> (and r1' (not r2'))
      (and g1' (not g2')))
  (=> (and (not r1') r2')
      (and (not g1') g2'))
  (=> (and r1' r2')
      ; the unconstrained value of `s` is used as non-deterministic choice
      (ite s' (and g1' (not g2'))
        (and (not g1') g2')))
  )
)
```

The next arbiter is similar to <tt>NonDetArbiter</tt> but grants requests a cycle later and does not use a local variable for the non-deterministic choice.

```scheme
(define-system DelayedArbiter
 :input ( (r1 Bool) (r2 Bool) )
 :output ( (g1 Bool) (g2 Bool) )
 :local ( (s Bool) )
 :init ( (not g1) (not g2) )  ; nothing is granted initially
 :trans (
    (=> (and (not r1) (not r2))
        (and (not g1') (not g2')))
    (=> (and r1 (not r2))
        (and g1' (not g2')))
    (=> (and (not r1) r2)
        (and (not g1') g2'))
    (=> (and r1 r2)
        (!= g1' g2'))
  )
)
```

Similar to <tt>NonDetArbiter</tt> but for requests expressed as integer events.

```scheme
(define-system IntNonDetArbiter
  :input ( (r1 (Event Int)) (r2 (Event Int)) )
  :output ( (g1 (Event Int)) (g2 (Event Int)) )
  :local ( (s Bool) )
  :init ( (= g1 g2 none) )
  :trans (
    (=> (= r1' r2' none)
        (= g1' g2' none))
    (=> (and (!= r1' none) (= r2' none))
        (and (= g1' r1) (= g2' none)))
    (=> (and (= r1' none) (!= r2' none))
        (and (= g1' none) (= g2' r2')))
    (=> (and (!= r1' none) (!= r2' none))
        (or (and (= g1' r1') (= g2' none))
          (and (= g1' none) (= g2' r2'))))
  )
)
```

<!--
; An event-triggered channel that arbitrarily loses its input data
(define-system LossyIntChannel
  :input ( (in (Event Int)) )
  :output ( (out (Event Int)) )
  :local ( (s Bool) )
  ; whether the input event is transmitted depends on s
  ; s is unconstrained so can take any value during execution
  :inv ( (= out (ite s in none)) )   
)

; A system that simply passes along the current input `in` when the `clock` input is true.
; When `clock` is false it outputs the value output the last time `clock` was true.
; Until `clock` is true for the first time it outputs the initial value of `init`.
(define-system StutteringClockedCopy
  :input ( (clock Bool) (init Int) (in Int) )
  :output ( (out Int) )
  :init ( (ite clock (= out s in) (= out s init)) )
  :trans ( (ite clock (= out' s' in) (= out' s' s)) )
)
-->

#### Composite Systems  - synchronous composition

A transition systems can be defined as the synchronous composition
of other systems by a command of the form:

<tt>
(define-system $S$ <br>
&nbsp; :input (($i_1$ $\sigma_1$) $\cdots$ ($i_m$ $\sigma_m$)) <br>
&nbsp; :output (($o_1$ $\tau_1$) $\cdots$ ($o_n$ $\tau_n$)) <br>
&nbsp; :local (($s_1$ $\sigma_1$) $\cdots$ ($s_p$ $\sigma_p$)) <br>
&nbsp; :subsys ($N_1$ ($S_1$ $\boldsymbol x_1$ $\boldsymbol y_1$)) <br>
&nbsp;&nbsp; $\cdots$<br>
&nbsp; :subsys ($N_q$ ($S_q$ $\boldsymbol x_q$ $\boldsymbol y_q$)) <br>
&nbsp; :init $I$<br>
&nbsp; :trans $T$<br>
&nbsp; :inv $P$<br>
)
</tt>

where

* <tt>:input</tt>, <tt>:output</tt>, <tt>:local</tt> <tt>:init</tt>, <tt>:trans</tt>, and <tt>:inv</tt> 
  are as in atomic system definitions;
* $q > 0$ and each $S_i$ is the name of a system other than $S$;
* the names $S_1 \ldots,S_q$ need not be all distinct;
* each $N_i$ is a local synonym for $S_i$, with $N_1,\ldots,N_q$ distinct;
* each $\boldsymbol x_i$ consists of $S$'s variables of the same type as $S_i$'s input;
* each $\boldsymbol y_i$ consists of $S$'s local/output variables of the same type as $S_i$' ’'s output;
* the directed subsystem graph rooted at $S$ is acyclic.

#### Composite System Semantics

For $k=1,\ldots, q$, let
$S_k = (I_k[\boldsymbol{i}_k,\boldsymbol{o}_k,\boldsymbol{s}_k],
        T_k[\boldsymbol{i}_k,\boldsymbol{o}_k,\boldsymbol{s}_k,
            \boldsymbol{i'}_k,\boldsymbol{o'}_k,\boldsymbol{s'}_k])$,
with the elements of $\boldsymbol{s}_1,\ldots, \boldsymbol{s}_q$ all mutually distinct.

Let
$\boldsymbol{i} = (i_1, \ldots, i_m)$,
$\boldsymbol{o} = (o_1, \ldots, o_n)$,
$\boldsymbol{s} = (s_1, \ldots, s_p) \circ \boldsymbol{s}_1 \circ \cdots \circ \boldsymbol{s}_q$,
and
$\boldsymbol{v}$ = $\boldsymbol{i} \circ \boldsymbol{o} \circ \boldsymbol{s}$.

Formally, a composit system $S$ introduced by the <tt>define-system</tt> command above
is a transition system whose behavior consists of all the (infinite) executions $(\mathcal I, \pi)$
over $\boldsymbol{v}$ such that

$$(\mathcal I, \pi) \models
  I_S[\boldsymbol{v}] \land \mathbf{always}\ T_S[\boldsymbol{v},\boldsymbol{v'}]
$$
where

* $I_S[\boldsymbol{v}] =
   I[\boldsymbol{v}] \land
   \bigwedge_{k=1}^{q} I_k[\boldsymbol{x}_k,\boldsymbol{y}_k,\boldsymbol{s}_k]
  $<br>
and
* $T_S[\boldsymbol{v},\boldsymbol{v'}] = 
   P[\boldsymbol{v}] \land T[\boldsymbol{v},\boldsymbol{v'}] \land
   \bigwedge_{k=1}^{q} T_k[\boldsymbol{x}_k,\boldsymbol{y}_k,\boldsymbol{s}_k,
                           \boldsymbol{x'}_k,\boldsymbol{y'}_k,\boldsymbol{s'}_k]
  $

#### Examples, composite systems

```scheme
;----------------
; Two-step delay
;----------------

;     +------------------------------------+
;     |   DoubleDelay                      |
;     |  +-----------+      +-----------+  |
;  in |  |           | temp |           |  | out
; ----+->|   Delay   |----->|   Delay   |--+---->
;     |  |           |      |           |  |
;     |  +-----------+      +-----------+  |
;     +------------------------------------+

; One-step delay
(define-system Delay :input ( (i Int) ) :output ( (o Int) )
  :local ( (s Int) )
  :inv (= s i) 
  :init (= o 0)
  :trans (= o' s)
)

; Two-step delay
(define-system DoubleDelay :input ( (in Int) ) :output ( (out Int) )
  :local ( (temp Int) )
  :subsys  (D1 (Delay in temp))
  :subsys  (D2 (Delay temp out))
)

;; DoubleDelay expanded
(define-system DoubleDelay
  :input ( (in Int) )
  :output ( (out Int) )
  :local ( 
    (temp Int)  
    (s1 Int)      ; from `(Delay in temp)`
    (s2 Int)      ; from `(Delay temp out)`
  )
  :inv (and
    (= s1 in)     ; from `(Delay in temp)`
    (= s2 temp)   ; from `(Delay temp out)`
  )
  :init (and
    (= temp 0)   ; from `(Delay in temp)`
    (= out 0)    ; from  `(Delay temp out)`
  )
  :trans (and
    (= temp' s1) ; from `(Delay in temp)`
    (= out' s2)  ; from `(Delay temp out)`
  )
)
; Example trace:
;   in = 1, 2, 3, 4, 5, 6, 7, ...
;   s1 = 1, 2, 3, 4, 5, 6, 7, ...
; temp = 0, 1, 2, 3, 4, 5, 6, ...
;   s2 = 0, 1, 2, 3, 4, 5, 6, ...
;  out = 0, 0, 1, 2, 3, 4, 5, ...
````

The next example defines a three-bit counter in terms of three identical one-bit counters.
The one-bit counter uses a latch.
The latch has a Boolean state represented by state variable <tt>s</tt> with arbitrary initial value.
The value of output <tt>out</tt> is always the previous value of <tt>s</tt>.
A set request (represented by input <tt>set</tt> being true) sets the new value of <tt>s</tt> to true
unless there is a concurrent reset request (represented by input <tt>reset</tt> being true).
In that case, the choice between the two requests is resolved arbitrarily using the value of
the unconstrained local variable <tt>b</tt>.
In the absence of either a set or a reset, the value of <tt>out</tt> is unchanged <tt>out</tt> .


````scheme
(define-system Latch  :input ( (set Bool) (reset Bool) )  :output ( (out Bool)) 
 :local ( (s Bool) (b Bool) )
 :init (and
   (= out b)
 )
 :trans (and
   (= out' s)
   (= s' (or (and set (or (not reset) b)) 
             (and (not set) (not reset) out)))
 )
)
````

The one-bit counter is implemented using the latch component modeled by `Latch`.
The counter goes from 0 (represented as <tt>false</tt>) to 1 (<tt>true</tt>)
with a carry value of 0, or from 1 to 0 with a carry value of 1 when
the increment signal <tt>inc</tt> is true.
It is reset to 0 (<tt>false</tt>) when the start signal is true.
The initial value of the counter is arbitrary.

````scheme
;        +------------------------------------------------------------+
;        |                                                            |
;        | +--------------------------------------------------------+ |
;        | |                                              +-------+ | |
;        +-|-----------------------------------|``-.  set |       | | |
;        | |                          |`-._    |    :---->|       | | |
;        | +->|``-.                +--|   _]o--|..-`      | Latch | | |
;        |    |    :--+----\``-.   |  |.-`          reset |       |-+-+--> out
;   inc -+----|..-`   |     )   :--+--------------------->|       |   |
;        |            | +--/..-`                          +-------+   |
;        |            | |                                             |
; start ----------------+                OneBitCounter                |
;        |            |                                               |
;        +------------+-----------------------------------------------+
;                     |    
;                     v carry

(define-system OneBitCounter :input ( (inc Bool) (start Bool) ) 
 :output ( (out Bool) (carry Bool) )
 :local ( (set Bool) (reset Bool) )
 :subsys (L (Latch set reset out))
 :inv (and 
   (= set (and inc (not reset)))
   (= reset (or carry start))
   (= carry (and inc out))
 )
)
````

The three-bit counter is a resettable counter obtained by cascading
three 1-bit counters.
The output is three Boolean values standing for the three bits,
with <tt>out0</tt> being the least significant one.

````scheme
(define-system ThreeBitCounter  :input ( (inc Bool) (start Bool) )
 :output ( (out0 Bool) (out1 Bool) (out2 Bool) ) 
 :local ( (car0 Bool) (car1 Bool) (car2 Bool) ) 
 :subsys (C1 (OneBitCounter inc start out0 car0))
 :subsys (C2 (OneBitCounter car0 start out1 car1)) 
 :subsys (C3 (OneBitCounter car1 start out2 car2))
)
````


#### Sanity requirements on _I<sub>S</sub>_ and _T<sub>S</sub>_

One way to address the deadlock state problem is to consider only infinite traces in the semantics (systems that are meant to reach a final state can be always modeled with states that cycle back to themselves and produce stuttering outputs).
In that semantics, the reachability of a deadlock state (a state with no successors in the transition relation) indicates an error in the system's definition.

Intuitively, for a system definition to define a system with no deadlocks:

1. Any assignment of values to the input variables can be extended to a total assignment (to all the unprimed variables) that satisfies _I<sub>S</sub>_.

2. For any reachable state $S$, any assignment to the primed input variables can be extended to a total assignment _s'_ so that _s,s'_ satisfies _T<sub>S</sub>_.


The first restriction above guarantees that the system can start at all. The second ensures that from any reachable state and for any new input the system can move to another state (_and so_ also produce output).

* A sufficient condition for (1) is that the following formula is valid in the (previously specified) background theory:  _∀ **i** ∃ **o** ∃ **s** I<sub>S</sub>_

* A sufficient condition for (2) is that the following formula is valid in the background theory: _∀ **i** ∀ **o** ∀ **s** ∀ **i'** ∃ **o'** ∃ **s'** T<sub>S</sub>_

  This condition is not necessary, however, since it need not apply to unreachable states. Let _Reachable[**i**, **o**, **s**]_ denote the (possibly higher-order) formula satisfied exactly by the reachable states of $S$. Then, a more accurate sufficient condition for (2) above would be the validity of the formula:

  _∀ **i** ∀ **o** ∀ **s** ∀ **i'** ∃ **o'** ∃ **s'** (Reachable[**i**,**o**,**s**] ⇒ T<sub>S</sub>)_

**Note:** In general, checking the two sufficient conditions above automatically can be impossible or very expensive (because of the quantifier alternations in the conditions).










### System verification command

<tt>(check-system $S$</tt>
<tt>&nbsp; :input ((_i<sub>1</sub> δ<sub>1</sub>_)  $\cdots$ (_i<sub>m</sub> δ<sub>m</sub>_))</tt>
<tt>&nbsp; :output ((_o<sub>1</sub> τ<sub>1</sub>_) $\cdots$ (_o<sub>n</sub> τ<sub>n</sub>_))</tt>
<tt>&nbsp; :local ((_s<sub>1</sub> σ<sub>1</sub>_) $\cdots$ (_s<sub>p</sub> σ<sub>p</sub>_))</tt>
<tt>&nbsp; :assumptions (_a<sub>1</sub> $\cdots$ a<sub>q</sub>_)</tt>
<tt>&nbsp; :fairness (_f<sub>1</sub> $\cdots$ f<sub>h</sub>_)</tt>
<tt>&nbsp; :reachable (_r<sub>1</sub> $\cdots$ r<sub>i</sub>_)</tt>
<tt>&nbsp; :invariants (_p<sub>1</sub> $\cdots$ p<sub>k</sub>_)</tt>
<tt>)</tt>

<!-- <tt>&nbsp; :conjectures (_c<sub>1</sub> $\cdots$ c<sub>j</sub>_)</tt> -->


where

* $S$ is the identifier of a previously defined system with _m_ inputs, _n_ outputs, and _p_ local variables;
* each _i<sub>j</sub>_ is a renaming of the corresponding input variable of $S$ of sort _δ<sub>i</sub>_;
* each _o<sub>j</sub>_ is a renaming of the corresponding output variable of $S$ of sort _τ<sub>i</sub>_;
* each _s<sub>j</sub>_ is a renaming of the corresponding local variable of $S$ of sort _σ<sub>i</sub>_;
* each _a<sub>j</sub>_ is a triple of the form <tt>(_n<sub>j</sub> l<sub>j</sub> A<sub>j</sub>_)</tt> 
  with <tt>_A<sub>j</sub>_</tt> a formula over input and local variables
  (called _environmental assumption_);
* each _f<sub>j</sub>_ is a triple of the form <tt>(_n<sub>j</sub> l<sub>j</sub> F<sub>j</sub>_)</tt>
  with <tt>_F<sub>j</sub>_</tt> a formula over input, output and local variables
  (called _fairness condition_);
* each _r<sub>j</sub>_ is a triple of the form <tt>(_n<sub>j</sub> l<sub>j</sub> R<sub>j</sub>_)</tt>
  where <tt>_R<sub>j</sub>_</tt> is a formula over input, output and local variables
  (called _reachability condition_);
* each _p<sub>j</sub>_ is a triple of the form <tt>(_n<sub>j</sub> l<sub>j</sub> P<sub>j</sub>_)</tt>
  where <tt>_P<sub>j</sub>_</tt> is a formula over input, output and local variables
  (called _invariance condition_);
* each _n<sub>j</sub>_ is fresh identifier
* each _l<sub>j</sub>_ is a string literal
* a formula is an (quantifier-free?) FOL formula over primed and unprimed variables
<!-- * each _c<sub>j</sub>_ is a triple of the form <tt>(_n<sub>j</sub> l<sub>j</sub> C<sub>j</sub>_)</tt> -->
<!-- * each _C<sub>j</sub>_ is a formula over input, output and local variables; -->

**Note:** The ids _n<sub>j</sub>_ are meant to be user-defined names for the corresponding condition. The strings _n<sub>j</sub>_ are labels that can be attached for convenience, especially when producing output.


Let _I<sub>S</sub>_ and _T<sub>S</sub>_ be respectively the initial state condition and transition predicate of $S$.

Let _A = A<sub>1</sub> \land \cdots \land A<sub>q</sub>_, _F = F<sub>1</sub> \land \cdots \land F<sub>h</sub>_, _R = R<sub>1</sub> \land \cdots \land R<sub>i</sub>_, _P = P<sub>1</sub> \land \cdots \land P<sub>k</sub>_.

The command above succeeds if both the following holds:

1. there is a trace of $S$ that satisfies all the environmental assumptions and fairness conditions, and reaches a state that satisfies all the reachability conditions; that is, if the following LTL formula is satisfiable (in the chosen background theory):

   _I<sub>S</sub> ∧ **always** T<sub>S</sub> ∧ **always** A ∧ **always eventually** F ∧ **eventually** R_

 2. every property _P<sub>j</sub>_ is invariant for $S$ under the environmental assumptions and the fairness conditions; that is, if the following LTL formula is valid (in the chosen background theory):

    _I<sub>S</sub> ∧ **always** T<sub>S</sub> ∧ **always** A ∧ **always eventually** F ⇒ **always** P_


<!-- Each conjecture _C<sub>j</sub>_ is a _possible_ auxiliary invariant of $S$, that is, _(I<sub>S</sub> ∧ **always** T<sub>S</sub>) ∧ **always** (A<sub>1</sub> \land \cdots \land A<sub>n</sub>) ⇒ **always** C<sub>i</sub>)_ is possibly valid. If it is indeed invariant, it may be used to help prove the properties _P_ invariant. -->



#### Examples

```scheme
;---------
; Arbiter
;---------

(define-system NonDetArbiter
  :input ( (r1 Bool) (r2 Bool) )
  :output ( (g1 Bool) (g2 Bool) )
  :local ( (s Bool) )
  :init ( (not g1) (not g2) )  ; nothing is granted initially
  :trans (
    (=> (and (not r1') (not r2'))
        (and (not g1') (not g2')))
    (=> (and r1' (not r2'))
        (and 'g1 (not g2')))
    (=> (and (not r1') r2')
        (and (not g1') g2'))
    (=> (and r1' r2')
        ; the unconstrained value of `s` is used as non-deterministic choice
        (ite s'
          (and g1' (not g2')
          (and (not g1') g2'))))
  )
)

(verify-system NonDetArbiter
  :input ( (r1 Bool) (r2 Bool) )
  :output ( (g1 Bool) (g2 Bool) )
  :properties (
    (p1 "Every request is immediately granted" ; not invariant
      (and (=> r1 g1) (=> r2 g2))
    )
    (p2 "In the absence of other requests, every request is immediately granted" ; invariant
      (=> (distinct r1 r2)
          (and (=> r1 g1) (=> r2 g2)))
    )
    (p3 "A request is granted only if present" ; invariant
      (and (=> g1 r1) (=> g2 r2))
    )
    (p4 "At most one request is granted at any one time" ; invariant
      (not (and g1 g2))
    )
    (p5 "In case of concurrent requests one of them is always granted" ; invariant
      (=> (and r1 r2) (or g1 g2))
    )
    (p6 "If there have been no requests so far then there have been no grants" ; invariant
      (=> (historically (and (not r1) (not r2)))
          (historically (and (not g1) (not g2))))
    )
  )
)

(verify-system NonDetArbiter
  :input ( (r1 Bool) (r2 Bool) )
  :output ( (g1 Bool) (g2 Bool) )
  :assumptions (
    (a1 "There are no concurrent requests"
      (not (and r1 r2))
    )
  )
  :properties (
    (p1 "Every request is immediately granted" ; invariant
      (and (=> r1 g1) (=> r2 g2))
    )
  )
)

;---------------
; 3-bit counter
;---------------

(define-fun toBit ((b Bool)) Int (ite b 1 0))

(define-fun toInt ( (b2 Bool) (b1 Bool) (b0 Bool) ) Bool
  (+ (* 4 (toBit b2)) (* 2 (toBit b1)) (toBit b0))
)

(verify-system ThreeBitCounter
  :input ( (inc Bool) (start Bool) )
  :output ( (o0 Bool) (o1 Bool) (o2 Bool) )
  :local ( (c0 Bool) (c1 Bool) (c2 Bool) )
  :properties (
    (p1 "Sanity-check invariant"
      (<= 0 (toInt o2 o1 o0) 7)
    )
    (p2 "A start signal resets the counter to 0 in the next"
      (=> (before start)
          (= 0 (toInt o2 o1 o0)))
    )
    (p2A "Alternative formulation of p2 as a transition invariant"
      (=> start
          (= 0 (toInt o2' o1' o0')))
    )
    (p3 "If no increment requests are ever sent, the counter stays at 0"
      (=> (historically (not inc))
          (= (toInt o2 o1 o0) 0))
    )
    (p4 "If there is an increment request and the counter is below 7
         then it will increase by 1 next"
      (let ( (n (toInt o2 o1 o0)) )
        (=> (and inc (< n 7))
            (= (toInt o2' o1' o0') (+ n 1))))
    )
  )
)

(define-system DelayedCounter
  :input ( (inc Bool) (start Bool) )
  :output ( (n Int) )
  :local ( (c Int) )
  :init (
    (= n 0)
    (= c (ite (and inc (not start)) 1 0))
  )
  :trans (
    (= n' c)
    (= c' (ite start' 0
            (ite (not inc') c
              (ite (= c 7) 0 (+ c 1)))))
  )
)

(define-system CountObserver
  :input ( (inc Bool) (start Bool) )
  :output ( (n1 Int) (n2 Int) )
  :local ( (o0 Bool) (o1 Bool) (o2 Bool) )
  :init (
    (= n1 (count o2 o1 o0))
    (ThreeBitCounter inc start o0 o1 o2)
    (DelayedCounter inc start n2)
  )
  :trans (
    (= n1' (count o2' o1' o0'))
    (ThreeBitCounter inc start o0 o1 o2)
    (DelayedCounter inc start n2)
  )
)

; more concisely
(define-system CountObserver
  :input ( (inc Bool) (start Bool) )
  :output ( (n1 Int) (n2 Int) )
  :local ( (o0 Bool) (o1 Bool) (o2 Bool) )
  :inv ( (= n1 (count o2 o1 o0)) )
  :compose (
    (ThreeBitCounter inc start o0 o1 o2)
    (DelayedCounter inc start n2)
  )
)

(verify-system CountObserver
  :input ( (inc Bool) (start Bool) )
  :output ( (n1 Int) (n2 Int) )
  :properties (
    (p1 "" (= n1 n2))                                 ; not invariant
    (p2 "" (since start (= n1 n2)))                   ; not invariant
    (p3 "" (=> (once start) (since start (= n1 n2)))) ; invariant
  )
)
```

## Extensions

### Contracts

[more]

#### Examples

##### Thermostat Controller

```scheme
;            +-----------------------------------+
;            |  ThermostatController             |
;            |                                   |
;            |    +--------------------+         |
; --up----------->|                    |         |
;            |    |   SetDesiredTemp   |--+----------set_temp-->
; --down--------->|                    |  |      |
;            |    +--------------------+  |      |
;            |                            v      |
;            |    +--------------------------+   |
; -switch-------->|                          |-------cool-->
;            |    |       ControlTemp        |   |
; -out_temp------>|                          |-------heat-->
;            |    +--------------------------+   |
;            |                                   |
;            +-----------------------------------+

;-------------------
; Global parameters
;-------------------
;
(define-const MIN_TEMP Real 40.0)
(define-const MAX_TEMP Real 100.0)
(define-const INI_TEMP Real 70.0)
(define-const DEADBAND Real 3.0)
(define-const DIFF Real 3.0)

(declare-enum-sort SwitchPos (Cool Off Heat))

;----------------
; SetDesiredTemp
;----------------
;
(define-system SetDesiredTemp
  :input ( (up_button Bool) (down_button Bool) )
  :output ( (set_temp Real) )
  :auxiliary (
    (inc Bool
      :def (and up_button (<= set_temp (- MAX_TEMP DIFF))))
    (dec Bool
      :def (and down_button (>= set_temp (+ MIN_TEMP DIFF))))
  )
  :assumptions (
    (a1 "Up/Down button signals are mutually exclusive"
      (not (and up_button down_button))
    )
  )
  :guarantees (
    (g0 "Initial set_temp"
      (initially (= set_temp INIT_TEMP))
    )
    (g1 "set_temp increment"
      (=> inc (= set_temp' (+ set_temp DIFF)))
    )
    (g2 "set_temp decrement"
      (=> dec (= set_temp' (- set_temp DIFF)))
    )
    (g3 "set_temp invariance"
      (=> (not (or inc dec))
        (= set_temp' set_temp))
    )
  )
)

;-------------
; ControlTemp
;-------------
;
(define-system ControlTemp
  :input ( (switch SwitchPos) (current_temp Real) (set_temp Real) )
  :output ( (cool_act_sig Bool) (heat_act_sig Bool) )
  :auxiliary (
    (cool_start Bool
      :def (and (= switch Cool)
             (> current_temp (+ set_temp DEADBAND)))
    )
    (cool_mode Bool
      :init cool_start
      :next (or cool_start'
              (and cool_mode
                (= switch' Cool)
                (> current_temp' set_temp')))
    )
    (heat_start Bool
      :def (and (= switch Heat)
             (< current_temp (- set_temp DEADBAND)))
    )
    (heat_mode Bool
      :init heat_start
      :next (or heat_start'
              (and heat_mode
                (= switch' Heat)
                (< current_temp' set_temp')))
    )
    (off_mode Bool
      :def (and (not cool_mode) (not heat_mode))
    )
  )
  :guarantees (
    (g1 "Cooling activation"
      (= cool_act_sig cool_mode)
    )
    (g2 "Heating activation"
      (= heat_act_sig heat_mode)
    )
    (g3 "Heating and cooling mutually exclusive"
      (not (and heat_act_sig cool_act_sig))
    )
  )
)

;----------------------
; ThermostatController
;----------------------
;
(define-system ThermostatController
  :input ( (switch SwitchPo) (up Bool) (down Bool) (in_temp Real) )
  :output ( (cool Bool) (heat Bool) (set_tempReal) )
  :inv (
    (SetDesireTemp up down set_temp)
    (ControlTemp switch in_temp set_temp cool heat)
  )
  :assumptions (
    (a1 "Up/Down button signals are mutually exclusive"
      (not (and up_button down_button))
    ) 

  )
  :auxiliary (
    (in_deadzone Bool
      :def (<= (- set_temp DEADBAND) in_temp (+ set_temp DEADBAND))
    )
    (system_off Bool
      :def (and (not cool) (not heat))
    )

  )
  :guarantees (
    (g1 "Initial temperature is in range":
      (<= MIN_TEM INIT_TEMP MAX_TEMP)
    ) 
    (g2 "Deadband and Diff are positive values"
      (and (> DEADBAND 0.0) (> DIFF 0.0))
    )
    (g3 "No activation signal is enabled if switch is in Off"
      (=> (= switch Off) (and (not cool) (not heat)))
    )
    (g4 "Cooling system is turned On only if switch is in Cool"
      (=> cool (= switch Cool))
    )
    (g5 "Heating system is turned On only if switch is in Heat"
      (=> cool (= switch Heat))
    )
    (g6 "Activation signals are never enabled at the same time"
      (not (and cool heat))
    )
    (g7 "set_temp is always in range"
      (<= MIN_TEMP set_temp MAX_TEMP)
    )
    (g8 "set_temp doesn't change if no button is pressed"
      (=> (and (not up') (not down')
        (= set_temp' set_temp))
    )     
    (g9 "set_temp doesn't decrease if the up button is pressed"
      (=> up'
        (>= set_temp' set_temp))
    )
    (g10 "set_temp doesn't increase if the down button is pressed":
      (=> down'
        (<= set_temp' set_temp))
    )
    (g11 "System is Off if indoor temperature is in the dead zone and system was Off in the previous step"
      (=> (and in_deadzone' system_off) system_off')
    )
    (g12 "Cooling system is On only if indoor temperature is higher than set_temp":
      (=> cool (> in_temp set_temp))
    )
    (g13 "Heating system is On only if indoor temperature is lower than set_temp":
      (=> heat (< in_temp set_temp))
    )
    (g14 "Cooling system is On if switch is in Cool and the indoor temperature is higher than set_temp plus deadband"
      (=> (and (= switch Cool) (> in_temp (+ set_temp DEADBAND)))
        cool)
    )
    (g15 "Heating system is On if switch is in Heat and the indoor temperature is lower than set_temp plus deadband"
      (=> (and (= switch Heat) (> in_temp (+ set_temp DEADBAND)))
        cool)
    )
    (g16 "Once cooling system is On, it remains On as long as set_temp has not been reached and switch is in Cool"
      (=> (and (since cool (= switch Cool))
            (> in_temp set_temp))
        cool))
    )
    (g17 "Once heating system is On, it remains On as long as set_temp has not been reached and switch is in Heat"
      (=> (and (since heat (= switch Heat))
            (< in_temp set_temp))
        heat))
    )
  )
)
```

## Parameters

[parameters as rigid variables]
