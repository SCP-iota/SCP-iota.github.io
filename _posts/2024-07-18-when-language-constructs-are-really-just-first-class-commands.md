---
layout: blog_post
category: software
title: When Language Constructs Are Really Just First Class Commands
description: "See how Bash shell scripting implements if statements in a non-traditional way"
---

In just about every programming language, there will be times you'll need to evaluate a boolean expression.
Whether it's a condition to an `if` statement or a loop, or just to store in a variable, booleans are the defining component of digital computers.

The way you evaluate an `if` statement looks a bit different in different languages:

```c
// C
if(condition) {
    // then...
}
```

```python
# Python
if condition:
    # then...
    pass
```

```bash
# Bash
if [ condition ]
then
    # then...
fi
```

Bash sure looks like the strangest one here, for a few reasons:

* There *must* be spaces separating the brackets from the condition expression
* The `then` keyword *must* be on a separate line (or preceded by a semicolon, if you prefer that style)
* The body ends with `fi`, following the Borne Shell's convention of ending blocks with the reverse of their names

The peculiarity that's most interesting under the surface is the first one: *why*, exactly, must the condition be padded with spaces.
The answer, it turns out, is not just style enforcement...

## Command Exit Codes

If you write Bash scripts, you might know that every command exits with an integer "exit code."
An exit code of 0 indicates success, which a nonzero exit code represents an application-specific error code for the calling script to handle.

In fact, you can omit the brackets of an `if` statement and use a command as a condition to check for success:

```bash
# Don't actually run this
if rm -rf /
    echo "You're doomed"
else
    echo "You got lucky"
fi
```

## Generalizations

This is where Bash takes the [Lisp](https://en.wikipedia.org/wiki/Lisp_(programming_language))-like approach of preferring to implement language features out of existing constructs rather than introducing new special constructs.

The `[` in a condition is... a separate command. Yes, there is a command called `[`, which expects a conditional expression followed by a `]` and either fails or succeeds depending on the result.
And theoretically, it wouldn't even have to require a `]` if not for wanting it to look like a real syntax construct.

It gets worse. The `[` command is actually an alias for the `test` command, which does exactly the same thing other than the fact that is does not require a trailing `]`. The implementation of the `test` command checks whether it was invoked as `test` or `[` to decide whether to be strict about that.

## There's a File for That

It still gets worse. Even though most shells implement `test` and `[` as built-in commands, you can still find both of them in your `/bin` or `/usr/bin` directory. On my system (running Ubuntu), they're copies - not symlinks, but two copied of the exact same `test` executable under two different names.

The idea that something as simple as checking a condition could involve execution a separate process raises concerns about efficiency, but these days most of that is handled by the shell's built in implementations, and the independent executables never need to get called... which really just raises concerns about bloat.