# Objective

The objective of this lab is to implement [constant folding](https://en.wikipedia.org/wiki/Constant_folding) for a subset of Java and use black-box testing to test its functional correctness. The implementation will use the [org.eclipse.jdt.core.dom](https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.jdt.doc.isv%2Freference%2Fapi%2Forg%2Feclipse%2Fjdt%2Fcore%2Fdom%2Fpackage-summary.html) to represent and manipulate Java.  The [constant folding](https://en.wikipedia.org/wiki/Constant_folding) itself should be accomplished with a specialized [ASTVisitor](https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.jdt.doc.isv%2Freference%2Fapi%2Forg%2Feclipse%2Fjdt%2Fcore%2Fdom%2Fpackage-summary.html). The program will take two arguments as input:

  1. the Java file on which to do constant folding; and
  2. an output file to write the result.

The program should only apply to a limited subset of Java defined below. It should fold any and all constants as much as possible. It should not implement **constant propagation** which is the topic of the next lab for the course.
 
The part of the program that does the actual folding should be specified and tested via black-box testing. A significant part of the grade is dedicated to the test framework and the tests (more so than the actual implementation), so be sure to spend time accordingly. In the end, the test framework and supporting tests are more interesting than the actual constant folding in regards to the grade.

# Reading

The [DOM-Visitor lecture](https://bitbucket.org/byucs329/byu-cs-329-lecture-notes/src/master/DOM-Visitor/) is a must read before starting this lab. You will also need the [DOMViewer](https://github.com/byu-cs329/DOMViewer.git) installed and working to view the AST for any given input file. Alternatively, there is an [ASTView plugin](https://www.eclipse.org/jdt/ui/astview/) for Eclipse available on the Eclipse Market Place that works really well. There is something similar for IntelliJ too.

# Constant Folding

[Constant folding](https://en.wikipedia.org/wiki/Constant_folding) is the process where constant expressions are reduced by the compiler before generating code. 
 
Examples:

  * `x = 3 + 7` becomes `x = 10`
  * `x = 3 + (7 + 4)` becomes `x = 14`
  * `x = y + (7 + 4)` becomes `x = y + 11`

Not that constant folding does not include replacing variables that reduce to constants. 

```java
x = 3 + 7;
y = x;
```

Constant folding for the above gives:

```java
x = 10;
y = x;
```

Constant folding may also apply to Boolean values.

```java
if (3 > 4) {
  x = 5;
} else {
  x = 10;
}
```

Should reduce to

```java
if (false) {
  x = 5;
} else {
  x = 10;
}
```

Which should reduce to

```java
x = 10;
```

In the above example, the constant folding removed **dead code** that was not reachable on any input.

Be aware that [short circuit evaluation](https://en.wikipedia.org/wiki/Short-circuit_evaluation) comes into play for constant folding. For example ```a.f() && false``` cannot be reducted to ```false``` because it is not known if the call to ```a.f()``` side effects on the state of the class, so it's call must be preseved. That said, ```a.y() && false && b.y()``` can be reduced to ```a.y() && false``` since the call to ```b.y()``` will never take place.  As another example, consider ```a.y() || b.y() || true || c.y()```. Here the call to ```c.y()``` will never take place so the expression reduces to ```a.y() || b.y() || true```.

It is also possible to remove literals that have no effect on the expression. For example ```a.y() && true``` reduces to ```a.y()```. Similarly, ```a.y() || false || b.y()``` reduces to ```a.y() || b.y()```.  Always be aware of [short circuit evaluation](https://en.wikipedia.org/wiki/Short-circuit_evaluation) in constant folding since method calls can side-effect on the state of the class, those calls must be presered as shown in the examples above. 

Here is another example of short circuit evaluation that requires some careful thought.

```java
if (a.f() || true) {
  x = 10;
}
```

This code could reduce to

```java
a.f();
if (true) {
  x = 10;
}
```

Which would then reduce to

```java
a.f();
x = 10;
```

That said however, the following example cannot reduce in the same way because if the call to ```a.y()``` returns true, then ```b.y()``` is never called. 

```java
if (a.f() || b.y() || true) {
  x = 10;
}
```

It could reduce to

```java
if (a.f() || b.y()) {
   ;
}
if (true) {
  x = 10;
}
```

Which would then reduce to

```java
if (!a.f()) {
  b.y();
}
x = 10;
```

This level of reduction (short-circuiting with three operands) is not required for this lab. It can be implemented, with appropriate tests, for extra credit if so desired.

## Some further notes on folding

Do not fold outside of `ParenthesizedExpressions` unless that expression contains as single `NumberLiteral` (e.g., `(10)`). If such is the case, replace the `ParenthesizedExpression` with a `NumberLiteral` (e.g. `(10)` becomes `10`).

Replace if-statements or other statements that are removed because of folding with an instance of the  `EmptyStatement`. Consider the below example.

```java
if (false) {
  x = 10;
}
```

The above code reduces to `;` which is the `EmptyStatement`. Using the `EmptyStatement` in the DOM avoids having to deal with empty blocks or empty expressions in if-statements as in the next example.

```java
if (x)
  if (false);
```

The above example reduces to

```java
if (x);
```

Removing empty statements is not required---completely uncompensated and not required.

# Java Subset

This course is only going to consider a very narrow [subset of Java](https://bitbucket.org/byucs329/byu-cs-329-lecture-notes/src/master/java-subset/java-subset.md) as considering all of Java is way beyond the scope and intent of this course (e.g., it would be a nightmare). The [subset of Java](https://bitbucket.org/byucs329/byu-cs-329-lecture-notes/src/master/java-subset/java-subset.md) effectively reduces to single-file programs with all the interesting language features (e.g., generics, lambdas, anonymous classes, etc.) removed. **As a general rule**, if something seems unusually hard then it probably is excluded from the [subset of Java](https://bitbucket.org/byucs329/byu-cs-329-lecture-notes/src/master/java-subset/java-subset.md) or should be excuded, so stop and ask before spending a lot of time on it.

# Where to Apply Constant Folding

Constant folding is **only applied** to the following types of expressions and statements (ordered roughly from easiest to hardest)

  * `ParenthesizedExpressions` that contain only a literal
  * Logical `PrefixExpressions` for `!`
  * Numeric `InfixExpressions` for `+` and `-` and if and only if all the operands are of type `Literal` including the extended operands
  * Binary relational `InfixExpressions` for `<` and `==`
  * Binary logical `InfixExpressions` for `||` and `&&`
  * `IfStatement`
  * `WhileStatement`
  * `DoStatement`

This set of expressions and statemenst are more narrow than what is allowed in the Java subset. That is OK. Folding is only applied to the above program features.  Also, folding should be applied iteratively until no furter reduction is possible.

# Lab Requirements

  0. Write a specification for each type of supported folding. The specification should include a model of the [org.eclipse.jdt.core.dom](https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.jdt.doc.isv%2Freference%2Fapi%2Forg%2Feclipse%2Fjdt%2Fcore%2Fdom%2Fpackage-summary.html) and a convenient notation to quantify over the supported types for folding in the [org.eclipse.jdt.core.dom](https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.jdt.doc.isv%2Freference%2Fapi%2Forg%2Feclipse%2Fjdt%2Fcore%2Fdom%2Fpackage-summary.html). 
  1. Implement a black-box functional test framework for each type of expression or statement that is supported that takes as input an [org.eclipse.jdt.core.dom](https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.jdt.doc.isv%2Freference%2Fapi%2Forg%2Feclipse%2Fjdt%2Fcore%2Fdom%2Fpackage-summary.html) and tests if the updated object after folding is as expected. Assume that any input [org.eclipse.jdt.core.dom](https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.jdt.doc.isv%2Freference%2Fapi%2Forg%2Feclipse%2Fjdt%2Fcore%2Fdom%2Fpackage-summary.html) conforms to the Java subset though you are not required to test this assumption. The tests should be organized in a way to make clear the black-box test methodology and its relation to the specification.
  2. Implement constant folding for each supported feature by implementing visitors on the [org.eclipse.jdt.core.dom](https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.jdt.doc.isv%2Freference%2Fapi%2Forg%2Feclipse%2Fjdt%2Fcore%2Fdom%2Fpackage-summary.html). Each supported feature should be implemented by a unique visitor. For example, implement one visitor to fold `ParenthesizedExpressions`, a second visitor to fold `PrefixExpressions`, a third visitor to fold numeric `InfixExpressions` with only literals, etc. The visitors are applied iteratively until no further reductions take place.

## Suggested order of attack:

Approach the lab is small steps starting with the easiest type of folding and then growing out. For example, start with just `ParethesizedExpressions` that contain only a literal. Write the specification. Create the test framework with tests. And then implement the actual visitor. Repeat the process until done.

Write the specification, test framework, tests first. These are the most important not just because they are worth more points but because in test driven development, if you write a good set of tests, then those tests set the agenda for the implementation and signal when the implementation is done.

Organize the tests in a way that communicates the reasoning behind the tests. Use the `@Nested`, `@Tag`, and `@DisplayName` to convey clearly how and why the tests are grouped.

For the test framework, here are some things to consider:

  * How will you determine that the output is correct (i.e., that constant folding happened correctly)? In the [DOM-Visitor](https://bitbucket.org/byucs329/byu-cs-329-lecture-notes/src/master/DOM-Visitor/) lecture, the `NumberLiteral` expressions were recorded after the folding and verified to meet a specific criteria. That is only a starting point and not a solution. Is there one method that will work for all the ways to fold?
  * How can you partition the input space to systematically cover ways that constant folding may appear and affect code?  In the [DOM-Visitor](https://bitbucket.org/byucs329/byu-cs-329-lecture-notes/src/master/DOM-Visitor/) lecture, it starts with a single expression with only two `NumberLiteral` operands. It then adds a single binary expression, with one operand being a binary expression with only `NumberLiteral` operands. Consider such a starting point. Also consider how constant folding might change code or remove code. These too should be part of the input partitioning exercise.
  * What would a boundary value analysis look like?
  * How can you isolate things to only test one feature at time? For example, it may not be wise to write a test to see if **dead code** is removed correctly that relies on the implementation for short-circuit evaluation of boolean expressions to be working as expected.

Do not implement more than is required by the test, and do not be afraid to create a lot of tests. Let the test define what needs to be implemented. Create a test for everything that needs to be implemented according to the input partitions. Add code to throw `UnsupportedException` for things not yet implemented as you code along. For example, if additions is the only thing implemented so far, then expressions with other operators should throw on exception. In this way it is easier to keep track of what is yet to be implemented.

# What to turn in?

Create a pull request when the lab is done. Submit to Canvas the URL of the repository.

# Rubric

| Item | Point Value | Your Score |
| ------- | ----------- | ---------- |
| Oracle(s) strong enough to determine when output is valid | 50 | |
| Tests that divide the input space in a reasoned, systematic, and somewhat complete way | 50 |
| Appropriate use of `@Nested`, `@Tag`, and `@DisplayName` to organize and communicate the test methodology | 30 | | 
| Implementation of constant folding | 60 | |
| Adherence to best practices (e.g., good names, no errors, no warnings, documented code, well grouped commits, appropriate commit messages, etc.) | 10 | |
| Extra credit short circuit reduction, at least two tests (+20) | 0 | |

Breakdown of Implementation **(60 points)**

  * **(10 points)** Numeric `InfixExpressions` (e.g., `+`, `-`,`*`, and `/`) when all operands are literals (including extended operands)
  * **(5 points)** Binary (i.e., no extended operands---exactly two operands) relational `InfixExpressions` (e.g. `<`, `>`, `<=`, `>=`, `==`, and `!=`)
  * **(10 points)** Binary logical `InfixExpressions` (e.g., `||`, `&&`)
  * **(5 points)** Logical `PrefixExpressions` (e.g., `!`)
  * **(10 points)** `IfStatement` (reduction and short-circuiting with two operands)
  * **(10 points)** `WhileStatement` (reduction and short-circuiting with two operands)
  * **(10 points)** `DoStatement` (reduction and short-circuiting with two operands)
  
