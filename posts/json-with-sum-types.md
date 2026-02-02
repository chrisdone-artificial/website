---
date: 2026-02-02
title: Proposal to extend JSON with sum types (variants)
---

A small syntactical addition that I propose to add to JSON is variants. 
It looks like this:

```lua
#"succeeded"
#"failed"("Problem flerbing the fobnicator.")
#"loading"({ "remaining": 3, "completed": 7 })
[ #"foo", #"bar"(1), #"boz"("Ok!") ]
```

I trust that the syntax is intuitive and follows norms that a hashtag is a way of labelling something.

The grammar is simply:

```bnf
root ::= string | number | ... | object | variant
variant ::= "#" string variant-arg?
variant-arg ::= "(" root ")"
```

It would be more convenient without the quotes, 
but JSON is primarily a transfer format and secondarily a human readable format.
The latter is what JSON5 is for. Additionally, it's faster to crunch through
JSON's explicitly delimited string literals than more elaborate rules.

I think this syntax gives a good balance of intention and practicality.
Sum types exist nicely in XML and static languages (Rust, Haskell),
and this is always covered by convention in JSON. It seems about
the right time to make this concept first-class.

