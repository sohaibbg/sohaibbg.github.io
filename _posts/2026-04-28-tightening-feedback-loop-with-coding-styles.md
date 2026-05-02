---
layout: post
title: "Shrinking the Search Space for Code Reviews"
date: 2026-04-28
categories: coding-practices
---

Tightening the human feedback loop is prominent now as a bottleneck with agent assisted coding. Whilst coding "practices" like continuous deployment and TDD are already espoused by many, I wanted to mention "styles", instead, of coding.

The core idea is learning to code in a style that makes the boundaries of change clear, such that when changes are made, typically by an AI agent, you can be confident that unexpected "side-effects" will not occur as a result. This entails using practices like guard clauses and immutability which come from functional programming.

Over time it will help evolve in you a sense of seeing code as states and outcomes, a level of abstraction higher than seeing code "imperatively", i.e. in loops and conditions. Working at a higher abstraction level = more comprehension with less effort, which is something that also goes far with LLMs.

The benefit is twofold: not only do code reviews become faster, but the codebase is left in a more maintainable state. Changes become easier to reason about, reducing both the cognitive load on human reviewers and token use with AI agents.

These examples depict real world usecases of existing code + a delta of change representing challenges with maintainability of code. By the way, "memory" features like Claude.md make inculcating these patterns trivial, it is your human brain you must train to see the patterns in this style of coding.

## 1. Early Return Pattern/Using Guard Clauses

Consider the following example, which I have frequently seen in codebases, of nested statements.

```java
public boolean assemblyCheck() {
    boolean result = true;
    if (pants.arePressed()) {
        if (belt.isBuckled()) {
            if (collar.isFixed()) {
                if (buttons.areFastened()) {
                    if (jacket.isBrushed()) {
                        if (cuffs.areAdjusted()) {
                            if (tie.isStraightened()) {
                                result = true;
                            } else {
                                teacher.straighten(tie);
                                result = false;
                            }
                        } else {
                            result = false;
                        }
                    } else {
                        student.remove(jacket);
                        result = false;
                    }
                } else {
                    result = false;
                }
            } else {
                teacher.fix(collar);
                result = false;
            }
        } else {
            result = false;
        }
    } else {
        result = false;
    }
    return result;
}
```

Suppose an agent made a change as the BasketService is now meant to be used. Can you spot the error?

```java
public boolean assemblyCheck() {
    boolean result = true;
    if (pants.arePressed()) {
        if (belt.isBuckled()) {
            if (jacket.isBrushed()) {
                if (collar.isFixed()) {
                    if (buttons.areFastened()) {
                        if (cuffs.areAdjusted()) {
                            if (tie.isStraightened()) {
                                result = true;
                            } else {
                                teacher.straighten(tie);
                                result = false;
                            }
                        } else {
                            result = false;
                        }
                    } else {
                        result = false;
                    }
                } else {
                    student.remove(jacket);
                    Jacket substituteJacket = basketService.fetchSubstitute();
                    student.wear(substituteJacket);
                    result = false;
                }
            } else {
                result = false;
            }
        } else {
            result = false;
        }
    } else {
        result = false;
    }
    return result;
}
```

Consider below the functionally equivalent case with guard clauses.

```java
public boolean assemblyCheck() {
    if (!shirt.isIroned()) return false;
    if (!shoelaces.areTied()) return false;
    if (!shoes.arePolished()) return false;
    if (!pants.arePressed()) return false;
    if (!belt.isBuckled()) return false;
    if (!collar.isFixed()) return false;
    if (!buttons.areFastened()) {
        student.remove(jacket);
        Jacket substituteJacket = basketService.fetchSubstitute();
        student.wear(substituteJacket);
        return false;
    }
    if (!jacket.isBrushed()) return false;
    if (!cuffs.areAdjusted()) return false;
    if (!tie.isStraightened()) {
        teacher.straighten(tie);
        return false;
    }
    return true;
}
```

Can you spot the error now? (The nested and early return code snippets are functionally equivalent.)

As computer scientists, our ability to break problems into sub problems before solving them is core to a systems approach. Similarly, in inductive reasoning in maths, we identify and get rid of simpler cases first to make our lives easier.

The error is that the `basketService` code is tied to the `collar.isFixed()` condition instead of the `jacket.isBrushed()` condition where it should be. Nesting clauses fling "else" clauses far down the line, making it non trivial to decipher which code is logically coupled to which case.

Where would you add the new condition, `pants.areTheRightSize()`?

In nested code, introducing another level of nesting at the deepest level can feel intuitive, before any else statements require you to shuffle around brackets. Switching up the order of if-cases is jarring with moving braces and logic to be deciphered to ensure correctness is maintained.

With the early-return style, reordering conditions in most IDEs is a simple select then Alt+Up or Alt+Down, versus nested code where braces have to be carefully aligned.

Consider how much easier it is to add a new condition to the early-return style code. It is much easier to notice in this style also that there is already a condition coupled with pants in the method (`if(!pants.arePressed())`) in this style, and factor it out into a separate method.

For example, here's the refactored version with pants checks extracted:

```java
public boolean assemblyCheck() {
    if (!shirt.isIroned()) return false;
    if (!shoelaces.areTied()) return false;
    if (!shoes.arePolished()) return false;
    if (!checkPants()) return false;
    if (!belt.isBuckled()) return false;
    if (!collar.isFixed()) return false;
    if (!buttons.areFastened()) {
        student.remove(jacket);
        Jacket substituteJacket = basketService.fetchSubstitute();
        student.wear(substituteJacket);
        return false;
    }
    if (!jacket.isBrushed()) return false;
    if (!cuffs.areAdjusted()) return false;
    if (!tie.isStraightened()) {
        teacher.straighten(tie);
        return false;
    }
    return true;
}

private boolean checkPants() {
    return pants.arePressed() && pants.areTheRightSize();
}
```

It is no surprise that in codebases where nesting is preferred, scopes are not regularly extracted and methods tend to run very long.

This harms readability: In 1 huge method, the search space for changes is much larger, compared to the same method split into 5, where you might safely be able to ignore scopes that haven't been touched by changes.

## 2. Smaller Methods and the Namespace Advantage

Refactoring into smaller methods has an often overlooked benefit: better naming. In long methods with large scopes, the namespace shrinks, leading to variable names with 3-4 prefixes that ultimately don't make immediate sense either. In smaller methods, you can use simpler names because the method name itself is part of your mental context window.

Consider this typical long method:

```java
public void processUserOrder() {
    // ... 50 lines of code ...
    
    boolean userOrderPaymentVerified = paymentService.verify(order);
    String userOrderPaymentTransactionId = payment.getTransactionId();
    boolean userOrderInventoryAvailable = inventory.check(order.items);
    String userOrderInventoryReservationId = inventory.reserve(order.items);
    boolean userOrderShippingAddressValid = validateAddress(user.shippingAddress);
    String userOrderShippingTrackingNumber = shipping.createLabel(order);
    
    if (userOrderPaymentVerified && userOrderInventoryAvailable && userOrderShippingAddressValid) {
        // ... more code with these long names ...
    }
}
```

Every variable needs a prefix (`userOrder` + `Payment` + `Verified`) because in a large scope with many concerns, you need to disambiguate. The method handles payment, inventory, and shipping all at once, so variable names become bloated.

Now contrast with smaller, focused methods:

```java
public void processUserOrder() {
    if (!verifyPayment(order)) return;
    if (!reserveInventory(order)) return;
    if (!scheduleShipping(order)) return;
    completeOrder(order);
}

private boolean verifyPayment(Order order) {
    boolean isVerified = paymentService.verify(order);
    String transactionId = payment.getTransactionId();
    
    if (isVerified) {
        order.setTransactionId(transactionId);
        return true;
    }
    return false;
}

private boolean reserveInventory(Order order) {
    boolean isAvailable = inventory.check(order.items);
    String reservationId = inventory.reserve(order.items);
    
    if (isAvailable) {
        order.setReservationId(reservationId);
        return true;
    }
    return false;
}

private boolean scheduleShipping(Order order) {
    boolean isAddressValid = validateAddress(user.shippingAddress);
    String trackingNumber = shipping.createLabel(order);
    
    if (isAddressValid) {
        order.setTrackingNumber(trackingNumber);
        return true;
    }
    return false;
}
```

In small methods, the method name can be relied upon to already be in your mental context window, so you don't need to repeat yourself. The freer namespace enables you to assign more specific, relevant and less ambiguous names.

## 3. Scattered State Mutation and Function Purity

Consider the following shopping basket implementation:

```java
public class ShoppingBasket {
    private List<LineItem> items;
    private double totalCost;
    
    public void addItem(String productId, int quantity, double unitPrice) {
        LineItem item = new LineItem(productId, quantity, unitPrice);
        items.add(item);
        totalCost += unitPrice * quantity;
    }
    
    public void removeItem(String productId) {
        LineItem item = findItem(productId);
        if (item != null) {
            items.remove(item);
            totalCost -= item.getUnitPrice() * item.getQuantity();
        }
    }
    
    public void updateQuantity(String productId, int newQuantity) {
        LineItem item = findItem(productId);
        if (item != null) {
            int oldQuantity = item.getQuantity();
            item.setQuantity(newQuantity);
            totalCost += item.getUnitPrice() * (newQuantity - oldQuantity);
        }
    }
    
    public double getTotalCost() {
        return totalCost;
    }
}
```

`totalCost` is a property of the class. Each method is like an event that updates its value. Can you spot the bug here?

Mutable state is typically obscuring two realities about your code:
1. Every control flow of every method where that property is mutated (`totalCost += whatever`) represents a scattered logic. This logic can be extracted from across those methods and unified into one, new, communicatively named method (such as `calculateTotalCost()` in our case). Doing this leaves methods lighter and more maintainable.
2. Your methods have deceptive return types. The output of your methods is not just the returned object/void, but also the delta of change to each mutated property.

Your job then must be to realize that there is a scope one level above this current scope that is both (1) using your methods, and (2) relying on the state they are mutating. Here is where you should process the mutating state.

Realize the object value in this scope, only when needed, in the proximity of where it is actually being used to make it easier to understand *how* its being used, and name it specific to this context, not reusing names from a scope below that breaks the mental model of encapsulation.

```java
// ViewModel uses calculateTotalCost() and names it for UI context
public class BasketView {
    public double projectedBillWidget(ShoppingBasket basket) {
        ...
        // the "total" in method name is specific to the
        // shopping basket scope, its a total over the
        // line items. variable name "lineItemsCost" is
        // specific to this context. this is better
        // encapsulation. in the view, "total"
        // may otherwise be confused with the basket
        // total, post discount total, maybe the payment
        // is split with a redeemed voucher and loyalty
        // points. a new variable automatically forces you
        // to be more specific about naming
        final lineItemsCost = basket.calculateTotalCost();
        final discounts = ...
        final costCharged = basketCost - discounts;
        return Column(
            "lineItemsCost": "$" + lineItemsCost.with2Decimals,
            "discounts": "- $" + discount.with2Decimals,
            "=========="
            "costCharged": "$" + costCharged.with2Decimals,
        );
    }
}

// Service uses it and names it for business context
public class OrderService {
    public OrderResponse processOrder(ShoppingBasket basket) {
        double chargedAmount = basket.calculateTotalCost() + x + y + z;
        // ... process payment ...
        return new OrderResponse(chargedAmount);
    }
}

public class OrderResponse {
    private final double chargedTotalCost;
    
    public OrderResponse(double chargedTotalCost) {
        this.chargedTotalCost = chargedTotalCost;
    }
}
```

Those who resist this approach often cite the memory cost of immutability over mutable state. But in most cases, defaulting to mutability for memory reasons is premature optimization. The garbage collection of immutable variables in small methods is trivial. If there actually is a memory constraint, profiling should be done and cache mechanisms could be considered, mutability being seen as one such cache mechanism.

Additionally, this is easier to maintain. Tomorrow if you realize you want to change totalCost based on time specific discounts for Christmas or Labor Day or so, you can do that swapping the calculated value with an output from a service that can be swapped at runtime. Just one maintenance scenario that would have been trickier to otherwise accomplish with deeply rooted mutating state.

### Function Purity as a Spectrum

The definition for functions was taught to me in grade school, a function is something that, for a given input, always maps it to the same output.

The input for a function is its arguments. You only need to understand the arguments passed and the method body to be able to make changes to it.

```java
public static double calculateDiscount(double price, double rate) {
    return price * rate;
}
```

The input for methods is their arguments + the properties the object its called on has access to. To make changes to a method, you need to understand the arguments, the method body, and additionally these object properties.

```java
public class PricingCalculator {
    private final double discountRate;
    
    public PricingCalculator(double discountRate) {
        this.discountRate = discountRate;
    }
    
    public double calculateDiscount(double price) {
        return price * discountRate;
    }
}

// Usage
PricingCalculator calculator = new PricingCalculator(0.1);
double discount = calculator.calculateDiscount(100.0);
```

If these object properties were immutable/final, you only need to look at one place, i.e. where the object was initialized, to understand how their value may vary. When these properties are *mutable* however, you have to understand *every* place where this property is mutated.

```java
public class ShoppingCart {
    private double discountRate;  // ← mutable state
    
    public ShoppingCart() {
        this.discountRate = 0.0;  // mutation #1
    }
    
    public void applyMemberDiscount() {
        this.discountRate = 0.1;  // mutation #2
    }
    
    public void applySeasonalDiscount() {
        this.discountRate = 0.15;  // mutation #3
    }
    
    public void applyLoyaltyBonus() {
        this.discountRate += 0.05;  // mutation #4
    }
    
    public double calculateDiscount(double price) {
        return price * discountRate;  // depends on mutations #1-4
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.applyMemberDiscount();
cart.applyLoyaltyBonus();
double discount = cart.calculateDiscount(100.0); // What is discountRate now?
```

Every place where this value is mutated gives rise to n more states that your function now has to cater to. Not catering to these states risks leaving your application in an invalid state. It may be fine today but in a year's time doing this will leave your system riddled with bugs regarding "unexpected" use cases.

It is often even much much worse than this example, because when folks are not careful about mutation, in reality they will be mutating state far beyond the scope of one class into other classes which mutate other classes, and building a mental model of this will be complex in a way that you could mathematically even then demonstrate.

The more immutable properties being used by your methods, the more "impure" your function is, impurity here referring to how dependent your output is on mutable, external variables. For a human or an AI agent, this makes it difficult to reason about, as the search space for changes explodes exponentially, and making changes to such classes becomes slow and risky.

Using the term "function purity" has a tendency to upset software engineers however, and you will find accusations of rigidity and impracticality common in such discussions at the mere mention of this word.

What is important to remember is that purity is a spectrum that you can slide both high and low on. Don't uproot your whole codebase to start implementing this advice. You don't have to be 100% pure 100% of the time. But you do need to understand what it means so you can use it as a warning sign of your code quality.

### When to Break the Rules

These guidelines aren't absolute. There are times when mutable state is the right choice: performance-critical loops, large data structures where copying is prohibitively expensive, stateful systems like game engines or UI frameworks where state machines are idiomatic.

But here's the key: **you yourself will be a bad judge of the exceptions**. The things that are obvious and simple to you as the author of the code will not be similarly obvious to another maintainer or yourself in a week's time.

These styles require learning to see code in a certain way. They don't ask you to refactor a whole codebase today. They're defaults, not additional structures, that keep you in flow state for longer by shrinking the search space for code reviews. Hopefully they will make coding more joyful and less adversarial with AI agents when you have to deal with the 23423th suggested code review of the day.

Comments welcome here;
https://www.linkedin.com/posts/sohaibbaig1_shrinking-the-search-space-for-code-reviews-activity-7456003268729692161-cDW6