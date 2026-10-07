### Open/Closed Principle(OCP)

## Table of Contents

- [Open/Closed Principle(OCP)](#openclosed-principleocp)
- [Strategy design pattern.](#strategy-design-pattern)
- [The Decorator design pattern](#the-decorator-design-pattern)


---

We will now go to the next principle on my list of the SOLID principles of Object Oriented software design--**The Open/Closed Principle**. This principle states that **A software artifact should be closed for modification but open for extension**.

At first, this definition seems to be a paradox. How can a software module be closed for modification but open for extension?? Well, we shall see how achieve this goal with the principles we shall discuss in this section.

This term was first coined in 1988 by **Bertran Meyer** in his book **Object-Oriented Software Construction (Prentice Hall)**. The modern definition of this principle was offered by Martin Roberts and goes as follows

> **Open for extension** : This means that the behavior of the module can be extended.
> As the requirements of the application change, we are able to extend the module
> with new behaviors that satisfy those changes. In other words, we are able to
> change what the module does. <br/> <br/> > **Closed for modification** : Extending the behavior of a module does not result
> in changes to the source or binary code of the module. The binary executable
> version of the module, whether in a linkable library, a DLL, or a Java .jar, remains untouched.

There are 2 exceptions to this rule. Code can be edited if :

1. Fixing bugs.

If a module contains a bug, we can either choose to write a new similar module without the bugs but this would be an overkill solution. So we tend to prefer fixing the buggy module to writing a new one.

2. Client awareness.

Another situation where it is possible to edit the source code of a module is when the changes don't affect the client of the module.This places an
emphasis on how coupled the software modules are, at all levels of granularity: between classes and
classes, assemblies and assemblies, and subsystems and subsystems.

If a change in one class forces a change in another, the two are said to be tightly coupled. Conversely, if a class can change in isolation without forcing other classes to change, the participating
classes are loosely coupled. At all times and at all levels, loose coupling is preferable. Maintaining
loose coupling limits the impact that the OCP has if you allow modifications to existing code that
does not force further changes to clients

To illustrate the OCP rule, we are going to use the following techniques

1. Strategy pattern
1. Decorator design pattern

#### Strategy design pattern.

---

> **Strategy Pattern** <br/> Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from the clients that use it.

This definition seems abstract enough but we are going to try explaininig it in the following example.
Consider the following class that is part of an e-commerce application. The class contains a method that selects the which payment to choose for settling a payment as shown below.

**Bad** :angry:

```python
class OnlineCart:
    def check_out(self, payment_type: str) -> None:
        if payment_type == 'creditCard':
            self.process_credit_card_payment()
        elif payment_type == 'payPal':
            self.process_paypal_payment()
        elif payment_type == 'GoogleCheckout':
            self.process_google_payment()
        elif payment_type == 'AmazonPayments':
            self.process_amazon_payment()
        else:
            pass

    def process_credit_card_payment(self):
        print('paying with credit card...')

    def process_paypal_payment(self):
        print('Paying with paypal...')

    def process_google_payment(self):
        print('Paying with google check out')

    def process_amazon_payment(self):
        print('Paying with amazon ...')
```

The above class is neither extendable nor flexible. If a new payment method comes up, the conditional logic will have to be changed and a new method added to the class. This class violets the OCP rule and thus needs to be refactored.

There are many ways to solve this simple problem but I will stick with the original solution proposed by the GoF programmers. We will use the **strategy pattern**. We will model each payment method as a class and we will use composition and inject in the payment strategy at run-time.

**Good** :smile:

```python
from abc import ABC, abstractmethod

class Payment(ABC):
    def __init__(self, payment_id: str):
        self.id = payment_id

    @abstractmethod
    def pay(self):
        pass

```

This is the interface that all payment strategies are supposed to implement. We are going to use to create a family of payment strategies.

**Good** :smile:

```python
class OnlineCart:
    def __init__(self, payment: Payment) -> None:
        if isinstance(payment, Payment):
            self.payment = payment
        else:
            raise AssertionError('Bad argument')

    def check_out(self):
        self.payment.pay()


```

The `OnLineCart` class no longer contains the conditional logic and all the corresponding methods have been pulled out. They will be implemented by the corresponding payment strategies as the following code snippet reveals.

**Good** :smile:

```python

class CreditCard(Payment):
    def __init__(self, *, card_number: str) -> None:
        Payment.__init__(self, card_number)

    def pay(self) -> None:
        print(f'Payment made with card number {self.id}')


class Paypal(Payment):
    def __init__(self, *, paypal_id: str) -> None:
        Payment.__init__(self, paypal_id)

    def pay(self) -> None:
        print(f'Payment made with paypal id {self.id}')


class GoogleCheckOut(Payment):
    def __init__(self, *, google_checkout: str) -> None:
        Payment.__init__(self, google_checkout)

    def pay(self) -> None:
        print(f'Payment made with google checkout with id {self.id}')


class AmazonPayment(Payment):
    def __init__(self, *, amazon_payment: str) -> None:
        Payment.__init__(self, amazon_payment)

    def pay(self) -> None:
        print(f'Payment made with amazon services using id {self.id}')


```

To use the `OnlineCart` class, we inject in the payment strategy to use for making the payment as shown in the following snippet.

```python
# we are paying using paypal
paypal : Payment = PayPal(paypal_id='ERTWF342T')
cart : OnlineCart = OnlineCart(paypal)
cart.check_out()
```

Note that the `OnlineCart` class no longer cares about which payment method is being used, it delegates that responsibility to the wrapped object. `OnlineCart` is now open for extension (we can change its behavior by passing in different objects) but it is closed for modification (we don't change its source code to add new functionality).

**Payment strategy family**

| Class | Relationship | Members |
| --- | --- | --- |
| `OnlineCart` | Holds a `Payment` strategy (aggregation). | Private `payment_strategy: Payment`; public `check_out(): void`. |
| `Payment` | Abstract parent of the strategy family. | Public `pay(): void`. |
| `CreditCard` | Inherits from `Payment`. | Public `pay(): void`. |
| `PayPal` | Inherits from `Payment`. | Public `pay(): void`. |
| `GoogleCheckOut` | Inherits from `Payment`. | Public `pay(): void`. |
| `AmazonPayment` | Inherits from `Payment`. | Public `pay(): void`. |

The strategy family consists of `Payment` and its four implementations; `OnlineCart` is the client outside
that family. The cart calls the selected strategy through the common `Payment` interface.

From this class structure, we can notice that once a new payment method shows up, we just create a new class for that method, inherit from `Payment` and inject it in `OnlineCart`. This code is flexible and extendable.

> **Note**: The same design could be achieved with lambda expressions though at times the logic in the respective strategiess may be complex enough that it is implemented in more than one function. This is why i decided to use this rather verbose method.

#### The Decorator design pattern

---

> **Decorator Pattern**<br/>Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to sub-classing for extending functionality.

The decorator design pattern was first proposed in 1994 in the seminal work of the Gang of Four book. It is a technique of adding capabilities to a class without changing its source code.
We are going to view this under various examples.

We are going to continue with our example in the previous section. Consider that we want to add some logging information after making the payment. There are many ways to solve this problem and one of them is to edit the `OnlineCart` class to add logging features as shown below;

```python
class OnlineCart:
    def __init__(self, payment: Payment) -> None:
        if isinstance(payment, Payment):
            self.payment = payment
        else:
            raise AssertionError('Bad argument')

    def check_out(self):
        self.payment.pay()
        print('Logging payment')
```

The added `print('Logging payment')` line puts console logging directly into `check_out()`.

This is a **code smell**. Notice that we have modified the class, this is violation of the Open/Closed principle. There is even a more serious problem than this one. In this case we are logging to the console, what will happen if we want to log to a database or to a text file? We will have to constantly open this class and modify it, this is serious violation of the OCP rule.

One solution to this problem is the **decorator pattern**. Decorators are just classes that wrapper other classes. The **wrapped classes** have exactly the same interface as the **wrapper classes**. We achieve this by using both **composition** and **inheritance** as the following class structure reveals.

| Class | Inherits from | Members |
| --- | --- | --- |
| `Component` | — | Abstract public `Operation(): void`. |
| `ConcreteComponent` | `Component` | Public `Operation(): void`. |
| `Decorator` | `Component` | Protected `component: Component`; public `Operation(): void` and `SetComponent(Component): void`. |
| `ConcreteDecoratorA` | `Decorator` | Private `addedState: String`; public `Operation(): void`. |
| `ConcreteDecoratorB` | `Decorator` | Public `AddedBehavior(): void` and `Operation(): void`. |

`Component` and `Decorator` are abstract. In addition to inheriting the component interface, `Decorator`
holds a reference to a wrapped `Component` (aggregation). `SetComponent` supplies that wrapped object.
`ConcreteDecoratorA` adds state; `ConcreteDecoratorB` adds behaviour. Their `Operation` methods can
extend the wrapped component's operation while keeping the same public interface.

In this case, the `Component` is an interface that is both supported by `ConcreteComponent` and `Decorator` this means that both `ConcreteComponent` and `Decorator` can be swapped without breaking existing code.

Notice also that the `Decorator` contains a `Component` inside it implying that it delegates some of its tasks to the wrapped component.

Let us use this technique to add logging capabilities to the `OnlineCart` class without having to modify it.

Looking very closely at the code for `OnlineCart`, notice that the `Payment` object is being injected during the instantiation of the class and so the `OnlineCart` class doesn't control what type of payment it receives (remember `Payment` is polymorphic).

```python
class OnlineCart:
    def __init__(self, payment: Payment) -> None:
        if isinstance(payment, Payment):
            self.payment = payment
        else:
            raise AssertionError('Bad argument')

    def check_out(self):
        self.payment.pay()
```

The injection points are the `payment: Payment` constructor parameter, the assignment
`self.payment = payment`, and the delegated call `self.payment.pay()`. The cart stores and calls the
supplied payment object without constructing a concrete payment implementation itself.

This means we can inject in anything that is similar to `Payment`. That is what the **Decorator pattern** is based on. **Dependency Injection** is the prerequisite to achieving all this flexibility. We shall cover dependency injection fully under the **Dependency Inversion Principle (DIP)**.

The following call paths show the idea behind the decorator pattern:

```text
Without a decorator: OnlineCart -> Payment
With a decorator:    OnlineCart -> Decorator -> Payment
```

The decorator intercepts the call to the payment object. It exposes the same interface as `Payment`,
so `OnlineCart` can call it without noticing the extra layer. The cart still depends on the `Payment`
abstraction; the object supplied at runtime is now a decorator that forwards the call to a wrapped payment.

Ok!! time for some code.

Below is the code for our abstract decorator class.

```python
class Decorator(Payment, ABC):
    def __init__(self, payment: Payment) -> None:
        if isinstance(payment, Payment):
            self.payment = payment
        else:
            raise AssertionError('Bad argument')

    @abstractmethod
    def pay(self):
        pass
```

Pay close attention to this class.

1. It uses multiple inheritance : This is because we need the class to both be abstract and still inherit from the `Payment` class.
2. It takes in a `Payment` dependency and inherits from `Payment`. This is typical of decorator classes.
3. The `pay()` method is abstract since concrete decorators will have to define their implementations.

We can now implement our `Logging` decorator that adds logging capabilities to the `OnlineCart` class without modifying it. Let's go!!!

```python
class ConsoleLoggingDecorator(Decorator):
    def __init__(self, payment: Payment):
        Decorator.__init__(self, payment)

    def pay(self):
        self.payment.pay()
        print(f'Logging Payment made with id {self.payment.id}')
```

Simple!!! This is the console logging decorator because it logs on the console using `print`. We can create a file logging decorator that logs into a text file as shown below.

```python
class FileLoggingDecorator(Decorator):
    def __init__(self, *, payment: Payment, filename: str):
        Decorator.__init__(self, payment)
        self.filename = filename

    def pay(self):
        self.payment.pay()
        _file = open(self.filename, 'a')
        _file.writelines(f'{self.payment.id} payment logged\n')
        _file.close()
```

The snippet shows the code that sets up the `OnlineCart` class to use the `ConsoleLoggingDecorator` and then the `FileLoggingDecorator`

```python
#using the console logger
credit_card: Payment = CreditCard(card_number='RTGW@#')
decorator: FileLoggingDecorator = ConsoleLoggingDecorator(payment=credit_card)
cart: OnlineCart = OnlineCart(decorator)
cart.check_out()

#using the file logger
credit_card: Payment = CreditCard(card_number='RTGW@#')
decorator: FileLoggingDecorator = FileLoggingDecorator(filename='dta.txt', payment=paypal)
cart: OnlineCart = OnlineCart(decorator)
cart.check_out()
```

**Note**: We first create a bare payment object and then wrap it in a decorator which we then inject into `OnlineCart` class. The decorator adds some capabilities (in this case logging) to the payment object.

We can add any functionality to the `OnlineCart` class without modifying it. Capabilities like Profiling, Laziness, Immutability, e.t.c.

The following class structure shows our work up to this point in time.

| Class | Relationship | Members |
| --- | --- | --- |
| `Payment` | Abstract payment interface. | Private `id: String`; public `pay(): void`. |
| `OnlineCart` | Holds a payment strategy, which can be a decorator. | Private `payment_strategy: Payment`; public `check_out(): void`. |
| `Decorator` | Abstract subclass of `Payment`; wraps another `Payment`. | Private `payment: Payment`; public `pay(): void`. |
| `PayPal` | Concrete subclass of `Payment`; a bare payment object. | Public `pay(): void`. |
| `CreditCard` | Concrete subclass of `Payment`; a bare payment object. | Public `pay(): void`. |
| `ConsoleLoggingDecorator` | Concrete subclass of `Decorator`. | Public `pay(): void`; adds console logging. |
| `FileLoggingDecorator` | Concrete subclass of `Decorator`. | Public `pay(): void`; adds file logging. |

The two bare payment classes form one group, and the two logging decorators form another. `OnlineCart`
holds the decorator, and the decorator holds the bare payment object; both are aggregation relationships.
Because `Decorator` also inherits from `Payment`, the cart's declared dependency remains `Payment`.
The call sequence is `OnlineCart.check_out()` → the selected decorator's `pay()` → the wrapped payment's
`pay()`, followed by the decorator's logging operation.

We can add more organization to the decorator hierachy by using the **Template pattern**. That will be a story for the next time.

