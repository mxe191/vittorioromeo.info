

As I mentioned in [my previous article](https://vittorioromeo.info/index/blog/passing_functions_to_functions.html) many features introduced in the latest C++ standards allow *functional patterns* to thrive in your codebase. Two ideas from that programming paradigm that I really like are [**currying**](https://en.wikipedia.org/wiki/Currying) and [**partial application**](https://en.wikipedia.org/wiki/Partial_application).

In this article we're going to:

* Introduce and briefly explain the two aforementioned concepts.

* Write a generic `constexpr` zero-overhead `curry` function in C++17.

* Analyze the generated assembly of the implementation to prove the lack of overhead.



### Currying

Let's begin by explaining currying.

> > In mathematics and computer science, **currying** is the technique of translating the evaluation of a function that takes multiple arguments *(or a tuple of arguments)* into evaluating a sequence of functions, each with a single argument.
>
> [*(from Wikipedia)*](https://en.wikipedia.org/wiki/Currying)

The following `add3` function is an example of a *non-curried* function, as its [*arity*](https://en.wikipedia.org/wiki/Arity) is $3$.

```cpp
auto add3(int a, int b, int c)
{
    return a + b + c;
}

add3(1, 2, 3); // Returns `6`.
```

We can *curry* it by returning nested [*lambda expressions*](http://en.cppreference.com/w/cpp/language/lambda).

```cpp
auto curried_add3(int a)
{
    return [a](int b)
    {
        return [a, b](int c)
        {
            return a + b + c;
        };
    };
}

curried_add3(1)(2)(3); // Returns `6`.
```

[**wandbox example**](http://melpon.org/wandbox/permlink/HGACYHh1sV3knQGZ)

As you can see, the arity of every lambda is $1$. This pattern is useful because it allows developers to intuitively *bind* arguments incrementally until the last one. If you're finding yourself constantly using the same arguments except one in a series of function calls, currying avoids repetition and increases readability.

```cpp
auto add2_one = curried_add3(1);
auto add1_three = add2_one(2);

add1_three(3); // Returns `6`.
add1_three(4); // Returns `7`.
add1_three(5); // Returns `8`.
```

[**wandbox example**](http://melpon.org/wandbox/permlink/7w4RUHqHEL1Y7Yx1)

Basically, `add1_three(5)` is equivalent to `add2_one(2)(5)`, which is equivalent to `curried_add3(1)(2)(5)`.

*([This short article](http://cukic.co/2013/08/07/curry-all-over-the-c11/) by Ivan Čukić has some additional interesting examples of currying in C++.)*



### Partial application

> > In computer science, **partial application** (or **partial function application**) refers to the process of fixing a number of arguments to a function, producing another function of smaller arity.
>
> [*(from Wikipedia)*](https://en.wikipedia.org/wiki/Partial_application)

Despite them being two separate concepts, *partial application* is very similar to *currying*. Even though I couldn't find a formal confirmation anywhere, I believe that thinking about *partial application* as a "generalized form of *currying*" can be helpful: instead of binding one argument and getting $arity - 1$ unary functions back, we can bind $n$ arguments at once and get another *partially-applicable* function with $arity - n$ arity.

Imagine we had add a `partial_add3` function which allowed *partial application* to sum three numbers:

```cpp
partial_add3(1, 2, 3); // Returns `6`.
partial_add3(1)(2, 3); // Returns `6`.
partial_add3(1, 2)(3); // Returns `6`.
partial_add3(1)(2)(3); // Returns `6`. (Currying!)
```

As you can see, we can decide how many arguments to bind *(including zero)*. We could easily implement this in C++17 using *recursion*, [*generic lambdas*](http://en.cppreference.com/w/cpp/language/lambda), [*`if constexpr(...)`*](http://en.cppreference.com/w/cpp/language/if#Constexpr_If), and [*variadic templates*](http://en.cppreference.com/w/cpp/language/parameter_pack). *(We'll also use a [fold expression](http://en.cppreference.com/w/cpp/language/fold) to compute the sum.)*

```cpp
template <typename... Ts>
auto partial_add3(Ts... xs)
{
    if constexpr (sizeof...(xs) == 3)
    {
        // Base case: evaluate and return the sum.
        return (0 + ... + xs);
    }
    else
    {
        // Recursive case: bind `xs...` and return another 
        return [xs...](auto... ys)
        {
            return partial_add3(xs..., ys...);
        };
    }
}
```

[**wandbox example**](http://melpon.org/wandbox/permlink/AFmdO0Cpkt5zRcJC)

Writing code that enables *currying* and *partial application* for every function is cumbersome. Let's write a generic `curry` function that, given a callable object `f`, returns a *curried*/*partially-applicable* version of `f`!



### C++17 `curry`

As mentioned in the beginning of the article, these are the goals for our `curry` function:

* Given a generic callable object `f`, invoking `curry(f)` returns a *curried*/*partially-applicable* version of `f`.

* If `f` is `constexpr`-friendly, the returned one will be as well.

* `curry` should not introduce any overhead compared to hand-written *currying*/*partial application*.



#### Credit where it's due

Please note that the design and implementation of `curry` that I am going to cover is a *heavily-modified version* of [this snippet that was tweeted by **Julian Becker**](https://twitter.com/awtem/status/804781466852950017) - in fact, it was that tweet that inspired me to write this article. **Thanks!**



#### Example usage

Before we analyze the *declaration* and *definition* of `curry`, let's take a look at some usage examples.

* **Nullary functions:**

    ```cpp
    auto greet = []{ std::puts("hi!\n"); };

    greet(); // Prints "hi!".
    curry(greet); // Prints "hi!".
    
    // Compile-time error:
    /* curry(greet)(); */
    ```

    As you can see, in the case of a *nullary callable object* `f`, invoking `curry(f)` calls the original object immediately. 

* **Unary functions**:

    ```cpp
    auto plus_one = [](auto x){ return x + 1; };

    plus_one(0); // Returns `1`.
    curry(plus_one)(0); // Returns `1`.

    // Returns a wrapper around `plus_one` that enables
    // currying/partial application.
    // `plus_one` is "perfectly-captured" in the wrapper.
    auto curried_plus_one = curry(plus_one);
    
    curried_plus_one(1); // Returns `1`.
    ```

    > What does *perfectly-captured* mean? 

    It means that if the captured object is an *lvalue*, it will be captured *by reference*. If the captured object is an *rvalue, it will be captured *by move*. I've written a comprehensive article on this topic: [**"capturing perfectly-forwarded objects in lambdas"**](http://vittorioromeo.info/index/blog/capturing_perfectly_forwarded_objects_in_lambdas.html).

* **Binary functions**:

    ```cpp
    auto add2 = [](auto a, auto b){ return a + b; };

    // All of the invocations below return `3`.
    add2(1, 2);
    curry(add2)(1, 2); // Partial application.
    curry(add2)(1)(2); // Currying.

    // Example of "binding" an argument:
    auto add_one = curry(add2)(1);
    add_one(2); // Returns `3`.
    add_one(3); // Returns `4`.
    ```

    You should be starting to see the pattern now...

* **$N$-ary functions**:

    ```cpp
    auto add3 = [](auto a, auto b, auto c)
    { 
        return a + b + c; 
    };

    // All of the invocations below return `6`.
    add3(1, 2, 3);
    curry(add3)(1, 2, 3);
    curry(add3)(1, 2)(3);
    curry(add3)(1)(2, 3);
    curry(add3)(1)(2)(3);
    ```

    The example above shows that *currying* and *partial application* can be freely combined. Let's see another example of that with a [`constexpr` lambda](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4487.pdf) of arity $5$.

    ```cpp
    auto add5 = [](auto a, auto b, auto c, 
                   auto d, auto e) constexpr
    { 
        return a + b + c + d + e; 
    };
    
    constexpr auto csum5 = curry(sum5);

    constexpr auto a = csum5(0, 1, 2, 3, 4, 5);
    constexpr auto b = csum5(0)(1, 2, 3, 4, 5);
    constexpr auto c = csum5(0, 1)(2, 3, 4, 5);
    constexpr auto d = csum5(0, 1, 2)(3, 4, 5);
    constexpr auto e = csum5(0, 1, 2, 3)(4, 5);
    constexpr auto f = csum5(0, 1, 2, 3, 4)(5);
    ```

    Note that the usages of `curry(sum5)` above are in no way exhaustive - more combinations such as `curry(sum5)(0, 1)(2, 3)(4, 5)` can be written, and every *intermediate step* can be given a name.

Now that you have an idea on how `curry` can be used, let's dive into its *declaration* and *definition*.

#### Declaration

Given the constraints listed earlier, we can easily write down the *declaration* of `curry`.

```cpp
template <typename TF>
constexpr decltype(auto) curry(TF&& f);
```

> Why `decltype(auto)` instead of `auto`? 

Because the *final step* of `curry` needs to return exactly what the original callable object does. Example:

```cpp
auto f = [](auto, auto) -> auto& 
{
    return some_global_variable; 
};

// OK - can return an additional "curry wrapper" by value.
auto step0 = curry(f); 

// Same as above.
auto step1 = step0('a');
auto step2 = step1('b');

// Now `step2` has to return a reference!
auto& that_same_global = step2();
```

Additionally, the `f` parameter is taken by [*forwarding-reference*](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4164.pdf). I will assume you're familiar with [*move semantics*](http://stackoverflow.com/questions/3106110/what-are-move-semantics), [`std::forward`](http://en.cppreference.com/w/cpp/utility/forward), and [**"forward captures"**](http://vittorioromeo.info/index/blog/capturing_perfectly_forwarded_objects_in_lambdas.html) for the rest of the article.



#### Definition

I'll show the complete definition of `curry` first, and then analyze all the parts one-by-one more closely.

```cpp
template <typename TF>
constexpr decltype(auto) curry(TF&& f) 
{
    if constexpr (std::is_callable<TF()>{}) 
    {   
        return FWD(f)();
    }
    else
    {
        return [xf = FWD_CAPTURE(f)](auto&&... partials) mutable constexpr 
        {
            return curry
            (
                [
                    partial_pack = FWD_CAPTURE_PACK_AS_TUPLE(partials), 
                    yf = std::move(xf)
                ]
                (auto&&... xs) constexpr                
                    -> decltype(forward_like<TF>(xf.get())(FWD(partials)..., 
                                                           FWD(xs)...))
                {
                    return apply_fwd_capture([&yf](auto&&... ys) constexpr 
                        -> decltype(forward_like<TF>(yf.get())(FWD(ys)...)) 
                    {
                        return forward_like<TF>(yf.get())(FWD(ys)...);
                    }, partial_pack, FWD_CAPTURE_PACK_AS_TUPLE(xs));
                }
            );
        };
    }
}
```

The first thing to notice is the **recursive** structure of `curry`.

```cpp
template <typename TF>
constexpr decltype(auto) curry(TF&& f) 
{
    if constexpr (std::is_callable<TF()>{}) 
    {
        return FWD(f)();
    }
    else
    {
        // ...
    }
}
```

The *base case* branch is taken when `std::is_callable<TF()>{}` evaluates to `true`. [`std::is_callable`](http://en.cppreference.com/w/cpp/types/is_callable) is a new C++17 *type trait* that checks whether or not a particular object types can be called with a specific set of argument types.

* If `std::is_callable<TF()>{}` evaluates to `false`, then it means that `TF` needs some arguments in order to be called - those arguments can be *curried*/*partially-applied*.

* If it evaluates to `true`, it means that there are no more arguments to *curry*/*partially-apply* in `f`. Therefore, `f` can be invoked to get the final result:

    ```cpp
    return FWD(f)();
    ```

    `FWD` is a macro that expands to `std::forward<decltype(f)>(f)`. It's being used as `TF` could have a [*ref-qualified*](https://akrzemi1.wordpress.com/2014/06/02/ref-qualifiers/) `operator()` that behaves in different ways depending on `f`'s value category.

We will now focus on the *recursive case* of `curry`. The first step is allowing *partial application* of arguments - since we don't know how many arguments will be bound in advance, a *generic variadic lambda* will be returned:

```cpp
return [xf = FWD_CAPTURE(f)](auto&&... partials) mutable constexpr 
{
    return curry(/* ... */);
}
```

The returned lambda will:

* Capture `f` by *forward capture* into `xf`.

* Accept any amount of *forwarding references* in the `partials...` pack. These arguments will be *bound* for subsequent recursive calls.

* Be marked as `mutable`: this is **important** as `xf` will be moved in the inner lambda's capture list.

* Be marked as `constexpr`: this allows `curry` to be used as a [*constant expression*](http://en.cppreference.com/w/cpp/language/constant_expression) where possible.

* Recursively call `curry` in its body, returning a new *curried*/*partially-applicable* function.

Let's now focus on the `return curry(/*...*/)` statement. We want to return a *curried* version of a new intermediate callable object where:

* The `partials...` pack values are *bound* for its invocation - these values will be captured by *forward capture* as `partial_pack`.

* The *forward-captured* `xf` from the "parent" lambda is captured *by move* into `yf`. `xf` doesn't need to be forwarded as `FWD_CAPTURE(f)` returns a movable wrapper that either stores an *lvalue reference* or a *value*.

```cpp
return curry
(
    [
        partial_pack = FWD_CAPTURE_PACK_AS_TUPLE(partials), 
        yf = std::move(xf)
    ]
    (auto&&... xs) constexpr                
        -> decltype(forward_like<TF>(xf.get())(FWD(partials)..., 
                                               FWD(xs)...))
    {
        // ...
    }
);
```

The lambda passed to `curry` will accept any number of *forwarding references* in the `xs...` pack that will be used alongside the captured `partials...` to call `f`. The expected function call can be easily understood by the lambda's *trailing return type*:

```cpp
//          Unwrap `f` from the `xf` `FWD_CAPTURE` wrapper and propagate
//          the original callable object's value category.
//          vvvvvvvvvvvvvvvvvvvvvvvvv
-> decltype(forward_like<TF>(xf.get())(FWD(partials)..., FWD(xs)...))
//                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//          Unpack both the `partials` and `xs` argument packs in a
//          single function call to `f`.
```

[`forward_like` is an utility function in my `vrm_core` library](https://github.com/SuperV1234/vrm_core/blob/437a0afb35385250cd75c22babaeeecbfa4dcacc/include/vrm/core/type_traits/forward_like.hpp) that *forwards* the passed argument with the same *value category* of the potentially-unrelated specified type.

The expression inside the above return type essentially means: *"invoke the original callable object by unpacking `partials...` and `xs...` one after another"*.

*(For some reason both `g++` and `clang++` throw a compilation error when `-> decltype(auto)` is used instead of the explicit one above. I am inclined to think this is a compiler bug.)*

Lastly, let's analyze the body of the lambda.

```cpp
return apply_fwd_capture([&yf](auto&&... ys) constexpr 
    -> decltype(forward_like<TF>(yf.get())(FWD(ys)...)) 
{
    return forward_like<TF>(yf.get())(FWD(ys)...);
}, partial_pack, FWD_CAPTURE_PACK_AS_TUPLE(xs));
```

Remember: we're trying to call `f` by unpacking both `partials...` and `xs...` **at the same time**. The `partials...` pack is stored in a special wrapper returned by `FWD_CAPTURE_PACK_AS_TUPLE`. The `xs...` pack contains the arguments passed to the lambda.

The `apply_fwd_capture` takes any number of wrapped *forward-capture pack wrappers* and uses them to invoke an user-provided callable object. The wrappers are unpacked at the same time, preserving the original value category. Since `xs...` is not wrapped, we're gonna explicitly do so by using the `FWD_CAPTURE_PACK_AS_TUPLE` macro.

In short, `apply_fwd_capture` will invoke the *`constexpr` variadic lambda* by expanding `partials...` and `xs...` correctly - those values will be then forwarded to the wrapped callable object `yf`.

**That's it!** Eventually the recursion will end as one of the steps will produce an intermediate callable objects that satisfies `std::is_callable<TF()>{}`, giving back a "concrete" result to the caller.




#### Generated assembly benchmarks

```cpp
sdgjs
```

#### Compiler bugs


```cpp
fgsdfd
```