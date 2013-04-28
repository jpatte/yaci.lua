# yaci.lua - Yet Another Class Implementation for Lua

original page: [YetAnotherClassImplementation](http://lua-users.org/wiki/YetAnotherClassImplementation) from the Lua-wiki (2007)
## Introduction

I've seen several implementations for classes which suggest how to use metatables to simulate 
object oriented aspects like intanciation or inheritance (see for example 
[ObjectOrientationTutorial](http://lua-users.org/wiki/ObjectOrientationTutorial), 
[LuaClassesWithMetatable](http://lua-users.org/wiki/LuaClassesWithMetatable), 
[InheritanceTutorial](http://lua-users.org/wiki/InheritanceTutorial), 
[ClassesAndMethodsExample](http://lua-users.org/wiki/ClassesAndMethodsExample) 
and [SimpleLuaClasses](http://lua-users.org/wiki/SimpleLuaClasses).
However I thought it should be possible to go even further than that by adding some additional features and facilities. 
This is why I suggest here yet another implementation, which is mainly based on the other ones
but with some additional stuff in it. I don't pretend it to be the best way but I think it could be useful 
for some other persons than me.

Note that this code has been designed to be as comfortable as possible to use; therefore this is certainly not 
the fastest way of doing things. It is rather complicated, it makes intensive use of metatables, upvalues, proxies... 
I tried to optimize it a lot but I'm not an expert thus maybe there were some shortcuts that I didn't know yet.

## Features

### Content

This code "exports" only 2 things: the base class `Object`, and a function `newclass()`.

### Class Definition

There are basically 2 ways of defining a new class: by calling `newclass()` or by using `Object:subclass()`. 
Those functions return the new class.

When creating a class you should specify a name for it. This is not absolutely required, but it could be helpful 
(for debugging purposes etc). If you don't give any name the class will be called "Unnamed". 
Having several unnamed classes is not a problem.

When you use `Object:subclass()`, the new class will be a direct subclass of `Object`. 
However `newclass()` accepts a second argument, which can be another superclass than `Object`.
If you don't specify any superclass the class `Object` will be chosen; that means that all classes 
are subclasses of `Object`. Note that each class also has `subclass()` method that you can use.

Let's take an example to illustrate this:
```lua
-- 'LivingBeing' is a subclass of 'Object'
LvingBeing = newclass("LivingBeing")

-- 'Animal' is a subclass of 'LivingBeing'
Animal = newclass("Animal", LivingBeing)

-- 'Vegetable' is another subclass of 'LivingBeing'
Vegetable = LivingBeing:subclass("Vegetable")

-- create some other classes...
Dog = newclass("Dog", Animal)
Cat = Animal:subclass("Cat")
Human = Animal:subclass("Human")
Carrot = newclass("Carrot", Vegetable)
```
Note that the exact code of `newclass()` is
```lua
function newClass(name, baseClass)
  baseClass = baseClass or Object
  return baseClass:subClass(name)
end
```
It was just added for convenience.

### Methods Definition

Methods are created in a rather natural way:
```lua
function Animal:eat()
  print "An animal is eating..."
end

function Animal:speak()
  print "An animal is speaking..."
end

function Dog:eat()
  print "A dog is eating..."
end

function Dog:speak()
  print "Wah, wah!"
end

function Cat:speak()
  print "Meoow!"
end

function Human:speak()
  print "Hello!"
end 
```
The method `init()` is treated as a constructor. So:
```lua
function Animal:init(name, age)
  self.name = name
  self.age = age
end

function Dog:init(name, age, master)
  self.super:init(name, age)   -- notice call to superclass's constructor
  self.master = master
end

function Cat:init(name, age)
  self.super:init(name, age)
end

function Human:init(name, age, city)
  self.super:init(name, age)
  self.city = city
end
```
Subclasses may call the constructor of their superclass through the field `super` (See below). 
Note that `Object:init()` exists but does nothing, so it is not required to call it.

### Events Definition

You may also define events for the class instances, exactly in the same way as for the methods:
```lua
function Animal:__tostring()
  return "An animal called " .. self.name .. " and aged " .. self.age 
end

function Human:__tostring()
  return "A human called " .. self.name .. " and aged " .. self.age .. ",
         living at " .. self.city
end
```
Any events could be used, excepted `__index` and `__newindex` which are needed for OO implementation. 
You can use this feature to define operators like `__add`, `__eq` etc. 
`__tostring` is a really useful event here, therefore the class `Object` implements a default version for it 
which simply returns a string "a xxx" where 'xxx' is the name of the instance's class.

### Instanciation

Each class has a method `new()`, used for instanciation. All arguments are forwarded to the instance's constructor.
```lua
Robert = Human:new("Robert", 35, "London")
Garfield = Cat:new("Garfield",  18)
```
The result is the same if you "call" the classes directly:
```lua
Mary = Human("Mary", 20, "New York")
Albert = Dog("Albert", 5, Mary)
```

### Classes methods

Besides `subclass()`and `new()`, each class exposes several other methods:

- `inherits()` can be used to check if a class inherits another class. 
  For example, `Human:inherits(Animal)` returns true, and `Vegetal:inherits(Dog)` returns false 
  (Mainly used for internal purposes)
- `name()` returns the class's name (the one you specified when you created it).
- `super()` returns the superclass
- `made()` is used to check if an instance implements this class or not. 
  For example, `Dog:made(Albert)` returns `true `while `Cat:made(Robert)` returns `false`; 
  however, `Animal:made(Albert)` and `Animal:made(Robert)` both return `true`. I preferred this way 
  to an `isa()` method in the instances (e.g. `Albert:isa(Dog)`), because `isa()` would require that `Albert` 
  in that case is really an instance (and not a string or a function etc), and we cannot be totally sure 
  of the variable's type without testing it (think of function arguments for example). 
  Here, `made()` will also check the argument's type for you.
- `virtual()` is used to declare abstract & virtual methods explicitely (see below).
- `cast()` & `trycast()` are used for casting. See below for details.

### Instances methods

Every instances permit access to the variables defined in the constructor of their class (and of their superclasses). 
They also have a `class()` method returning their class, and a field `super` used to access the superclass's members 
if you overrode them. For example:
```lua
A = newclass("A")
function A:test() print(self.a) end
A:virtual("test") -- declare test() as being virtual; see below
function A:init(a) self.a = a end

B = newclass("B", A)
function B:test() print(self.a .. "+" .. self.b) end
function B:init(b) self.super:init(5) self.b = b end

b = B:new(3)
b:test()         -- prints "5+3"
b.super:test()   -- prints "5"
print(b.a)       -- prints "5"
print(b.super.a) -- prints "5"
```
The superclass's members are created (and initialized) when the `self.super:init()` method is called. 
You should generally call this method at the beginning of the constructor to initialize them.
Note that as `b` is an instance of `B`, `b.super` is simply an instance of `A` (So be careful, 
here `super` is dynamic, not static).

### Static variables

Each time you define a new method for a class it is registered into a `static`" table ; this way 
we cannot mix class methods with class services. This table is accessible through a `static` field. 
This is mainly done to permit access to static variables in classes. Example:
```lua
A = newclass("A")
function A:init(a) self.a = a end
A.test = 5   -- a static variable in A

a = A(3)
prints(a.a)           -- prints 3
prints(a.test)        -- prints 5
prints(A.test)        -- prints nil (!)
prints(A.static.test) -- prints 5
```

### Virtual methods

Class methods are not virtual by default, which mean they are not implicitely overridden by potential 
subclass implementations. To declare a method as being virtual you have to explicitely declare them by using 
the `virtual()` method of their class. The call to `virtual()` must be written outside any method, 
and after the method definition:
```lua
A = newclass("A")

function A:whoami()
  return "A"
end
A:virtual("whoami") -- whoami() is declared virtual

function A:test()
  print(self:whoami())
end

B = newclass("B", A)

function B:whoami()
  return "B"
end
  -- no need to use B:virtual() here
  
myB = B()
myB:test() -- prints "B"
```
It is also possible to declare some methods as abstract (i.e. pure virtual methods); you just have to 
call `A:virtual()` with the name of the abstract method without defining it. 

An error will be raised if you try to call it without having defined it lower in the hierarchy. 
Here is an example:
```lua
A = newclass("A")

A:virtual("whoami") -- whoami() is an abstract method

function A:test()
  print(self:whoami())
end

B = newclass("B", A)

function B:whoami() -- define whoami() here
  return "B"
end
					
myB = B()
myB:test() -- will print "B"

myA = A()  -- no error here! 
myA:test() -- but will raise an error here
```

### Private attributes

By default, subclasses inherit all methods and attributes defined by their parent class(es).
This can lead to some confusion when defining several attributes sharing the same name at different levels
of the hiearchy:
```lua
A = newclass("A")

function A:init()
  self.x = 42  -- define an attribute here for internal purposes
end

function A:doSomething()
  self.x = 0   -- change attribute value
  -- do something here...
end


B = A:subclass("B")

function B:init(x)
  self.super:init()   -- call the superclass's constructor
  self.x = x          -- B defines an 'x' attribute. Problem: 'x' is actually already defined by A!
end

function B:doYourJob()
  self.x = 5
  self.doSomething()
  print(self.x)       -- prints "0": 'x' has been modified by A because A defined it first
end
```
It is possible to define "private" attributes in a class depending on the order these attributes are initialized. 
Note that "private" isn't the best terms here (because there is no real protection mechanism);
we should rather talk about "shared" and "non shared" attributes between a class and its subclasses. 
You should also note that this distinction is made by the subclass itself (and not by the superclass), 
which can decide (in its constructor) which attributes of the superclass should be eventually inherited 
from the superclass or privately overridden. As a rule of thumb, you should probably always define a class' attributes
before calling its superclass' constructor.

Let's look at the same example, with a slight change in `B:init()`:
```lua
A = newclass("A")

function A:init()
  self.x = 42  -- define an attribute here for internal purposes
end

function A:doSomething()
  self.x = 0   -- change attribute value
  -- do something here...
end


B = A:subclass("B")

function B:init(x)
  self.x = x          -- B defines a private 'x' attribute
  self.super:init()   -- call the superclass's constructor
end

function B:doYourJob()
  self.x = 5
  self.doSomething()
  print(self.x)       -- prints "5": 'x' has not been modified by A
  print(self.super.x) -- prints "0": this is the 'x' attribute that was used by A
end
```
You can see that the different behaviours of the attributes 'x' and 'y' come from the order of initialisation 
in the constructor. The "first" class that defines an attribute will get possession of that attribute, 
even if some superclasses declare an attribute with the same name "later" in the initialisation process. 
I personnaly suggest to initialise all "non shared" attributes at the beginning of the constructor, 
then call the superclass' constructor, then eventually use some of the superclass' methods. 
On the contrary if you want to access an attribute defined by a superclass, you may not set its value 
before the superclass' constructor has done it.

### Castings

Castings are useful if you need to access a specific (non virtual) method from a method located higher 
in a class hierarchy. This can be done with the `cast()` and `trycast()` class methods. Here is a simple example:
```lua
A = newclass("A")

function A:foo()
  print(self.x)         -- prints "nil"! There is no field 'x' at A's level
  selfB = B:cast(self)  -- explicit casting into a B
  print(selfB.x)        -- prints "5"
end


B = newclass("B",A)

function B:init(x) 
	self.x = x
end

myB = B(5)
myB:foo()
```
`C:cast(x)` tries to find the "sub-objet" or "super-object" in 'x' corresponding to the class `C`, 
by searching higher and lower in the hierarchy. Intuitively, we will have 
`myB.super == A:cast(myB)` and `myB == B:cast(myB.super)`. Of course this works with more 
than 2 levels of inheritance. If the casting fails, an error will be raised.

`C:trycast(x)` does exactly the same except that it simply returns `nil` when casting is impossible 
instead of raising an error. `C:made(x)` actually returns `true` if `C:trycast(x)` does not return `nil`, 
i.e if casting is possible.

## Comments
Any feedback, comments and suggestions are appreciated. Don't hesitate to fork this code and adapt it to your needs!
