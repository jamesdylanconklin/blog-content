---
date: '2025-10-05T10:43:39-07:00'
draft: true
title: 'Rolling dice with syntax trees'
---

## Gambling without Stakes

One of the projects I almost always do when learning or returning to a language or framework is a dice roller. The first was a monstrosity in C for which I learned the unique pain of dealing with random number generator seeds. Incarnations followed in Perl, in Python, [again in Python](https://github.com/jamesdconklin/py-roll), and finally in ruby as part of a [personal virtual tabletop app](https://github.com/jamesdconklin/OnceUponATable/blob/master/app/models/message.rb).

Once you no longer have to deal with the minutiae of personally managing RNGs, two linked issues pop up. One is the complexity of dealing with parsing and evaluating roll strings at the same time. If a monolithic script does both in parallel, it becomes rather difficult to add or adjust functionality later on. The other issue is the morass of regular expressions one tends to use when rolling their own parser, which are often decipherable only by the author, and even then only at the time of the writing.

{{< highlight python >}}
# Who wrote this? Oh, right. Yuck.
if re.search(r'^[^\)]*\(.*\)[^\(]*$', eval_string):
        left, right = _paren_slice(eval_string, level)
        eval_string = "%s%d%s" % (eval_string[:left-1],
                                  eval_roll(eval_string[left:right], level),
                                  eval_string[right+1:])
{{< /highlight >}}

Something I've wanted to try for a while, more acutely now that I'm returning to Python for more than rough scripting purposes, is an approach that separates parsing from evaluation. Build a syntax tree and map this tree to a homomorphic-ish, evaluable structure. The idea crystalized into something a little more actionable when I revisited language classifications and grammars from my favorite course back at McGill - is there a pre-existing tooling to convert a grammar for dice roll strings into a ready-to-use parser?

As is becoming a common workflow, I outsourced this question to Copilot. It suggested several options that tackled this very use case across Python, JavaScript, and Ruby, but [Lark](https://github.com/lark-parser/lark) seemed the most prmising. 

## Planning

For a quick turnaround on the initial development, I figured these steps made the most sense:
- Learn enough about Lark grammars to write one for roll strings.
- Define basic tests so that any changes I or Copilot make can be quickly sanity-checked
- Transform the parser's tree output into a mirroring structure of `EvaluatorNode` instances.
- Define `evaluate()` functions for each subclass of `EvaluatorNode`.

There's some flex in later nice-to-haves, like verbose results or a richer individual die atom syntax, but I wanted this solid baseline first. 

## Creating the Grammar

So what sort of rolls do we want to support? The most obvious pieces are `x?dy` atoms, along with integers and basic arithmetic. My most fully-featured stab at this problem space also handled parentheticals and lists of rolls through space-separated expressions, so those also go in the grammar. 

In the end, I settled on this:

{{< highlight python >}}
GRAMMAR = """
start: list_expression -> root_result
list_expression: expression | expression LIST_SEP list_expression -> list_expression
expression: "(" expression ")" -> parens
          | expression OPERATOR_AS expression -> binary_op_as
          | expression OPERATOR_MD expression -> binary_op_md
          | DICE_ROLL -> dice_roll
          | FLOAT -> float_num
          | INTEGER -> integer
          | NATURAL_NUM -> natural_num

DICE_ROLL: /([1-9]\d*)?d([1-9]\d*|[Ff])/i
OPERATOR_AS: "+" | "-"
OPERATOR_MD: "*" | "/"
FLOAT: /-?\d*\.\d+/
INTEGER: /-?\d+/
NATURAL_NUM: /[1-9]\d*/
LIST_SEP: /\s+/

%import common.WS_INLINE
%ignore WS_INLINE  # Only ignore inline whitespace in expressions
"""
{{< /highlight >}}

There's some extra annotation in there, but for the most part this matches what you could specify with an [EBNF Grammar](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form). The arrow aliasing is very useful later on for hooking into the Transformer class we'll write to create an evaluation tree. 

There were a few gotchas. Namely, using the `lalr` parser lead to some collisions within dice roll atom tokens when the number matcher greedily gobbled up the leading numeral in e.g. a `"3d6"` string, leaving nothing to match the remainder. Solution was to switch to `earley`. I ran into similar issues trying to prevent collisions between `list_expression` matches and `DICE_ROLL` atoms with an earlier version of the `DICE_ROLL` pattern that matched `NATURAL_NUM? "d" NATURAL_NUM` instead of using regex. The general allowance of whitespace meant that something like `"3 d6"` could mean either "roll 3d6" or "roll d6 3 times without summing them". And so was born the one bit of uglyish regex in this project.

## Tests as Guide Rails

With the grammar done, creating the parser is a one-liner:

{{< highlight python >}}
parser = Lark(GRAMMAR, parser='earley')
{{< /highlight >}}

While I was eager to get to the actual eval stage, I also wanted some basic tests to canonize certain expectations of how our roll strings should be evaluated, and to give me a little confidence when I set Copilot off the leash in agent mode. Luckily, one very useful application of an unleashed agent is creating a bedrock of use-cases. Copilot created most of the sample roll strings in `test_cases.py`, although its initial stab at the actual testing logic was "will it parse". I wanted a slightly more vigorous assertion.

In playing around with the parser, I was pleasantly surprised to learn that Lark's `Tree` class implemented its own equality checker. I could generate an expected parse tree and compare directly to actual. This required creating a few helpers in the test cases file and a minor headache the first time I defined an expected tree for a non-trivial input string, and I'll probably curse nested parentheses for the rest of the week, but once this groundwork was done, I was able to point Copilot at what I'd done for the first few examples and politely ask it to continue the pattern.

## The Giving Tree

Progress accelerated after the parser specs were written. Jury's out on whether that's because I stopped writing specs or because I could write the actual code with a little more no-longer-reckless abandon.

The final step in this first iteration was to create an evaluation tree from the parse tree. `"6 2d6+6"` parses out to the following tree, but that doesn't yet help us spit out numbers.

{{< highlight python >}}
parser.parse('2*(d6+4)').pretty()) 
root_result
  list_expression
    integer     6
     
    list_expression
      binary_op_as
        dice_roll       2d6
        +
        integer 6
{{< /highlight >}}

What we need is a mirroring tree where each node exposes an `evaluate()` function that recursively calls `evaluate()` on its children to return a final result. To lay the barest groundwork for later result analysis, these evaluators should create a results tree instead of percolating up raw results, and so I created a nearly-naked `ResultNode` class with `raw_result` and `children` attributes.

After explaining the reasoning behind the results tree and implementing one of the [more complex examples](https://github.com/jamesdylanconklin/ast_roller/blob/d26cecdcfbd308d14930704e198be29e08449425/src/ast_roller/evaluators.py#L27), I set Copilot loose. Again, just a few hiccups. 

Note that the `root_result` and `parens` matchers are not represented in `EvaluatorNode` classes. Their role is structural; once the tree's been built, they've done their job and can be omitted from the mapped evaluator tree. 

## That's (almost) all, folks!

I played with the roller in the python repl a bit before hooking it into a lambda. This involved dipping my toe back into python packaging and several kludges I'm not comfortable with on the importing end because I'm not ready to publish the roller to PyPi just yet. I'll leave imagining the relevant carnage as an exercise to the viewer and invite you to play with the roller below!

{{< dice-roller >}}

## Key Takeaways

- Copilot is very useful for tool discovery

  "I want to accomplish x. Has someone already published a package doing x in my language of choice" can short-circuit a lot of Googling.

- Copilot can help create its own leash

  Copilot made it very easy to create some foundation test cases, which made it easier to check things hadn't broken more than expected when asking it to make changes in agent mode. 

- Copilot is great at monkey-see, monkey-do

Tackling one part of a series of similar problems, highlighting my work, and explaining my thought process to the LLM assistant seemed to prime it well to attempt the rest of that series. Even if it didn't often accomplish everything perfectly, it almost always got most of the drudgery and boilerplate out of the way. 

- Logic separation really helps with readability.

  Go figure. Foundational tenet of the profession, but still obvious in comparison to the last time I did this in Python a decade or so ago. The isolated grammar can even be explained to non-technical folk, perhaps so long as you spot them a beer or some yardwork first, and the eval nodes are similarly simple and acessible. It probably also wouldn't hurt later efforts to add features. I can pretty easily see a new language feature boiling down to a few lines in the grammar with one top-level match and an evaluator class for that match node - the pattern is already present in this first iteration.

## Coda

The code discussed and demoed in this article can be found [here](https://github.com/jamesdylanconklin/ast_roller/blob/main/src/ast_roller). I'll see about polishing it up, adding a `__main__` function for terminal use, and packaging it in case anyone feels like using it.

