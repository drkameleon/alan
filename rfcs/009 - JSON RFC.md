# 009 - JSON RFC

## Current Status

### Proposed

2020-09-22

### Accepted

YYYY-MM-DD

#### Approvers

- Luis de Pombo <luis@alantechnologies.com>

### Implementation

- [ ] Implemented: [One or more PRs](https://github.com/alantech/alan/some-pr-link-here) YYYY-MM-DD
- [ ] Revoked/Superceded by: [RFC ###](./000 - RFC Template.md) YYYY-MM-DD

## Author

- David Ellis <david@alantechnologies.com>

## Summary

With functioning (though bare) HTTP client and server implementations, Alan is ready to communicate with outside systems. But it needs to be able to "speak their language." JSON is both the most ubiquitous interchange format and in a positive coincidence, one of the simpler interchange formats to parse and emit. Tackling this will help make Alan more immediately useful and also expose limitations that exist in the language that should be addressed (and are likely blockers for writing an Alan compiler in Alan, as well).

## Expected SemBDD Impact

If Alan was at 1.0.0 or greater, this would be a minor update to the language.

## Proposal

JSON is a schemaless interchange format with a simple implicit type system. It is slightly simplified from Javascript's syntax and inherits its strengths and weaknesses.

The only valid types are strings, "numbers", boolean true/false, null, arrays and objects, where arrays can be heterogeneous collections of any of these and objects are key-value pairs where the key is always a string and the value can be any of these types. Numbers are essentially always 64-bit floating point numbers but can be "integer-like" and treated like 32-bit integers.

This makes JSON neither a subset nor a superset of what is currently expressable in Alan. The Number type is clearly a subset versus Alan, but Alan has no support for mixed typing within Arrays and HashMaps, making JSON a superset there.

```json
{ "json": [ 0.0, "fucks given" ], "absolutely: true }
```

Beyond coming up with an internal representation of the JSON object, we also need serialization of that representation to a JSON string and we need a parsing library to use to implement JSON parsing with. These are both complicated because the "natural" representation of these algorithms is recursive. The `recurse` mechanism in the [Sequential Algorithms RFC](./007 - Sequential Algorithms RFC.md) partially resolves this (directly recursive functions are clear, but recursive-decent graphs of recursive functions are not resolved by that syntax), which means this is awkward to do in Alan.

Solving this, however, opens up not just JSON parsing, but parsing Alan in Alan, which will make self-hosting possible.

Therefore, breaking this proposal into three sections, the JSON representation in Alan code, the serialization approach from that representation back to a JSON string, and the parser approach to turn a JSON string into a valid representation in Alan.

### Representation

The natural representation of the JSON type is a recursive type, something like this:

```
type JSON = string | float64 | bool | void | Array<JSON> | HashMap<string, JSON>
```

which, if Alan had support added for recursive types, could be written something like:

```ln
type JSON = Either<string, Either<float64, Either<bool, Either<void, Either<Array<JSON>, HashMap<string, JSON>>>>>>
```

This would be ridiculously awkward to work with directly, though it could be covered up with wrapper functions, such as:

```ln
fn isString(j: JSON): bool = j.isMain()
fn isVoid(j: JSON): bool {
  if j.isAlt() {
    const alt1 = j.getAlt()
    if alt1.isAlt() {
      const alt2 = alt1.getAlt()
      if alt2.isAlt() {
        const alt3 = alt2.getAlt()
        if alt3.isMain() {
          return true
        }
      }
    }
  }
  return false
}
```

But I hope you can see from even this trivial example that the performance wouldn't be amazing. The recursive nature of the type system could also require usage of `@std/seq` to do anything non-trivial with it, as well. So this is not the approach we'll take.

Instead `JSON` will be a non-recursive type where the contents are actually a JSON string, and all operations involving it are simply applying the parser and serializer to it. This doesn't mean the API involving it needs to be complicated or even that different from working with regular objects, it just acknowledges a truth about JSON: being untyped, *any* operation on it can fail, but you can also apply *almost any* operation to it and expect it to potentially succeed, as well.

Since there's already [a good pattern for fallible APIs even for basic arithmetic](./004 - Runtime Error Elimination RFC.md) the functions on the `JSON` type can work the same way. For instance, support for implicitly converting the JSON to an int64 and adding it to another int64 is simply (after RFC 004 is implemented):

```ln
fn toInt64(j: JSON): Result<int64> = toInt64(j.jsonstr)

fn add(a: JSON, b: int64): Result<int64> = add(a.toInt64(), b)
```

And now if you do `print(myJsonBlob + 1)` it would either print a number or the error message that it could not parse an integer from the JSON. You could also write more explicit checking and execution code like:

```ln
if myJsonBlob.isInt64() {
  print(myJsonBlob + 1)
} else {
  print("Custom error message")
}
```

Accessing array or hashmap values would be done through a `get` method, where if provided an int64 it assumes it's a JSON array and if provided a string that it's a JSON object.


### Serialization

There are generally two kinds of situations where you want to serialize out JSON: constructing a JSON payload for an API and providing a JSON representation of your own data structures. The former works decently well just using the "Representation" APIs, manipulating the JSON type by first creating a new JSON array or object (say `newJsonObject()`) and then `set` some values on it.

But for the latter, you'd want to be able to call `variable.toJson()` and have it "just work." There may be issues with serializing things like `Array<Array<int64>>` because the interface types would likely cause the compiler to generate a recursive call stack that another part of the compiler would block to prevent potential infinite looping.

Defining a recursive function to handle this is likely the only way, but the implementation of `recurse` does not support interface types (as the actual type to be operated on could vary from recursive call to recursive call, the opposite problem of directly-recursive function calls the compiler blocks for "normal" functions.

A more detailed description of the proposed changes, what they will solve, and why it should be done. Diagrams and code examples very welcome!

Reviewers should *not* bring up alternatives in this portion unless the template is not being followed by the RFC author (which the RFC author should note in the PR with a detailed reason why). Reviewers should also not let personal distaste for a solution be the driving factor behind criticism of a proposal, there should be some rationale behind a criticism, though you can still voice your distaste since that means there's probably *something* there that perhaps another reviewer could spot (but distate on its own should not block).

Most importantly, be civil on both proposals and reviews. `alan` is meant to be an approachable language for developers and if we want to make it better we need to be approachable to each other. Some parts of the language may have been mistakes, but they certainly weren't intentional and all parts were thought over by prior contributors. New proposals come from people who see something that doesn't sit well with them and they have put forth the energy to write a proposal and we should be thankful that they care and want to make it better.

Ideally everyone can come to a refined version of the RFC that satisfies all arguments and is better than what anyone person could have come up with, but if an RFC is divisive, the "winning" side should be gracious, and the "losing" side should hopefully accept that the proposal was contentious and that there are multiple programming languages for a reason. 

### Alternatives Considered

After proposing the solution, any and all alternatives should be listed along with reasons why they are rejected. 

Authors should *not* reject alternatives just because they don't "like" them, there should be a more solid reason

Reviewers should *not* complain about a lack of detail in the alternative descriptions especially if that is their own preferred solution -- they should attempt to positively describe the solution and bring their own arguments and proof for it.

## Affected Components

A brief listing of what part(s) of the language ecosystem will be impacted should be written here.

## Expected Timeline

An RFC proposal should define the set of work that needs to be done, in what order, and with an expected level of effort and turnaround time necessary. *No* multi-stage work proposal should leave the language in a non-functioning state.

