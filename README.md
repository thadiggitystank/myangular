# Summary

Outline the features covered in each chapter

## Chapter 1 - Scopes and Dirty-checking

* The two-sided process underlying Angular’s dirty-checking: $watch and $digest.
* The dirty-checking loop and the TTL mechanism for short-circuiting it.
* The difference between reference-based and value-based comparison.
* Exception handling in the Angular digest.
* Destroying watches so they won’t get executed again.

## Chapter 2 - Scope Methods

* $eval - $eval doesn't have much purpose yet. Later it will allow passing of string expressions, and be responsible for compiling and executing them. It also has the added benefit of being explicit that code is dealing with the scope.
* $apply - standard way for integrating external code into angular lifecycle. Simple executes a method (which gets access to scope object) and triggers a digest.
* $evalAsync - allows scheduling of a function to run later in the same digest. Preferable to $timeout because $timeout is beholden to browser event loop scheduling. $evalAsync ensures that it will run in the digest and prevent unnecessary re-rendering. $evalAsync should also schedule a digest if none is running.
* Scope Phases - lets angular internals query when a digest is running. '$digest' and '$apply' are two of them, otherwise it will be null.
* $applyAsync - coalesce many $apply invocations. Rather than happen immediately, they are scheduled to run soon. This also guards you against calling $apply while another digest is running, which will throw an exception. applyAsync in that sense is a safer operation and has the same desired effect. Note that it always defers the invocation. Main goal is optimization, to execute invocations that are scheduled in quick succession in one digest.
* $$postDigest - schedules a function to run after the next digest completes. Unlike apply, does not schedule a digest.
* $watchGroup - takes several watch functions and fires listener when any of them change. It should also defer the listener call to a moment when all watches have been checked so we don't run them multiple times.

## Chapter 3 - Scope Inheritance

Relies heavily on Javascript prototypal inheritance with a few added bells and whistles. We will go over both regular and isolated forms of scope. 

* Root Scope - this is effectively what we have been working with to date. A root scope has no parent. 
* $new - used to make child scopes. A child scope by default shares the properties of its parent's scope. 'inheritance' describe block has a good set of tests describing the behavior we should expect from a child scope. 
* attribute shadowing - A consequence of prototypal inheritance is that attributes on the child can 'hide' attributes on the parent. This is referred to as attribute shadowing. This can be confusing when a child wants to modify a parent's member. The workaround for this is, on the parent, to wrap the attribute in an object. Then the child can set the property on the parent.
* Separated watches - with current implementation, all watchers are stored on root scope. This means, any time we want to trigger a digest we have to trigger all watchers in the scope hierarchy. 
* Recursive Digest - we don't want watches to run **up** the hierarchy but we do want them to run **down** the hierarchy. Therefor, scopes need to know what children they have. We will implement this using a $$children array, while angular implements this as a linked list ($$nextSibling, $$prevSibling, etc) for performance reasons. 
* $apply and $evalAsync should trigger digests from root. 
* Isolated Scopes - we create it from a parent, but do not set the parent to its prototype, thus disabling prototypal inheritance. This is handled by passing true to the $new call. 
* Substitue parent scope - scopes also allow the ability to provide some other scope to be the parent of the new scope, while still maintaining the normal inheritance chain. 
* Destroying scope - means that all of its watchers are removed and that the scope itself is remove from the $$children of its parent.

## Chapter 4 - Watch Collections

* Provide an efficent way to watch for groups of changes in objects and arrays. Has the array changed? Have items been added/removed/reordered. We are doing this with value based equality checking but it has to deep watch the entire graph. $watchCollection will be an optimized version of the value-based $watch we already have. We will now effectively have watches that specialize in certain types of data structures.
* Think of the watchCollection as having two top level conditional branches, for objects and arrays. Non-collection watches should just defer to $watch implementation.

## Chapter 5 - Scope Events

* Pub/sub messaging - scope event system will mimic this pattern. It should also respect scope hierarchy (up or down). When you go up, you are 'emitting' an event. When you go down, you are 'broadcasting' an event. 
* Registering event listeners - uses the $on method. This method lives on the scope object. Listeners of $on will receive both emitted and broadcasted events. 

## Chapter 6 - Literal expressions

Much of the angular expression system can be implemented in a few lines of javascript:

```javascript
function parse(expr) {
    return function(scope) {
        with (scope) {
            return eval(expr);
        }
    }
}
```

This is problematic though, the use of both `with` and `eval` is frowned upon. This also doesn't support filters, which use the `|` character, which `eval` will interpret as a bitwise OR operator.

Because of this, we will implement our own parser, lexer and AST to handle expression compilation. In the process, we will skip some nicesseties for simplicity sake. Namely, error messages will not be detailed when parsing goes wrong, and we will only implement compiled mode of HTML Content Security Policy and not interpreted mode. 

![angularjs expression cheatsheet](expression-cheatsheet.jpg)

* We will start our implementation by only considering literal expressions, simple data expressions that represent themselves, like numbers, strings, arrays, etc.
* Our implementation will contain four objects that turn expression strings into functions: Lexer, AST Builder, AST Compiler, and a Parser.
* The Lexer takes the original expression string and returns an array of tokens parsed from that string. For example `a + b` would return tokens for `['a', '+', 'b']`.
* The AST Builder takesn the array of tokens generated by the lexer and builds up an Abstract Syntax Tree (AST) from them. The tree represents the syntactic structure of the expression as nested JS objects. For example, the tokens `['a', '+', 'b']` would result in:

```javascript
{
    type: AST.BinaryExpression,
    operator: '+',
    left: {
        type: AST.Identifier,
        name: 'a'
    },
    right: {
        type: AST.Identifier,
        name: 'b'
    }
}
```
* The AST Compiler takes the AST and compiles it into a javascript function that evaluates the expression represented in the tree. For example, the AST above would result in:

```javascript
function(scope){
    return scope.a + scope.b;
}
```
* The Parser is responsible for combining the low-level steps mentioned above. It doesn't do very much itself, but delegates heavy lifting to the other components. 

### Takeaways

* That the expression parser runs internally in three phases: Lexing, AST building, and AST compilation.
• That the end result of the parsing process is a generated JavaScript function.
• How the parser deals with integers, floating point numbers, and scientific notation.
• How the parser deals with strings.
• How the parser deals with literal booleans and null.
• How the parser deals with whitespace - by ignoring it.
• How the parser deals with arrays and objects, and how it recursively parses their contents

## Chapter 7 - Lookup and Function Call Expressions

Our expression parser can interpret literals but isn't yet very useful. Angular expressions in the view are typically used to either access data on the scope or manipulate data on the scope. In this chapter we will add those capabilities. We will also implement security measures to prevent dangerous expressions from getting through.

* Simple attribute lookup