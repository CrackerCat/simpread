> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zerotypic.medium.com](https://zerotypic.medium.com/introducing-wilhelm-376dbfae9fdf#)

> wilhelm is a Python API providing a better interface for working with Hex-Rays.

Introducing wilhelm
===================

[![](https://miro.medium.com/fit/c/56/56/0*LmMlLJ_ufEyu5j-A.jpg)](/?source=post_page-----376dbfae9fdf--------------------------------)[

zerotypic

](/?source=post_page-----376dbfae9fdf--------------------------------)[

2 days ago·7 min read

](/introducing-wilhelm-376dbfae9fdf?source=post_page-----376dbfae9fdf--------------------------------)![](https://miro.medium.com/max/1400/1*VIWMmQhEXzH2WZoVRwVG_g.png)wilhelm in action.

wilhelm is a reversing tool I’ve been working on for a long while, which I’ve finally decided to release. It’s a Python API that provides (in my opinion) a better interface for working with Hex-Rays. In particular, I designed it with the IDAPython REPL (aka console) in mind; while it works fine in scripts, it’s meant to be used interactively to quickly automate some analysis while reversing.

Currently, wilhelm is very much a work in progress, and many features that I plan to add haven’t been written yet. However, the parts that already do exist have proven quite useful to me while I do my work, so I decided that other people might find it useful too. In particular, I think the API for accessing the abstract syntax tree (AST) of a decompiled function comes in really handy when reversing large applications, and the AST node selection feature (known as WilPath) allows one to automate a lot of otherwise menial tasks.

Besides for AST access, wilhelm also provides an event system (necessary when working interactively as the IDA database can change), a qualified name management system, and will in the future include a higher-level type system. You can try wilhelm out by cloning the repository on GitHub; I haven’t gotten down to packaging it yet, but it should work if you just append the right directory to IDAPython’s `sys.path`.

**Check out wilhelm on GitHub** [**here**](https://github.com/zerotypic/wilhelm)**.**

An Example
==========

The best way to show what wilhelm can do is through a simple example. I decided to randomly choose some binary and start analyzing it in IDA. I ended up looking at a copy of `libstagefright.so` , taken from some old Android firmware.

Quick introduction to _libstagefright_: _libstagefright_ is a media processing library that is part of the Android Open Source Project (AOSP). A function of interest in _libstagefright_ is `android::MPEG4Extractor::parseChunk()` : it is part of the MPEG4 decoder, and parses chunks (aka boxes) in an MPEG4 stream. Each chunk begins with 4 bytes indicating its size, and then 4 bytes indicating the type of the chunk, often referred to as the chunk’s _fourcc_. `parseChunk()` checks the chunk’s fourcc and performs different actions based on it. Most of the time, fourcc values consist of four printable ASCII characters, such as `ftyp` or `moov`, which make them easy to identify.

Here’s what some of the fourcc parsing code looks like when freshly decompiled by Hex-Rays (the fourcc value is stored in the local variable `fourcc2`):

![](https://miro.medium.com/max/60/1*6MFmIk34qYhrKOw877PCvQ.png?q=20)![](https://miro.medium.com/max/875/1*6MFmIk34qYhrKOw877PCvQ.png)![](https://miro.medium.com/max/1400/1*6MFmIk34qYhrKOw877PCvQ.png)

As you can see, Hex-Rays represents the literal fourcc values as 32-bit integers. This makes the code harder to read if you’re familiar with the fourcc values represented as ASCII strings. We can manually convert the integers to ASCII strings by pressing the “R” hotkey, but this would take a while as `parseChunk()` is a pretty huge function (the pseudocode is 4,937 lines long). I’m feeling lazy; could we write a few lines of Python code to automate this?

Accessing the AST
-----------------

We can, with wilhelm’s help. The first thing we want to do is identify the components of the AST that we wish to manipulate. These would be the literal numeric expressions that appear as part of a binary operation expression (binop) where the left-hand-side (LHS) operand is the local variable “`fourcc2`”.

We can access a decompiled function’s AST using wilhelm:

The `func` object is the function at the current cursor position (_screen_ea_), and its AST can be accessed via `func.body`, which is the top-level block statement of the function. For example, we could access the first statement in the function using `func.body[0]`, and (because this is an if-statement), access its LHS expression with the following:

WilPath Selectors
-----------------

wilhelm allows nodes within the AST to be filtered and navigated, in a fashion similar to how you can filter and navigate through elements in an HTML/XML document. The optional WilPath feature (which we enabled by passing `wilhelm.Feature.PATH` to `wilhelm.initialize()`) allows us to use an [XPath](https://en.wikipedia.org/wiki/XPath)-inspired DSL to quickly return a list of nodes that meet a specific criteria.

_(Note that if you don’t want to use WilPath, you can still filter and navigate through nodes using methods on the_ `NodeList` _class.)_

The WilPath DSL consists of a sequence of _selectors_, which get applied to a list of AST nodes. A selector is either a _navigator_ selector, which maps nodes in the list to other nodes, or a _filter_ selector, which returns a subset of nodes in the list meeting some criteria. See the [docstring](https://github.com/zerotypic/wilhelm/blob/main/python/wilhelm/path.py) in `path.py` for more information.

When WilPath is loaded, lists of nodes (`NodeList` objects) have a `select()` function can be used to apply a WilPath to the list. Let’s start with the function’s top-level block:

This returns all if-statements that are direct children of the block, _i.e._ at the top level. `select()` returns a `NodeList`, which is a lazy container that does not compute its contents until necessary. By creating a list out of the `NodeList`, we can force it to evaluate and see it contains two statements. To make the rest of the code easier to read, we define a function to call select and evaluate the returned `NodeList`:

Now, what we are looking for are binops in the entire function. We can search for all binops using the following WilPath:

Using the _all_ navigator (`*/`) allows us to search for all descendants of the function body, and not just immediate children.

We don’t want just any binary operation though; only those that involve `fourcc2`. We can select for that using the following:

In this expression, we use the attribute navigator `.e_lhs` to select the LHS operands of all found binops. We then filter these operands to include only those that are local variable expressions, and further filter those to only include those where the local variable is `fourcc2`.

This brings us closer to our goal. However, what we have is a list of local variable expressions referencing the local variable `fourcc2`, but not the numeric expressions we want, which are actually the right-hand side (RHS) operands of the parent binops. We can select these by doing the following:

We use the _subpath_ filter (`[]`) to hold on to our reference to the parent binary operation expression (the filter will return only nodes for which the WilPath in the brackets successfully selects something). Then, we select the RHS operand using the attribute navigator, and filter to include only literal number expressions.

Converting to ASCII Strings
---------------------------

We now have a list of literal numeric expressions which should all be fourcc values. The final thing to do is to ask Hex-Rays to display these numbers as ASCII strings instead.

Unfortunately, this is not as straightforward as it should be, because I can’t seem to find a function that does it automatically (when working with the disassembly, you can just use the `op_chr()` function). I had to write a helper function to do this:

The function uses the Hex-Rays API to modify the user-defined number formats associated with the function. I’ll probably include the code to do this in a future version of wilhelm. (If anybody knows of a better way of doing this, please let me know!)

One-Liner Automation
--------------------

Once this helper function is defined, we can then complete our task and convert all fourcc values to ASCII with (almost) a single line of Python:

I couldn’t seem to find a way to get Hex-Rays to update the pseudocode view to reflect the new number formats, but re-decompiling the function (_e.g._ pressing F5) appears to do the trick. The final result, with a total of 99 fourcc literals converted:

![](https://miro.medium.com/max/60/1*vSrs0Er5zYCf8jVCtWO89w.png?q=20)![](https://miro.medium.com/max/875/1*vSrs0Er5zYCf8jVCtWO89w.png)![](https://miro.medium.com/max/1400/1*vSrs0Er5zYCf8jVCtWO89w.png)