> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.can.ac](https://blog.can.ac/2020/04/11/writing-an-optimizing-il-compiler-for-dummies-by-a-dummy/)

> Before I begin this series of blog posts, I would like to add a small disclaimer. I have no prior ......

_Before I begin this series of blog posts, I would like to add a small disclaimer. I have no prior experience or academic knowledge when it comes to compiler development so I might not use the correct jargon or state of the art algorithm, but nonetheless, I wanted to share my journey working on VTIL which is a project I started aiming to make the translation of a virtual machine architecture –e.g. VMProtect– into x86-64 easier. You will be able to find the source code for most of the things I will mention [here](https://github.com/vtil-project)._

0x0: Why?
---------

I started this journey with one single goal, reverse engineering VMProtect, mostly because I was sick and tired of every so-called security company buying this product, which is priced at an astonishing 250$, slapping it on their “product”, and calling it a day since no one will bother checking it. Now the initial reversal took roughly 36 hours, which wasn’t so bad, but what came after that was a month(+) long journey of optimizing it back to x86-64 code so I could use the output with common analysis tools. Surprise surprise, IDA turns out to be not as good at optimizations as I thought. Let alone failing to group a simple combination of operators `~((~x&~x)+y)&~((~x&~x)+y => ~(~x+y) => x-y`, it failed on me for the [simplest](https://i.imgur.com/oVNdZwK.png) things, as it [usually](https://i.imgur.com/m2fZp3m.png) does.

I did not want to use LLVM because it didn’t seem like the right tool for the job (translation in specific and not optimization obviously) [and because I’m a stubborn fan of reinventing the wheel], and I did not want to use Z3 because the API was not so appealing and I kept getting told that it was not that good at simplification, which lead to me deciding to develop an entire toolchain with its own instruction set, expression simplifier, and optimizing compiler designed around binary translation. I’ll start this series with symbolic expressions since that’s where most of the optimization logic boils down to.

0x1: Symbolic Expressions?
--------------------------

Now what is a symbolic expression really and what are we using them for? VTIL uses symbolic expressions to describe the value a register or a memory location holds after the execution of a certain instruction, expressed in a static-single-assignment like form by tagging the operands with the source virtual instruction pointer, `VIP`, to compensate for the non-SSA ISA VTIL has.

This is useful for many reasons such as exploring the virtual-machine’s control-flow since you have to resolve the jump destination to continue processing the virtual instruction stream, producing an output that does not immediately make IDA suicide or performing basically any kind of bitwise or arithmetic optimization to clean up the obfuscated code.

Generating one can be done with ease from even a non-SSA architecture such as the one we use for VTIL. Consider the following basic block instance:

```
movq   t2,   t0             |  mov  r8,  rax
orq    t2,   t1             |  or   r8,  rcx
movq   t3,   t1             |  mov  r9,  rcx
andq   t3,   t0             |  and  r9,  rax
notq   t3                   |  not  r9
andq   t3,   t2             |  and  r9,  r8
```

_I’ve written the “equivalent” routine in_ _x86-64 on the right side just to clarify what each VTIL instruction translates into._

Let’s try to express the value of `t3` after the whole block is executed. Reading the stream backward, `t3` is equal to `(t3@6+t2@2)`, `t3@6` is equal to `~t3@5`, which is `~(t3@4&t0)`, which is `~(t1@0&t0@0)`, and so on. Following the same logic for the whole expression, we get `(t0|t1)&~(t0&t1)`. To put it simply, you trace it back the entire routine until you hit an instruction you cannot resolve the result of, or reach the end of the block. This, of course, gets harder when you start implementing cross-block optimization, especially with loops and conditional jumps, or partial data read/write, but they are out of the scope of this post. (I will be explaining all quirks of it in another post, hopefully, this week along with a bunch of data optimization techniques.)

Now that we got the gist of generating one, let’s try simplifying it. Simplifying an expression such as this is weird enough, very easy to do by head since we do this process almost automatically, but hard to describe how and why to a computer since we do “post-mortem” analysis when explaining our steps and start saying “Well x was y so I did this then I did this and here’s the result”, which is never easy enough to generalize to _any_ expression. This has always been the reverse of my experience with computers in general so I personally find this task very, very interesting.

Since I have apparently skipped middle school maths and jumped to calculus straight and am incapable of describing operator properties in a formal way, we’ll _reverse engineer_ a human brain instead. When I saw this expression, I started reading from the beginning, skipped to the operator noting the left-hand side operand, saw a paranthesis so started reading it in the same way until the whole expression is parsed. This kind of parsing reminds me of a tree so I’m going to make the assumption that easiest way to represent these expressions is by using a tree.

![](https://blog.can.ac/wp-content/uploads/2020/04/Untitled-Document-1.png)`(t0|t1)&~(t0&t1)` in tree format.

Try simplifying it now noting your actions as you go. As a fellow human being, I skipped `LHS`, and started following `[&].RHS`, repeated the same thing, following `[~&].RHS`, since both operands were unknown variables to me, I jumped back to `[&].LHS`, and then `[|].RHS` into`[|].LHS` both of which I skipped again since operands were unknown again. Now I find myself trying to understand what it really does, one side of the `AND` is taking `OR` of both variables, which means at least one of them must be set (at boolean level) to have an outcome of one, and other side is taking `NOT AND`, which means they can’t be set at the same time, so I’ll write down `t0^t1` and be done with it, tada, all of this optimized into a simple `xorq`.

That was a rather horrifying experience if you ask me since I just randomly came up with rules which I could not possibly list if you asked me to (correction: [apparently I could](https://vtil.org/repo/Core/blob/master/VTIL-SymEx/simplifier/directives.hpp)). But the general steps seem to be recursively simplifying the operands and trying to merge them together as a part of this simplification process if we failed to transform the whole tree.

0x2: Small steps, structure design.
-----------------------------------

First of all, we know that the simplification process is mostly recursive pattern matching, giving it a try, and repeating, with certain rules to make it more interesting. You really don’t want to use a tree that does deep-copies or any copies at all by default in this situation if you do not want to stand in front of your computer for 30 minutes waiting for the simplification, so I started by writing a simple copy-on-write reference implementation making sure I do not copy anything at all if possible, which can be found [here](https://vtil.org/repo/Core/blob/master/VTIL-Common/util/copy_on_write.hpp).

Another thing you want to absolutely avoid is repeated calculations, so I will also add a hash to the expression structure which will be assigned at the constructor of the expression in a `O(1)` way that will not require the traversal of the tree and a trivially implemented lookup map to serve as the cache of the expression simplifier. Since VTIL is thread-safe and we do not want a global lock to impair the performance with most likely no benefit–since the same temporaries will not be used across threads–this lookup map will be declared `thread-local`.

The structure looks something like this in my head so far:

```
struct expression
{
	?int??_t?                    value;
	
	size_t                       hash;
	bool                         simplified?;
	
	operator_id                  op;
	shared_reference<expression> lhs; // Will be null for unary operations.
	shared_reference<expression> rhs;
	...
};
```

I already have a VMProtected sample lifted to VTIL, so I just went ahead and generated an expression for the branching destination of a “_simple_” Jcc to use as a sample across our tests:

**Hey! Click to expand this cute little JCC ![](https://s.w.org/images/core/emoji/14.0.0/svg/1f642.svg)** 

Since this expression is practically unreadable at the moment, let’s ignore it for now. After implementing the cache check and recursing into operands bit, starting with constant-folding seems like a logical choice. However I will implement this with a _slight_ twist.

0x3: Overcome your fear of the unknown.
---------------------------------------

Expressing constant values in expressions as, well…, constants work fine sure. But what about the rest? Say you are evaluating `(A>>1)&1`, is the value of this expression really that “unknown” since you know that a high majority of the bits are zero? Do you really want to recursively pattern match every unit operand of the `((A>>1)&1)|0x21` just to optimize it into `0x21`? No, and you don’t.

Let’s try representing the value as a bit vector instead with three basic properties.

```
// Bit-vector holding 0 to 64 bits of value with optional unknowns.
//
class bit_vector
{
    // Value of the known bits, mask of it can be found by [::known_mask()]
    // - Guaranteed to hold 0 for unknown bits.
    //
    uint64_t known_bits = 0;

    // Mask for the bit that we do not know.
    // - Guaranteed to hold 0 for known bits and for all bits above bit_count.
    //
    uint64_t unknown_bits = 0;

    // Number of bits this vector contains.
    //
    uint8_t bit_count = 0;
```

Let’s refer to known bits as (where `1` is the value that generates a set bit where unset bits indicate unknown bits and optionally `n` as the bit index) and unknown bits as (following the same convention). will not be defined by itself in most cases as it can be resolved by a simple .

All basic bitwise operators are implemented trivially in this convention as it can be seen in equations , , , and .

Similarly shifting and rotation can be expressed with the help of a few branches, as seen in equations , and . Rest of the operators can also be implemented in solutions ranging from `O(n²)` to `O(1)` with some bitwise magic, but I will not bore you with math any further, let’s move on.

0x4: Classification of simplification directives.
-------------------------------------------------

I’ve implemented a structure that acts like an expression but instead is used for pattern creating purposes so that we can create simplification directives, `directive::instance`, there are two major benefits to this choice over hand-written logic:

1.  Directives created in a structured way will let us write verification routines much easier, later on, say using a SMT solver or a simple fuzzer, since we have access to the complete logic of the simplifier described in a structured way.
2.  All of the directives are in one [neat and tidy](https://vtil.org/repo/Core/blob/master/VTIL-SymEx/simplifier/directives.hpp) place where we can manage them easily.

Now that we can finally get into writing simplification directives, let’s talk about what simplification is.

Consider `~A|~B` vs `~(A&B)`, which one is simplified? Say you pick the 2nd one since it has 2 operations vs 3. What about `A&(B<<C)` where A is a constant? Do we leave it as is or rewrite as `((B&(A>>C))<<C)`, or rather if do not rewrite it, how does one simplify `0x40&(ZF&1<<6)` as`(ZF&1)<<6` since `1&1==1`?

A simple answer to this question, sadly, does not exist. However, we can create an arbitrary definition of a “complexity” and optimize the reward function created based on it, which will give us a direction to go towards. VTIL’s definition of an expression complexity is defined as shown below in the equation like this:

Basically this definition of , punishes depth in trees in an exponentially-growing way and prefers constants over variables and smaller constants over huge constants in a way that would not hurt negative constants. I also additionally add a x2 multiplier if the arithmetic / bitwise hints do not match so that for instance `NEG` is preferred instead of `NOT` when used in an arithmetic expression. We can classify simplifiers in 3 different categories now:

1.  **Universal Simplifiers**  
    This type of simplification directive will reduce the cost of the expression with no exceptions, what this means is that when we simplify an expression based on any of the directives in this list and recurse, we have no possibility of creating an infinite loop, which makes these the first step of any simplification flow.
2.  **Join Directives**  
    Join directives basically present alternative forms of directives that may have a positive or a negative effect on the final cost based on the way lower nodes are structured, a great example is `A(B&C) => (A|B)&(A|C)`, this can go both ways, good and bad, based on the full expression, the methods of handling these directives will be discussed when we start discussing them.
3.  **Pack/Unpack Directives**  
    Packing directives create very high-level operators from bitwise and arithmetic operators such as the conversion of `__if`, this is a great thing for the user-end (a.k.a. compiler-end), but not exactly the best news for the simplifier itself as each additional operator we implement means we need to implement join directives and universal simplifiers for them, which is a lot of work. This class of directives deals with this problem by keeping all expressions unpacked via the unpacking directives, unless explicitly requested by the user to `prettify`, which I find to be a very fitting name for simplifications that are not necessarily simple, but preferred by humans and thus code written by them.

With a quick implementation of a bunch of universal simplifiers, we get the monstrous expression from Section #0x2 into this form:

```
(((_ucast(((0x40&~((((((_ucast(D:1, 0x40)<<0xb)|(_ucast(H:1, 0x40)<<0x7))|(_ucast(J:1, 0x40)<<0x6))|(_ucast(B:1, 0x40)<<0x4))|(_ucast(F:1, 0x40)<<0x2))^(((((_ucast(E:1, 0x40)<<0xb)|(_ucast(I:1, 0x40)<<0x7))|((_ucast((_cast(K:1, 0x40)+0x145515abbb), 0x40)<<0x6)&0x40))|(_ucast(C:1, 0x40)<<0x4))|(_ucast(G:1, 0x40)<<0x2))))>>0x6), 0x1)?0x14000b964)+(~_ucast(((0x40&~((((((_ucast(D:1, 0x40)<<0xb)|(_ucast(H:1, 0x40)<<0x7))|(_ucast(J:1, 0x40)<<0x6))|(_ucast(B:1, 0x40)<<0x4))|(_ucast(F:1, 0x40)<<0x2))^(((((_ucast(E:1, 0x40)<<0xb)|(_ucast(I:1, 0x40)<<0x7))|((_ucast((_cast(K:1, 0x40)+0x145515abbb), 0x40)<<0x6)&0x40))|(_ucast(C:1, 0x40)<<0x4))|(_ucast(G:1, 0x40)<<0x2))))>>0x6), 0x1)?0x14000b6de))+relocation:64)
```

Not bad!

0x5: Join directives.
---------------------

One problem we need to solve before anything with join directives is the cost of building and trying each join directive. I accomplish this by adding two special constructs that change how directives work in general. First one is special directives that grant us the ability to make choices at evaluation time if a specific condition is satisfied which adds 4 essential operators:

1.  **Accepting <x> only if it simplifies, `!x`**  
    This becomes useful as it lets us avoid the construction completely if a chosen partition cannot be simplified, avoiding all costs.
2.  **Attempt to simplify <x> before it reaches caller, `s(x)`**  
    This lets us simplify certain bits before the recursive-simplifier starts working again and most importantly before the complexity comparison is made, so we don’t get to lose anything by not simplifying before the complexity checks.
3.  **If and only if, `__iff(condition, directive)` and or, `__or(x, y)`**  
    These combined let you have fully conditional output essentially letting you define logic when combined with already existing comparison operators.
4.  **Access to mask interface from `math::bit_vector`**  
    Accessing the value of , and is an exceptional helper when simplifying bitwise expressions.

Other one is the ability to deny the construction of certain expressions if the directive will not match, which basically speculatively creates the conditional segments of the directives within the `directive::match` routine itself. This basically allows for fast verification of directive conditions and thanks to our cache, we do not suffer any drops in speed since 2nd time this directive is built, simplified version will be pulled out of the cache.

Now we can do complex simplifications like the following, see if you can spot the problem with the last directive.

```
{ (A>>B)<<C,                                          __iff(B>=C, s(!((-1>>B)<<C)&(A>>!(B-C)))) },
{ (A>>C)<<B,                                          __iff(B>=C, s(!((-1>>C)<<B)&(A<<!(B-C)))) },
{ A|(B<<U),                                           !(!(A>>U)|B)<<U|s(A&((1<<U)-1)) },
...
```

Our last problem is the elimination of infinite-loops, which we have already solved by a previously implemented feature without even noticing.

![](https://blog.can.ac/wp-content/uploads/2020/04/Untitled-Document-4-1024x416.png)See the problem with this perfectly valid directive?

Now the directive is technically not as safe as it should be, as it doesn’t check if the OR it appends actually has any effect but this is fine. As we’ve implemented the directive cache and reference and set it within the routine BEFORE simplification is done, the same expression calling into the `::simplify` routine will get a quick simplify failed and be kicked out, so this problem is solved for us… by us?

I will, again, implement a bunch of directives of this type and see how the output improves just like the previous section.

```
(((0x14000b964&~((0x1&~(_ucast(J:1, 0x40)^_ucast(K:1, 0x40)))+-0x1))+(0x14000b6de&((0x1&~(_ucast(J:1, 0x40)^_ucast(K:1, 0x40)))+-0x1)))+relocation:64)
```

Quite a decent improvement over the last version again :).

0x6: _Prettifying_.
-------------------

Now the output of the previous is surely the simplest form it will ever get to, but as human beings, we seem to like pretty expressions, so as the final stage we add a prettyfier, which converts certain groups into high-level functions we understand easier with no real added value. The only trick here is that this flag to “_prettify_“, should not propagate to operands and that after the join stage is done, regardless of the _prettify_ flag we should attempt to _unprettify_. With those two things in mind, we can implement as many high level operators as we can.

With the simple two-line introduction of the `__if` operator, we get the final output of

**`(((~(J:1^K:1)?0x14000b964)+((J:1^K:1)?0x14000b6de))+relocation:64)`**

The lesson to learn here is that weeks of work and debugging do not go to waste even when you feel the least hopeful ¯\_(ツ)_/¯.

-0x1^0xF8: Conclusion
---------------------

Surely, there are numerous reasons LLVM keeps getting recommended again and again and again. Getting this kind of complex logic working correctly in stable form takes at least weeks, let alone the fact that ours is a very reduced form. However I still find this challenge really interesting and since this was pretty much the hardest part of it all, given that I already have implemented a lot of instruction-stream optimizations and a complete compiler on my previous project, porting them to VTIL should not take much time and hopefully we will have a great release sometime soon.

**TL;DR:** If you feel like reinventing the wheel, go ahead, it’s fun. But you can always contribute to ~LLVM~ VTIL instead, why bother reinventing the wheel? /s

**P.S.:** I’d like to thank [@daax_rynd](https://twitter.com/daax_rynd) and [@vm_call](https://twitter.com/vm_call) for the mental support they have provided during this rough time.