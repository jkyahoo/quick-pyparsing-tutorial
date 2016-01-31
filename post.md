## A Parsing Problem

To work through this tutorial, we will address a hypothetical problem. Given a string of alphabetic characters, expand any ranges to return a list of all the individual characters.

For instance given:

    A-C,X,M-P,Z

return the list of individual characters:

    A,B,C,X,M,N,O,P,Z

For simplicity of this tutorial, we will limit ourselves to single, upper-case, alphabetic characters.  At the end, I'll suggest some enhancements that you'll be able to tackle on your own.

## Parsing Options in Python

Python offers several built-in mechanisms for slicing and dicing text strings:

 -- str class - the str class itself includes a number of methods such as split, join, startswith, and endswith that are suitable for many basic string problems

 -- re and regex modules - the builtin re module and the externally available enhancement regex bring all the power, speed, complexity, and arcana of regular expressions to crack many difficult parsing problems

 -- shlex module

These features are good for many quick text-splitting and scanning tasks, but can suffer when it comes to ease of maintenance or extensibility. 


## Approaching a problem with Pyparsing

Whether using pyparsing or another parsing library, it is always good to start out with a plan for the parser. In parsing terms, this is called a BNF, for Backus-Naur Form. This plan helps structure your thinking and the resulting implementation of your parser. Here is a simplified BNF for the letter range problem:

    letter ::= 'A'..'Z'
    letter_range ::= letter '-' letter
    letter_range_item ::= letter_range | letter
    letter_range_list ::= letter_range_item [',' letter_range_item]...

This translates almost directly to pyparsing Python code:
```python
from pyparsing import *

letter = oneOf(list(alphas))
letter_range = letter + '-' + letter
letter_range_item = letter_range | letter
letter_range_list = letter_range_item + ZeroOrMore(',' + letter_range_item)
```

Using parseString to run this parser:
```python
print(letter_range_list.parseString(test))
```

gives

    ['A', '-', 'C', ',', 'X', ',', 'M', '-', 'P', ',', 'Z']

While this shows that we did in fact parse everything in the input, we still have to do some cleaning up, and also add the expansion of the `A-C` type ranges.

First thing is to strip out the commas. They are important during the parsing process, but not really necessary for the parsed results. Pyparsing provides a class Suppress for this type of expression, that needs to be parsed, but should be suppressed from the results:
```python
letter_range_list = letter_range_item + ZeroOrMore(Suppress(',') + letter_range_item)
```

As it happens, this `expr + ZeroOrMore(Suppress(delimiter) + expr)` pattern occurs so frequently, that pyparsing includes a helper method `delimitedList(expr, delim)` (with a default delimiter of `','`) that supports this pattern:

```python
letter_range_list = delimitedList(letter_range_item)
```

With this change, our output is now cleaner:

    ['A', '-', 'C', 'X', 'M', '-', 'P', 'Z']

The last step is to expand the sub-ranges `'A-C'` and `'M-P'`. We could certainly walk this list of parsed items, looking for the `'-'` symbols:
```python
def expand_range(from_, to_):
    alphas = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    from_loc = alphas.index(from_)
    to_loc = alphas.index(to_)
    return list(alphas[from_loc:to_loc+1])

out = []
letter_iter = iter(parsed_output)
letter = next(letter_iter, '')
while letter:
    if letter == '-':
        to_ = next(letter_iter)
        out.extend(expand_range(last, to_)[1:])
    else:
        out.append(letter)
        last = letter
    letter = next(letter_iter, '')
```

But doesn't it feel like we are retracing steps that the parser must have already taken? When pyparsing ran our parser to scan the original input string, it had to follow a very similar process to recognize the `letter_range` expression. Why not have pyparsing run this expansion process at the same time that it has parsed out a `letter_range`?

As it happens, we can attach a callback to an expression; in pyparsing applications, this callback is called a *parse action*. Parse actions can serve various purposes, but the feature we will use in this example is to replace the parsed tokens with a new set of tokens. We'll still utilize the `expand_range` function defined above, but now calling it will be more straightforward:
```python
def expand_parsed_range(t):
    return expand_range(t[0], t[2])
letter_range.addParseAction(expand_parsed_range)
```

And now our returned list reads:

    'A', 'B', 'C', 'X', 'M', 'N', 'O', 'P', 'Z']

## One more thing...

While this accomplishes our goal, there is one pyparsing "best practice" that I'd like to incorporate into this example. Whenever we access parsed data using numeric indices like in
```python
return expand_range(t[0], t[2])
```

we make it difficult to make changes in our grammar at some time in the future. For instance, if the grammar were to change and add an optional field in this expression, our index of 2 might have to be conditionalized depending on whether that field were present or not. It would be much better if we could refer to these fields by name.

We can attach names to parsed fields by modifying the definition of the letter_range expression from:
```python
letter_range = letter + '-' + letter
```

to:
```python
letter_range = letter('start') + '-' + letter('end')
```

With this change, we can now write our parse action using these results names:
```python
def expand_parsed_range(t):
    return expand_range(t.start, t.end)
```

Here's a different one yeah...

And now if the definition of `letter_range` changes by adding other fields, our starting and ending fields will still be processed correctly, without having to change any existing code. Parsers that use results names are much more robust and maintainable over time.

## The Complete Parser

Here is the full working parser example:
```python
from pyparsing import oneOf, alphas, delimitedList

def expand_range(from_, to_):
    alphas = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    from_loc = alphas.index(from_)
    to_loc = alphas.index(to_)
    return list(alphas[from_loc:to_loc+1])

letter = oneOf(list(alphas))
letter_range = letter('start') + '-' + letter('end')

def expand_parsed_range(t):
    return expand_range(t.start, t.end)
letter_range.addParseAction(expand_parsed_range)

letter_range_list = delimitedList(letter_range | letter)

test = "A-C,X,M-P,Z"
print(letter_range_list.parseString(test))
```

## Potential extensions

Here are some extensions to this parser that you can try on your own:

 -- Change the syntax of a range from "A-C" to "A:C"; how much of your program did you have to change?

 -- Add support for lower-case letters and numeric digits
 
 -- Add a parse action to `letter_item_list` to return a sorted list, with no duplicates
 
 -- Add validation that `letter_range` is proper (do not allow degenerate ranges like "J-J", out of order ranges "X-M", or mixed ranges like "T-4")
 
 -- Write another parser to handle ranges of integers instead of letters
 
 -- Write a series of tests, and run them using `letter_range_list.runTests`


## For More Information

You can find more information on pyparsing by visiting the pyparsing wiki, at http://pyparsing.wikispaces.com.
