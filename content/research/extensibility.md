+++
title = "Qualifying the Extensibility of Language Frameworks"
date = "2024-08-21"
math = true
+++

# Introduction

Programming languages incorporate assistive tools like user-defined
abstractions, type-checkers, meta-programming and compiler optimizations
to write programs that are as accurate and performant as possible, while
remaining concise and legible to human eyes. Compilation of
feature-packed languages becomes complex, with compilation performance
being important in ensuring productivity and ease of use. As language
features are added, language implementations must also be updated.

Systems for extensibly specifying languages, which we'll collectively
call *Extensible Language Frameworks*, or *ELFs*, offer language
designers the ability to add language features sequentially, without
rewriting existing language features. ELFs may be their own standalone
language, or more often, are *domain-specific languages* (languages
designed to support a specific purpose, *DSLs* for short) or libraries
borrowing useful behavior of some implementing language, known as a
*host language*. In this writeup, we'll refer to the language specified
in a particular usage of an ELF as the *object language*, and the ESL
itself as the *meta language*.

# Extensibility

In order to understand the differences between ELFs, we develop a system for
describing extensibility. We take inspiration from formal methods for
specifying programming languages. Broadly, models of programming
languages specify language structure and behavior by respectively
defining the concrete syntax and semantics of a language.

Syntactic specifications dictate which symbols and combinations thereof
are valid programs in a given language. The names of features, binding
forms, and lexical structures of a programming language are part of this
syntax spec, with every lexical phrase abiding by the spec being a
well-structured language form that semantic specifications can then
reason about. Syntax may be grouped, allowing us to refer to classes of
valid syntax that, for example, represent expressions, types, and so
forth. Syntax variables allow syntactic forms to be parameterized on all
possible instances of syntax of a particular class; for example the
variable `e` in the syntax specification `(add1 e)` may be used to
denote any possible syntax which is a member of the class of
expressions. We will use *term* to refer to any piece of syntax which
abides by a given object language's syntax specification, and will not
consider specific approaches to specifying syntax, reasoning about
syntax specifications generally so long as they support the abstractions
of syntax classes and syntax variables.

Semantic specifications are imposed on instances of legal program
syntax, encoding properties of a language like which pieces of syntax
are indeed well-formed programs, and how those well-formed programs
behave when they're executed. Models of language semantics may be
abstractly thought of as relations over valid syntactic forms, where the
inclusion of syntax in a relation holds some meaning. For example,
consider a binary relation `has-type` which relates syntax representing
terms to syntax representing types; a pair of syntaxes representing a
term and a type are a member of this binary relation if the term syntax
has a type which is represented by the type syntax. By defining this
relation, we encode the semantics of "having a type:" a term has a
specific type because the pair of term and type is in the `has-type`
relation, and we check if an arbitrary term is well typed by proving
that the pair of it and the type we believe it to be are a member of the
relation. Relations may also be defined in terms of each other; for
example, the `has-type` relation may rely on a unary `well-formed-type`
relation, which specifies which syntaxes may even represent meaningful
types.

In pursuit of adding new features to an object language, an ELF user
would like to be able to add new syntactic forms with which to use them;
we denote this *syntactic extension*, by which one modifies the
specification for the syntactic forms and classes that could potentially
make up a program. Here, the implementer of an ELF has a decision: is
extension strictly additive, adding either classes or cases but not
modifying existing syntax, or does an extension have the privilege of
modifying existing syntax? We'll call the former *weak* and the latter
*strong* syntactic extensibility. Strong extensibility need not be
strictly more desirable than weak extensibility, and depending on the
goals of an ELF, it may not be possible to maintain assumed invariants,
or manage the implementation of strong extensibility while providing a
friendly interface.

*Semantic extension* is how an ELF allows a user to provide new meaning
to syntactic forms. Consider a simple extension, where syntax has been
added that represents the new terms `true` and `false`, and their new
type `Bool` in a language which has an existing semantic relation
`has-type`. To complete the new language feature, the `has-type`
relation would need to be extended to include members of the `has-type`
relation which allow us to type-check our new syntax. We call this kind
of extension *term-level semantic extension,* which allows us to encode
meaning for new syntactic forms under existing semantics. We also
introduce an analogous concept of weak and strong term-level semantic
extensibility, the latter allowing for redefinition of relations over
existing terms; for example changing the type of numbers from `Natural`
to `Integer`. Weak syntactic extensions may require strong semantic
extensions in order to achieve desired behavior. For example, sound
gradual typing is a typing discipline which allows for static
imprecision in typing that induces run-time constraint checks. Often presented in literature as a modification of
some base type system, the run-time semantics of a gradually typed
language differ from those of the base language, changing how existing
terms are interpreted. Should we wish to add gradual types to a simply
typed system, we would syntactically need only add syntax representing
the unknown type (a weak extension), but semantic changes supporting
this new syntax must be made to existing typing relations (strongly
extending term-level semantics).

An ELF user may also wish to extend semantic definitions in a way that
requires the creation of new relations, adding or modifying dependencies
between existing relations to integrate extensions into an existing
object language. We'll call this *judgement-level semantic extension*,
which in contrast to term-level extension, extends what meanings we can
ascribe to syntactic forms rather than interpreting new syntactic forms
within existing relations. In our previous example, term-level extension
allowed us to add new type in a system which already contained a
`has-type` relation, but adding the `has-type` relation to an untyped
system would be a judgement-level extension. Similarly, we distinguish
between weak and strong judgement-level semantic extensions. We
summarize our extensibility classes in figure [1](#fig:classes).

<figure id="fig:classes">
<table>
<thead>
<tr>
<th style="text-align: center;"></th>
<th style="text-align: center;"><strong>Syntactic</strong></th>
<th colspan="2" style="text-align: center;"><strong>Semantic</strong></th>
</tr>
<tr>
<th style="text-align: center;"></th>
<th style="text-align: center;"></th>
<th style="text-align: center;">Term-level</th>
<th style="text-align: center;">Judgement-level</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center;"><strong>Weak</strong></td>
<td style="text-align: center;">Addition of syntactic forms and groupings</td>
<td style="text-align: center;">Addition of syntax to existing semantic relations</td>
<td style="text-align: center;">Addition of new semantic relations</td>
</tr>
<tr>
<td style="text-align: center;"><strong>Strong</strong></td>
<td style="text-align: center;">+ modification of existing syntactic forms and groupings</td>
<td style="text-align: center;">+ modification of existing associations</td>
<td style="text-align: center;">+ modification of existing relations</td>
</tr>
</tbody>
</table>
<figcaption>Extensibility classes</figcaption>
</figure>

Tangential to these classes, we introduce the concept of
*quasi-extensibility*, where an ELF provides a means by which an object
language can be extended, provided such extensions *can be expressed
using existing object language functionality*. Macro systems, for
example, display weak syntactic quasi-extension paired with weak
semantic quasi-extension: an extension creates a new syntactic form
(which must have already been legal in the object language) whose
behavior is defined in terms of existing syntax and that syntax's
existing semantics, and must therefore only have behavior already
expressible in existing object language semantics.

Related is the concept of *syntactic abstractions* used to formalize the expressive power of
macros, described by [Felleisen](https://homepage.cs.uiowa.edu/~jgmorrs/eecs762f19/papers/felleisen.pdf), in relation to a given object language, as the set of all named instances
of valid object language syntax. Felleisen's syntactic abstractions impose the same restrictions
as our idea of quasi-extensibility: a language is quasi-extensible in
both syntax and semantics if it can support syntactic abstractions. We
contrast this with our concept of syntactic and semantic extensibility,
which allows for an ELF user to add syntax and semantics to an object
language not expressible using solely syntactic abstractions.

# The Expression Problem

The classes of extensibility we've defined map well to the *expression
problem*, which as described by [Wadler](https://homepages.inf.ed.ac.uk/wadler/papers/expression/expression.txt), is the problem of achieving
extensibility of both data and addition of functions which operate on
that data. In the context of programming languages, the data domain
comprises syntactic language forms, with the function domain comprising
language semantics, namely type checking, elaboration and interpretation
algorithms. Designing ELFs which are both term-level and judgement-level
semantic extensible requires one to solve the expression problem by
allowing for the extensibility of data (that being, syntactic and
term-level semantic extension: adding syntactic forms, with new cases
describing their semantics under *existing* judgements) and adding
functions on that data (judgement-level semantic extension: new
judgements being expressed as new functions on the data representation
of language forms). In supporting *either* term-level semantic extension
*or* judgement-level semantic extension, one does not need to solve the
expression problem: term-level extension is neatly modelled with
traditional object-oriented programming (*OOP*) structures (where syntax
maps to classes, and relations to methods called by a visitor, but
adding new methods is hard), and judgement-level extension maps neatly
to typical functional patterns (where writing new functions which
operate on an existing datatype is easy, but adding new cases to union
types requires modification of existing functions). In desiring both, we
require the addition of both new language forms with semantics encoded
as instances of existing interfaces, and the addition of new interfaces
(read: judgements) to existing forms.

In Wadler's presentation, a solution to the expression problem should
allow for extension without modification of existing code (in our
application, allowing for separate compilation of a language base and
language extensions). This criterion provides a good delineation between
an *extension* and a *modification*; if the original language no longer
exists as a distinct entity&mdash;a distinct unit of compilation in the host
language&mdash;the object language has not been extended, it has been
modified.