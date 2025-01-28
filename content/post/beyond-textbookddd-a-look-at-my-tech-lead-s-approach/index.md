---
title: "Beyond Textbook DDD: A Look at My Tech Lead’s Approach"
description: 
date: 2025-01-29T08:51:46+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories:
  - Domain-Driven Design
tags:
  - ddd
  - domain-driven-design
  - clean-code
---

In my previous article, “[Combining Clean Code Principles with Domain-Driven Design](../combining-clean-code-principles-with-domain-driven-design/index.md)”, we discussed how DDD and Clean Code naturally reinforce each other—emphasizing clarity, separation of concerns, and well-modeled domain logic. 

In a typical “textbook” DDD setup, we often see:

* **Entities** annotated with `@Entity` in the *domain* layer,
* Rich domain logic inside those entities or, if needed, in a domain service,
* And direct usage of those domain entities for both reading and writing.

However, my team’s tech lead has taken a different route. Below is an exploration of what he told me, how I researched the approach, and the conclusion I’ve arrived at about its suitability for our specific scenario.

---

## 1. The Tech Lead’s Statement

> “Modifying a field in the same object across many methods will lead to tons of bugs in the future.”

This was how the conversation started. Our tech lead warned about the dangers of “passing around a mutable `@Entity` object” that JPA/Hibernate manages. His concern was that multiple methods might inadvertently change the entity’s state, leading to unintended side effects or tricky concurrency bugs.

Instead, he proposed a design like this:

Keep `@Entity` classes in a `po` (persistent object) package, not in the domain layer.
When data is read, convert that `@Entity` object into an **immutable DTO** (in our case, a **Java Record**).
Perform business logic using that **immutable** data structure.
If changes are needed, produce a **new** DTO/Record (rather than mutating the old one).
Finally, **convert** back to the `@Entity` object right before persisting to the database.

**Key Rationale**:

* By isolating the JPA entity, we decouple domain processing from the “live” Hibernate-managed object.
* By using **immutable** records, we drastically reduce the chance that multiple methods inadvertently corrupt shared state.

## 2. Research: How This Differs from “Textbook” DDD

I dove into various DDD articles to see how typical DDD recommends structuring entities and logic. Usually, **textbook DDD** suggests:

1. Entities in the **domain layer** (often annotated with `@Entity` in a typical Spring/Hibernate environment).
2. **Rich** domain behavior inside those entities (e.g., `order.addItem(...)`, `order.cancelOrder()`).
3. Possibly returning these same domain objects to the outside world, or mapping them to simple DTOs if you want to hide details or flatten data for the UI/API layer.

The difference in our tech lead’s approach:

* He **splits** the “domain model” from the “persistence model.” Instead of a single `@Entity` that acts as both domain and DB representation, he keeps them in a separate `po` package.
* He **doesn’t** place complex domain logic directly in the `@Entity`; that logic might be in domain services (for multi-entity operations) or is computed using these **immutable** record objects (acting almost like “detached” domain data).
* After domain rules complete, we only map the record back into the `@Entity` form before saving.

## 3. Implementation Details & Benefits

### 3.1 Immutable DTOs / Records
The use of **immutable** data structures (Java Records) means we never have partial updates in flight. Each transformation is a new object, a new snapshot of state. This suits:

* **Functional / Reactive** style logic,
* **Concurrency** safety (no unexpected side effects from multiple threads or methods altering the same entity instance),
* **Clear data flow**, as changes only happen by creating new records rather than mutating old ones.

## 3.2 Isolation from JPA State Management

Because we don’t keep the `@Entity` object “alive” in memory across multiple domain operations, we avoid accidental flushes or lazy-loading surprises. The domain logic works with plain, in-memory data structures (records), unaware of any ongoing session or proxy mechanics. We only reattach to the DB session at the final step.

## 3.3 Potential Downsides

Of course, it’s not all sunshine:

1. **Additional Mapping**: Converting back-and-forth between `@Entity` (PO) and record-based domain data can be tedious. Some teams use libraries like MapStruct to reduce boilerplate.
2. **Risk of Entities Turning “Anemic”**: If we never put any domain behavior in the @Entity class, it effectively becomes just a data structure for the DB. This is okay if the real domain model lives elsewhere, but we must ensure we do indeed have a “real” domain model or domain services with the necessary business logic.
3. **Learning Curve**: New developers might get confused about which class is the “real entity.” Clear docs or naming conventions help.

## 4. Is This Approach “Suitable”?
**DDD** is more about **aligning code** with **domain concepts** than about dogmatically following a single pattern. The text-book approach of “one entity class with JPA annotations + domain methods” works well in many cases—especially simpler or smaller projects. However:

* **For larger, complex, or highly concurrent systems**, the separation of domain logic from the JPA layer is a recognized and sometimes recommended pattern.
* **Tech leads** often choose this “double-model” approach if they’ve encountered tricky issues with JPA state management or concurrency in the past. The immutability step can help reduce entire categories of bugs.

Conclusion: It’s not “best” in an absolute sense—there is no single “best” architecture. But it can be **suitable** (and quite powerful) for:

* Teams who want a pure domain model, free from ORM.
* Systems dealing with concurrency, where immutable data helps avoid confusion.
* Projects with complex transformations where the domain logic is easier to express outside the constraints of a “live” entity.

## 5. Final Thoughts

**Textbook DDD** suggests putting domain logic in your entities, with `@Entity` classes living in the domain layer. That’s perfectly valid, especially when the overhead of mapping or immutability isn’t necessary.

**Our tech lead’s approach** takes a more separated stance:

1. `@Entity` for raw persistence (in a `po` package),
2. **Immutable Records** for domain logic transformations,
3. “Pure” domain services to handle multi-entity rules,
4. Mapping between these layers when saving or loading from the database.

While it requires extra care (and extra classes), it also **reduces side effects** and clarifies exactly when data is persisted versus just used in domain logic. If your domain is large or the logic is complex enough to warrant the overhead, this pattern can keep code safer and more expressive.

Ultimately, **both approaches** are part of the DDD toolbox. Evaluate your project’s scale, complexity, concurrency needs, and team preferences to pick which one is more suitable. As with all architectural decisions, the real goal is to produce **clean, maintainable code** that expresses the domain clearly and handles change gracefully.