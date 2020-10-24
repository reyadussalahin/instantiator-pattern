# Instantiator Pattern
The problem of dependencies between software components or classes is a well known problem and also several well known solutions exist to solve this problem. Some popular approaches to solve the dependency problems are `Designing by Contracts`, `Dependency Injection`, `Service Locator Pattern`, `Autowiring(dependency injection by a third party service locator)` etc. But each of these approaches has its own pros and cons. And sometimes, the cons are responsible for adding new complexities to the program(of course, for bad decisions) that, the approach itself becomes more of a problem than solution.  
  
Instantiator pattern tries to solve the dependency problem in such a way that it can overcome the limitation of each of the other approaches.  
  
Let's dive into details:
 - First I'll try to discuss a problem
 - Then I'll try to explain what the limitations of each approaches
  - After that, I'll explain what Instantiator pattern is and how it solves the problem

## The problem
When designing softwares, we often break softwares into components or classes to wrap up similiar kinds or tightly coupled functionalities. Let's say, in our imaginary project, we have several such components or classes. Let's say, there is such a component which has the same functionlity as a third party component. As we want to develop our software as fast as possible, we decided to go with third party component for now, but later we'll replace it with our own implementation of that component. For discussions sake, let's define a consumer class for the thrid party component as follows(note: I'll be using php, but if you're familiar with any oop language, you should be fine):

```php
// note: It's the easiest possible code, we could write to use the ThirdPartyComponent class. This is the first solution come to our mind, cause it's natural, just creating dependency as we go forward. No extra thinking!

// the following line is equivalent to java or python import
use Path\To\ComponentByThirdParty;

// class that dependents on component
class ComponentConsumer
{
    public function usesComponent()
    {
        // ...
        // ...
        // other necessary codes
        // here, $a is third party component's dependency
        $component = new ComponentByThirdParty($a);
        // ...
        // ...
        // some other codes
    }
}
```


## The Existing solutions and their limitations
I am going to discuss about what is the problems with the above code and each existing solutions step by step. And also I'll show you, how each of the existing solutions tries to solve the problems and fails in one aspect or another.  
  
### The first problem: Or the compatibility problem
The first problem with the above(natural) solution is the compatibility problem. If we want to replace `ComponentByThirdParty` class with our own implementation, then both must have same public methods and also, the corresponding methods signatures must be same and perform same tasks.  
  
### Design by contract: The solution to the first problem
To solve the issues with the first problem, we must enforce a contract that must be carried out by both `ComponentByThirdParty` and `ComponentByUs(let it be the name of our own implementation)` classes. But it's possible that, third party component has already gone out of the way. So, we can introduce an adapter class `CompatibleComponentByThirdParty`, that conforms `ComponentByThirdParty` to implement `ComponentInterface(let it be the name of the interface that both should implement)`. So,

**ComponentInterface** sample implementation:

```php
interface ComponentInterface
{
    public function methodA(): void;
    public function methodB(): RetClass;
    // ...
    // ...
    // other methods
}
```

**CompatibleComponentByThirdParty** sample implementation:

```php
use Path\To\ComponentByThirdParty;

class CompatibleComponentByThirdParty implements ComponentInterface
{
    private $thirdPartyComponent;

    // constructor
    public function __construct($a)
    {
        $this->thirdPartyComponent = new ComponentByThirdParty($a);
    }

    public function methodA(): void
    {
        $this->thirdPartyComponent->someMethod();
    }

    public function methodB(): RetClass
    {
        // ...
        $ret = $this->thirdPartyComponent->someOtherMethod();
        // ...
        return $ret;
    }

    // ...
    // ...
    // other methods
}
```

So, design by contract along with adapter pattern solves the first problem for us. We can now just replace `CompatibleComponentByThirdParty` with `ComponentByUs` to use our own implementation of component. But this is not the end. New problem arises.

### The second problem: Or the problem with replacement/edit
Now, if we want to swap `CompatibleComponentByThirdParty` with `ComponentByUs`, then we've to go into the code and manually replace or edit `CompatibleComponentByThirdParty` to `ComponentByUs`. And also, we've to add import statement for `ComponentByUs`. Now, this is a tedious task, and leads to several problems:
 - Hardcoding dependencies into code
 - We've to change all the places where `ComponentImplementedByThirdParty` has been instantiated. And this could be a lot of places distributed in a lot of files
 - Changing already written codes often introduces bugs
 - (language specific) fixing imports

### Service Locator: The solution to second problem
As, the first solution leads us to perform very tedius task to get our job done, it's not very useful for a large project where we've to deal with a lot of files. So, to solve the issue with the second problem, we can instantiate the objects using a service locator. So, its get very easy to replace one component with another. We can rewrite `ComponentConsumer` as follows:

```php
class ComponentConsumer
{
    public function usesComponent()
    {
        // ...
        $component = ServiceLocator::get(ComponentInterface::class, [ $a ]);
        // ...
    }
}
```

and service locator interface which is implemented by service locator could be as follows:

```php
interface ServiceLocatorInterface
{
    public static function add(string $className, Closure $closure): void;
    public static function get(string $className, ...$args): mixed;
}
```

And one could map one interface to a concrete implementation as follows:

```php
ServiceLocator::add(ComponentInterface::class, function($a) {
    return new \Path\To\ComponentByThirdParty($a);
});
```

So, we can see that, `ServiceLocator` gives us the flexibility we needed. We have totally isolated the object creation from our code. And, now we've only one place where the changes occurs. So, to replace `ComponentByThirdParty` with `ComponentByUs` can be done just by mapping `ComponentInterface::class` to `ComponentByUs`:

```php
ServiceLocator::add(ComponentInterface::class, function($a) {
    return new \Path\To\ComponentByUs($a); // note the changes
});
```

Pretty easy and fun huh! Now, just wait a bit. This also has problems. Let's discover them.


### The third problem: Testing problem
Service locator leads us to a path where testing becomes harder. When testing we often need an environment where we could easily inject dependencies(specially fake external dependencies like a fake database or fake file system) easily. But service locator does not provide the facility for us. We can't just inject a fake facility without changing service locator. And changing service locator back and forth for testing purposes, is not the appropriate solution.

### The fourth problem: Runtime error
Service locator often leads to runtime errors, cause object not intantiated directly, rather it is instantiated inside a method or closure later at runtime, which may lead to runtime error.

### The fifth problem: The cognitive load
Let's look at folllowing code properly:

```php
// here $a is a dependency
$component = ServiceLocator::get(ComponentInterface::class, [ $a ]);
```

We've to pass dependency `$a` to properly construct the `ComponentByUs` object, but we don't know the `ComponentByUs`'s constructor signature by looking at the above line. Many one uses `IDE` for this purpose, so that they can instantly see what are the dependencies to construct an object or call a method. To construct an object that implements `ComponentInterface` either we've to consult `ComponentInterface`'s documentation or go see it's implementation or we just have to remind all the dependencies. This problem may create distraction or cognitive load.


### Dependency Injection: The rescuer from third, fourth and fifth problem
Now, dependency injection is really an elegant solution to solve the above problems. Rather then resolving dependencies inside the method, dependency injection says, let's just push the dependencies into the method. In this way, testing would be easier, and there would be no cognitive load(at least, for the writer of the method). So, we could write `ComponentConsumer` as follows:

```php
class ComponentConsumer
{
    public function usesComponent(ComponentInterface $component)
    {
        // ...
        // do you stuff with component
        // ...
    }
}
```

That's just pretty neat, huh! No worry about which component class we're using, third party or our, no worry about how to construct it, just great, the writer of the method is pretty happy! But hold on, there still should be someone who has to intantiate component object. So, who would do it? Let's go to our next problem.


### The sixth problem: Responsibility and diameter of knowledge
So, someone has to be responsible for creating object that implements `ComponentInterface`. Naturally, it becomes the responsibility of the consumer of the `usesComponent`'s method. So, let's say there is some code as bellow:

```php
// $b, $c are dependencies for creating an object of class A
$a = new A($b, $c);
$component = new \Path\To\ComponentByUs($a);
$componentConsumer->usesComponent($component);
```

Now, let's examine the three lines of codes. For writing the three lines how much knowledge one should have. Well, (s)he has to know about dependency of A, know about dependency of `ComponentByUs` and also (s)he has to know about dependency of the methods (s)he is calling of `$componentConsumer` object. Now, if you ask me, I think that's a lot of knowledge one need just to write three lines of code. And I don't want that. The much more you have to know to write some code, the much more burden will be on you and the much more possibility of that error would occur.


### The seventh problem: Dependency hell
Let's just assume, you have faced a very odd case, where class `A` dependends of class `B` and `C`. `B` depends on `D` and `E`. `C` depends on `F`. `D` depends on `G`. All dependencies are constructor dependencies. It's a mess right. Let's see it in a picture:

```
                      A
                     / \
                    B   C
                   / \   \
                  D   E   F
                 /
                G
```

So, to construct an object of A, we have to do as follows:

```php
$c = new C(new F());
$b = new B(
    new D(new G()),
    new E()
);
$a = new A($b, $c);
```

Very bad design, huh! What is the odd that this could happen? Well, it's not about the odds that we should be worried about, rather, we should be worried about that, we're keeping a hole in our design procedure, that this could happen. In reality, it could lead to a much worse dependency hell.  
  
### Autowiring: An attemp to solve the sixth and seventh problem
Autowiring is just a hybrid of `Service Locator` and `Dependency Injection` pattern. Autowiring checks all the argument types in a method's signature and if the type is registered in `Service Locator`, then if just creates the object and return. So, you can just assume that, if the above classes `A, B, C, D, E, F, G` are just registered in the service locator, you don't have to worry about dependency handling, autowiring would do that for you. So, you just call `ComponentConsumer::usesComponent` method as follows:

```php
$componentConsumer->usesComponent();
```

You don't even have to be worried about creating any object that implements `ComponentInterface`. Autowiring does that job for you.  
  
Well, that's very charming. I think, it just solved all the problems. But does it? Well, try to remember `Autowiring needs a Service Locator`. So, it comes with `the problems of Service Locator`. And, we're doomed again.  
  
We just observe that, we've actually found a lot of solutions for dependency handling, but each of them tries to solve the problem partially. And among all of them `Dependency Injection` is the most usable solution, but one must be experienced to design `Dependency Injection` properly, so that, (s)he may not create a dependency hell.  
  
Okay, let's move forward and try to find a new solution.  
  
## The ultimate solution: The Instantiator pattern
Before going on describing Instantiator pattern, let's list all problems first to see what type of solution we need:
1. The compatibility problem
2. The replacement/edit problem
3. The testing problem
4. The runtime error
5. The cognitive load
6. The responsilibity and the diameter of knowledge complexity
7. The dependency hell

So, what we've to do is just propose a way that could solve those problem provided above and we're done.  
  
### The solution ideas which lead to Instantiator pattern:
To solve the above described problems, I have proposed some simple rules, that would help us either avoid the problems or solve the problems. The rules are:

1. Whenever possible, one should resolve it's own dependency rather than injected by others. This helps us eliminate dependency problem. And also keeps the diameter of knowledge as small as possible.
2. The second idea depends on observation: `How often do we need to switch between implementation? 1. for testing, and 2. for switching production grade components`. So, one should be able to switch between components based on modes. This solves replacement/edit problem and testing problem.
3. Each instantiator will provide a client interface which will be used to instantiate object. This will help solve cognitive load.
4. For solving compatibility, we'll use our first solution, a common contract.
5. The only problem that remains is `runtime error`. As Instantiator is a lazy object generator pattern, we can not solve it directly, but we can easily solve it using a `simple unit test`.

### Description of Instantiator pattern:
I am not great at generating diagrams, so I'll try to explain by words: 

1. Instantiator class would be an abstract class
2. Instantiator class would be extended by other class for use
3. Children of Instantiator class would provide two method: one is `register method`, other one is of their own choice for getting instance. `register method` is an abstract method of Instantiator class, which the user must implement, it would provide `closures or factories that returns desired objects`.
4. Intantiator would contain an static variable `private static $container` to point to a container which would be used to `contain the closures or factories that produces proper objects`.
5. Instantiator class will have two states. A global state and a local state. Global state targets to "default" mode by default and local state would be set by `Child of Instantiator class`. If global state is missing, then loacl state would point to global state at that time. Instantiator would also have a fallback variable, which states that if a mode is missing, then the program should fallback to "default" mode or not. Generally, one should often let fallback to true.

And that's it.  
  
Let's view a look at how a simple child of Instantiator class would look like:

```php
class DatabaseInstantiator extends Instantiator
{
    protected function register()
    {
        $this->instance([
            "default" => function($p, $q, $r) {
                return new \Path\To\Database($p, $q, $r);
            },
            "test" => function($p, $q, $r) {
                return new \Path\To\FakeDatabase($p, $q, $r);
            }
        ]);
    }

    // you can choose any name you want get, create etc..., but getInstance
    // cause, getInstance method is provided by Instantiator and can't be overridden
    public function get(P $p, Q $q, R $r): DatabaseInterface
    {
        return $this->getInstance($p, $q, $r);
    }
}
```
When mode is set to "test", `DatabaseInstantiator::get($p, $q, $r)` would return `FakeDatabase` object and when in "default" mode, it would return `Database` object.

If you want to define your `DatabaseInstantiator` have a constructor, then you must call parent constructor as following:

```php
class DatabaseInstantiator extends Instantiator
{
    public function __construct($mode=null, $fallback=null)
    {
        parent::__construct($mode, $fallback);
        // ...
        // do other stuffs
    }
    // ...
    // ...
    // register and other methods you need
}
```

And the structure of Instantiator could be like:
```php
abstract class Instantiator implements InstantiatorInterface
{
    private static $globalMode;
    private static $globalFallback;
    private static $container;

    // various setter, getter and access checker for
    // global variables
    // ...
    // ...


    // local properties and methods of instantiator
    // local state variables
    private $childMode;
    private $childFallback;

    public function __construct(?string $mode=null, ?bool $fallback=null)
    {
        $this->childMode = ($mode === null) ? self::globalMode() : $mode;
        $this->childFallback = ($fallback === null) ? self::globalFallback() : $fallback;
    }

    // instance and singleton method implementation
    protected function instance(array $factories): void
    {
        // ...
    }

    protected function singleton(array $factories): void
    {
        // ...
    }

    // `register` is an abstract method which must be implemented by
    // user. It should be used to provide closures for instantiating
    // objects for various modes
    abstract protected function register();

    // method to get and set local mode and fallback
    public function setMode(string $mode): void
    {
        $this->childMode = $mode;
    }

    public function getMode(): string
    {
        return $this->childMode;
    }

    public function setFallback(bool $fallback): void
    {
        $this->fallback = $fallback;
    }

    public function getFallback(): bool
    {
        return $this->childFallback;
    }

    final protected function getInstance(...$args)
    {
        // method that accesses $container and return generated object
        // this can not be overridden
    }
}
```

One can use the given DatabaseInstantiator as follows:

```php
// default mode
$dbInstantiator = new DatabaseInstantiator();
$db = $dbInstantiator->get($p, $q, $r);
// here is instance of Database, can be checked as follows
var_dump($db instanceof Database); // prints true

// and if one want to use "test", then
// mode="test", fallback=true
// cause, if "test" mode is not found, it'll use
// the "default" mode factory or closure
$dbInstantiator = new DatabaseInstantiator("test", true);
$db = $dbInstantiator->get($p, $q, $r);
// here is instance of Database, can be checked as follows
var_dump($db instanceof DatabaseFake); // prints true
```


I hope this gives an idea how Instantiator pattern works. For a full version of Instantiator implementation visit this [repository](https://github.com/rs-world/instantiator).


# Contributing
If you know how to improve Intantiator pattern or make it much more easier to use then send a pull request. You can also contact me at reyadussalahin@gmail.com with your ideas. I would love to have a talk.
