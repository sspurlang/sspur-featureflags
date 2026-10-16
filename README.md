# featureflags

Feature flags for SSPUR services: boolean and multivariate flags (string, integer and number variants), explicit allow and deny lists, targeting rules over context attributes, reusable segments, prerequisites, and stable percentage rollouts and weighted splits. Evaluation is a pure function of a snapshot and a context and returns OpenFeature-style details: the value, the variant, the reason (`STATIC`, `DEFAULT`, `TARGETING_MATCH`, `SPLIT`, `DISABLED`, `PREREQUISITE_FAILED`, `ERROR`) and an error code (`FLAG_NOT_FOUND`, `TYPE_MISMATCH`, `TARGETING_KEY_MISSING`, `GENERAL`, ...). Definitions are a JSON format that is validated with errors that name the exact path, and they can be loaded from a file or kept in SQLite with versioned, optimistic-concurrency updates and an audit history.

## Install

```console
$ sspur add featureflags
```

## Example

`flags.json`:

```json
{"flags": [
  {"key": "new-checkout",
   "variants": [{"name": "on", "boolean": true}, {"name": "off", "boolean": false}],
   "off_variant": "off",
   "targets": [{"variant": "off", "keys": ["user-13"]}],
   "rules": [{"id": "staff", "when": {"segment": "staff"}, "serve": {"variant": "on"}}],
   "fallthrough": {"split": [{"variant": "on", "weight": 25}, {"variant": "off", "weight": 75}]}},
  {"key": "theme",
   "variants": [{"name": "light", "string": "light"}, {"name": "dark", "string": "dark"}],
   "off_variant": "light",
   "rules": [{"id": "night-owls", "when": {"attr": "hour", "op": "ge", "values": ["20"]}, "serve": {"variant": "dark"}}],
   "fallthrough": {"variant": "light"}}
 ],
 "segments": [{"key": "staff", "rules": [{"attr": "email", "op": "ends_with", "values": ["@example.com"]}]}]}
```

`main.ssp`:

```sspur
use featureflags.{context, with_attr, evaluate, is_on, str_value, reason_name}

fn main() -> Unit ! fs, db, log, time, fail[DbErr], fail[featureflags.StoreErr]
= match featureflags.load_file("flags.json")
  | err(e) => log("bad flags: {e}")
  | ok(s) => do
    ada = context("user-1").with_attr("email", "ada@example.com")
    d = evaluate(s, "new-checkout", ada)
    log("ada: {d.value}, {reason_name(d.reason)}, rule {d.rule}")
    log("user-13: {is_on(s, "new-checkout", context("user-13"), true)}")
    on = (0..10000).filter(i => is_on(s, "new-checkout", context("user-{i}"), false)).len
    log("rollout: {on} of 10000 users")
    log("theme at 21h: {str_value(s, "theme", context("u").with_attr("hour", "21"), "light")}")
    log("missing flag: {reason_name(evaluate(s, "nope", ada).reason)}")
    c = db.open(":memory:")
    featureflags.open_store(c)
    v = featureflags.import_snapshot(c, s, "deploy", now())
    f = featureflags.put_flag(c, featureflags.bool_flag("kill-switch", false), some(0), "ada", now())
    log("store version {featureflags.store_version(c)}, kill-switch v{f.version}")
    log("history: {featureflags.history(c, "flag", "kill-switch").map(e => "{e.action} by {e.actor}")}")
```

```console
$ sspur run main.ssp
ada: some(BoolV{v: true}), TARGETING_MATCH, rule some("staff")
user-13: false
rollout: 2521 of 10000 users
theme at 21h: dark
missing flag: ERROR
store version 4, kill-switch v1
history: ["put by ada"]
```

## How a flag is evaluated

1. A disabled flag serves its `off_variant` (`DISABLED`).
2. Each prerequisite flag must evaluate to one of its listed variants, or the flag serves `off_variant` (`PREREQUISITE_FAILED`). Prerequisite cycles are a validation error, and an `ERROR` if a snapshot with one is evaluated anyway.
3. Explicit targets: a context whose targeting key is listed gets that variant (`TARGETING_MATCH`, rule `target:VARIANT`). Use them as allow and deny lists.
4. Rules in order: the first whose condition holds serves its variant (`TARGETING_MATCH`) or its split (`SPLIT`).
5. Otherwise the fallthrough: `STATIC` when the flag has no targets or rules, else `DEFAULT`, or `SPLIT` for a split.

Typed accessors (`is_on`, `str_value`, `int_value`, `num_value` and their `*_details` forms) return the caller's default on any error, with the error in the details, so a missing or mistyped flag never breaks a request.

## Bucketing

A split hashes `flag key + "." + salt + "." + bucketing key` with SHA-256 and maps the first four bytes to a bucket from 0 to 99999; the weights (any non-negative integers, such as percentages or parts of 100000) are laid out in order over the buckets. So a context always gets the same variant in every process and in both tiers, buckets are uniform (the tests check 20000 keys in ten bands, and a 10% and 30% rollout of 5000 keys), and raising a rollout's percentage keeps everyone who already had the new variant. Changing the flag's `salt` reshuffles it; `bucket_by` buckets by another attribute (an organization id, say), and a context without that attribute gets `TARGETING_KEY_MISSING`.

## Conditions

`AttrClause{attr, op, values}` compares a context attribute (`key` is the targeting key unless the attributes define `key`). Operators: `equals`, `not_equals` (one value), `in`, `not_in`, `contains`, `not_contains`, `starts_with`, `ends_with`, `matches` (any of the values; `matches` takes regular expressions, which the stdlib runs in linear time, so a pattern can't hang a server), `lt`, `le`, `gt`, `ge` (numbers), `before`, `after` (ISO 8601 times), `exists`, `not_exists`. A missing attribute fails every operator except `not_exists`, negated ones included. Combine with `all`, `any` and `negate`; `{"segment": "staff"}` tests segment membership (excluded keys never, included keys always, else any of its rules).

## API

### Contexts and values
- `Context{key, attrs}`, `context(key)`, `with_attr(c, name, value)`, `attr(c, name)`.
- `FlagValue` (`BoolV`, `StrV`, `IntV`, `NumV`), `type_name(v)`.

### Definitions
- `Flag`, `Segment`, `Rule`, `Target`, `Prereq`, `Serve` (`Fixed{variant}`, `Weighted{weights, bucket_by}`), `Clause` (`AttrClause`, `SegmentClause`, `AllClauses`, `AnyClause`, `NotClause`, `AlwaysClause`), `Op`.
- Builders: `bool_flag(key, enabled)`, `string_flag(key, enabled, variants, default, off)`, `add_rule`, `add_target`, `add_prereq`, `with_fallthrough`, `rule(id, attr, op, values, serve)`, `rollout(on, off, pct)`.
- `Snapshot{flags, segments, version}`, `snapshot(flags, segments)`: an immutable value that request handlers can share.
- `problems(s)`: every validation problem with its path; empty when valid.

### Evaluation
- `evaluate(s, key, ctx)` gives `Details{key, value, variant, reason, error, message, rule, flag_version}`. `evaluate_all(s, ctx)`: every flag, for bootstrapping a client.
- `is_on`, `str_value`, `int_value`, `num_value`; `bool_details`, `str_details`, `int_details`, `num_details` give `Resolved[T]{value, details}`.
- `Reason`, `reason_name`, `ErrorCode`, `error_name` (OpenFeature spellings).
- `clause_holds(s, c, ctx)`, `in_segment(s, seg, ctx)`, `bucket(flag, salt, key)`, `split_variant(weights, bucket)`.

### JSON format
- `decode(text)` (validated snapshot, or an error such as `flags[1].rules[0].when.op: unknown operator 'eqals'`), `encode(s)`, `load_file(path)`, `fingerprint(s)` (SHA-256 of the definitions, to tell whether anything changed).
- `decode_flag`, `encode_flag`, `decode_segment`, `encode_segment`: one definition.
- Wire types: `DefinitionsJson`, `FlagJson`, `VariantJson` (a name and exactly one of `boolean`, `string`, `integer`, `number`), `ClauseJson` (exactly one of `attr`/`op`/`values`, `segment`, `all`, `any`, `negate`, `always`), `ServeJson` (`variant`, or `split` with optional `bucket_by`), `WeightJson`, `RuleJson`, `SegmentJson`. `op_name`, `parse_op`.

### SQLite store
- `open_store(c)`: create the tables (`featureflags_flags`, `featureflags_segments`, `featureflags_audit`, `featureflags_meta`).
- `put_flag(c, f, expected, actor, at)`, `delete_flag`, `put_segment`, `delete_segment`: each a transaction that keeps the whole store valid (a flag can't name a missing segment, and a flag or segment still in use can't be deleted), with optimistic concurrency (`expected` is the version the caller read; a concurrent change is `Conflict`) and an audit entry.
- `import_snapshot(c, s, actor, at)`: replace everything, auditing each change.
- `load_store(c)`, `store_version(c)` (one row; compare with `Snapshot.version`), `reload_if_changed(c, have)`.
- `evaluate_stored(c, key, ctx)`: evaluate from the store, reading only the flag, its prerequisites and their segments; cheap enough for every request of a handler that shares the `Db`.
- `history(c, kind, key)`, `changes_since(c, version)`: `AuditEntry{id, kind, key, action, definition, version, actor, at}`.
- `StoreErr`: `Invalid{msg}`, `Conflict{key, expected, actual}`, `NotFound{key}`; raised from an `http.serve` handler they answer 400, 409 and 404.

## Out of scope

JSON-object variants (store JSON text in a string variant), semantic-version operators, streaming updates to remote clients, experiment analytics and exposure events (log `Details` yourself), and a hosted admin UI. Handlers can't keep a mutable cache between requests, so a service either shares one immutable `Snapshot` (and restarts, or polls `reload_if_changed` from a loop outside the handlers) or calls `evaluate_stored` per request.

## License

MIT
