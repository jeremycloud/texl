# Mozjexl [![CircleCI](https://img.shields.io/circleci/project/github/mozilla/mozjexl.svg)](https://github.com/mozilla/mozjexl)

Javascript Expression Language: Powerful context-based expression
parser and evaluator Mozjexl is a fork of Jexl for use at Mozilla,
specifically as a part of SHIELD and Normandy.

## Quick start
Use it with promises or callbacks:

```javascript
var context = {
    name: {first: 'Sterling', last: 'Archer'},
    assoc: [
        {first: 'Lana', last: 'Kane'},
        {first: 'Cyril', last: 'Figgis'},
        {first: 'Pam', last: 'Poovey'}
    ],
    age: 36
};

// Filter an array
mozjexl.eval('assoc[.first == "Lana"].last', context).then(function(res) {
    console.log(res); // Output: Kane
});

// Do math
mozjexl.eval('age * (3 - 1)', context, function(err, res) {
    console.log(res); // Output: 72
});

// Concatenate
mozjexl.eval('name.first + " " + name["la" + "st"]', context).then(function(res) {
    console.log(res); // Output: Sterling Archer
});

// Compound
mozjexl.eval('assoc[.last == "Figgis"].first == "Cyril" && assoc[.last == "Poovey"].first == "Pam"', context)
    .then(function(res) {
        console.log(res); // Output: true
    });

// Use array indexes
mozjexl.eval('assoc[1]', context, function(err, res) {
    console.log(res.first + ' ' + res.last); // Output: Cyril Figgis
});

// Use conditional logic
mozjexl.eval('age > 62 ? "retired" : "working"', context).then(function(res) {
    console.log(res); // Output: working
});

// Transform
mozjexl.addTransform('upper', function(val) {
    return val.toUpperCase();
});
mozjexl.eval('"duchess"|upper + " " + name.last|upper', context).then(function(res) {
    console.log(res); // Output: DUCHESS ARCHER
});

// Transform asynchronously, with arguments
mozjexl.addTransform('getStat', function(val, stat) {
    return dbSelectByLastName(val, stat); // Returns a promise
});
mozjexl.eval('name.last|getStat("weight")', context, function(err, res) {
    if (err) console.log('Database Error', err.stack);
    else console.log(res); // Output: 184
});

// Add your own (a)synchronous operators
// Here's a case-insensitive string equality
mozjexl.addBinaryOp('_=', 20, function(left, right) {
    return left.toLowerCase() === right.toLowerCase();
});
mozjexl.eval('"Guest" _= "gUeSt"').then(function(val) {
    console.log(res); // Output: true
});
```

## Installation

For Node.js or Web projects, type this in your project folder:

    yarn add mozjexl

Access Mozjexl the same way, backend or front:

    import mozjexl from 'mozjexl';

## All the details
### Unary Operators

| Operation | Symbol |
|-----------|:------:|
| Negate    |    !   |

### Binary Operators

| Operation        |      Symbol      |
|------------------|:----------------:|
| Add, Concat      |         +        |
| Subtract         |         -        |
| Multiply         |         *        |
| Divide           |         /        |
| Divide and floor |        //        |
| Modulus          |         %        |
| Power of         |         ^        |
| Logical AND      |        &&        |
| Logical OR       |   &#124;&#124;   |

### Comparisons

| Comparison                 | Symbol |
|----------------------------|:------:|
| Equal                      |   ==   |
| Not equal                  |   !=   |
| Greater than               |    >   |
| Greater than or equal      |   >=   |
| Less than                  |    <   |
| Less than or equal         |   <=   |
| Element in array or string |   in   |

#### A note about `in`:
The `in` operator can be used to check for a substring:
`"Cad" in "Ron Cadillac"`, and it can be used to check for an array element:
`"coarse" in ['fine', 'medium', 'coarse']`.  However, the `==` operator is used
behind-the-scenes to search arrays, so it should not be used with arrays of
objects.  The following expression returns false: `{a: 'b'} in [{a: 'b'}]`.

### Ternary operator

Conditional expressions check to see if the first segment evaluates to a truthy
value. If so, the consequent segment is evaluated.  Otherwise, the alternate
is. If the consequent section is missing, the test result itself will be used
instead.

| Expression                        | Result |
|-----------------------------------|--------|
| "" ? "Full" : "Empty"             | Empty  |
| "foo" in "foobar" ? "Yes" : "No"  | Yes    |
| {agent: "Archer"}.agent ?: "Kane" | Archer |

### Native Types

| Type     |            Examples            |
|----------|:------------------------------:|
| Booleans |         `true`, `false`        |
| Strings  | "Hello \"user\"", 'Hey there!' |
| Numerics |      6, -7.2, 5, -3.14159      |
| Objects  |        {hello: "world!"}       |
| Arrays   |       ['hello', 'world!']      |

### Groups

Parentheses work just how you'd expect them to:

| Expression                          | Result |
|-------------------------------------|:-------|
| (83 + 1) / 2                        | 42     |
| 1 < 3 && (4 > 2 &#124;&#124; 2 > 4) | true   |

### Identifiers

Access variables in the context object by just typing their name. Objects can
be traversed with dot notation, or by using brackets to traverse to a dynamic
property name.

Example context:

```javascript
{
    name: {
        first: "Malory",
        last: "Archer"
    },
    exes: [
        "Nikolai Jakov",
        "Len Trexler",
        "Burt Reynolds"
    ],
    lastEx: 2
}
```

| Expression        | Result        |
|-------------------|---------------|
| name.first        | Malory        |
| name['la' + 'st'] | Archer        |
| exes[2]           | Burt Reynolds |
| exes[lastEx - 1]  | Len Trexler   |

### Collections

Collections, or arrays of objects, can be filtered by including a filter
expression in brackets. Properties of each collection can be referenced by
prefixing them with a leading dot. The result will be an array of the objects
for which the filter expression resulted in a truthy value.

Example context:
```javascript
{
    employees: [
        {first: 'Sterling', last: 'Archer', age: 36},
        {first: 'Malory', last: 'Archer', age: 75},
        {first: 'Lana', last: 'Kane', age: 33},
        {first: 'Cyril', last: 'Figgis', age: 45},
        {first: 'Cheryl', last: 'Tunt', age: 28}
    ],
    retireAge: 62
}
```

| Expression                                    | Result                                                                                |
|-----------------------------------------------|---------------------------------------------------------------------------------------|
| employees[.first == 'Sterling']               | [{first: 'Sterling', last: 'Archer', age: 36}]                                        |
| employees[.last == 'Tu' + 'nt'].first         | Cheryl                                                                                |
| employees[.age >= 30 && .age < 40]            | [{first: 'Sterling', last: 'Archer', age: 36},{first: 'Lana', last: 'Kane', age: 33}] |
| employees[.age >= 30 && .age < 40][.age < 35] | [{first: 'Lana', last: 'Kane', age: 33}]                                              |
| employees[.age >= retireAge].first            | Malory                                                                                |

### Transforms

The power of Mozjexl is in transforming data, synchronously or asynchronously.
Transform functions take one or more arguments: The value to be transformed,
followed by anything else passed to it in the expression. They must return
either the transformed value, or a Promise that resolves with the transformed
value. Add them with `mozjexl.addTransform(name, function)`.

```javascript
mozjexl.addTransform('split', function(val, char) {
    return val.split(char);
});
mozjexl.addTransform('lower', function(val) {
    return val.toLowerCase();
});
```

| Expression                                 | Result                |
|--------------------------------------------|-----------------------|
| "Pam Poovey"&#124;lower&#124;split(' ')[1] | poovey                |
| "password==guest"&#124;split('=' + '=')    | ['password', 'guest'] |

#### Advanced Transforms
Using Transforms, Mozjexl can support additional string formats like embedded
JSON, YAML, XML, and more.  The following, with the help of the
[xml2json](https://github.com/buglabs/node-xml2json) module, allows XML to be
traversed just as easily as plain javascript objects:

```javascript
var xml2json = require('xml2json');

mozjexl.addTransform('xml', function(val) {
    return xml2json.toJson(val, {object: true});
});

var context = {
    xmlDoc:
        "<Employees>" +
            "<Employee>" +
                "<FirstName>Cheryl</FirstName>" +
                "<LastName>Tunt</LastName>" +
            "</Employee>" +
            "<Employee>" +
                "<FirstName>Cyril</FirstName>" +
                "<LastName>Figgis</LastName>" +
            "</Employee>" +
        "</Employees>"
};

var expr = 'xmlDoc|xml.Employees.Employee[.LastName == "Figgis"].FirstName';

mozjexl.eval(expr, context).then(function(res) {
    console.log(res); // Output: Cyril
});
```

### Context

Variable contexts are straightforward Javascript objects that can be accessed
in the expression, but they have a hidden feature: they can include a Promise
object, and when that property is used, Mozjexl will wait for the Promise to
resolve and use that value!

### API

#### mozjexl.Jexl
A reference to the Jexl constructor. To maintain separate instances of Jexl
with each maintaining its own set of transforms, simply re-instantiate with
`new mozjexl.Jexl()`.

#### mozjexl.addBinaryOp(_{string} operator_, _{number} precedence_, _{function} fn_)
Adds a binary operator to the Jexl instance. A binary operator is one that
considers the values on both its left and right, such as "+" or "==", in order
to calculate a result. The precedence determines the operator's position in the
order of operations (please refer to `lib/grammar.js` to see the precedence of
existing operators). The provided function will be called with two arguments:
a left value and a right value. It should return either the resulting value,
or a Promise that resolves to the resulting value.

#### mozjexl.addUnaryOp(_{string} operator_, _{function} fn_)
Adds a unary operator to the Jexl instance. A unary operator is one that
considers only the value on its right, such as "!", in order to calculate a
result. The provided function will be called with one argument: the value to
the operator's right. It should return either the resulting value, or a Promise
that resolves to the resulting value.

#### mozjexl.addTransform(_{string} name_, _{function} transform_)
Adds a transform function to this Jexl instance.  See the **Transforms**
section above for information on the structure of a transform function.

#### mozjexl.addTransforms(_{{}} map_)
Adds multiple transforms from a supplied map of transform name to transform
function.

#### mozjexl.getTransform(_{string} name_)
**Returns `{function|undefined}`.** Gets a previously set transform function,
or `undefined` if no function of that name exists.

#### mozjexl.eval(_{string} expression_, _{{}} [context]_, _{function} [callback]_)
**Returns `{Promise<*>}`.** Evaluates an expression.  The context map and
callback function are optional. If a callback is specified, it will be called
with the standard signature of `{Error}` first argument, and the expression's
result in the second argument.  Note that if a callback function is supplied,
the returned Promise will already have a `.catch()` attached to it.

#### mozjexl.removeOp(_{string} operator_)
Removes a binary or unary operator from the Jexl instance. For example, "^" can
be passed to eliminate the "power of" operator.

## Hacking

```shell
$ yarn install
$ yarn test
```

### Precommit hook

Mozjexl provides a config for
[Therapist](http://therapist.readthedocs.io/en/latest/overview.html). Install
it with Pip, and then run `therapist install` in this repo to set it
up. It will automatically format your Javascript with Prettier, and
run ESLint checks before committing your code.

## Building a JSM for use in mozilla-central ##
```shell
$ yarn build
```

The result will be in `vendor/mozjexl.jsm`.  This is for use in `mozilla-central/toolkit`, so it is not
minified as it will be compressed in the omnijar file.

## License
Mozjexl is licensed under the MIT license. Please see `LICENSE.txt` for full
details.

## Credits
Jexl was designed and created at [TechnologyAdvice](http://technologyadvice.com).
