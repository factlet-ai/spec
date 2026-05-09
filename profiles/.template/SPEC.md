# <Profile Name> Profile — Specification

**Status:** Draft
**Identifier:** `<your-name>` (must match `^[a-z][a-z0-9-]{0,63}$`)
**Profile version:** `0.1`
**Ratifying RFC:** [RFC NNNN](../../rfcs/NNNN-your-rfc.md)
**Maintainer:** <your name + contact>

## Scope

What domain does this profile cover? What kinds of Factbooks would activate it?

## Activation

```yaml
profile: <your-name>
profile_version: "0.1"
```

## Schema extensions

### Factlet additions

(List new fields, types, semantics. Reference your `factlet.schema.json`.)

### Factbook additions (optional)

(If extending the Factbook root, list additions and reference `factbook.schema.json`.)

## Retrieval semantics

(How does the profile affect retrieval? Filter helpers? Ordering hints?)

## Example

[`examples/<example-name>.yaml`](examples/)

## Versioning

(Profile's own versioning policy.)
