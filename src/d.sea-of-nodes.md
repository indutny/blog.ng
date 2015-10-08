---
title: Sea of Nodes
date: 2015-10-08
---

## Outline

1. Compilers are translating
2. Text, AST - not ok for optimizations, need for data-flow graph
3. Usually control graph is totally separated from data-flow graph, CFG
4. Sea-of-nodes - same graph, two kinds of edges
5. Graph reductions
6. Reachability + const prop example
7. Scheduling stuff back in blocks
8. Generating machine code

## Brief intro

This post is going to be about the sea-of-nodes compiler concept that I have
recently learned.

While it is completely not necessary, it may be useful to take a peek at the
some of my previous posts on JIT-compilers before reading this:

* [How to start JIT-ting][0]
* [Allocating numbers][1]
* [SMIs and Doubles][2]
* [Deoptimize me not, v8][3]

## Compilers = translators

Compilers is something that every Software Engineer uses several times a day.
Surprisingly even people who consider themselves to be far from writing the
code, still use the compilers quite heavily throughout their day. This is
because most of the web depends on client-side code execution, and many of such
client-side programs are passed to the browser in a form of the source code.

Here we come to important thing: while source code is (usually) human-readable,
it looks like a complete garbage to your laptop/computer/phone/...'s CPU. On
other hand, machine code, that computers **can** read, is (almost always) not
human-readable. Something should be done about it, and the solution to this
problem is provided by the process called **translation**.

Trivial compilers do a single pass of _translation_: from the source code to the
machine code. More usual compilers do at least two passes: from the source code
to [Abstract Syntax Tree][4] (AST), and from [AST][4] to machine code. [AST][4]
in this case acts like an _Intermediate Representation_ (IR), and as the name
suggests, [AST][4] is just another form of the same source code.

There is no limit on the layer count. Each new layer brings the source program
closer to how it will look like in machine code.

## Optimization layers

However, not all layers are used just for translation. Many compilers are also
try to optimize the human-written code. (There is always a balance between code
elegance and code performance).

Take the following JavaScript code, for example:

```javascript
for (var i = 0; i < arr.length; i++)
  arr[i] += 1;
```

If the compiler would translate it to the machine code straight out of [AST][4],
it may look this (in very abstract and detached from reality instruction set):

```
i = 0;
loop {
  // Load `.length` field of arr
  tmp = loadArrayLength(arr);
  if (i >= tmp)
    break;

  // Check that `i` is between 0 and `arr.length`
  checkRange(arr, i);

  // Load value
  tmp = load(arr, i);

  // Store modified value
  store(arr, i, tmp + 1);

  i += 1;
}
```

It may not be obvious, but this code is far from optimal. Length does not really
change inside of the loop, and the range checks are not necessary at all.
Ideally, it should look like this:

```
i = 0;
len = loadArrayLength(arr);
loop {
  if (i >= tmp)
    break;

  tmp = load(arr, i);
  store(arr, i, tmp + 1);
  i += 1;
}
```

Let's try to imagine how we could do this.

Suppose we have an [AST][4] at hand, and we try to generate the machine code
straight out of it:

_(NOTE: Generated with [esprima][6])_
```javascript
{ type: 'ForStatement',

  //
  // This is `var i = 0;`
  //
  init:
   { type: 'VariableDeclaration',
     declarations:
      [ { type: 'VariableDeclarator',
          id: { type: 'Identifier', name: 'i' },
          init: { type: 'Literal', value: 0, raw: '0' } } ],
     kind: 'var' },

  //
  // `i < arr.length`
  //
  test:
   { type: 'BinaryExpression',
     operator: '<',
     left: { type: 'Identifier', name: 'i' },
     right:
      { type: 'MemberExpression',
        computed: false,
        object: { type: 'Identifier', name: 'arr' },
        property: { type: 'Identifier', name: 'length' } } },

  //
  // `i++`
  //
  update:
   { type: 'UpdateExpression',
     operator: '++',
     argument: { type: 'Identifier', name: 'i' },
     prefix: false },

  //
  // `arr[i] += 1;`
  //
  body:
   { type: 'ExpressionStatement',
     expression:
      { type: 'AssignmentExpression',
        operator: '+=',
        left:
         { type: 'MemberExpression',
           computed: true,
           object: { type: 'Identifier', name: 'arr' },
           property: { type: 'Identifier', name: 'i' } },
        right: { type: 'Literal', value: 1, raw: '1' } } } }
```

This JSON could also be visualized:
![AST][7]

This is a tree, so it is very natural to traverse it from the top to the bottom,
generating the machine code as we visit the AST nodes. The problem with this
approach is that the information about variables is very sparse, and is spread
through the different tree nodes.

To safely move the length lookup out of the loop we need to know that the array
length does not change between the loop's iterations. Humans can do it easily
just by looking at the source code, but the compiler needs to do lots of work to
figure this out of the AST.

As many other compiler problems, this one is commonly solved by introducing one
more intermediate representation layer. What's needed to optimize this
particular case is a thing called data-flow graph. Instead of talking about
syntax-entities (like `for loop`s, `expressions`, ...), we should talk about the
data itself (read variables values), and how it changes through the program.

## Data-flow Graph

In our particular example, the data we are interested in is the value of
variable `arr`. We want to be able to easily observe all uses of it to verify
that there are no out-of-bands accesses and other length-changing uses of that
array.

This is accomplished by introducing def-use (definition and uses) relationship
between the different data values, and it means that the value is declared once
(_node_), and is used somewhere to create new values (_edge_). Obviously,
connecting different values together will form a **data-flow graph** like this:

![Data-flow Graph][8]

Note the red `array` box in this vast graph. The solid arrows going out of it
represent uses of this value. By iterating them compiler can tell that `array`
value is used at:

- `loadArrayLength`
- `checkRange`
- `load`
- `store`

It could also follow back the dashed lines (these lines are very important, and
we will discuss them a bit later) going to `checkRange` to verify that
both `load` and `store` are "guarded" by it, and won't cause array length to
change. Compiler concludes that it is safe to move `loadArrayLength` out of
the loop.

By going through the graph further, we may observe that the value of `ssa:phi`
node is always between `0` and `arr.length`, so the `checkRange` may be removed
altogether too.

Pretty neat, isn't it?

## Control Flow Graph

We just used some form of [data-flow analysis][9] to figure out information from
the program, to be able to make safe assumptions about how it could be
optimized.

This _data-flow representation_ is very useful in many other cases too. The only
problem is that by turning our code into this kind of graph, we made a step
backwards in our representation chain (from the source code to the machine
code). This intermediate representation is less suitable for generating machine
code than even the AST.

The reason is that the machine is just a sequential list of instructions, which
CPU executes one-after-another, and graph does not nearly look like something
similar. In fact, there is no enforced ordering in it at all.

Usually, this is solved by grouping the graph nodes into blocks, and calling the
thing a [Control Flow Graph][10] (CFG). Example:
```
b0 {
  i0 = literal 0
  i1 = array
  i2 = jump ^b0
}
b0 -> b1
b1 {
  i3 = ssa:phi ^b1, i0, i14
  i4 = loadArrayLength i1
  i5 = jump ^i3
}
b1 -> b2
b2 {
  i6 = cmp "<", i3, i4
  i7 = if ^b2, i6
}
b2 -> b3, b4
b3 {
  i8 = checkRange ^b3, i1, i3
  i9 = load ^i8, i1, i3
  i10 = literal 1
  i11 = add i9, i10
  i12 = store ^i9, i1, i3, i11
  i13 = literal 1
  i14 = add i3, i13
  i15 = jump ^i12
}
b3 -> b1
b4 {
  i16 = return ^b4
}
```

It is called graph not without the reason. These `bXX` are blocks, and
`bXX -> bYY` represent a graph edge. Let's visualize it:

![CFG][12]

As you can see, there is code before the loop in block `b0`, loop header in
`b1`, loop test in `b2`, loop body in `b3`, and exit node in `b4`.

Translation to machine code is very easy from this form. We just replace `iXX`
identifiers with CPU register names (in some sense, CPU registers are sort of
variables, but CPU has just fixed amount of them, so we need to be careful to
not run out of them), and generating machine code for each instruction,
line-by-line.

[CFG][10] has data-flow relations, it also has ordering, so we can do both
data-flow analysis and machine code generation out of it... but optimizing it,
moving things around blocks is very complicated and error-prone.

Instead, Clifford Click and Keith D. Cooper [proposed to use][11] an approach
called **sea-of-nodes**, the very topic of this blog post!

## Sea-of-Nodes

Remember our fancy data-flow graph with dashed lines? These lines is what makes
that Intermediate Representation **sea-of-nodes**.

Instead of grouping nodes in blocks and ordering them, we choose to declare the
control dependencies as the dashed lines in a graph. If we will take that graph,
remove everything non-dashed, and group things a bit we will get:

![Control-flow part of Sea-of-Nodes][13]

With a bit of imagination and node reordering, we can see that this graph is the
same as the simplified CFG graphs that we have just seen above:

![CFG][12]

Let's take another look at the **sea-of-nodes** representation:

![Sea-of-Nodes][8]

The striking difference

[0]: /4.how-to-start-jitting
[1]: /5.allocating-numbers
[2]: /6.smis-and-doubles
[3]: /a.deoptimize-me-not
[4]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[5]: https://en.wikipedia.org/wiki/SIMD
[6]: https://github.com/jquery/esprima
[7]: /images/ast.svg
[8]: /images/data-flow.svg
[9]: https://en.wikipedia.org/wiki/Data-flow_analysis
[10]: https://en.wikipedia.org/wiki/Control_flow_graph
[11]: http://www.researchgate.net/profile/Cliff_Click/publication/2394127_Combining_Analyses_Combining_Optimizations/links/0a85e537233956f6dd000000.pdf
[12]: /images/cfg.svg
[13]: /images/control-flow-sea.svg
