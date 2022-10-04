# DSSA Data Gathering & Warehousing
---

**Instructor**: Carl Chatterton <br>
**Term**: Fall 2022 <br>
**Module**: 2 <br>
**Week**: 5

---
# Working with Classes in Python

![img](/assets/img/stop.jfif)

---

A Python __class__ is a blueprint from which objects are created. An __object__ is a collection of data (variables) and methods (functions). A class provides a way of building data and functionality together. Classes let developer's do a few things:
1. Creating a new class creates a new type of object, allowing new instances of that type to be made. 
2. Each class instance can have attributes attached to it for maintaining its state. 
3. Class instances can also have methods (defined by their class) for modifying their state.

### Define a Python Class

To define a function in python we use `def`, to define python classes we use the `class` keyword. The below line creates a python class then created a new instance of the class by calling it.

```python
# Defines the class
class SomeObject:
    """
    The first string inside of a class is called a docstring
    """
    pass

# Creates a new instances of the class
obj = SomeObject()
```

### Adding Attributes to a Python Class

Adding attributes to a class is the same as creating variables

```python
# Defines the class
class SomeObject:
    """
    The first string inside of a class is called a docstring
    """

    # Lets add some attributes to the class
    someAttribute = 40
    anotherAttribute = "This is an attribute"
    _encapsulatedAttribute = "Cannot be modified"

# Creates a new instances of the class
obj = SomeObject()
```

### Adding Methods to a Python Class
A Method is a essentially a python function that is tied to the class instance. 

```python
# Defines the class
class SomeObject:
    """
    The first string inside of a class is called a docstring
    """

    # Lets add some attributes to the class
    someAttribute = 40
    anotherAttribute = "This is an attribute"
    _encapsulatedAttribute = "Cannot be modified"

    # This is a method that prints two attributes
    def printStrings(self):
        print(self.anotherAttribute, _encapsulatedAttribute)
```

### Constructors in Python Classes

Class functions that begin with double underscore __ are called special functions as they have special meaning.

One common constructor is the __init__() function. This special function gets called whenever a new object of that class is instantiated. `__init__()` is normally use it to initialize all the attributes (variables).


The below example creates a class with two attributes name and age using the `__init__()` constructor
```python
# Create a class called person
class Person:
    """
    This class is used to create objects
     of a persons first name and age
    """

    def __init__(self, name, age):
        self.name = name
        self.age = age

# Creates a new instances of the class
p = Person("John", 36)

# Print the new instances attributes
print(p.name)
print(p.age)
```

### The `self` Argument
The self argument is a reference to the current instance of the class, and is used to access variables that belongs to the class.

It does not have to be named self , you can call it whatever you like, but it has to be the first parameter of any function in the class:

```python
# Create a class called person
class Person:
    """
    This class is used to create objects
     of a persons first name and age
    """

    def __init__(someName, name, age):
        someName.name = name
        someName.age = age

# Creates a new instances of the class
p = Person("John", 36)

# Print the new instances attributes
print(p.name)
print(p.age)
```

### Modify Class Attributes or Delete the Class
We can modify class attributes in the same way we can modify variables

```python
# Create a class called person
class Person:
    """
    This class is used to create objects
     of a persons first name and age
    """

    def __init__(self, name, age):
        self.name = name
        self.age = age

# Creates a new instances of the class
p = Person("John", 36)

# Print the new instances attributes
print(p.age)

# Modify the attributes value and print again
p.age = 40
print(p.age)

# Delete the instance
del p

```

### The `pass` Statement
class definitions cannot be empty, but often we need to create a placeholder for code or functionality to be added later. Put in the pass statement to avoid getting an error.

```python
class Person:
    pass
```

### Other built in attributes of Python Classes

Along with the other attributes, a Python class also contains some built-in class attributes which provide information about the class.

| **Num** | **Attribute** | **Description** |
|---|---|---|
| 1 | \_\_dict\_\_ | It provides the dictionary containing the information about the class namespace\. |
| 2 | \_\_doc\_\_ | It contains a string which has the class documentation |
| 3 | \_\_name\_\_ | It is used to access the class name\. |
| 4 | \_\_module\_\_ | It is used to access the module in which, this class is defined\. |
| 5 | \_\_bases\_\_ | It contains a tuple including all base classes\. |


```python
class Student:    

    def __init__(self,name,id,age):    
        self.name = name    
        self.id = id    
        self.age = age    

    def display_details(self):    
        print("Name:%s, ID:%d, age:%d"%(self.name,self.id))    

s = Student("John",101,22)    
print(s.__doc__)    
print(s.__dict__)    
print(s.__module__)    
```
Returns
```
None
{'name': 'John', 'id': 101, 'age': 22}
__main__
```

### Abstract Classes

An abstract class is a class that is declared abstract —it may or may not include abstract methods. 

Abstract classes <u>cannot be instantiated</u>, but they can be subclassed. When an abstract class is subclassed, the subclass usually provides the implementation details for all of the abstract methods in its parent class.

![img](/assets/img/abstract-class-example.png)


```python

# we use the abc library to define abstract base classes (abc)
from abc import ABCMeta, abstractmethod
 

# The abstract class cannot be use only subclassed (inheritance)
class AbstractBaseClassExample(metaclass=ABCMeta):):
    
    @abstractmethod
    def do_something(self):
        raise NotImplementedError("Must Be implemented by the inheriting classes.")


# We can now provide the implementation in the concrete class
class ConcreteSubclass(AbstractBaseClass):

    def do_something(self):
        print("OOP Is lots of fun!")

# Lets see what happens when we forget to provide implementation details to a concrete class 
# that inherits from an abstract base class
class BadConcreteSubclass(AbstractBaseClass):

   def do_something_else(self):
      print('I forgot to include the implementation for do_something!!!')


x = ConcreteSubclass()
x.do_something()

y = BadConcreteSubclass()
y.do_something_else()



    # Now lets see this in action
>>> x = ConcreteSubClassExample()
>>> x.do_something()
OOP Is lots of fun!


>>> y = BadConcreteSubclass()
>>> y.do_something_else()
"examples.py", line 32, in <module>
    y = BadConcreteSubclass()
TypeError: Can't instantiate abstract class BadConcreteSubclass with abstract method do_something
```

__Wrapping up__

An __abstract base class__ is a class that is used as a blueprint for other classes. Abstract base classes are a powerful feature in Python since they help you define a blueprint for other classes that may have something in common.

A __concrete class__ is a class that has an implementation for all of its methods. They cannot have any unimplemented methods. It can also extend an abstract class or implement an __interface__ as long as it implements all their methods. It is a complete class and can be instantiated.

Interfaces and Abstract classes are similar but have their  differences. This really refers to how interfaces are defined and enforced in other programming languages like Java. In python a we simply use abstracted base classes and set the metaclass to `abc.ABCMeta` to prevent the abstract base class from being instantiated until all methods are defined.


### Mixin Classes

A _Mixin__ is a class that contains methods for use by other classes without having to be the parent class of those other classes. In python, mixins are used for multiple inheritance. 

There are two main situations where mixins are used:

* You want to provide a lot of optional features for a class.
* You want to use one particular feature in a lot of different classes

Mixins should be narrow in scope and not meant to be extended. They are meant to be "mixed in" to any other class using without affecting the inheriting class while still offering some beneficial functionality

___Think of (mixins as) an interface that is already implemented.___ 

```python

# This is our base (or super) class
class PlainPizza:

    # the parent class gives us this constructor
    def __init__(self):
         self.toppings = []

# This is our mixin 
class OlivesMixin:

    # the mixin gives us olive functionality
    def add_olives(self):
         print("Adding olives!")
         self.toppings += ['olives']


# This is our mixin 
class SausagesMixin:

    # the mixin gives us sausage functionality
    def add_sausage(self):
         print("Adding sausage!")
         self.toppings += ['sausage']


# We inherit the Parent Class Pizza but also can use the Mixin for added functionality
class MyFriendsPizza(OlivesMixin, PlainPizza):

    def prepare_pizza(self):
         self.add_olives()


class MyPersonalPizza(OlivesMixin, SausagesMixin, PlainPizza):

    def prepare_pizza(self):
         self.add_olives()
         self.add_sausage()
        
        
>>> # Lets give it a try
>>> my_friends_pizza_order = MyFriendsPizza()
>>> my_pizza_order = MyPersonalPizza()

>>> # Makes the friends pizza order
>>> my_friends_pizza_order.prepare_pizza()
Adding olives!

>>> # Makes my pizza order
>>> my_pizza_order.prepare_pizza()
Adding olives!
Adding sausage!

```