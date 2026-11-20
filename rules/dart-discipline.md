---
# Path-scoped (Claude Code): auto-loads only when a matching file is read or edited.
paths:
  - "**/*.dart"
---

# Dart / Flutter Discipline

The standing rules for Dart and Flutter code, on top of the standard Dart style rules. `/dart-audit-code`, `/dart-improve-code` and the `dart-reviewer` enforce them; this file is the single source of truth. Put project-specific rules in it.

## Rule 1: Behaviour preservation

Every change preserves observable behaviour unless the user asked for a behaviour change. A quiet behaviour change dressed as a refactor is the most expensive regression: reviewers trust the framing and skim.

- Don't "improve" algorithms, data structures or control flow that work.
- Don't add abstractions, base classes, mixins, generics or extensions the current call sites don't need.
- Don't rename public APIs or move declarations between files unless asked.
- A worthwhile refactor bigger than a local edit: **report it, don't do it**. When in doubt, leave it alone.

## Rule 2: No `late`

`late` turns a compile-time null-safety error into a runtime `LateInitializationError`. Forbidden in application code. Instead, in order:

1. `final` with an initialiser at the declaration.
2. `final`, set in the constructor initializer list (`Foo() : _x = computeX();`).
3. If the value really arrives later, nullable (`Foo? _foo`), or return it from a method instead of caching it.
4. In `initState`, a `final` field if the type allows, else nullable.

The only exception: `late final` for a value impossible to compute eagerly that is part of a documented external contract (a generated binding).

## Rule 3: Avoid unnecessary nulls

`T?` is a contract with every caller; use it only when "no value" is part of the domain.

- "Always set before use" is a constructor parameter, not nullable. A list that might be empty is `const []`. A string that might be missing is usually `''`.
- A function that can't run with `null` takes a non-nullable parameter; the caller guards.
- Return `T?` only when the caller must tell "no value" apart from `T`. Never `null` for errors: throw or return a result type.
- Handle a justified null at the **boundary** where it enters, not with `?.` and `??` scattered through the body.

## Rule 4: Single source of truth for names and keys

Any string used as a map key, JSON field, query parameter, storage key, collection or table name, route, event, flag or DB column is **declared once**: a repeated literal fails as a silently missing field. A closed set of NAMES is an enum (4a); the KEYS of one record are constants (4b).

### 4a: A closed set of names is an enum, and `.name` is the string

Collections, preference slots, events, flags, jobs, permissions, error codes, asset names: an enum whose UPPER_SNAKE member IS the string.

```dart
enum CacheCollection { ITEMS, GROUPS, ACTIVITIES_ENVELOPE }

final store = stringMapStoreFactory.store(CacheCollection.ITEMS.name); // 'ITEMS'
final parsed = CacheCollection.values.valueOf(raw) ?? (throw FormatException('Unknown collection: $raw'));
```

- Never type the name twice: not `static const ITEMS = 'items';`, not `ITEMS('items')`.
- An outsider's spelling (a file name, URL segment, vendor word) comes from ONE getter: `String get fileBase => name.toLowerCase();`. Constructor parameters carry only what the name can't (a label, a vendor number).
- Parse with one helper that ignores case and returns null on a miss (for example df_type's `X.values.valueOf(s)`); decide at the call what a miss means. Never `values.firstWhere((e) => e.name == s)` in a `try`, `byName`, or a `switch` on the raw string.
- Const maps key by the member, not its string. Changing a stored spelling is a data migration, never a drive-by rename.

### 4b: The keys of one record are constants

The field names of one JSON object, map, template or query string are constants (generated `XxxFieldNames` where a generator owns the shape):

```dart
abstract final class UserKeys {
  static const id = 'id';
  static const name = 'name';
}

final name = json[UserKeys.name] as String;
```

- The same literal as a map key in two places is a bug. Group related keys in an `abstract final class`; if its members name things rather than fields, it's 4a.
- Name the constant exactly as its string (`createdAt = 'createdAt'`). A key used in one file is a file-scope `const _kRouteName = 'home';`.

## Rule 5: Defend at boundaries, trust internals

- Validate where data enters: HTTP responses, files, user input, IPC, DB rows. Downstream code assumes well-formed data.
- No null, range or type checks or try/catch inside private helpers called only from validated code; an `assert` documenting an invariant is fine.
- Don't wrap every `await` in try/catch: catch only where there's a meaningful response (retry, fallback, user message, structured log).
- No `if (foo == null) return;` on non-nullable values. Prefer types to runtime checks: three states are an enum or sealed class, not a checked `String`.
- A check whose failure mode you can't name is comprehension cost: delete it.

## Rule 6: Short, simple, dumb

The reader is tired.

- One screen-sized function beats two clever helpers; a flat `if/else` beats a three-subclass hierarchy; three similar lines beat a parametric helper.
- Inline a function called from one place that doesn't aid naming.
- Boring constructs: `for` over `fold` with side effects, `if` over an asymmetric ternary, a named method over a multi-line closure.
- More than 3 levels of nesting: extract a guard clause or a named helper.
- Not a licence for new abstractions: simplify inline.

## Rule 7: Readability over shortcuts

- No cryptic abbreviations: `controller` not `ctrl`, `context` not `ctx`, `colorScheme` not `cs`, `description` not `desc`.
- No `_buildFoo()` helper methods: extract a `Widget` class.
- No `dynamic` outside JSON (`Map<String, dynamic>`). No `var` on fields, parameters or return types (fine on locals with an obvious right side).
- `as` casts only at a boundary (`json['x'] as String`); deeper in, a cast means a missing abstraction or a pattern match.
- No magic numbers (`const _kMaxRetries = 3;`). Doubles are `16.0`, never `16`.

## Rule 8: Consistent patterns

The same problem is solved the same way everywhere; uniformity beats picking the locally best pattern.

- One result shape for repositories, one state library, one way of extracting UI sections, one error boundary per layer, one JSON spelling matching the back end.
- Two ways of doing the same thing: flag it and let the human pick. Never rewrite either side unilaterally.

## Rule 9: Anti-pattern catalogue

Flag these, and fix where safe. The "why" says whether an edge case applies.

- **`late` field**: runtime crash instead of a compile error (Rule 2).
- **Magic-string map key**: a typo is a silent bug (Rule 4).
- **String literal as enum value** (`if (status == 'ACTIVE')`): use an enum or sealed class.
- **Name typed twice** (`static const ITEMS = 'items';`, `ITEMS('items')`): use `.name` (Rule 4a).
- **Hand-rolled enum lookup** (`firstWhere((e) => e.name == s)`, `byName`, `asNameMap()[s]`, a string `switch`): the project's one lookup helper, miss handled at the call.
- **`dynamic` in application code**: defeats the type system.
- **`!` without context**: if it's safe, why is the type nullable? **`?? throw` on a never-null value**: the type is wrong.
- **`_buildFoo()` helper**: extract a `Widget`. **Business logic in `build()`**: move it to services, repositories or state.
- **`setState` beside a state library (or the reverse)**: pick one.
- **`print` / `debugPrint`**: use the project's logger.
- **Silent try/catch**: at least log; preferably let it propagate.
- **Fire-and-forget `Future` without `unawaited()`**: state the intent.
- **Public `StreamController`**: expose the `Stream`, keep the controller private.
- **Repeated literal lists / maps as defaults**: `const` once at file scope.
- **Generic name** (`data`, `info`, `helper`, `manager`, `util`): name the thing.
- **`if (x) return true; else return false;`**: `return x;`.
- **Unnamed boolean argument** (`doThing(true)`): a named parameter or an enum.
- **`Map<String, dynamic>` past the boundary**: decode into a model at the boundary.
- **More than 4 positional constructor args**: make them named.
- **`initState` field that could be a constructor `final`**: make it `final`.
- **Re-implementing a utility a dependency already has**: check `pubspec.yaml` first.
- **`Widget? _foo` cached across rebuilds**: recompute or extract.
- **String-concatenated paths / URLs**: `path.join`, `Uri`.
- **`identical` where `==` was meant (or the reverse)**: reference vs value; almost always `==`.
