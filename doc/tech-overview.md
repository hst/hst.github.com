---
layout: default
title: Technical overview
---

# Technical overview

This document provides an overview of the source code and APIs of the
HST library, and some of the rationale behind its design.

## Data structures and basic types

Any CSP refinement checker is going to need good support for various
data structures — most notably sets of integers (to hold events and
states) and sets of these sets (to hold refusals).  Indeed, much of
the publicly available literature on FDR describes its approach for
dealing with large instances of these sets.  One assumption that I
made while designing and writing HST is that with the advances in
processing power and memory sizes over the past decade or so, the
details of the data structure implementations would be less important.
The library is not at the point yet where it's clear whether that's
true or not, but the data structure APIs should be separated enough
from the rest of the library that more thoughtful implementations
could be dropped in without requiring a major refactoring of the code.

That shouldn't imply that the data structure library I use didn't have
much thought put into it; just that it wasn't me who did the thinking.
HST uses the [Judy library](http://judy.sourceforge.net/), which
provides a low-level C API for sets of integers and (integer →
integer) maps.  Built on top of this is the [judyhash
libary](http://judyhash.sourceforge.net/), which provides
implementations of the STL `set` and `map` interfaces using Judy
structures as its backing store.  The project pages for these two
libraries claim very good speed and storage requirements for widely
varying usage patterns, which, if correct, should lend them well to
the kinds of sets that we'll need for a refinement checker.  Again,
this is a hypothesis that needs verification!

Finally, HST contains some classes that build upon judyhash, providing
a more convenient API (and providing an abstraction barrier that
should let us replace Judy and judyhash should it be necessary).

### intset.hh

Defines the `intset_t` class, which represents a set of integers.
Provides methods for all of the basic set operations, and a `>=`
comparison operator based on subset containment.  This class also
defines a hash function for sets based on [Zobrist
hashes](http://en.wikipedia.org/wiki/Zobrist_hashing) (defined in
_zobrist.hh_).  Zobrist hashes have the beneficial property that,
given a hash for the set _a_, you can find the hash for (_a_ ∪
{__i__}) with a single XOR operation — you don't have to recalculate
the whole thing from the elements of the new set.

### intsetset.hh

Defines the `intsetset_t` class, which represents a set of
`intset_t`s.  It defines the same methods as `intset_t`.

### types.hh

Defines the basic types used by the rest of the library: `state_t` and
`event_t`.  They're both `typedef`ed to be integers.  This header also
defines the `stateset_t` and `alphabet_t` classes, which are
`typedef`s for `intset_t`.

### eventmap.h

Defines the `eventmap_t` class, which represents an (`event_t` ↔
`event_t`) relation.  It's called an event “map” even though it's not
technically a map, since it's not functional.  Anyway, it defines
methods for adding event pairs to the map, for finding the relational
image _μ_(_e_), and for iterating through all of the pairs.  Its
primary purpose is as one of the arguments in the CSP renaming
operator.

### event-stateset-map.hh and state-stateset-map.hh

These two classes provide a convenient wrapper around two maps that
are used pretty often: (`event_t` → `stateset_t`) and (`state_t` →
`stateset_t`).  Technically, since `event_t` and `state_t` are the
same underlying type, this could be defined as a single class, but
they're not.


## Labeled transition systems

The key data structure that we need is the labeled transition system
(LTS), which is represented by the `lts_t` class (defined in the
_lts.hh_ header).  The LTS graph itself is represented by a (`state_t`
→ `event_t` → `stateset_t`) nested map.  The class provides two helper
methods for referencing into this map: `deref1`, which takes a
`state_t` and yields an `event_stateset_map_t`; and `deref2`, which
takes a `state_t` and `event_t` and yields a `stateset_t`.  (More
accurately, they return `shared_ptr`s to the result, to prevent
extraneous copying.)  Both methods are also provided in two flavors.
The first is a `const` version that returns a `NULL` pointer if the
argument doesn't exist in the graph.  The second is non-`const`, and
creates an empty result if the argument doesn't already exist.  If you
have a non-`const` reference to an `lts_t`, and want the `deref`
methods to return a `NULL` pointer if the argument doesn't exist, you
must use a cast to ensure that you call the `const` version of the
method.

The `lts_t` class also keeps track of _finalized_ states, which are
states where you've fully specified its outgoing edges.  The
`add_edge` methods can only be called for _from_ states that are not
yet finalized.

Lastly, the class defines several iterators that can be used to
inspect the LTS graph:

* `from_state_iterator` — iterates through all of the _from_ states in
  the LTS graph

* `state_events_iterator` — iterates through the _initials_ set for a
  particular _from_ state (i.e., the set of events for which there is
  an outgoing edge from the state)

* `event_target_iterator` — iterates through all of the states that
  can be reached from a _from_ state by performing a particular
  _event_

* `state_pairs_iterator` — combines the behavior of the
  `state_events_iterator` and `event_target_iterator`; given a _from_
  state, it returns an (`event_t`, `state_t`) pair for each outgoing
  edge


## Normalized LTSs

The specification side of a CSP refinement must be normalized before
the refinement check can take place.  A _normalized_ LTS is one where
we guarantee that there is at most one outgoing edge for any (_state_,
_event_) pair.  The states in the normalized LTS, therefore, must
represent _sets_ of states from the original LTS.

When we normalize an LTS, we need to know which event is τ, since we
need to calculate τ-closures in order to correctly determine the sets
of source states that a normalized state refers to.  We could say that
τ is always event 0; however, we've chosen a different strategy and
allow the user of the `normalized_lts_t` class to specify which event
is τ.  In the case of our CSP scripts, this will in fact always be
event 0, but it seemed a better abstraction not to rely on this.

The algorithm that we use to normalize an LTS is described in
[\[1\]](#bib1), and is broken down into the following basic operations:

* **τ closure** (_normalization/closure.cc_): calculates the set of
  states that can be reached from a set of initial states by following
  any sequence of τ events.

* **divergence testing** (_normalization/divergent.cc_): finds the
  _divergent_ states in an LTS.  A state is divergent if it is part of
  a τ cycle, or if a τ cycle is τ-reachable from it.

* **prenormalization** (_normalization/prenormalize.cc_): calculates a
  _prenormalized_ LTS given a special _initial state_.  The basic idea
  is to find the sets of source states that are all reachable from the
  initial state by following the same sequence of actions.  This
  result is not fully normalized, though, because there can still be
  some prenormalized states that are semantically identical.

* **bisimulation** (_normalization/bisimulate.cc_): finds the
  prenormalized states that are semantically identical.  Starts by
  assuming all prenormalized states with the same initial behavior are
  identical, and repeatedly refines this equivalence relation by
  separating states that don't actually have the same behavior.
  Eventually this bottoms out at a fixed-point, which is the maximal
  bisimulation relation.

* **normalization** (_normalization/normalize.cc_): uses the
  bisimulation relation to “merge” together prenormalized states that
  are behaviorally equivalent.  The end result is a fully normalized
  LTS.

From the point of view of a client of the `normalized_lts_t` class,
only the prenormalization and normalization steps must be called
explicitly; the others are called automatically during this process.
Two steps must be followed:

1. Call the `prenormalize` method for the specification process's
   initial state.  (If you want to perform multiple refinement checks
   on the same script, you can call the `prenormalize` method multiple
   times, once for each specification process.)

2. Once all of the necessary states have been prenormalized, call the
   `normalize` method.

Note that once you call the `normalize` method, you the normalized LTS
is “locked in”, and you can't prenormalize any more source states.
You can use the `clear` method on the normalized LTS, though, to reset
things so that you can prenormalize more source states.  However, this
throws away any existing normalization that's been done, so you should
really try to prenormalize everything that you need before calling the
`normalize` method.


## CSP scripts

The `csp_t` class creates the predefined τ and ✓ events and the
predefined `STOP` and `SKIP` processes.  It also defines the
operational semantics of the various CSP operators in terms of an LTS.
Finally, it includes support for memoizing the processes as they're
constructed; for instance, this means that calculating P ||| Q twice
(with the same operand processes) will only actually calculate the
interleaving once, and return the same result process for each
invocation.  This also works for the temporary processes that can be
created while unwinding a complex operator; an interleaving, for
instance, will encounter the same subprocesses many times while it's
expanded.  By memoizing these intermediate processes, the process tree
of the overall interleaving is much smaller.

The operator statements correspond with the operator statements in the
CSP₀ language.  Some of the processes require that their operands be
finalized; others do not.  The operators that do not require finalized
operands are what support CSP₀'s limited recursion.  More details can
be found in the [CSP₀ reference](csp0.html).

Each operator has two methods for constructing it.  In the first, we
assume that a state has already been allocated for the result, and we
just add the appropriate edges to that state to represent the
semantics of the operator.  In the second, we allocate a new process
state before calculating the necessary edges.  It's assumed that these
newly-allocated processes don't need to be externally visible to the
CSP₀ script; if they did, they would've been preallocated by a
`process` statement, and the first version of the operator method
would've been called instead.  Therefore, the new processes are
created (in the `add_temp_process` method) with names beginning with a
“`%`”, which is not a valid CSP₀ identifier character.

Finally, the `csp_t` instance contains a symbol table that allows you
to look up a process state by its CSP₀ name, and there are also
methods that give you access to the underlying `lts_t` and
`normalized_lts_t` instances.

The `csp_t` class only defines the structure of a CSP script; the
refinement checks are performed by stand-alone functions defined in
the _assertions.hh_ header.  (Furthermore, these functions take in
`lts_t` and `normalized_lts_t` instances, rather than a `csp_t`
instance, since in theory we can perform a refinement check on any
valid LTS, regardless of whether it came from a CSP script.)

There aren't yet any helper methods in `csp_t` for performing the
normalization or refinement checks; please see the _src/bin/csp0.cc_
and _tests/refinement/test-traces-refinement.cc_ files to see the
correct sequence of method calls that are needed.


## References

<a name="bib1"/>
<div class="reference">
  <div class="citation_number">[1]</div>
  <div class="citation">
    A.&nbsp;W. Roscoe.  “Model-checking CSP”.  In A. W. Roscoe,
    editor, <em>A classical mind: Essays in honour of
    C. A. R. Hoare</em>, pages 353–378.  Prentice-Hall, 1994.
  </div>
</div>
