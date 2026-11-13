# GAP-61: Algebraic Static Analysis for Queries

## Overview

This proposal defines a reusable API for statically analyzing GraphQL operations.
An analysis supplies a small algebra; a shared engine handles GraphQL semantics
and computes a sound summary when that algebra satisfies the specified laws.

The normative [draft](./DRAFT.md) defines the algebra, engine behavior,
soundness obligations, and APIs for analysis with or without request variables.

## Context

The proposal grew out of work on verified GraphQL operation analysis. Cost,
response-size, authorization, and policy analyses all need to reason about the
same fragments, runtime types, Boolean directives, field collection, and nested
selections. A common engine lets each analysis define only its own summary
domain and transfer operations.

The [IBM GraphQL Cost Directives
specification](https://ibm.github.io/graphql-specs/cost-spec.html) is the
canonical motivator. Maximum response size provides a smaller example of the
same API.

This GAP intentionally does not standardize an analysis algorithm, engine precision,
or individual analyses such as the IBM cost model.

## Status

**Proposal.** Initial draft; not yet sponsored.

The main open question is whether the algebra and engine boundary is the right
portable API for GraphQL implementations in different languages. Feedback from
authors and users of existing query-complexity, cost, and authorization tools is
especially useful.

## Challenges and drawbacks

The principal limitations are that algebra laws are not automatically enforced
by mainstream type systems, sound bounds may be conservative, symbolic argument
values may be unbounded, and some analyses rely on trusted external assumptions
such as list-size limits. The draft discusses the effect of each limitation on
the engine's guarantee.

## References and prior art

- [Static Analysis for GraphQL, Verified in
  Lean](https://duckki.github.io/2026/08/30/static-analysis-for-graphql-verified-in-lean.html)
  introduces this algebraic approach and its IBM cost case study.
- [`graphql-lean`](https://github.com/duckki/graphql-lean) defines the formal
  model and reusable soundness proofs.
- [`graphql-static-analysis-js`](https://github.com/duckki/graphql-static-analysis-js)
  is the reference implementation for the source-oriented API.
- [`graphql-static-analysis-rs`](https://github.com/duckki/graphql-static-analysis-rs)
  is a Rust implementation of the same architecture.
- [GraphQL field
  collection](https://spec.graphql.org/September2025/#sec-Field-Collection)
  defines the execution behavior modeled by collected field groups.
- [IBM GraphQL Cost Directives](https://ibm.github.io/graphql-specs/cost-spec.html)
  is the canonical motivating analysis.
