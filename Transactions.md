# Spring Transactions – What I Learned Today

> This document captures my learning from real confusion and debugging while understanding Spring transactions. It is not a tutorial; it is a record of how my mental model evolved.

---

## 1. The Initial Confusion

I believed that knowing Spring annotations was enough to understand backend behavior.
I knew `@Transactional`, but I did not truly understand **what happens when things fail**.

The confusion started when I saw this behavior:

* Owner / Order saved successfully ✅
* Dependent operation (Pet / Email) failed ❌
* Previously saved data disappeared ❌

There was no obvious error pointing to *why* this happened.

---

## 2. Key Realization: Transactions Are Business Units of Work

A transaction is **not** tied to a method or a database call.

It represents a **business unit of work**.

If any part of that unit fails, Spring rolls back **everything** by default.

This led to an important insight:

> Transactions do not care about importance.
> They care about boundaries.

---

## 3. Default Transaction Behavior (Propagation.REQUIRED)

Spring’s default transaction propagation is:

```java
@Transactional // propagation = REQUIRED
```

Meaning:

* If a transaction already exists → join it
* If not → create a new one

Effect:

* All methods share the **same fate**
* One failure causes a rollback of all changes

This explains why:

* Order save was undone when email sending failed

---

## 4. Why `REQUIRES_NEW` Did Not Work Initially

I tried to fix the issue by using:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
```

But it **still did not work**.

This led to a deeper understanding of how Spring actually works.

---

## 5. Spring Is Proxy-Based (Critical Insight)

Spring applies `@Transactional` using **runtime proxies** (AOP).

Transactions only work when:

* A method is invoked **through the Spring proxy**

### The self-invocation problem

When one method calls another method **inside the same class**:

```java
this.methodB();
```

* The call bypasses the proxy
* Transactional annotations are ignored
* The method behaves like a plain Java (POJO) method

No proxy → no transaction → no propagation rules

---

## 6. Why Transactional Methods Should Be in Different Services

To ensure transactional behavior:

* Calls must cross **Spring-managed bean boundaries**

Correct approach:

* Business flow in one service
* Isolated transactional logic in another service

This ensures:

* The call goes through the proxy
* `REQUIRES_NEW` actually starts a new transaction
* Failures are properly isolated

---

## 7. Separation of Concerns (Reinforced Understanding)

* Controller: handles HTTP concerns
* Repository: handles database access only
* Service: defines business flow and transaction boundaries

Transactions belong in the **service layer** because:

* Only services understand the full business workflow
* Controllers are not reusable entry points
* Repositories are too granular

---

## 8. What Actually Makes a Backend Engineer (Key Shift)

I realized that backend engineering is **not about syntax**.

It is about:

* Understanding execution flow
* Anticipating failures
* Designing safe rollback boundaries
* Knowing how the framework behaves at runtime

---

## 9. Why This Learning Matters in the AI Era

AI can:

* Generate CRUD code
* Add annotations
* Scaffold projects

AI cannot:

* Predict transactional side effects
* Understand proxy bypass issues
* Design failure-isolated systems

These require **human system thinking**.

---

## 10. Final Takeaway

> I stopped trusting annotations blindly.
> I started understanding how systems behave when things break.

This shift—from syntax to system behavior—is the most important learning from today.
