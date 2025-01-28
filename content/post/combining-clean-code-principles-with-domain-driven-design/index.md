---
title: "Combining Clean Code Principles With Domain Driven Design"
description: 
date: 2025-01-28T11:20:56+08:00
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

Domain-Driven Design (DDD) aims to model complex business logic in a way that reflects real-world domains, while Clean Code focuses on readability, maintainability, and reducing “code smells.” Although DDD and Clean Code come from different authors and traditions, they complement each other perfectly. This post explores how to keep a DDD-based project clean by applying code-smell detection and good architectural practices together.

## 1. Quick Overview of DDD Layers
A typical layered DDD architecture (in a Java/Spring Boot context) might look like this:

1. Domain Layer

   * **Entities**: Objects with identity and life cycle.
   * **Value Objects**: Immutable objects identified by their attributes, not identity.
   * **Domain Services**: Stateless services containing domain logic that doesn’t naturally fit in a single entity.
   * **Repositories** (interfaces) to retrieve and persist entities.

2. Application Layer

   * **Application Services** or “use-cases” that coordinate domain objects, external systems, and transactions.
   * They do not contain heavy domain rules; they orchestrate them.

3. Infrastructure Layer

   * Implementations of repositories (e.g., Spring Data `JpaRepository`), integrations with external systems, messaging, etc.
   * DDD places domain concepts at the center (hence “domain-driven”). **Clean Code** techniques ensure each part remains readable, testable, and well-organized.

## 2. Clean Code Principles that Matter Most in DDD

### 2.1 Single Responsibility & Avoiding “Anemic” Models

* In **Clean Code**, classes should have a **single responsibility**. In DDD, that means each **Entity** or **Value Object** focuses on domain modeling, not on infrastructure or UI logic.
* Avoid “anemic” domain models—where Entities only have getters/setters and no real business behavior. Instead, allow your **Entities** to carry domain logic that directly relates to their state or invariants (e.g., order.addItem() updates the total).

### 2.2 Avoiding Code Smells in Entities

* **Long Method / Large Class**: Keep Entities or domain services small and cohesive. If an entity class grows huge, consider splitting it or rethinking your aggregates.
* **Primitive Obsession**: In the domain layer, prefer Value Objects for meaningful concepts (e.g., Address, Money, Email) rather than raw strings or primitives. This clarifies intent.


### 2.3 Keep Infrastructure Out of Your Entities

* A frequent “clean code” smell is mixing concerns in a single class. In DDD, Entities should **not** contain direct database or networking logic. Instead, keep that in repositories (or other infrastructure components).
* Having an `@Entity` annotation in Spring Boot is typically acceptable. But do `not` add direct SQL queries, HTTP calls, or logging frameworks inside your domain entity. This separation respects single responsibility and maintains a pure domain focus.

## 3. Use-Cases and the Application Layer

### 3.1 What Are “Use-Cases”?

* Use-cases are operations your system supports: “Place an order,” “Cancel a reservation,” etc. In DDD, they often live in **Application Services** that orchestrate domain objects, external systems, and transactions.

### 3.2 Clean Code in Application Services

* **Meaningful Names**: Give your application service methods names that reflect the domain action: `placeOrder(...)`, `cancelOrder(...)`, `registerNewUser(...)`.
* **Short Methods, No Bloated Scripts**: If your use-case method becomes massive, that’s a **Long Method** smell. Decompose logic into smaller private methods or delegate domain rules to Entities/Domain Services.

### 3.3 Example Flow

```java
@Service
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final OrderDomainService orderDomainService;

    public OrderApplicationService(OrderRepository repo,
                                   PaymentGateway gateway,
                                   OrderDomainService domainService) {
        this.orderRepository = repo;
        this.paymentGateway = gateway;
        this.orderDomainService = domainService;
    }

    public void placeOrder(PlaceOrderCommand command) {
        // Create a new domain entity or load a draft
        Order order = new Order(...);

        // Delegate domain logic: e.g. validating or adding items
        orderDomainService.validateOrder(order);

        // Possibly call external Payment Gateway
        paymentGateway.charge(order.calculateTotal());

        // Finally, persist
        orderRepository.save(order);
    }
}
```

Here, the application service is coordinating the **use-case** but pushing domain logic down to the **domain layer**. This keeps code clean, short, and well-separated.

## 4. Domain Services: Where Shared Logic Lives
Sometimes, logic needs to span multiple aggregates or doesn’t quite belong in a single entity. That’s where Domain Services come in.

### 4.1 Avoid “Anemic” or “God” Entities

* If an entity becomes too big, that’s a **Large Class** smell.
* If an entity is too small (just data), but it truly has domain invariants, consider push those invariants/logic into the entity. The sweet spot is a rich model, but only with rules that make sense for that entity.

### 4.2 Example: OrderDomainService

```java
public class OrderDomainService {

    public void validateOrder(Order order) {
        // e.g. check inventory, pricing policy, etc.
        if (order.getItems().isEmpty()) {
            throw new IllegalStateException("Cannot place an order with no items");
        }
        // more domain checks...
    }
}
```

This approach keeps the domain rule consistent, testable, and free from infrastructure concerns.

## 5. Strategy Pattern in the Domain
A frequent Clean Code recommendation is to replace **switch statements** on type codes with **polymorphism**. In DDD, we often do this by introducing domain **policies** or **strategies**.

### 5.1 Example: Order Creation Policies
Suppose an order can be created in four different ways: by customer, by employee, by chairman, and by funder. Each creation path might have different rules. Instead of scattering `if/else` blocks around, we define a **Strategy** interface:

```java
public interface OrderCreationPolicy {
    Order createOrder(OrderRequest request);
}
```

Then we provide separate implementations:

```java
public class CustomerOrderCreationPolicy implements OrderCreationPolicy {
    public Order createOrder(OrderRequest request) {
        // logic specifically for a customer
        return new Order(...);
    }
}

public class EmployeeOrderCreationPolicy implements OrderCreationPolicy {
    public Order createOrder(OrderRequest request) {
        // logic for employee
        return new Order(...);
    }
}
// ... and so on
```

We pick the strategy based on an enum or a field in the entity, reducing big switch statements and keeping each policy self-contained.

## 6. Data Clumps and Value Objects

Data Clumps appear when multiple fields commonly travel together (like `street`, `city`, `state`, `zip`) but are passed around as separate parameters. **Clean Code** suggests grouping them into a single object; **DDD** calls that object a **Value Object**.

### 6.1 Example: Address as a Value Object

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String state;
    private String zip;

    // constructor, getters, maybe some validation
}
```

Rather than polluting an `Order` or `Customer` entity with many separate address fields, we encapsulate them. This not only reduces parameter lists (another code smell) but also enriches the domain with expressive objects.

## 7. Infrastructure vs. Domain
In DDD, it’s important to keep your domain layer pure (or close to it):

* **Entities**, **Value Objects**, and **Domain Services** handle **business logic**.
* **Infrastructure code** (DB queries, external calls, etc.) stays in **repositories** or **adapters**.

**Clean Code** also warns against mixing responsibilities. Using a **Repository** (Spring Data’s `JpaRepository`, for instance) helps us follow the **Interface Segregation** principle and keeps the domain logic free of direct database code.

## 8. Key Takeaways
1. **Entities/Value Objects Are Not Just Data Bags**

They enforce invariants. For example, `order.addItem()` can update totals, check limits, and ensure business rules remain consistent.

2. **Domain Services Handle Multi-Entity Workflows**

They implement domain logic that spans multiple aggregates (e.g., calculating discounts across multiple orders, shipping logic that involves both `Order` and `Inventory`).

3. **Application Services are “Glue Code”**

They wire domain logic to infrastructure (database, external APIs). They coordinate use-cases like `placeOrder()`, but avoid containing complex domain rules themselves.

4. **Why Entities Might Seem “Simple”**

A tech lead might keep an entity narrowly focused on its own data and direct invariants. Larger or cross-cutting processes go to domain services. It’s about **responsibility boundaries**, not a rigid rule. The entity deals with its own state changes; domain services tackle bigger workflows.

## 9. Final Thoughts
**DDD** and **Clean Code** reinforce each other: DDD provides a robust conceptual framework (entities, value objects, domain services, etc.), while Clean Code ensures your implementation is **readable, maintainable, and testable**. By:

* Designing rich **Entities/Value Objects** that enforce business rules,
* Leveraging **Domain Services** for multi-entity logic,
* And keeping **Application Services** as orchestrators between domain and infrastructure,

you’ll build a system that’s both **deeply aligned** with its domain and clean in its code structure. This alignment leads to clearer communication with domain experts, simpler testing, and easier feature evolution.
