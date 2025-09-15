---
layout: default
title: Sean Tang
---

# Operator Precedence

See code at [here](https://github.com/SeanTang001/control_code).

This project was inspired by an attempt at a [turing tarpit](https://esolangs.org/wiki/Turing_tarpit) configuration language during one of my internships. I ended up getting stuck on implementing operator precedence. I decided to revisit the problem during the summer. I corrected some of the spelling here.


## 7/24

We propose a programming language of the following form:

```
module test(int hello, bool foo) {
    if hello < 10 { # basic logic
        return true
    }
    for i in range(100) { # unroll optimization
        return false
    }
    end
}

a = hello + 2
a = hello & foo
...
```

The grammar (in rough terms) looks like this:
```ebnf
m ::= module id (t id, ... , t id) {s, .. s}
s ::= { s ... s } | id = e | if (e) s else s
e ::= p1 && p2 | p1 < p2 | p1 + p2 | p1 - p2 | p1 * p2 | p1[ p2 ] | p.length | p.id(e, ... , e) | p
p ::= c | true | false | id | !e | (e)
c ::= 1|2|3|4|5|6|7|8|9
id ::= char
```

This needs to be fully expanded out so the recursive parser can actually parse nesting

## 7/26

We want to implement arithmetics expression. We want to also implement precedence in it.

```enbf
e ::= term binop e | term
term ::= num | id | ( e )
```

### Tree Example

Ideally:
```
1 + 2 * 4-> 

   +
 /   \
1     *
   /    \    
   2     4
    
```

The computational graph has the plus sign at root, and the subsequent expression below; there is currently no precedence implemented. 

Maybe we should rewrite the binary to have a "binary-op" terminal so the result is a bit more easier to transform.

### Lexer

I want to write a tokenizer. Something like:

```ebnf
TOKEN ::= all-char/num
SEMICOLON ::= ;
LCB ::= {
RCB ::= }
LB ::= (
RB ::= )
PERIOD ::= ,
```

Ended up doing this with regex

### Operator Precedence / Associativity / Ambiguity

I plan on putting this into the grammar:

```enbf
e = e1
e1 = e2 (*|/) e2
e2 = e3 (+|-) e3
e3 = id
```

From GPT:

```ebnf
<expr> ::= <equality>

<equality> ::= <additive> <equality_tail>
<equality_tail> ::= ("==" | "!=") <additive> <equality_tail> | ε

<additive> ::= <multiplicative> <additive_tail>
<additive_tail> ::= ("+" | "-") <multiplicative> <additive_tail> | ε

<multiplicative> ::= <unary> <multiplicative_tail>
<multiplicative_tail> ::= ("*" | "/") <unary> <multiplicative_tail> | ε

<unary> ::= ("-" | "!") <unary> | <primary>

<primary> ::= "(" <expr> ")" | <number> | <id>
```

From Google:
```
expression = term, { ("+" | "-"), term } ;
term       = factor, { ("*" | "/"), factor } ;
factor     = primary, { "^", factor } ; // For right-associativity of exponentiation
primary    = number | "(", expression, ")" ;
```

Too complex to reason about. Might try an alternative that uses shunting yard on intermediate form of expression tree.

https://en.wikipedia.org/wiki/Left_recursion#Removing_left_recursion

https://langdev.stackexchange.com/questions/2648/how-can-the-shunting-yard-algorithm-be-adapted-to-handle-currying

## 7/27
Finished operator precedence / associativity. Ended up modifying the grammar and visitor. 

To illustrate the change, let's do an example, take:

```
1 * 2 + 3
```

With previous grammar of:

```
_e ::= _s _binop _st | _num
_st ::= _s | _s _binop _st | epsilon
```

This gets converted roughly into:

```
e = 1 *   |
        2 + 3  
```

The key is the fact that our grammar would group `2+3` in `_st`. This is both right associative (which is incorrect for math expressions which are left associative) and violates operator precedence.

A more explicit example is:

```
e = 1 - |
      2 - 3
```

which is `1-(2-3)=2`; the correct parse should be `(1-2)-3=-4`.

### Implementing Left Associativity

Let's take an alternative grammar, where we try to remedy first the associativity problem.

```
_e ::= _add  _addt
_addt ::= (-|+) _add _addt | _num | epsilon
```

This ends up parsing out into:

```
e = 1  |
      - 2  |
          - 3
```

Now this requires a bit of effort to convert back into
an AST form, but generally, its not hard to see an evaluation where:
```
1 + (-2) + (-3) = -4
```
as a result of the parse stricture. This nice. But we still do not resolve the issue of operator precedence. 

### Implementing Operator Precedence

Let's take this grammar snippet, where we implement layering for "+" and "*":


```
_e ::= _add
_add ::= _mul  _addt
_addt ::=  (+|-) _mul _addt | epsilon
_mul ::= _num _mult
_mult ::= (*|/) _num _mult | epsilon
```

Notice how `_mul` is "nested" within `_add`; this achieves the prioritization of `(*|/)` above `(+|-)` as whenever there is `*` present in the expression, we must result parse as:

```
_num |
    * _num
```

Let examine our starting example again:

```
1 * 2 + 3
```

Under this new grammar, it will be parsed into:

```
_e =    _add
       /      \ 
    _mul    _  addt -----
    /   \       |  \     \
    1   _mult   +  _mul   _addt
        /    \      |        \    
        *    2      _num     epsilon
                      |
                      3
```

This tree would most likely be traversed as:
```
(1*2) + 3
```

## 7/27 Part 2

Next thing to work on can be to add control statements. We already have function calls.
Next to consider is if/else, and for (no while because we don't do that)

Maybe keywords like "return" can also be added;

Specifically, the grammar had the following expansion:

```ebnf
_s ::= if ( _e ) { _sl }|_s
```

## 7/28

Discovered a bug in my LL(k) parser; it does not handle null terminals correctly (tried to add a new production to describe multiple programs).

Made a band-aid solution for now (checks if the production one layer below is nullable); need to find a more permanent solution.

Some useful context on the `if len(tokens) == 0 or len(production_rule) == 0:` magic:

```
    if len(tokens) == 0 and len(production_rule) == 0:
        return tokens, []
    if len(tokens) == 0 and len(production_rule) > 0:
        tokens = [[]]
    if len(tokens) > 0 and len(production_rule) == 0:
        return acceptor(tokens, [tokens])
```
Ended up doing a bit of rewrite; now code does the checking of null terminals on entry to make_parser, and all other calls to acceptor are just pure continuations. 

## 7/29
I notice a problem with the recursive parser: we're hitting recursion limit for longer inputs. 

Really sucks because AST generation code is highly dependent on the grammar structure atm and its kind of messy to update. 

If there was an easier way to express the grammar along with the corresponding rules to rule when parsing it would be nice. 

In the meanwhile, I'm going to consider some low-hanging fruit to achieve (maybe a visitor that converts from AST directly into the IR; I think we have the necessary amount of grammar for a basic output. Then we can continue onto doing the CS6120 content too.)

Anyways, let's consider some parser optimizations. 

### Memoizing?

Let's see if memoizing helps.

It did not; each call to the function is unique because it contains all previous tokens that have not been parsed yet; each call is a unique combo of tokens and potential grammar

### Stack?

We now try a stack based approach; seemed promising but involves a pretty big rewrite.

### Implement *?

What if we modify our grammar specification to allow for the OneOrMore-esque definition (this way we can avoild the tail grammar stuff we use) - this could involve a loop where we attempt to parse the rest of it until we hit an unparseable result, and then we attempt to finish it off with regular recursion. 

It make sense to keep the expression part of the grammar nested because the tree like structure is useful (versus the statements which are more like )

### Hybrid Approach

Implement *, and then use a simpler grammar to describe expressions. During AST generation we can parse them and not worry about hitting recursion limit. 

## 8/1

Attempted to implement *; actually quite difficult and could mess with code; Essentially, we rely on the behavior of recursive calls to 'jump' to the next correct position in the grammar's production rule. 

## 8/2

Was able to successfully implement. We use `<nonterminal>**` to represent `*`. Now we can parse as many modules as we like, and do not need to worry about recursion limit. 

Final grammar has been simplified to:

```
grammar = [
    "_prog",
    {
        "_prog":[
            ["_m**"]
        ],
        "_m": [
            ["module", "_id", "(","_pp**", "_p",")", "{", "_s**", "}"],
        ],
        "_pp": [ 
            ["_p", ","],
            E
        ],
        "_p":[
            ["_d"],
            E
        ]
        "_d": [
            ["_t", "_id"]
        ],
        "_t": [
            ["int"], ["bool"]
        ],
        "_id": [
            ["next_word"]
        ],
        "_s": [
            ["_d", "=", "_e", ";"],
            ["_id", "=", "_e", ";"],
            ["_d", ";"],
            ["_if"],
        ],
        "_if":[
            # ["if", "(", "_e", ")","{", "_sl","}", "_iftail"]
            ["if", "(", "_e", ")","{", "_s**","}"]
        ],
        # "_iftail":[
        #     ["_elif", "_iftail"],
        #     ["_else"],
        #     E
        # ],
        # "_elif":[
        #     ["elif", "(", "_e", ")","{", "_sl","}"],
        #     E
        # ],
        # "_else":[
        #     ["else", "{", "_sl", "}"],
        #     E
        # ],
        # equality: additive ((== | !=) additive)*
        "_e": [
            ["_add", "_eq_tail"]
        ],
        "_eq_tail": [
            ["==", "_add"],
            ["!=", "_add"],
            ["<=", "_add"],
            [">=", "_add"],
            ["<", "_add"],
            [">", "_add"],
            E
        ],

        # additive: multiplicative ((+ | -) multiplicative)*
        "_add": [
            ["_mul", "_add_tail"]
        ],
        "_add_tail": [
            ["+", "_mul", "_add_tail"],
            ["-", "_mul", "_add_tail"],
            E
        ],
        
        # multiplicative: unary ((* | /) unary)*
        "_mul": [
            ["_unary", "_mul_tail"]
        ],
        "_mul_tail": [
            ["*", "_unary", "_mul_tail"],
            ["/", "_unary", "_mul_tail"],
            E
        ],

        # unary: (! | -) unary | primary
        "_unary": [
            ["!", "_unary"],
            ["-", "_unary"],
            ["_primary"]
        ],

        # primary: (expr) | number | id
        "_primary": [
            ["(", "_e", ")"],
            ["_num"],
            ["_id"],
            ["_id", "(", "_pp**", "_p", ")"]
        ],

        "_num":[
            ["next_num"]
        ]
    }
]
```