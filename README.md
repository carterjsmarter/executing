# executing

[![Build Status](https://github.com/alexmojaki/executing/workflows/Tests/badge.svg?branch=master)](https://github.com/alexmojaki/executing/actions) [![Coverage Status](https://coveralls.io/repos/github/alexmojaki/executing/badge.svg?branch=master)](https://coveralls.io/github/alexmojaki/executing?branch=master) [![Supports Python versions 2.7 and 3.4+, including PyPy](https://img.shields.io/pypi/pyversions/executing.svg)](https://pypi.python.org/pypi/executing)

This mini-package lets you get information about what a frame is currently doing, particularly the AST node being executed.

* [Usage](#usage)
    * [Getting the AST node](#getting-the-ast-node)
    * [Getting the source code of the node](#getting-the-source-code-of-the-node)
    * [Getting the `__qualname__` of the current function](#getting-the-__qualname__-of-the-current-function)
    * [The Source class](#the-source-class)
* [Installation](#installation)
* [How does it work?](#how-does-it-work)
* [Is it reliable?](#is-it-reliable)
* [Which nodes can it identify?](#which-nodes-can-it-identify)
* [Libraries that use this](#libraries-that-use-this)

## Usage

### Getting the AST node

```python
import executing

node = executing.Source.executing(frame).node
```

Then `node` will be an AST node (from the `ast` standard library module) or None if the node couldn't be identified (which may happen often and should always be checked).

`node` will always be the same instance for multiple calls with frames at the same point of execution.

If you have a traceback object, pass it directly to `Source.executing()` rather than the `tb_frame` attribute to get the correct node.

### Getting the source code of the node

For this you will need to separately install the [`asttokens`](https://github.com/gristlabs/asttokens) library, then obtain an `ASTTokens` object:

```python
executing.Source.executing(frame).source.asttokens()
```

or:

```python
executing.Source.for_frame(frame).asttokens()
```

or use one of the convenience methods:

```python
executing.Source.executing(frame).text()
executing.Source.executing(frame).text_range()
```

### Getting the `__qualname__` of the current function

```python
executing.Source.executing(frame).code_qualname()
```

or:

```python
executing.Source.for_frame(frame).code_qualname(frame.f_code)
```

### The `Source` class

Everything goes through the `Source` class. Only one instance of the class is created for each filename. Subclassing it to add more attributes on creation or methods is recommended. The classmethods such as `executing` will respect this. See the source code and docstrings for more detail.

## Installation

    pip install executing

If you don't like that you can just copy the file `executing.py`, there are no dependencies (but of course you won't get updates).

## How does it work?

Suppose the frame is executing this line:

```python
self.foo(bar.x)
```

and in particular it's currently obtaining the attribute `self.foo`. Looking at the bytecode, specifically `frame.f_code.co_code[frame.f_lasti]`, we can tell that it's loading an attribute, but it's not obvious which one. We can narrow down the statement being executed using `frame.f_lineno` and find the two `ast.Attribute` nodes representing `self.foo` and `bar.x`. How do we find out which one it is, without recreating the entire compiler in Python?

The trick is to modify the AST slightly for each candidate expression and observe the changes in the bytecode instructions. We change the AST to this:

```python
(self.foo ** 'longuniqueconstant')(bar.x)
```
    
and compile it, and the bytecode will be almost the same but there will be two new instructions:

    LOAD_CONST 'longuniqueconstant'
    BINARY_POWER

and just before that will be a `LOAD_ATTR` instruction corresponding to `self.foo`. Seeing that it's in the same position as the original instruction lets us know we've found our match.

## Is it reliable?

Yes - if it identifies a node, you can trust that it's identified the correct one. The tests are very thorough - in addition to unit tests which check various situations directly, there are property tests against a large number of files (see the filenames printed in [this build](https://travis-ci.org/alexmojaki/executing/jobs/557970457)) with real code. Specifically, for each file, the tests:
 
 1. Identify as many nodes as possible from all the bytecode instructions in the file, and assert that they are all distinct
 2. Find all the nodes that should be identifiable, and assert that they were indeed identified somewhere

In other words, it shows that there is a one-to-one mapping between the nodes and the instructions that can be handled. This leaves very little room for a bug to creep in.

Furthermore, `executing` checks that the instructions compiled from the modified AST exactly match the original code save for a few small known exceptions. This accounts for all the quirks and optimisations in the interpreter. 

## Which nodes can it identify?

Currently it works in almost all cases for the following `ast` nodes:
 
 - `Call`, e.g. `self.foo(bar)`
 - `Attribute`, e.g. `point.x`
 - `Subscript`, e.g. `lst[1]`
 - `BinOp`, e.g. `x + y` (doesn't include `and` and `or`)
 - `UnaryOp`, e.g. `-n` (includes `not` but only works sometimes)
 - `Compare` e.g. `a < b` (not for chains such as `0 < p < 1`)

The plan is to extend to more operations in the future.

## Projects that use this

### My Projects

- **[`stack_data`](https://github.com/alexmojaki/stack_data)**: Extracts data from stack frames and tracebacks, particularly to display more useful tracebacks than the default. Also uses another related library of mine: **[`pure_eval`](https://github.com/alexmojaki/pure_eval)**.
- **[`futurecoder`](https://futurecoder.io/)**: Highlights the executing node in tracebacks using `executing` via `stack_data`, and provides debugging with `snoop`.
- **[`snoop`](https://github.com/alexmojaki/snoop)**: A feature-rich and convenient debugging library. Uses `executing` to show the operation which caused an exception and to allow the `pp` function to display the source of its arguments.
- **[`heartrate`](https://github.com/alexmojaki/heartrate)**: A simple real time visualisation of the execution of a Python program. Uses `executing` to highlight currently executing operations, particularly in each frame of the stack trace.
- **[`sorcery`](https://github.com/alexmojaki/sorcery)**: Dark magic delights in Python. Uses `executing` to let special callables called spells know where they're being called from.

### Projects I've contributed to

- **[`IPython`](https://github.com/ipython/ipython/pull/12150)**: Highlights the executing node in tracebacks using `executing` via [`stack_data`](https://github.com/alexmojaki/stack_data).
- **[`icecream`](https://github.com/gruns/icecream)**: 🍦 Sweet and creamy print debugging. Uses `executing` to identify where `ic` is called and print its arguments.
- **[`friendly_traceback`](https://github.com/friendly-traceback/friendly-traceback)**: Uses `stack_data` and `executing` to pinpoint the cause of errors and provide helpful explanations.
- **[`python-devtools`](https://github.com/samuelcolvin/python-devtools)**: Uses `executing` for print debugging similar to `icecream`.
- **[`sentry_sdk`](https://github.com/getsentry/sentry-python)**: Add the integration `sentry_sdk.integrations.executingExecutingIntegration()` to show the function `__qualname__` in each frame in sentry events.
- **[`varname`](https://github.com/pwwang/python-varname)**: Dark magics about variable names in python. Uses `executing` to find where its various magical functions like `varname` and `nameof` are called from.
