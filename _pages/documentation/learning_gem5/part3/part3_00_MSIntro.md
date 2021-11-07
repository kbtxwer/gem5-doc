---
layout: documentation
title: Introduction to Ruby
doc: Learning gem5
parent: part3
permalink: /documentation/learning_gem5/part3/MSIintro/
author: Jason Lowe-Power
---


Introduction to Ruby
====================

Ruby comes from the [multifacet GEMS
project](http://research.cs.wisc.edu/gems/). Ruby provides a detailed
cache memory and cache coherence models as well as a detailed network
model (Garnet).

Ruby is flexible. It can model many different kinds of coherence
implementations, including broadcast, directory, token, region-based
coherence, and is simple to extend to new coherence models.

Ruby is a mostly drop-in replacement for the classic memory system.
There are interfaces between the classic gem5 MemObjects and Ruby, but
for the most part, the classic caches and Ruby are not compatible.

In this part of the book, we will first go through creating an example
protocol from the protocol description to debugging and running the
protocol.

Before diving into a protocol, we will first talk about some of the
architecture of Ruby. The most important structure in Ruby is the
controller, or state machine. Controllers are implemented by writing a
SLICC state machine file.

SLICC is a domain-specific language (Specification Language including
Cache Coherence) for specifying coherence protocols. SLICC files end in
".sm" because they are *state machine* files. Each file describes
states, transitions from a begin to an end state on some event, and
actions to take during the transition.

Each coherence protocol is made up of multiple SLICC state machine
files. These files are compiled with the SLICC compiler which is written
in Python and part of the gem5 source. The SLICC compiler takes the
state machine files and output a set of C++ files that are compiled with
all of gem5's other files. These files include the SimObject declaration
file as well as implementation files for SimObjects and other C++
objects.

Currently, gem5 supports compiling only a single coherence protocol at a
time. For instance, you can compile MI\_example into gem5 (the default,
poor performance, protocol), or you can use MESI\_Two\_Level. But, to
use MESI\_Two\_Level, you have to recompile gem5 so the SLICC compiler
can generate the correct files for the protocol. We discuss this further
in the compilation section \<MSI-building-section\>

Now, let's dive into implementing our first coherence protocol!
