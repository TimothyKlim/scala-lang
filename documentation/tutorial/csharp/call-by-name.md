---
layout: page
title: Scala's call-by-name for C# Programmers

disqus: true
by: Ivan Towlson
---



## Call by Name

A parameter passed by name is not evaluated when it is passed to a method.
It is evaluated -- and re-evaluated -- when the called method uses
the parameter; specifically when the called method requests the value of
the parameter by mentioning its name. This is the key to being able to
define your own control constructs.

The *Java Virtual Machine* -- on which Scala runs -- passes both,
primitives as well references to objects, to methods by value.
Nonetheless, Scala enables passing arguments to a method by name.
(The Scala compiler invisibly "wraps" the argument as an anonymous function.)

Consider this piece of logging code:

    def complicatedComputation = {
      val result = ... // compute
      log(isDebugEnabled(), printDebugMessage(result, "DEBUG " + result + ": "))
    }

    def printDebugMessage(result: Object, message: String) = {
      ...
    }

Ideally, the second argument should only be evaluated if the method
`isDebugEnabled` evaluates to true.

A naive implementation would compute the second argument eagerly and
cause potentially expensive computation of debug output even if
`condition` evaluated to `false`:

    def log(condition: Boolean, action: Unit) { // action gets executed before log is called
      if(condition) ...
    }

An improved implementation might use a function for the second
parameter:

    def log(condition: Boolean, body: () => Unit) {
      if(condition) body
    }

The C# implementation looks almost similar:

    public void Log(bool condition, Action body)
    {
      if (condition)
      {
        body();
      }
    }

Both implementations share the issue that calling the method becomes
syntactically cumbersome:

    // Scala
    log(42==42, () => printDebugMessage(...))

    // C#
    Log(42==42, () => printDebugMessage(...))

Scala provides the possibility to not evaluate arguments eargely while
preserving a visually pleasing syntax at the call site:

    def log(condition: Boolean, body: => Unit) {
      if(condition) body
    }

    log(42==42, printDebugMessage(...))

### Not Just Another Calling Convention

Why does this matter? It matters because there are functions you can't
implement using pass by value or pass by reference, but you can implement
using pass by name.

Suppose, for example, that C# didn't have the `while` keyword.
You'd probably want to write a method that did the job instead:

    public static void While(bool condition, Action body)
    {
      if (condition)
      {
        body();
        While(condition, body);
      }
    }

What happens when we call this?

    long x = 0;
    While(x < 10, () => x = x + 1);

C# evaluates the arguments to `While` and invokes the `While` method with
the arguments `true` and `() => x = x + 1`.  After watching the CPU sit
on 100% for a while you might check on the value of `x` and find it's
somewhere north of a trillion. *Why?* Because the condition argument was
*passed by value*, so whenever the `While` method tests the value of
condition, it's always `true`. The `While` method doesn't know that
condition originally came from the expression `x < 10`; all `While` knows
is that condition is `true`.

For the `While` method to work, we need it to re-evaluate `x < 10` every
time it hits the condition argument.  While needs not the value of the
argument at the call site, nor a reference to the argument at the call
site, but the actual expression that the caller wants it to use to generate
a value.

### Short-Circuit Evaluation

Same goes for short-circuit evaluation. If you want short-circuit
evaluation in C#, your only hope if to get on the blower to Anders
Hejlsberg and persuade him to bake it into the language:

    bool result = (a > 0 && Math.Sqrt(a) < 10);
    double result = (a < 0 ? Math.Sqrt(-a) : Math.Sqrt(a));

You can't write a function like `&&` or `?:` yourself, because C# will
always try to evaluate all the arguments before calling your function.

Consider a VB exile who wants to reproduce his favourite keywords in C#:

    bool AndAlso(bool condition1, bool condition2)
    {
      return condition1 && condition2;
    }

    T IIf<T>(bool condition, T ifTrue, T ifFalse)
    {
      if (condition)
        return ifTrue;
      else
        return ifFalse;
    }

But when C# hits one of these:

    bool result = AndAlso(a > 0, Math.Sqrt(a) < 10);
    double result = IIf(a < 0, Math.Sqrt(-a), Math.Sqrt(a));

it would try to evaluate all the arguments at the call site, and pass the
results of those evaluations to `AndAlso` or `IIf`.  There's no
short-circuiting. So the `AndAlso` call would crash if a were negative,
and the `IIf` call if a were anything other than 0. Again, what you want is
for the `condition1`, `condition2`, `ifTrue` and `ifFalse` arguments to be
evaluated by the callee if it needs them, not for the caller to evaluate
them before making the call.

### Using Pass By Name in Scala

Let's see the custom while implementation again, this time with Scala
*pass by name* parameters:

    def myWhile(condition: => Boolean)(body: => Unit): Unit =
      if (condition) {
        body
        myWhile(condition)(body)
      }

We can call this as follows:

    var i = 0
    myWhile (i < 10) {
      println(i)
      i += 1
    }

Unlike the C# attempt, this prints out the numbers from 0 to 9 and then
terminates as you'd wish.

Pass by name also works for short-circuiting:

    import math._

    def andAlso(condition1: => Boolean, condition2: => Boolean): Boolean =
      condition1 && condition2

    val d = -1.234
    val result = andAlso(d > 0, sqrt(d) < 10)

The `andAlso` call returns `false` rather than crashing, because
`sqrt(d) < 10` never gets evaluated.

What's going on here?  What's the weird colon-and-pointy-sticks syntax?
What is actually getting passed to `myWhile` and `andAlso` to make this work?

The answer is a bit surprising. Nothing is going on here. This is the
normal Scala function parameter syntax. There is no *pass by name* in Scala.

Here's a bog-standard *pass by value* Scala function declaration:

    def myFunc1(i: Int): Unit = ...

Takes an integer, returns void: easy enough. Here's another:

    def myFunc2(f: Int => Boolean): Unit = ...

Even if you've not seen this kind of expression before, it's probably not
too hard to guess what this means. This function takes a *function from
`Int` to `Boolean`* as its argument. In C# terms,
`void MyFunc2(Func<int, bool> f)`. We could call this as follows:

    myFunc2 { (i: Int) => i > 0 }

So now you can guess what this means:

    def myFunc3(f: => Boolean) : Unit = ...

Well, if `myFunc2` took an *Int-to-Boolean* function, `myFunc3` must be
taking a "blank-to-Boolean" function -- a function that takes no arguments
and returns a `Boolean`.  In short, a conditional expression. So we can
call `myFunc3` as follows:

    val j = 123
    myFunc3 { j > 0 }

The squirly brackets are what we'd expect from an anonymous function, and
because the function has no arguments Scala doesn't make us write
`{ () => j > 0 }`, even though that's what it means really.  The anonymous
function has no arguments because `j` is a captured local variable, not an
argument to the function. But there's more. Scala also lets us call
`myFunc3` like this:

    val j = 123
    myFunc3(j > 0)

This is normal function call syntax, but the Scala compiler realises that
`myFunc3` expects a nullary function (a function with no arguments) rather
than a `Boolean`, and therefore treats `myFunc3(j > 0)` as shorthand for
`myFunc3(() => j > 0)`. This is the same kind of logic that the C# compiler
uses when it decides whether to compile a lambda expression to a delegate
or an expression tree.

You can probably figure out where it goes from here:

    def myFunc4(f1: => Boolean)(f2: => Unit): Unit = ...

This takes two functions: a conditional expression, and a function that
takes no arguments and returns no value (in .NET terms, an `Action`).
Using our powers of anticipation, we can imagine how this might be called
using some unholy combination of the two syntaxes we saw for calling
`myFunc3`:

    val j = 123;
    myFunc4(j > 0) { println(j); j -= 1; }

We can mix and match the `()` and `{}` bracketing at whim, except that we
have to use `{}` bracketing if we want to batch up multiple expressions.
For example, you could legally equally well write the following:

    myFunc4 { j > 0 } { println(j); j -= 1; }
    myFunc4 { println(j); j > 0 } (j -= 1)
    myFunc4 { println(j); j > 0 } { j -= 1 }

And we'll bow to the inevitable by supplying a body for this function:

    def myFunc5(f1: => Boolean)(f2: => Unit): Unit =
      if (f1()) {
      f2()
      myFunc5(f1)(f2)
    }

Written like this, it's clear that `f1` is getting evaluated each time we
execute the if statement, but is getting passed (as a function) when
`myFunc5` recurses. But Scala allows us to leave the parentheses off
function calls with no arguments, so we can write the above as:

    def myFunc5(f1: => Boolean)(f2: => Unit): Unit =
      if (f1) {
        f2
        myFunc5(f1)(f2)
      }

Again, type inference allows Scala to distinguish the *evaluation of
`f1`* in the if statement from the *passing of `f1`* in the `myFunc5`
recursion.

And with a bit of renaming, that's `myWhile`.  There's no separate
*pass by name* convention: just the usual closure behaviour of capturing
local variables in an anonymous method or lambda, a bit of syntactic sugar
for nullary functions (functions with no arguments), just like C#'s
syntactic sugar for property getters, and the Scala compiler's ability to
recognise when a closure is required instead of a value.

In fact, armed with this understanding of the Scala "syntax," we can
easily map it back to C#:

    void While(Func<bool> condition, Action body)
    {
      if (condition())
      {
        body();
        While(condition, body);
      }
    }

    int i = 0;
    While(() => i < 10, () =>
    {
      Console.WriteLine(i);
      ++i;
    });

The syntax for *calling* the `While` method in C# is more cumbersome and
less natural than the syntax for calling `myWhile` in Scala.
Calling `myWhile` in Scala was
like using a native language construct. Calling While in C# required a
great deal of clutter at the call site to prevent C# from trying to treat
`i < 10` as a once-and-for-all value, and to express the body at all.