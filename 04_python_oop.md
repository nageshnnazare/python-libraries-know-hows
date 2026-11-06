# 04 - Object-Oriented Programming

Classes bundle data (attributes) with behavior (methods). You will read a lot of
OOP code in libraries even if you write mostly functions. This file also covers
dataclasses, which are the pragmatic default for data-holding objects.

## 1. Why classes

```
   Procedural:  data and functions are separate; you pass data around.
   OOP:         data and the functions that operate on it live together.

   ┌─────────────────────────────┐
   │ class BankAccount           │
   │  state:  balance, owner     │  <- attributes (data)
   │  behavior: deposit(), ...   │  <- methods (functions bound to state)
   └─────────────────────────────┘
```

Use a class when you have state that multiple functions share and mutate, or when
you want multiple independent instances. For stateless transforms, a function is
better.

## 2. Defining a class

```python
class BankAccount:
    """A simple bank account."""

    def __init__(self, owner, balance=0):   # constructor / initializer
        self.owner = owner        # instance attribute
        self.balance = balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("amount must be positive")
        self.balance += amount
        return self.balance

    def withdraw(self, amount):
        if amount > self.balance:
            raise ValueError("insufficient funds")
        self.balance -= amount
        return self.balance

acct = BankAccount("Ada", 100)   # __init__ runs automatically
acct.deposit(50)                 # 150
acct.owner                       # 'Ada'
```

### `self` explained

`self` is the instance the method is called on. Python passes it automatically:
`acct.deposit(50)` is really `BankAccount.deposit(acct, 50)`.

```
   acct.deposit(50)
       │
       └──> BankAccount.deposit(self=acct, amount=50)
                                 └─ the instance is bound to self
```

You must declare `self` as the first parameter of instance methods. Forgetting it
is a very common beginner error.

## 3. Class vs instance attributes

```python
class Dog:
    species = "Canis familiaris"    # class attribute: shared by ALL instances
    def __init__(self, name):
        self.name = name            # instance attribute: unique per instance

a = Dog("Rex"); b = Dog("Fido")
a.species, b.species       # both 'Canis familiaris' (shared)
a.name, b.name             # 'Rex', 'Fido' (independent)

Dog.species = "changed"    # affects all instances that didn't override it
a.species                  # 'changed'
```

```
             ┌───────────────────────┐
             │ class Dog             │
             │  species = "..."      │  <- one copy, shared
             └──────────┬────────────┘
              instances │
        ┌───────────────┼───────────────┐
        v                               v
   ┌──────────┐                   ┌──────────┐
   │ a        │                   │ b        │
   │ name=Rex │                   │ name=Fido│
   └──────────┘                   └──────────┘
```

Mutable class attributes are a trap (all instances share the same list):

```python
class Bad:
    items = []             # shared! appending via one instance affects all
class Good:
    def __init__(self):
        self.items = []    # per-instance, correct
```

## 4. Methods: instance, class, static

```python
class Circle:
    pi = 3.14159

    def __init__(self, radius):
        self.radius = radius

    def area(self):                     # instance method: uses self
        return self.pi * self.radius**2

    @classmethod
    def from_diameter(cls, d):          # class method: uses cls, alt constructor
        return cls(d / 2)               # cls = the class itself

    @staticmethod
    def describe():                     # static method: no self/cls
        return "A round shape"

c = Circle(5)
c.area()                    # 78.53...
c2 = Circle.from_diameter(10)   # alternative constructor -> radius 5
Circle.describe()           # 'A round shape'
```

```
   instance method  def m(self):   operates on a specific instance
   class method      @classmethod  def m(cls):  operates on the class
                                     (common for alternative constructors)
   static method     @staticmethod def m():  just namespaced; no self/cls
```

## 5. Encapsulation and properties

Python has no true private attributes; it uses naming conventions.

```python
class Account:
    def __init__(self, balance):
        self.owner = "public"       # public
        self._internal = 0          # single _ : "internal, don't touch" (convention)
        self.__private = balance    # double __ : name-mangled to _Account__private
```

`__name` triggers name mangling (`_ClassName__name`) to avoid accidental override
in subclasses; it is not real privacy.

### Properties: computed attributes with validation

```python
class Temperature:
    def __init__(self, celsius=0):
        self._celsius = celsius

    @property
    def celsius(self):              # getter: access as attribute, not method
        return self._celsius

    @celsius.setter
    def celsius(self, value):       # setter: validation on assignment
        if value < -273.15:
            raise ValueError("below absolute zero")
        self._celsius = value

    @property
    def fahrenheit(self):           # computed, read-only
        return self._celsius * 9/5 + 32

t = Temperature(25)
t.celsius            # 25  (calls getter, looks like attribute access)
t.celsius = 30       # calls setter (validates)
t.fahrenheit         # 86.0 (computed on the fly)
# t.fahrenheit = 100 # AttributeError: no setter defined
```

Properties let you start with a plain attribute and later add validation or
computation without changing the public interface. This is the Pythonic
alternative to Java-style getters/setters.

## 6. Inheritance

A subclass inherits attributes and methods from a parent (base) class and can add
or override them.

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def speak(self):
        return "..."
    def describe(self):
        return f"{self.name} says {self.speak()}"

class Dog(Animal):
    def speak(self):                 # override
        return "Woof"

class Cat(Animal):
    def __init__(self, name, indoor=True):
        super().__init__(name)       # call parent __init__
        self.indoor = indoor
    def speak(self):
        return "Meow"

Dog("Rex").describe()    # 'Rex says Woof'  (describe uses overridden speak)
Cat("Tom").speak()       # 'Meow'
```

```
        ┌──────────┐
        │  Animal  │   base class
        │ speak()  │
        └────┬─────┘
       ┌─────┴─────┐
       v           v
   ┌──────┐    ┌──────┐
   │ Dog  │    │ Cat  │   subclasses override speak()
   └──────┘    └──────┘
```

`super()` accesses the parent class; use it to extend (not fully replace) parent
behavior, especially in `__init__`.

### isinstance and issubclass

```python
d = Dog("Rex")
isinstance(d, Dog)         # True
isinstance(d, Animal)      # True (Dog IS an Animal)
issubclass(Dog, Animal)    # True
type(d) is Dog             # True (exact type, ignores inheritance)
```

Prefer `isinstance` over `type(x) ==` because it respects inheritance.

### Method Resolution Order (MRO) and multiple inheritance

```python
class A: 
    def m(self): return "A"
class B(A): 
    def m(self): return "B"
class C(A): 
    def m(self): return "C"
class D(B, C):    # multiple inheritance
    pass

D().m()           # 'B'
D.__mro__         # (D, B, C, A, object) -- the lookup order (C3 linearization)
```

Python resolves methods along the MRO. Multiple inheritance is powerful but easy
to misuse; prefer composition unless you specifically need mixins.

## 7. Composition (often better than inheritance)

"Has-a" instead of "is-a": build objects from other objects.

```python
class Engine:
    def start(self):
        return "engine started"

class Car:
    def __init__(self):
        self.engine = Engine()      # Car HAS an Engine
    def start(self):
        return self.engine.start()  # delegate
```

```
   Inheritance:  Car IS-A Vehicle       (tight coupling)
   Composition:  Car HAS-A Engine       (flexible; swap parts freely)
```

Rule of thumb: favor composition; reach for inheritance only for genuine subtype
relationships and shared interfaces.

## 8. Dunder (magic) methods

Special methods with double underscores customize how objects behave with built-in
operations. These make your classes feel native.

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):                 # unambiguous, for developers/debugging
        return f"Vector({self.x}, {self.y})"

    def __str__(self):                  # readable, for end users (print)
        return f"({self.x}, {self.y})"

    def __eq__(self, other):            # == comparison
        return (self.x, self.y) == (other.x, other.y)

    def __add__(self, other):           # + operator
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):          # * operator
        return Vector(self.x * scalar, self.y * scalar)

    def __len__(self):                  # len()
        return 2

    def __getitem__(self, i):           # indexing v[0], v[1], enables iteration
        return (self.x, self.y)[i]

    def __abs__(self):                  # abs()
        return (self.x**2 + self.y**2) ** 0.5

v = Vector(1, 2) + Vector(3, 4)   # Vector(4, 6)
v * 2                             # Vector(8, 12)
print(v)                          # (4, 6)   (uses __str__)
v                                 # Vector(4, 6)  (uses __repr__ in REPL)
Vector(1, 2) == Vector(1, 2)      # True
abs(Vector(3, 4))                 # 5.0
```

```
   Operation you write   Dunder that runs
   ──────────────────────────────────────
   a + b                 a.__add__(b)
   a == b                a.__eq__(b)
   len(a)                a.__len__()
   a[i]                  a.__getitem__(i)
   str(a) / print(a)     a.__str__()
   repr(a)               a.__repr__()
   a < b                 a.__lt__(b)
   x in a                a.__contains__(x)
   for x in a            a.__iter__()
```

`__repr__` vs `__str__`: `__repr__` is for developers (should ideally be valid
Python to recreate the object); `__str__` is for users. If you define only one,
define `__repr__` (it is the fallback for both).

## 9. Dataclasses (the pragmatic default)

For classes that mostly hold data, `@dataclass` auto-generates `__init__`,
`__repr__`, `__eq__`, and more. Prefer this over hand-writing boilerplate.

```python
from dataclasses import dataclass, field

@dataclass
class Point:
    x: float
    y: float
    label: str = "origin"        # default value

p = Point(1.0, 2.0)
p                        # Point(x=1.0, y=2.0, label='origin')  (auto __repr__)
Point(1, 2) == Point(1, 2)   # True  (auto __eq__)

@dataclass(frozen=True)          # immutable + hashable (usable as dict key/set)
class Config:
    lr: float = 0.01
    epochs: int = 10

@dataclass
class Team:
    name: str
    members: list = field(default_factory=list)   # correct mutable default
```

`field(default_factory=list)` is the dataclass-correct way to default to a mutable
value (avoids the shared-list trap from file 03).

Trade-off: dataclasses add zero dependencies and eliminate boilerplate. Use them
for records, configs, and simple value objects. For behavior-rich classes, a
normal class is fine.

## 10. Class practical patterns for data science

```python
# A reusable, stateful transformer (mirrors scikit-learn's API design):
class StandardScaler:
    def __init__(self):
        self.mean_ = None
        self.std_ = None

    def fit(self, data):
        self.mean_ = sum(data) / len(data)
        var = sum((x - self.mean_) ** 2 for x in data) / len(data)
        self.std_ = var ** 0.5
        return self                        # enable method chaining

    def transform(self, data):
        if self.mean_ is None:
            raise RuntimeError("call fit() before transform()")
        return [(x - self.mean_) / self.std_ for x in data]

    def fit_transform(self, data):
        return self.fit(data).transform(data)

scaler = StandardScaler()
scaler.fit_transform([1, 2, 3, 4, 5])
```

This fit/transform pattern with stored `_`-suffixed learned parameters is exactly
scikit-learn's convention (file 14).

## Exercises

1. Build a `Rectangle` class with a `.area` property and a `from_square(side)`
   classmethod.
2. Add `__add__`, `__repr__`, and `__eq__` to a `Money` class (amount + currency);
   raise if currencies differ on `+`.
3. Convert the `Point` class to a frozen dataclass and use it as a dict key.
4. Model `Shape` -> `Circle`/`Square` with a polymorphic `area()` and a function
   that sums areas of a mixed list.
5. Explain why `class C: data = []` is dangerous and fix it.

Proceed to `05_python_advanced.md`.
