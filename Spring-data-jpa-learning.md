
# JPA Relationship Confusions I Faced (Cart ↔ CartItem)

I feel  , this is something that confuses most of the people learning spring data jpa for the first time .

Using:

```java
Cart  ->  CartItem
```

---

# 1️) Owning Side ≠ Parent

This was one of my confusion.

In this mapping:

```java
@OneToMany(mappedBy = "cart")
private List<CartItem> cartItemList;
```

```java
@ManyToOne
@JoinColumn(name = "cart_id")
private Cart cart;
```

* `CartItem` is the **owning side** (because it has `@JoinColumn`)
* `Cart` is still the **parent** (because it contains the collection)

Owning side only means:

> “I control the foreign key column.”

It does NOT mean:

> “I am the parent.”

These are two different concepts.

---

# 2️) Why Saving Parent Sometimes Fails

Scenario:

```java
cart.addItem(item);
cartRepository.save(cart);
```

If `CascadeType.PERSIST` is missing:

You get:

```
org.hibernate.TransientPropertyValueException
object references an unsaved transient instance
```

Reason:

* CartItem is new (transient).
* Hibernate refuses to automatically persist it.

Lesson:

If parent creates child → you need `CascadeType.PERSIST`.

---

# 3️) Deleting Parent Without Cascade REMOVE

Scenario:

```java
cartRepository.delete(cart);
```

If `CascadeType.REMOVE` is missing:

You get:

```
ConstraintViolationException
Foreign key constraint fails
```

Reason:

* CartItem still references Cart.
* Database blocks parent deletion.

Important:

This has NOTHING to do with owning side.

It’s purely about cascade configuration.

---

# 4️) Removing Item From Collection Is NOT Same As Deleting It

This is where most people get surprised.

```java
cart.getCartItemList().remove(item);
cartRepository.save(cart);
```

Without:

```java
orphanRemoval = true
```

Hibernate does NOT delete the row.

Instead it does:

```sql
UPDATE cart_item SET cart_id = NULL
```

Not DELETE.

That was unexpected to me .

---

# 5️ ) When orphanRemoval Is Required

If business logic says:

> When item is removed from cart, it should disappear completely

Then:

```java
orphanRemoval = true
```

is required.

With orphanRemoval:

```sql
DELETE FROM cart_item WHERE id = ?
```

Without it:

Only FK becomes NULL (if allowed).

---

# 6️⃣ Why FK NOT NULL Causes Errors

If `cart_id` is NOT NULL in database
and orphanRemoval = false

Removing item from collection causes:

```
ConstraintViolationException
```

Because Hibernate tries:

```sql
UPDATE cart_item SET cart_id = NULL
```

Database rejects it.

This is not a JPA bug.
It’s FK constraint enforcement.

---

# 7️⃣ Cascade REMOVE vs orphanRemoval (The Real Difference)

This confused me a lot.

| Action                       | Needs Cascade REMOVE | Needs orphanRemoval |
| ---------------------------- | -------------------- | ------------------- |
| Delete parent                | Yes                  | No                  |
| Remove child from collection | No                   | Yes                 |

So:

* Cascade REMOVE → parent deletion propagation.
* orphanRemoval → relationship cleanup.

They are not interchangeable.

---

# 8️⃣ Removing Child + setCart(null)

If I do:

```java
cart.getCartItemList().remove(item);
item.setCart(null);
```

Without orphanRemoval:

Hibernate updates FK to NULL.

It does NOT delete the row.

Because changing relationship ≠ removing entity.

---

# 9️) The Mental Model That Finally Worked For Me

Instead of memorizing rules, I now think like this:

* Who controls FK? → Owning side.
* Who propagates operations? → Cascade.
* Who deletes detached children? → orphanRemoval.


---

# 10 ) What I Now Default To

For parent → child lifecycle (like Cart → CartItem), I usually use:

```java
@OneToMany(mappedBy = "cart",
           cascade = CascadeType.ALL,
           orphanRemoval = true)
```

Because in most real-world cases:

* Saving parent should save children.
* Deleting parent should delete children.
* Removing child from parent should delete that child.

This keeps object model and database consistent.

--
