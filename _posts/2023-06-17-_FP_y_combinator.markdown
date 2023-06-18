---
layout: post
title: "Derive Y Combinator from an Object Oriented Perspective"
date: 2023-06-17 +0800
categories: Functional Programming
---

Recently I am studying the metaprogramming of Python. There are some very important concepts: decorator, metaclass, etc. As a C++ lover, I can't stop me to associate everything with C++ even when exploring the features of other programming languages. Last week I considered if I can simulate the Python style decorator in C++. In Python we can modify or enhance a function's capability simply by adding an `@` above its definition. For example, we can make a naive recursive searching caches its results to avoid repeated evaluation.

```python
import functools

# This fibonacci function has only O(N) complexity!
@functools.cache
def fibonacci(a):
    if a == 1:
        return 0
    elif a == 2:
        return 1
    else:
        return fibonacci(a - 2) + fibonacci(a - 1)
    
for i in range(1, 50):
    print(fibonacci(i), end=' ')
```

How to implement such magic in C++? After thinking for several hours, I came up with the following codes.

```c++
#include "decorator.hpp"

struct Fibonacci : deco::Decorate<Fibonacci, deco::Memorization> {
  uint64_t Call(auto& self, uint32_t a) {
    if (a == 1) {
      return 0;
    } else if (a == 2) {
      return 1;
    } else {
      return self(a - 1) + self(a - 2);
    }
  }
};

int main() {
  Fibonacci fib;
  for (int i = 1; i < 50; i++) {
    std::cout << fib(i) << ' ';
  }
}
```

Here are the full codes: [deco_cpp](https://github.com/coyorkdow/deco_cpp)

My C++ decorator utilizes CRTP(curiously recurring template parameter), yet this is not the topic we are discussing today. You may have noticed that in my `Call` method, the recursion doesn't invoke `Call` itself, instead it calls an additional parameter `self`, the type of which is undetermined. That's the key. In our program, when a recursive function is decorated by certain decorators. We must ensure that every recurring call invokes the "decorated" function, rather the "original" function. However, C++ is a static programming language. And the functions (including member functions) are not objects. Which means if you directly call a function, the stuff you called is determined in compile time. If we examine the disassembly we will find the corresponding instruction calls a fixed address, which cannot be changed in runtime.

We can indeed make a function object (i.e., a callable object) in C++. But what we calling is still a member function, and member functions are not the part of a class instance. Besides, every member function has a implicit parameter `this`. For instance, when we use a `std::function` instance, the invoking actually pass the pointer to the instance itself as an argument, and the invoked function is `std::function<T>::operator()`.

The decorator, essentially, is a high order function which takes original function as argument. In C++, we cannot write such a function which can hack into the definition of another function and alter the invoking target which determined in compiler time. So, technically we cannot use recursion in a decoratable function. We have to find some other approaches. And my approach is to give the decoratable an extra parameter to represent "self". When we need recursion, we instead to call this "self" parameter.

It's appears that such decoratable function has some same properties with lambda. They both cannot directly invoke themselves, need some tricks to get the job done. But there are also have some mechanisms to "decorate" these stuff so that we can call them same as the regular functions. Now, it turns out that after a long prologue, we finally come to our topic, the Y combinator. Here I am excited to tell you that this is the first time I use it in my real life programming. And I found it's very easy to understand the details of the Y combinator if we start from the views of "regular programming languages", or, the object oriented perspective.

Briefly speaking, given a function which has the form of

```c++
f = [](auto g) {
  return [](auto... x) { /* do something and calls g(x...) */ }
};
```

The Y combinator is a high order function that takes `f` as an argument and returns a function `fix_f`. We can use `fix_f` as a regular recursive function, and invoking `fix_f(x...)` is equivalent to invoking `f(g)(x...)`. Please be aware that due to the Currying, `f` is actually equivalent to the following function.

```c++
[](auto g, auto... x) { /* do something and calls g(x...) */ }
```

The Y combinator is also known as the fixed point combinator. And `fix_f` is a fixed point of `f`. Why is it called "fixed point"? Consider that the `f` itself is not a recursive function, the recursion is performed by its return function. But in the returned function, it calls `g` instead itself. So if our function really does a recurring call, then what `f` returns must be same as `g`, where the `g` is a function that represents the recursive call. So, `fix_f` is exactly the `g` what we desired. Moreover, `fix_f(x...)` is equivalent to `f(fix_f)(x...)`. Hence, we have
```
fix_f = f(fix_f)
```
And no matter we repeated call `f` how many times, the result will no longer change. So, we call `fix_f` is a fixed point of function `f`.

Now we can start our derivation of Y combinator. But first let's write some codes in Python to implement what we describe just now. It's quiet simple.

```python
class Y:
    def __init__(self, f):
        self.f = f

    def __call__(self, *args):
        return self.f(self, *args)
```

What we need to do is translate this `Y` to the form of lambda calculus. `Y` is a class in Python but we should view it as a function. Let's do this step by step. And remember the principle of Currying: you can split a function call with multiple argument into a chain of multiple function calls where each call only take one argument.

At the beginning, we have
```python
Y = lambda f: f(y) # Step 1
```
Where the `y` is the instance of `Y`. But Where do we get this parameter? The solution is to add a new parameter by making `Y` a higher order lambda. And We will find later that the `y` can only be next to the `f`.
```python
Y = lambda f: lambda y: f(y) # Step 2
```
Now we get a internal lambda. Once `Y` accepts an argument `f`, we can do some "initialization" to pass `Y`'s "instance" into the internal lambda. Please be aware that the `y` is the "instance" of `Y`, not the `Y` itself. Therefore, the argument should only consist of the part that come after the `f`, as the `Y(f)` corresponding to the instantiation of class `Y` in our python codes.
```python
Y = lambda f: (lambda y: f(y))(lambda y: f(y)) # Step 3
```
The step 3 expression looks familiar, doesn't it? But it still has some difference than the correct form. What's wrong it is? The fact is we didn't notice that a member function has an additional parameter to indicate "itself". Although in Python a member function has explicit `self`, the argument passing when invoking the method is still implicit. If we only pass `y` into `f`, when `f` calls `y` it should actually calls `y(y)`. So we have to add the step of "pass itself as an argument" in our lambda.
```python
Y = lambda f: (lambda y: f(y(y)))(lambda y: f(y(y))) # Step 4
```

Now we finally get an expression which has a same form of the

$$Y = \lambda f.(\lambda x.f(x\ x)(\lambda x.f(x\ x)))$$

But is this the end of our topic? If we try to use the step 4 lambda to evaluate something, for example
```python
factorial = Y(lambda g: lambda n: 1 if n <= 1 else n * g(n - 1))
```
We will surprisingly found that our program will crashes due to the infinite loop. And it just on the step of get the fixed point, we don't even start to calculate the factorial! To figure it out what happened, let's rewrite our `Y` to a normal function. And for convenience, I put our class version `Y` below too.

```python
# Which is corresponds to the lambda above
def Y(f):
    def y(y):
        return f(y(y))
    return y(y)

class Y:
    def __init__(self, f):
        self.f = f

    def __call__(self, *args):
        return self.f(self, *args)
```

Do we find anything abnormal? There is a `y(y)` in our function `Y`. Once `Y` takes an argument it will be executed immediately without any condition, it's where the infinite loop occurs. But on the other hand, in the class `Y`, the corresponding expression is `self(self)`, which is executed after the `f` called. More specifically, `self(self)` is executed by `f`, not by `Y`. And it is conditional, for example, let `y` denotes `lambda g: lambda n: 1 if n <= 1 else n * g(n - 1)`. The `self(self)` only comes while the n is greater than 1.

To make up this issue. We have to modify our lambda so that it can do a lazy evaluation. We come up with our fifth lambda expression.

```python
Y = lambda f: (lambda y: lambda x: f(y(y))(x))(lambda y: lambda x: f(y(y))(x))
```

This time, `y(y)` will only be executed when `f` is called. We use `x` to indicate the argument of `f`. Thanks to the currying, This combinator is compatible with the multi arguments function. For instance.
```python
F = lambda g: lambda x: lambda y: x + y if x + y <= 1 else x + y + g(x - 1)(y - 2)
f = Y(F)
print(f(10)(9))
```

But it's also has a drawback as it cannot accept a function without argument. The only way we fix it is by making all functions are called without argument at the first level after the currying.
