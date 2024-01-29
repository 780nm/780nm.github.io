+++
title = "Dependent Types for Parallel Operations - Proof of Concept"
date = "2023-12-21"
math = true
+++

The following is an excerpt of a class project I completed for CPSC 521, Parallel Computing. I thought it might be fun to model a small portion of Erlang --- a message-passing language initially built for the telecommunications industry --- in Agda, a dependently typed language, allowing me to prove some correctness properties for parallel operations. This excerpt also includes an overview of dependent types, easing into the key concepts.

Sources can be found [here,](https://github.com/780nm/dep-erl) and the original text (typeset for Markdown) can be found below.

# Introduction

Computation power is a hot commodity, and day by day the world demands more of it, done
quicker and cheaper. As sequential computation threatens to plateau in performance, we turn to
parallel computing in order to solve increasingly larger computational problems, providing they can
be massaged to fit a parallel paradigm.

To demonstrate ways a naive programmer might be tripped up in a parallel world, we turn to
a classical example of a parallel operation, the [reduction operator](https://en.wikipedia.org/wiki/Reduction_operator). Our implementing language
of choice is [Erlang](https://www.erlang.org/), a gradually typed, impure functional message-passing language well-suited for
distributed and parallel computing applications. Erlang’s [Dialyzer](https://www.erlang.org/doc/man/dialyzer.html) offers static type checking and
analysis, which we’ll defeat in the following example:

```erlang
-type procInfo() :: { PPid :: pid() | {pid()}, CPids :: [pid()],
   I :: non_neg_integer()}.
-type combine(DataT) :: fun((DataT, DataT) -> DataT).

-spec reduce(procInfo(), combine(DT), DT) -> DT.
reduce(ProcInfo, CombineFun, Value) -> % implementation available on Github
```

Note, in particular, that our `combine` function, which computes the merged result of two parallel branches of our reduction tree, is statically enforced to have a reasonable type — it’s a two-argument function which returns a single argument of the same type as its inputs.
Suppose we attempt to reduce a random array with the following function:

```erlang
red_fun() -> fun(X, Y) -> X + Y / 2 end.
```

Statically, this choice of `combine` is satisfactory - it’s not only of the right type, but in Dialyzer’s
opinion, is good to go:

```
sean@Sullivan:~$ dialyzer reduce.erl
  Checking whether the PLT /home/sean/.dialyzer_plt is up-to-date... yes
  Proceeding with analysis... done in 0m0.09s
done (passed successfully)
```

But, unsatisfactorily, arbitrary instances of our parallel `reduce` yield disagreeing results due to the fact that `red_fun` is not associative, breaking a fundamental assumption of the reduce operator.
Types did not allow us to catch this bug, and while testing infrastructure might, we could instead tighten our specifications for what properties `combine` must uphold, explicitly defining a more appropriate specification where a test suite would offer an implicit approximation.

# Specifications

It’s important to note that it’s unfair to blame Dialyzer for allowing this bug to slip through the cracks.
Dialyzer verified that the type we expected for `combine` — specifically the type `fun((DataT, DataT) -> DataT)` — is upheld by `red fun`. Whether that type is appropriately specific to prevent the class of bug we’ve demonstrated is not within scope of Dialyzer’s analysis.
This begs the question, is there any way to specify more stringent requirements for combine using types?

Indeed, there is, but we need to enrich our type system. Dependently typed languages are a class of languages which allow types to depend on values. Of particular interest is the fact that dependent types allow us to use [propositional logic](https://en.wikipedia.org/wiki/Propositional_calculus) to encode and prove properties of our functions.
Take for example the associative property, for which we provide the following instance in [Agda](https://wiki.portal.chalmers.se/agda/pmwiki.php), a dependent dialect of Haskell.

```agda
associative : (X : Set) -> (f : (X -> X -> X)) -> Set
associative X f = (c b a : X) -> (f a (f b c)) ≡ (f (f a b) c)
```

Breaking this definition down, we see that it makes use of the $\equiv$ type,
which allows us to reflect syntactic equality back into a term in our language,
using [syntactic](https://en.wikipedia.org/wiki/Syntax_(programming_languages))judgements of sameness in order to reason about sameness of programs.
Indeed, proving associativity by this definition requires us to argue, based on the syntactic definitions of our program,
that we may transform the syntax on the left-hand side of $\equiv$ to that on the right without changing the value it reduces to.


Note that associativity is defined as a function from any three arguments to an equality.
Intuitively, we may understand this as the goal that for any values of `a`, `b` and `c`, we are able to prove an application of `f` is associative.
If we are able to produce a function of this type,
then it must be the case that `f` is indeed associative, and we haven't merely cherry-picked values for which associativity appears to hold.

Dependent types allow us to type our reduction function thusly:
```agda
eReduce : {X : Set} -> (t : Term (X -> X -> X) 0 []) -> (associative X (recoverTerm t)) -> (Term ⊤ 0 [])
```

where `Term` is a type we use to carry around additional information for the purposes of translating our functions back into Erlang,
as we'll elaborate upon in the next section. We've therefore strengthened the specificity of our `reduce`
function's type, allowing us to statically encode the fact that `reduce` can only be called on associative functions.

# Implementation

A significant difficulty in adding dependent types to Erlang is the fact that Erlang is gradually typed,
with type annotations being optional --- often inferred from guard expressions --- and mainly for the purpose of static analyses rather than compile time checks.

This is not to say that all hope is lost. Gradual dependent languages with support for propositional equality are in active research and would suit this application well, with the significant caveat of greatly reduced performance.
Suppose we didn't want to rewrite Erlang's entire type system for a proof of concept,
what options might we have?

We could attempt to embed some purely functional subset of Erlang into an existing dependent type theory.
Agda, which has a mature dependent type system in which we can encode a subset of Erlang's arithmetic operations, is a natural choice.
We present the following Agda data definition:

```agda
fnOfType : ∀ {order} -> (Vec Set order) -> Set -> Set 
fnOfType (Y :: b) X = fnOfType b (Y -> X)
fnOfType [] X = X

fnOfStr : ℕ -> Set
fnOfStr 0 = String
fnOfStr (suc n) = String -> (fnOfStr n)

data Term (X : Set) : (order : ℕ) -> (Vec Set order) -> Set where
  term : ∀ o -> (b : Vec Set o) -> (fnOfType b X) -> (fnOfStr o) -> (Term X o b)
```

The `Term` datatype encodes the following information:
the number of free variables it contains, their types, a function from values to an Agda term (capturing binding semantics),
and a function from strings representing the Erlang syntax for those bindings to a string representing the Erlang syntax of the term,
which we may use to compile our Agda code into Erlang.
the `bind` form allows us to lift Agda's function bindings into the language of our syntax by giving them a name in our Erlang elaboration. 

Indeed, each closed `Term` contains both the Agda term we may prove properties of and, assuming our translation is correct,
the Erlang syntax which represents that Agda term.
By defining Erlang language constructs using our `Term` datatype,
we can embed a DSL in Agda which we may then compose in a well-typed fashion:

```agda
eNum : ℕ -> (Term ℕ 2 (ℕ :: ℕ :: []))
eNum n = term 2 (ℕ :: ℕ :: []) (λ x y -> n) (λ x y -> natShow n)

_e+_ = eOp (λ a b -> a + b) (λ a b -> a +s+ (" + " +s+ b))
_e*_ = eOp (λ a b -> a * b) (λ a b -> a +s+ (" * " +s+ b))

-- see source for all definitions

redFun = eBind "X" (eBind "Y" (eVar0N e+ (eVar1N e* (eNum 0))))

assoc : (a b c : ℕ) -> ((a + (b * 0)) + (c * 0)) |$\equiv$| (a + ((b + (c * 0)) * 0)) 
assoc a b c rewrite *-zeroʳ (b + (c * 0))
   rewrite *-zeroʳ b
   rewrite *-zeroʳ c
   rewrite +-identityʳ a = +-identityʳ a

-- Type an example for reduce, an associativity proof satisfying our constraint
reduction = eReduce redFun assoc

-- Spit out some Erlang:
main : Main
main = run (writeFile "test.erl" (recoverStr reduction))
```

Above we see the use of rewrite rules to syntactically prove the associative property for our choice of reduction function,
$X + (Y \cdot 0)$, making use of the Agda standard library's proofs for simple arithmetic equalities.
Note that `redFun` is defined in terms of our DSL, and in particular we've managed to avoid directly constructing our `Term` datatype.
Our reduction is well typed, consuming a function which is both associative and consumes two values of the same type it produces.
On the last line, we output the Erlang syntax representing our Agda term.

# Conclusions

We succeeded in making a micro DSL in Agda for typing simple arithmetic Erlang functions on natural numbers,
proving that dependent types offer us a way of catching the bug we postulated in section 1.
The Agda DSL approach is not feasible, however, for a full scale implementation.
Variable binding is difficult to model, making for a poor user experience,
and Agda's lack of support for gradual types means one would always be restricted to a purely functional subset of Erlang,
and not be able to reason about effectful code.

Personally, I learned a lot about modelling binding in Agda,
as I needed to model functions and bound variables for the DSL.
I experienced the difficulties of modelling language semantics in Agda in general
as I went down many false paths and typed myself into a corner,
making it impossible in any number of ways to write `bind`, the function introduction form.
I ultimately settled on a uniform `Term` type,
with curried application of open variables via `bind`,
allowing me to push arguments one at a time from the set of open variables into the type of the term,
adding an additional argument to the function type.
This reversed the order of binding relative to the ordered vector of types,
which was difficult to work with and made associativity proofs require some care.

# Future Work

The natural approach for future work in this area is to enrich Erlang's type system with gradual dependent types.
In order for this to be sound, we must investigate the interaction between effects and gradual dependent type theory,
in particular with respect to propositional equality.

Several questions arise --- how do we prevent performance blowup associated with sound gradual typing?
Is it better to have unsound types and use dependent types as a static specification? 
What does a gradual check on associativity look like?
Depending on complexity in theory and penalty in performance,
it may be prudent to omit insertion of runtime checks based on gradual type boundaries,
and use gradual dependent types to encode requirements, which could be formally proven but also ignored via a type cast.
This would greatly simplify the type theory at the expense of soundness,
but may be sufficient to augment our static analyses and catch our bug.