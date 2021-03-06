# Expressions

Expression :
  - FunctionExpression
  - ConditionalExpression
  - ControlExpression
  - LogicalExpressionOr

PrimaryExpression :
  - ReferenceExpression
  - UnitExpression
  - TupleExpression
  - RecordExpression
  - ListExpression
  - MemberExpression
  - CallExpression
  - WrappedExpression
  - BlockExpression but not WrappedExpression

WrappedExpression : `(` Expression TypeAnnotation? `)`

The organization of this section might be a bit confusing. We have {Expression} which forms what is effectively a grammar linked-list until we arrive at {PrimaryExpression}. This is because there are complicated [order-of-operations](https://en.wikipedia.org/wiki/Order_of_operations) rules we encode in our grammar. Once we get to {PrimaryExpression} we’re left with simple expressions that have a clear order-of-operations.

Note: Most of the syntax of {WrappedExpression} is ambiguous with {BlockExpression}. In the case where the two are ambiguous {WrappedExpression} wins. In practice, this doesn’t matter since the behavior is exactly the same for the ambiguous syntax. We don’t combine the two since `(x: T)` is valid syntax but not `(x = 42; x: T)`.

TODO: Literals. Strings and numbers.

## Reference Expression

ReferenceExpression :
  - BindingIdentifier
  - BindingPatternHole

We need {BindingPatternHole} for expression/pattern symmetry but they serve **no** purpose in our expressions so we error when we see them.

Note: We might consider using {BindingPatternHole} to curry functions. e.g. `myFunction(_, _, c)(a, b)`.

## Unit Expression

UnitExpression : `(` `)`

## Tuple Expression

TupleExpression : `(` TupleExpressionElementList `)`

TupleExpressionElementList :
  - TupleExpressionElement `,` TupleExpressionElement `,`?
  - TupleExpressionElement `,` TupleExpressionElementList

TupleExpressionElement : Expression TypeAnnotation?

## Record Expression

RecordExpression : `{` RecordExpressionPropertyList? `}`

RecordExpressionPropertyList :
  - RecordExpressionProperty `,`?
  - RecordExpressionProperty `,` RecordExpressionPropertyList

RecordExpressionProperty :
  - Identifier TypeAnnotation? RecordExpressionPropertyInitializer?

RecordExpressionPropertyInitializer: `=` Expression

Note: An empty {RecordExpression} (syntax: `{}`) is the same as a {UnitExpression} (syntax: `()`).

## List Expression

ListExpression : `[` ListExpressionItemList? `]`

ListExpressionItemList :
  - ListExpressionItem `,`?
  - ListExpressionItem `,` ListExpressionItemList

ListExpressionItem :
  - Expression
  - `...` OperandExpression

## Match Expression

MatchExpression : `match` Expression `{` MatchCaseList `}`

MatchCaseList :
  - MatchCase LineSeparator?
  - MatchCase LineSeparator MatchCaseList

MatchCase : Pattern MatchCaseCondition? `->` MatchCaseBody

MatchCaseCondition : `if` Expression

MatchCaseBody :
  - Expression
  - ControlStatement

Note: {MatchExpression} cases are surrounded by curly brackets (`{}`) instead of parentheses (`()`). This is aesthetically different from other Brite “blocks.” It makes sense here because our case list is categorically different from what normally goes between parentheses—statements or expressions. This has the side-effect of making parsing {MatchExpression} easier.

## Match Condition Expression

MatchConditionExpression : OperandExpression [lookahead != LineTerminator] `match` Pattern

This feature enables programmers to easily use pattern matching on their data structure at the expression level. Adding a whole new convenient avenue for refining human data into computer types.

Note: Any {BindingPattern} in the {Pattern} will only be bound for code reachable if the {MatchConditionExpression} evaluated to true. If there are two {MatchConditionExpression} in a {BinaryExpressionLogicalOr} and they have the same {BindingPattern}s then they must have the same type.

Note: We force the expression and `match` to be on the same line to avoid syntactic ambiguity.

## Block Expression

BlockExpression : `(` BlockExpressionStatementList `)`

BlockExpressionStatementList :
  - Statement LineSeparator?
  - Statement LineSeparator BlockExpressionStatementList

Note: We could change the syntax so that tuples also accept statement lists. However, then you can write programs with non-obvious operator precedance like: `(x = 1; x, y = 2; y)`. The program `((x = 1; x), (y = 2; y))` is always much clearer so we force that syntax.

Note: In {WrappedExpression} we allow an optional {TypeAnnotation}. However, unwrapped annotations are not allowed in expression statements. For consistency we force you to add parentheses around your annotated expressions. You can’t write the program `(x = 1; x: T)`, so you must write `(x = 1; (x: T))`.

## Member Expression

MemberExpression : PrimaryExpression `.` Identifier

## Call Expression

CallExpression : PrimaryExpression GenericArguments? CallExpressionArguments

CallExpressionArguments : [lookahead != LineTerminator] `(` CallExpressionArgumentList? `)`

CallExpressionArgumentList :
  - Expression `,`?
  - Expression `,` CallExpressionArgumentList

Note: {CallExpressionArguments} notes that there cannot be a {LineTerminator} in between the expression we are calling and the arguments. Since on a newline the arguments would be ambiguous with a {TupleExpression} statement.

Note: {GenericArguments} introduces ambiguity with {RelationalExpression} for LR(1) parsers. Brite implementations will have to deal with this. For example `f < x > ()` could be interpreted as two {RelationalExpression}.

## Function Expression

FunctionExpression :
  - Function
  - Identifier `->` FunctionBody

Note: You may notice the potential for significant syntactic ambiguity. Consider `(a`. Is that the start of a tuple expression `(a, b)` or the start of a function expression `(a, b) -> a + b`? We address this in [Pattern Expression Symmetry](#sec-Pattern-Expression-Symmetry).

## Conditional Expression

ConditionalExpression :
  - `if` Expression `then` Expression ConditionalExpressionAlternate?

ConditionalExpressionAlternate : `else` Expression

Note: Our syntax forbids `a + if x then y else z + b` since {ConditionalExpression} may not be an operand to a binary expression like {AdditiveExpression}. This resolves syntactic ambiguity forcing the programmer to wrap their conditional like so `a + (if x then y else z) + b`.

## Control Expression

ControlExpression :
  - ReturnExpression
  - `break`
  - `continue`

ReturnExpression :
  - `return` [lookahead LineTerminator]
  - `return` [lookahead != LineTerminator] Expression

Controls the execution of a Brite program. The most common control expression, {ReturnExpression}, allows the programmer to finish the execution of their function early and return the argument passed to the expression. `break` allows the programmer to stop the execution of the loop they are currently in. `continue` allows the programmer to skip the current iteration of the loop they are currently in.

Since Brite strongly encourages functional programming, one won’t often see the use of these control expressions as they are only necessary in imperative programming styles. One of the beauties of Brite is that it elegantly allows for both functional and imperative styles.

Note: {ControlExpression} is an expression instead of a statement so that they may be placed anywhere in a expression that is conditionally executed. For example `if x then return y` or `x || continue`.

Note: {ReturnExpression} may only have an {Expression} argument if that expression is on the same line as the `return` token. If there are no more tokens on the same line as the `return` then {ReturnExpression} receives no argument.

## Logical Expression

LogicalExpressionOr :
  - LogicalExpressionOr `||` LogicalExpressionAnd
  - LogicalExpressionAnd

LogicalExpressionAnd :
  - LogicalExpressionAnd `&&` EqualityExpression
  - EqualityExpression

## Equality Expression

EqualityExpression :
  - EqualityExpression `==` RelationalExpression
  - EqualityExpression `!=` RelationalExpression
  - RelationalExpression

Checks whether two values are equal to each other based on some equality interface defined in the standard library.

Note: Brite does not support “referential equality” unlike other languages we take inspiration from like JavaScript and OCaml. Since referential equality presumes a certain object layout at runtime. Instead we do deep equality checks. Referential equality checks may be still used underneath the hood as an optimization technique for applicable data structures.

Chained equality expressions of the same kind are treated as a test of the equality for multiple values. That is `a == b == c` is the same as `a == b && b == c`. This makes testing equality for a tripple quite simple.

Note: The chaining feature assumes a proper implementation of the equality interface that is transitive since `a == b == c` is only rewritten to `a == b && b == c`. We assume `a == c` so we don’t check that assumption.

## Relational Expression

RelationalExpression[WithoutLessThan] :
  - [~WithoutLessThan] RelationalExpression `<` AdditiveExpression
  - RelationalExpression[WithoutLessThan] `>` AdditiveExpression
  - RelationalExpression `<=` AdditiveExpression
  - RelationalExpression `>=` AdditiveExpression
  - AdditiveExpression

Checks the ordering relationship between two values based on some ordering interface defined in the standard library.

Note: We disallow the syntax in {RelationalExpression} for `a < b > c` since angle brackets (`<>`) are commonly used to denote other things. Such as generic type arguments or XML markup.

Chained relational expressions in the same direction are treated as a test on the ordering of all the elements. That is `a < b < c` is the same as `a < b && b < c`. This makes testing if a value is in a given range quite easy, for example: `0 <= x <= 20`.

Note: The chaining feature assumes a proper implementation of the ordering interface that is transitive since `a < b < c` is only rewritten to `a < b && b < c`. We assume `a < c` so we don’t check that assumption.

## Additive Expression

AdditiveExpression :
  - AdditiveExpression `+` MultiplicativeExpression
  - AdditiveExpression `-` MultiplicativeExpression
  - MultiplicativeExpression

## Multiplicative Expression

MultiplicativeExpression :
  - MultiplicativeExpression `%` UnaryExpression
  - MultiplicativeExpression `*` UnaryExpression
  - MultiplicativeExpression `/` UnaryExpression
  - UnaryExpression

## Unary Expression

UnaryExpression :
  - `-` UnaryExpression
  - `!` UnaryExpression
  - PrimaryExpression
