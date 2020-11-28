---
layout: post
title: Go map literal performance
published: true
---

### Go Map Literals performance

During recent go code benchmarking I noticed that function which returns
a map literal

```go
    // some code above
    return map[string]float {
        "key1": SOME_COMPUTED_ABOVE_VALUE,
        "key2": SOME_COMPUTED_ABOVE_VALUE,
        // more keys here
        "keyN": SOME_COMPUTED_ABOVE_VALUE,
    }
```

calls [`hashGrow`](https://golang.org/src/runtime/map_faststr.go) much more then expected.
I actually expected none cause the size of the map known as a compile time.

So I changed the code above to

```go
    // some code above
    result := make(map[string]float, SIZE) // SIZE >= N
    result["key1"] = SOME_COMPUTED_ABOVE_VALUE
    result["key2"] = SOME_COMPUTED_ABOVE_VALUE
    // more keys here
    result["keyN"] = SOME_COMPUTED_ABOVE_VALUE
    return result
```

and it actually sped up my benchmark ~1/3

To confirm the issue I developed a set of [benchmarks](https://github.com/trams/goplayground/blob/main/map_make_test.go) and showed the problem


| Map size | Map literal perf | Map make perf |
|-------|--------|---------|
| 5 | 235 ns/op | 240 ns/op |
| 9 | 716 ns/op | 453 ns/op |
| 65 | 7346 ns/op | 3401 ns/op |
| 513 | 59357 ns/op | 24774 ns/op |

So one can see that for sufficiently big maps we have 2x speedup.


Here are full result. This investigation is not yet over.
For example, I do not yet know why sometimes it makes 2 allocations but in other cases 3 allocations even when using `make`

```
BenchmarkMap/0003-8              6375036               187 ns/op             256 B/op          2 allocs/op
BenchmarkMap/0005-8              4897664               240 ns/op             256 B/op          2 allocs/op
BenchmarkMap/0009-8              2684498               453 ns/op             464 B/op          2 allocs/op
BenchmarkMap/0017-8              1434456               854 ns/op             954 B/op          2 allocs/op
BenchmarkMap/0033-8               807938              1609 ns/op            1868 B/op          2 allocs/op
BenchmarkMap/0065-8               375810              3401 ns/op            4176 B/op          3 allocs/op
BenchmarkMap/0129-8               198102              6541 ns/op            8272 B/op          3 allocs/op
BenchmarkMap/0257-8               102302             12072 ns/op           14417 B/op          3 allocs/op
BenchmarkMap/0513-8                50647             24774 ns/op           28752 B/op          3 allocs/op
BenchmarkMapStringLiteral/0003-8                 6525012               188 ns/op             256 B/op          2 allocs/op
BenchmarkMapStringLiteral/0005-8                 5099382               235 ns/op             256 B/op          2 allocs/op
BenchmarkMapStringLiteral/0009-8                 1587074               716 ns/op             672 B/op          3 allocs/op
BenchmarkMapStringLiteral/0017-8                  730539              1688 ns/op            1633 B/op          4 allocs/op
BenchmarkMapStringLiteral/0033-8                  344728              3432 ns/op            3594 B/op          6 allocs/op
BenchmarkMapStringLiteral/0065-8                  177174              7346 ns/op            8020 B/op          9 allocs/op
BenchmarkMapStringLiteral/0129-8                   84253             14291 ns/op           16324 B/op         11 allocs/op
BenchmarkMapStringLiteral/0257-8                   41516             28957 ns/op           30744 B/op         12 allocs/op
BenchmarkMapStringLiteral/0513-8                   20548             59357 ns/op           61348 B/op         22 allocs/op
```