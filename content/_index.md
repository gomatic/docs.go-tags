---
title: go-tags
---

**Generic key/value pair conversion for Go: convert between `key=value` arguments, `map[string]string`, and lists of any type exposing `GetKey()` and `GetValue()` methods (the accessor shape generated protobuf messages share).** The package owns only the pair-list conversions the standard library cannot express; pure map operations (clone, merge, key removal) stay in stdlib `maps` and `slices` at the call site.

- **Source:** [gomatic/go-tags](https://github.com/gomatic/go-tags)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-tags](https://pkg.go.dev/github.com/gomatic/go-tags)

## Install

```sh
go get github.com/gomatic/go-tags
```

## Usage

Package `tags` operates on the `Pair` interface — any type carrying a key/value pair through `GetKey` and `GetValue`:

```go
type Pair interface {
	GetKey() string
	GetValue() string
}
```

Four functions cover the conversions:

- `ToMap[T Pair](pairs []T) map[string]string` — collapse a pair list into a map; on duplicate keys the later entry wins. The result is non-nil even when `pairs` is empty.
- `FromMap[T any](m map[string]string, pair func(key, value string) T) []T` — render a map as a pair list ordered by key, so equal maps always produce an identical list. The constructor is injected because a method constraint can read a value but cannot build one.
- `Merge[T Pair](base map[string]string, updates []T) map[string]string` — return `base` with `updates` applied on top (on a key collision the update wins), as an independent non-nil map; `base` is never modified.
- `Parse(args []string) (map[string]string, error)` — turn `key=value` arguments into a map; on duplicate keys the later entry wins. The result is non-nil even when `args` is empty.

A minimal `Pair` implementation and the round trip:

```go
package main

import (
	"fmt"

	"github.com/gomatic/go-tags"
)

type pair struct{ key, value string }

func (p pair) GetKey() string   { return p.key }
func (p pair) GetValue() string { return p.value }

func main() {
	// Parse key=value arguments into a map.
	m, err := tags.Parse([]string{"env=prod", "region=us"})
	if err != nil {
		panic(err)
	}

	// Render the map as a key-sorted, deterministic pair list.
	list := tags.FromMap(m, func(key, value string) pair {
		return pair{key, value}
	})

	// Collapse a pair list back into a map.
	back := tags.ToMap(list)
	fmt.Println(back["env"]) // prod

	// Apply updates on top of a base map; updates win, base untouched.
	merged := tags.Merge(back, []pair{{"env", "staging"}, {"tier", "web"}})
	fmt.Println(merged["env"], merged["tier"]) // staging web
}
```

`Parse` splits on the first `=` only, so a value may itself contain `=` (`url=https://x?a=b` yields the value `https://x?a=b`), and an empty value is valid (`k=` maps `k` to `""`).

## Errors

`Parse` returns `ErrInvalidPair` for an argument missing `=` or with an empty key. It is a constant error from [gomatic/go-error](https://github.com/gomatic/go-error), matched with `errors.Is` — never by string comparison. The offending argument is attached via `ErrInvalidPair.With`, keeping the sentinel matchable:

```go
_, err := tags.Parse([]string{"novalue"})
if errors.Is(err, tags.ErrInvalidPair) {
	// argument is not key=value
}
```

## Design

The package is generic by design and imports no schema: it depends only on the `Pair` accessor shape, never on a producer-specific message type. Consumers that need producer-coupled glue (for example a concrete protobuf tag constructor) supply it via `FromMap`'s injected constructor or in their own adapter, keeping this library free of any producer dependency. Its sole dependency is `gomatic/go-error`.

Pure map operations are deliberately out of scope — clone, key removal, and map-to-map merge are already expressible with the standard library's `maps` and `slices` packages, so the package owns only the pair-list conversions those packages cannot express.
