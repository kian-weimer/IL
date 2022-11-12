---
title: Model Checking Intermediate Language
author: Cesare Tinelli
date: 2022-06-27
---

--------------------------------------------------------------------------------

# Model Checking Intermediate Language (Draft)

The Model Checking Intermediate Language (IL, for short) is an intermediate
language meant to be a common input and output standard for model checker
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
* simple translations to lower level modeling languages such as
  [Btor2](https://link.springer.com/chapter/10.1007/978-3-319-96145-3_32).

Based on these principles, IL provides no direct support for many of the features
offered by current hardware modeling languages such as VHDL and Verilog
or more general purpose system modeling languages
such as SMV, TLA+, PROMELA, Simulink, SCADE, Lustre.
However, it strives to offer enough capability so that problems expressed
in those languages can be reduced to problems in IL.

IL is an extension the 
[SMT-LIB language](https://smtlib.cs.uiowa.edu/language.shtml)
with new commands to define and verify systems.
It allows the definition of multi-component synchronous or
asynchronous reactive systems.
It also allows the specification and checking of reachability conditions
(or, indirectly state and transition invariants) and deadlocks,
possibly under fairness conditions on input values.

IL assumes a discrete and linear notion of time and hence has 
a standard trace-based semantics.

Each system definition:

* defines a _transition system_ via the use of SMT formulas,
  imposing minimal syntactic restrictions on those formulas;
* is parametrized by a _state signature_, a sequence of typed variables;
* partitions state variables into input, output and local variables;
* can be expressed as the synchronous or asynchronous composition of other systems
* is _hierarchical_, i.e., may include (instances of) previously defined systems
  as subsystems;
* can encode both synchronous and asynchronous system composition.

The current focus on _finite-state_ systems.
However, the language has been designed to support the specification
of infinite-state system as well.

## Technical Preliminaries

The base logic of IL is the same as that of SMT-LIB:
many-sorted first-order logic with equality and let binders.
We refer to this logic simply as FOL.
When we say _formula_, with no further qualifications, we refer
to an arbitrary formula of FOL (possibly with quantifiers and let binders).

We say that a formula is _quantifier-free_ if it contains no occurrences
of the quantifiers $\forall$ and
$\exists$.
We say that it is _binder-free_ if it is quantifier-free and also contains
no occurrences of the let binder.

The _scope_ of binders and the notion of _free_ and _bound_ (occurrences of)
variables in a formula are defined as usual.

### Notation

If $F$ is a formula and
$\boldsymbol{x} = (x_1, \ldots, x_n)$ a tuple of distinct variables,
we write $F[\boldsymbol{x}]$ or $F[x_1, \ldots, x_n]$ to express the fact
that every variable in $\boldsymbol{x}$ is free in $F$
(although $F$ may have additional free variables).
We write $\boldsymbol{x} \circ \boldsymbol{y}$ to denote the concatenation
of tuple $\boldsymbol{x}$ with tuple $\boldsymbol{y}$.
When it is clear from the context, given a formula $F[\boldsymbol{x}]$ and
a tuple $\boldsymbol{t} = (t_1, \ldots, t_n)$ of terms of the same type
as $\boldsymbol{x} = (x_1, \ldots, x_n)$,
we write $F[\boldsymbol{t}]$ or $F[t_1, \ldots, t_n]$ to denote the formula
obtained from $F$ by simultaneously replacing each occurrence of $x_i$
by $t_i$ for all $i=1,\ldots,n$.

A formula may contain _uninterpreted_ constant and function symbols,
that is, symbols with no constraints on their interpretation.
For most purposes, we treat uninterpreted constants as free variables and
treat uninterpreted function symbols as _second-order_ free variables.

### Transition systems

Formally, a transition system $S$ is a pair of predicates of the form

$$S = (
   I_S [\boldsymbol i, \boldsymbol o,  \boldsymbol s],~
   T_S [\boldsymbol i, \boldsymbol o, \boldsymbol s,
        \boldsymbol{i'}, \boldsymbol{o'}, \boldsymbol{s'}]
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
  $\boldsymbol{i},\boldsymbol{o},\boldsymbol{s},
   \boldsymbol{i'},\boldsymbol{o'},\boldsymbol{s'}$.

>**Note:**
For convenience, but differently from other formalizations, a (full) state
of system $S$is expressed by a valuation of the variables
$\boldsymbol{i},\boldsymbol{o},\boldsymbol{s}$.
In other words, input and output variables are automatically _stateful_
since the transition relation formula can access old values of inputs and outputs
in addition to the old values of the local state.
This means that, technically, $S$ is a closed system.
The designation of some state variables as input or output is, however, important
when combining systems together, to capture which of the state values are shared
between two systems being combined, and how.
>
> **Note:**
Similarly to Mealy machines, the initial state condition is also meant to specify
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

  Defines the model's _data logic_, that is, the background theories
  of relevant data types
  (e.g., integers, reals, bit vectors, and so on)
  as well as the language of allowed logical constraints
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
&nbsp; :input (($i_1$ $\delta_1$) $\cdots$ ($i_m$ $\delta_m$)) <br>
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
  by convention by appending $'$ to the names of the current-state variables 
  $i_j$, $o_j$, and $s_j$;
* $I$ is a one-state formula over the unprimed system's variables 
  (input, output and local state variables)
  that expresses a constraint on the initial states of $S$;
* $T$ is a two-state formula over all of the system's variables (primed and unprimed)
  that expresses a constraint on the state transitions of $S$;
* $P$ is a one state formula over all of the _unprimed_ system's variables
  that expresses a constraint on all reachable states of $S$;
* all attributes are optional and their order is immaterial except that
  <tt>:input</tt>, <tt>:output</tt>, and <tt>:local</tt> must occur before
  <tt>:init</tt>, <tt>:trans</tt>, and <tt>:inv</tt>;
* the default value for a missing attribute is the empty list <tt>()</tt>
  for <tt>:input</tt>, <tt>:output</tt>, and <tt>:local</tt>, 
  and <tt>true</tt> for <tt>:init</tt>, <tt>:trans</tt>, and <tt>:inv</tt>.
  
Syntactically, the system identifier, the input, output and local variables are 
SMT-LIB symbols.
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
We could allow multiple occurrences of the :inv attribute with conjunctive semantics.
Should we allow multiple occurrences of the :trans attribute but with _disjunctive_
semantics?
The rationale would be to facilitate the recognition of systems defined
by alternative sets of transitions.

#### Atomic System Semantics

Let
$\boldsymbol{i} = (i_1, \ldots, i_m)$,
$\boldsymbol{o} = (o_1, \ldots, o_n)$,
$\boldsymbol{s} = (s_1, \ldots, s_p)$,
and
$\boldsymbol{v}$ = $\boldsymbol{i},\boldsymbol{o},\boldsymbol{s}$.

Formally, an atomic system $S$ introduced by the <tt>define-system</tt> command above
is a transition system whose behavior consists of all the (infinite) executions 
$(\mathcal I, \pi)$ over $\boldsymbol{v}$ such that

$$(\mathcal I, \pi) \models
  \underbrace{I[\boldsymbol{v}] \land P[\boldsymbol{v}]}_{I_S} \land
  \mathbf{always}\ (\underbrace{T[\boldsymbol{v},\boldsymbol{v'}] \land P[\boldsymbol{v'}]}_{T_S})
$$

> **Note:**
The relation expressed by the formula $T$ is not required to be functional
over $\boldsymbol{i},\boldsymbol{o},\boldsymbol{s},\boldsymbol{i'}$,
thus allowing the modeling of non-deterministic systems.
>
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
>
> **Note:**
Systems are meant to be progressive: every reachable state has a successor
with respect $T_S$.
However, they may not be because of the generality of $T$ and $P$.
In other words, it is possible to define deadlocking systems.
(See later for more details on deadlocked states.)

#### Examples

(Adapted from "Principles of Cyber-Physical Systems" by R. Alur, 2015)

When reading these examples, it is helpful to keep in mind that, intuitively,
in the <tt>:init</tt> formulas the input values are given and the local and output
values are to be defined with respect to them.
In contrast, in the <tt>:trans</tt> formulas the new input values, and old input,
output and local values are given, and the new local and output values are
to be defined.

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

The next arbiter is similar to <tt>NonDetArbiter</tt> but grants requests
a cycle later and does not use a local variable for the non-deterministic choice.

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
* each $\boldsymbol y_i$ consists of $S$'s local/output variables of the same type
  as $S_i$' ’'s output;
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

Formally, a composite system $S$ introduced by the <tt>define-system</tt> command above
is a transition system whose behavior consists of all the (infinite) executions 
$(\mathcal I, \pi)$ over $\boldsymbol{v}$ such that

$$(\mathcal I, \pi) \models
  I_S[\boldsymbol{v}] \land \mathbf{always}\ T_S[\boldsymbol{v},\boldsymbol{v'}]
$$
where

* $I_S[\boldsymbol{v}] =
   I[\boldsymbol{v}] \land
   \bigwedge_{k=1}^{q} I_k[\boldsymbol{x}_k,\boldsymbol{y}_k,\boldsymbol{s}_k]$ <br>
and
* $T_S[\boldsymbol{v},\boldsymbol{v'}] = 
   P[\boldsymbol{v}] \land T[\boldsymbol{v},\boldsymbol{v'}] \land
   \bigwedge_{k=1}^{q} T_k[\boldsymbol{x}_k,\boldsymbol{y}_k,\boldsymbol{s}_k,
                           \boldsymbol{x'}_k,\boldsymbol{y'}_k,\boldsymbol{s'}_k]$

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

The next example defines a three-bit counter in terms of three identical
one-bit counters.
The one-bit counter uses a latch.
The latch has a Boolean state represented by state variable <tt>s</tt> with arbitrary
initial value.
The value of output <tt>out</tt> is always the previous value of <tt>s</tt>.
A set request (represented by input <tt>set</tt> being true) sets the new value
of <tt>s</tt> to true unless there is a concurrent reset request
(represented by input <tt>reset</tt> being true).
In that case, the choice between the two requests is resolved arbitrarily using
the value of the unconstrained local variable <tt>b</tt>.
In the absence of either a set or a reset, the value of <tt>out</tt> is unchanged.

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

#### Sanity requirements on $I_S$ and $T_S$

Because of the infinite trace semantics every system defined in IL is expected
to execute forever.
This is not a limitation in practice because systems that are meant to reach
a final state can be always modeled with states that cycle back to themselves and
produce stuttering outputs.
In such semantics, the reachability of a _deadlocked state_
(i.e., a state with no successors in the transition relation) indicates the presence
of an error in the system's definition.

Intuitively, for a system definition to define a system with no deadlocks:

1. Any assignment of values to the input variables can be extended
   to a total assignment (to all the unprimed variables) that satisfies $I_S$.

2. For any reachable state $S$, any assignment $s$ to the primed input variables
   can be extended to a total assignment $s'$ so that $s,s'$ satisfies $T_S$.

The first restriction above guarantees that the system can start at all.
The second ensures that from any reachable state and for any new input
the system can move to another state (_and so_ also produce output).

* A sufficient condition for (1) is that the following formula is valid
  in the (previously specified) background theory:

  $$\forall \boldsymbol{i}\, \exists \boldsymbol{o}\, \exists \boldsymbol{s}\, I_S$$

* A sufficient condition for (2) is that the following formula is valid
  in the background theory:

  $$\forall \boldsymbol{i}\, \forall \boldsymbol{o}\, \forall \boldsymbol{s}\,
    \forall \boldsymbol{i'}\,
    \exists \boldsymbol{o'}\, \exists \boldsymbol{s'}\, T_S
  $$

  This condition is not necessary, however, since it need not apply to unreachable
  states.
  Let $\textrm{Reachable}[\boldsymbol{i}, \boldsymbol{o}, \boldsymbol{s}]$ denote
  the (possibly higher-order) formula satisfied exactly by the reachable states
  of $S$.
  Then, a more accurate sufficient condition for (2) above would be the validity of the formula:

  $$\forall \boldsymbol{i}\, \forall \boldsymbol{o}\, \forall \boldsymbol{s}\,
    \forall \boldsymbol{i'}\,
    \exists \boldsymbol{o'}\, \exists \boldsymbol{s'}\,
     \textrm{Reachable} \Rightarrow T_S[\boldsymbol{i}, \boldsymbol{o}, \boldsymbol{s}]
  $$

> **Note:**
In general, checking the two sufficient conditions above automatically can be
impossible or very expensive because of the quantifier alternations in the conditions.

### System verification command


<tt>
(check-system $S$ <br>
&nbsp; :input (($i_1$ $\delta_1$) $\cdots$ ($i_m$ $\delta_m$)) <br>
&nbsp; :output (($o_1$ $\tau_1$) $\cdots$ ($o_n$ $\tau_n$)) <br>
&nbsp; :local (($s_1$ $\sigma_1$) $\cdots$ ($s_p$ $\sigma_p$)) <br>
&nbsp; :assumption ($a$ A) <br>
&nbsp; :fairness ($f$ $F$) <br>
&nbsp; :reachable ($r$ $R$) <br>
&nbsp; :current ($c$ $C$) <br>
&nbsp; :query ($q$ ($g_1$ $\cdots$ $g_q$)) <br>
)
</tt>

where

* $S$ is the identifier of a previously defined system with $m$ inputs, $n$ outputs,
  and $p$ local variables;
* all attributes are optional and their order is immaterial except that
  <tt>:input</tt>, <tt>:output</tt>, <tt>:local</tt>, have to come
  before the other attributes [may not need this restriction];
* all attributes except <tt>:input</tt>, <tt>:output</tt>, <tt>:local</tt>
  are repeatable.
* $\boldsymbol{i} = (i_1, \ldots, i_m)$ is a (possible) renaming of the corresponding
  input variables of $S$ of sort $\boldsymbol{\delta} = (\delta_1, \ldots, \delta_m)$;
* $\boldsymbol{o} = (o_1, \ldots, o_n)$ is a (possible) renaming of the corresponding
  output variables of $S$ of sort $\boldsymbol{\tau} = (\tau_1, \ldots, \tau_n)$;
* $\boldsymbol{s} = (s_1, \ldots, s_p)$ is a (possible) renaming of the corresponding
  local variables of $S$ of sort $\boldsymbol{\sigma} = (\sigma_1, \ldots, \sigma_n)$;
* $a$, $r$, $f$, $c$, and $q$ are identifiers;
  each $g_i$ ranges over such identifiers;
* $A$ is (non-temporal) formula over the system variables
  $\boldsymbol{i}$, $\boldsymbol{o}$, $\boldsymbol{s}$, and $\boldsymbol{i'}$
  expressing an _environmental assumption_ on $\boldsymbol{i'}$;
* $F$ is (non-temporal) formula over the system variables
  $\boldsymbol{i}$, $\boldsymbol{o}$, $\boldsymbol{s}$, and $\boldsymbol{i'}$
  expressing a _fairness condition_ on $\boldsymbol{i'}$;
* $R$ is (non-temporal) formula over all the system variables, primed and unprimed,
  expressing a state _reachability condition_;
* $C$ is (non-temporal) formula over all the unprimed system variables
  expressing a state _initiality condition_;

> **Note:**
The list of identifiers for a query $a$ contains all the elements of the query.
When the command contains more than one instance of the attributes
<tt>:assumption</tt>, <tt>:reachable</tt>, <tt>:fairness</tt> and <tt>:current</tt>,
the list of elements of a query $q$ can refer to any of the identifiers
in those attributes.


#### Semantics

Each query in the <tt>check-system</tt> command asks for the existence of a trace.
Specifically, given such a command for a system $S$
let $I_S$ and $T_S$ be the initial state and transition predicate of $S$ 
_modulo_ the variable renamings in the command.

[to be continued]
