# brainfuck-macros
Macro expander for generating Brainfuck programs

Usage
=====

	compile.py inputFile outputFile

Note that only a single input file can be specified; if you want to make
use of macro libraries, you could, for instance, first concatenate all source
files together, and pass the result to compile.py.

Language reference
==================

The output is generated from the input by performing these transformations:
* Remove comments. Comments start with a '#' and end with a newline.
* Evaluate the resulting file as a code block (see below), using an empty
  namespace.
* Evaluate save-and-recall commands (see below).
* Remove anything that is not Brainfuck code.
* Optimize the Brainfuck code a bit.


Code block
---------
A code block can contain the following elements:
* Macro definitions
* Macro invocations
* Variable invocations
* Code (Brainfuck commands and save-and-recall commands)
* Whitespace

Whitespace is ignored, except where it is necessary to separate other
syntactic elements. For instance, 'a b' is syntactically different from 'ab',
but 'a \t\n   b' is syntactically equal to 'a b'.

Code block evaluation uses a namespace as input. A namespace contains a
name -> value mapping, where the value is a piece of code.

A code block is evaluated in the following way:
* Macro definitions are read and removed.
* Any macro invocations that match macro definitions in the code block are
  evaluated. This is performed repeatedly, until there are no more matches.
* Any variable invocations that match namespace elements are evaluated.
  This is performed repeatedly, until there are no more matches.
* Any invocation that does not match anything is left unchanged.


Macro definition
----------------
Syntax: name([variable0[;variable1[;...]]]){block}

name, variable0, variable1, ... are identifiers. block is a code block.


Macro invocation
----------------
Syntax: name([value0[;value1[;...]]])

name is an identifier; a macro invocation matches a macro definition if their
names are equal.

value0, value1, ... are pieces of code. They are evaluated prior to the
evaluation of the macro invocation, as part of the evaluation of the code block
that contains the macro invocation, using the same namespace as that code block.

A namespace is constructed using variable0, variable1, ... from the macro
definition as names and value0, value1, ... from the invocation as values.

Evaluation of the macro invocation consists of evaluation the code block of the
macro definition using the newly constructed namespace, and replacing the
macro invocation with the resulting code.

Variable invocation
-------------------
Syntax: name

name is an identifier; a variable invocation matches a namespace element if
their names are equal.

Evaluation of the variable invocation consists of replacing the variable
invocation with the value of the namespace element.

Identifier
----------
An identifier has to conform to the following rules:
* It does not contain whitespace. Note that this allows whitespace to be used
  to unambiguously separate identifiers from other identifiers and other syntax
  elements.
* It does not contain any of the characters in "{}();". Note that this allows
  the use of identifiers in macro definitions and invocations without separating
  whitespace.
* It contains at least one character that is not a Brainfuck command or a
  save-and-recall command. Note that this allows to distinguish identifiers from
  literal code.

Save-and-recall commands
------------------------
Save-and-recall commands are used as a tool to remember and move back to certain
memory locations. These are processed by walking in a left-to-right direction
through the code, while keeping track of a *stack* and a *current position*.
Initially, the stack is empty and the current position is zero.

The following commands exist:
* ! ("push"): The current position is pushed to the stack. Subsequently, the
  current position is set to zero. The command is removed from the code.
* ? ("recall"): The command is replaced by the number of '>' or '<' indicated
  by the current position: '<' for a positive current position or '>' for a
  negative current position. Subsequently, the current position is set to zero.
* ~ ("pop"): The current position is popped from the stack. The command is
  removed from the code.
* &gt; : The current position is incremented. The command is kept in the code.
* &lt; : The current position is decremented. The command is kept in the code.

Note that this works quite intuitively on simple code, but mixing
save-and-recall with complicated Brainfuck code (especially when '>' and '<' are
not matched inside a loop) can give unexpected results. In those cases it is
usually a good approach to push before the complicated code, pop afterwards,
and manually ensure that the complicated code always ends at the same memory
location as where it started.

Examples
========

Comments
--------
Input:
	+       #x = 1
	> ++ <  #y = 2

	# Add x and y:
	[       #while x
	-       #    x--
	> + <   #    y++
	]       #(Result: y += x; x = 0)
Output:
	+>++<[->+<]


Save-and-recall
---------------
Input:
	!
	?  +   #x = 1
	?> ++  #y = 2

	# Add x and y:
	?  [       #while x
	?  -       #    x--
	?> +       #    y++
	?  ]       #(Result: y += x; x = 0)

	?~
Output:
	+>++<[->+<]
Note: the pushing and popping might not be needed here, but it is good
practice for when we are going to write macros.


Simple macros
-------------
Input:
	x(){}
	y(){>}

	!
	? x() +   #x = 1
	? y() ++  #y = 2

	# Add x and y:
	? x() [       #while x
	? x() -       #    x--
	? y() +       #    y++
	? x() ]       #(Result: y += x; x = 0)

	?~
Output:
	+>++<[->+<]


Function macros
---------------
Input:
	# Add x to y:
	# y += x; x = 0
	addTo(x;y)
	{
		!
		? x [       #while x
		? x -       #    x--
		? y +       #    y++
		? x ]
		?~
	}

	x(){}
	y(){>}

	!
	? x() +   #x = 1
	? y() ++  #y = 2

	? addTo(x(); y())

	?~
Output:
	+>++<[->+<]
Note: inside the addTo macro, x and y are variables, so we have to use variable
invocations ("x" and "y") instead of macro invocations("x()" and "y()").


Control structures
------------------
Input:
	0(){[-]}

	#if(x) code1 else code2
	ifelse(x;code1;code2;stack)
	{
		t0(){stack}
		t1(){stack >}

		!
		? t0() 0() +      #t0 = 1
		? t1() 0()        #t1 = 0
		? x    [          #if x
		?          code1  #    code1
		? t0()     -      #    t0--
		? x        [      #    while x
		? t1()         +  #        t1++
		? x        -]     #        x--
		? x    ]
		? t1() [          #while t1
		? x        +      #    x++
		? t1() -]         #    t1--
		? t0() [          #if t0
		?          code2  #    code2
		? t0() -]         #    t0--
		?~
	}


	ifelse(>>;++++++++;--------;>>>)
Output:
	>>>[-]+>[-]<<[<<++++++++>>>-<[>>+<<-]]>>[<<+>>-]<[<<<-------->>>-]<<<
This demonstrates, among other things:
* Macros can be defined inside macro definitions. Since reading and removing of
  macro definitions is the first step in evaluating a code block, their scope is
  limited to the inside of the macro.
* Code can be passed as argument value.
* Macros defined outside a macro can be used inside the macro: in this example,
  0() is used inside ifelse(). This works because, in the initial evaluation of
  the ifelse() invocation, 0() is unmatched, and it is kept un-evaluated.
  However, evaluation of the global code block keeps iterating, and in the next
  iteration, 0() is recognized and evaluated.

