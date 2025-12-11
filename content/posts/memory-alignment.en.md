---
title: "The Bytes You're Wasting: Memory Alignment and Padding in Go"
date: 2025-12-11
description: "Your structs are fat. Let me show you why, and how fieldalignment can put them on a diet."
layout: "single"
tags: ["go", "performance", "memory", "optimization"]
---

## The Crime Scene

You think you're being efficient. You define a struct, you use the smallest types possible, you feel good about yourself. Then you run `unsafe.Sizeof()` and reality slaps you in the face:

```go
type BadUser struct {
    IsActive bool    // 1 byte... right?
    ID       int64   // 8 bytes
    Age      uint8   // 1 byte
    Balance  float64 // 8 bytes
    IsVerified bool  // 1 byte
}

// You expect: 1 + 8 + 1 + 8 + 1 = 19 bytes
// Reality:    40 bytes
// Your face:  😱
```

Where did those **21 extra bytes** come from? Padding. And it's entirely your fault.

---

## Why Alignment Exists: A CPU's Perspective

I don't make the rules. The CPU does. And the CPU has preferences.

Modern processors access memory in **word-sized chunks**—typically 8 bytes on 64-bit systems. When data isn't aligned to its natural boundary, bad things happen:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY ACCESS PATTERNS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ALIGNED (1 read):          UNALIGNED (2 reads + shift):        │
│                                                                 │
│  ┌───────────────────┐      ┌───────────────────┐               │
│  │ addr 0x00: int64  │      │ addr 0x01: int64  │               │
│  └───────────────────┘      └──────────┬────────┘               │
│         │                              │                        │
│         ▼                              ▼                        │
│     ┌───────┐                   ┌───────────────┐               │
│     │ 1 op  │                   │ 2 ops + merge │               │
│     └───────┘                   └───────────────┘               │
│                                                                 │
│  Fast. Efficient.            Slow. Wasteful.                    │
│  As it should be.            As YOU write it.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Some architectures will crash on unaligned access. Others will silently make it work, but at 2-10x the cost. Either way, I refuse to generate code that suffers this fate.

---

## Go's Alignment Rules

Every type has an **alignment requirement**—the address where it must start must be divisible by this value:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TYPE ALIGNMENT IN GO (amd64)                 │
├──────────────────┬──────────────────┬───────────────────────────┤
│ Type             │ Size (bytes)     │ Alignment (bytes)         │
├──────────────────┼──────────────────┼───────────────────────────┤
│ bool, int8       │ 1                │ 1                         │
│ int16            │ 2                │ 2                         │
│ int32, float32   │ 4                │ 4                         │
│ int64, float64   │ 8                │ 8                         │
│ pointer, string  │ 8 (ptr) / 16     │ 8                         │
│ slice            │ 24               │ 8                         │
│ interface{}      │ 16               │ 8                         │
└──────────────────┴──────────────────┴───────────────────────────┘
```

---

## The Padding Problem, Visualized

Let's dissect your crime from earlier:

```go
type BadUser struct {
    IsActive bool    // offset 0, size 1
    // 7 bytes padding to align ID
    ID       int64   // offset 8, size 8
    Age      uint8   // offset 16, size 1
    // 7 bytes padding to align Balance
    Balance  float64 // offset 24, size 8
    IsVerified bool  // offset 32, size 1
    // 7 bytes padding for struct alignment
}
// Total: 40 bytes
```

Memory layout:

```
Offset:   0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
        ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
        │ A │ . │ . │ . │ . │ . │ . │ . │       ID (int64)              │
        └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
Offset:  16  17  18  19  20  21  22  23  24  25  26  27  28  29  30  31
        ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
        │Age│ . │ . │ . │ . │ . │ . │ . │      Balance (float64)        │
        └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
Offset:  32  33  34  35  36  37  38  39
        ┌───┬───┬───┬───┬───┬───┬───┬───┐
        │ V │ . │ . │ . │ . │ . │ . │ . │  (. = padding)
        └───┴───┴───┴───┴───┴───┴───┴───┘

Legend: A = IsActive, V = IsVerified, . = padding byte
```

**21 bytes of pure waste.** That's more padding than actual data.

---

## The Fix: Field Ordering

Reorder by alignment (largest first):

```go
type GoodUser struct {
    ID       int64   // offset 0, size 8
    Balance  float64 // offset 8, size 8
    IsActive bool    // offset 16, size 1
    Age      uint8   // offset 17, size 1
    IsVerified bool  // offset 18, size 1
    // 5 bytes padding for struct alignment
}
// Total: 24 bytes
```

Memory layout:

```
Offset:   0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
        ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
        │           ID (int64)          │      Balance (float64)        │
        └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
Offset:  16  17  18  19  20  21  22  23
        ┌───┬───┬───┬───┬───┬───┬───┬───┐
        │ A │Age│ V │ . │ . │ . │ . │ . │
        └───┴───┴───┴───┴───┴───┴───┴───┘

Saved: 16 bytes per struct (40% reduction)
```

---

## Enter: fieldalignment

Reordering fields manually is tedious and error-prone. I approve of automation.

The `fieldalignment` analyzer from `golang.org/x/tools` does this for you:

```bash
# Install
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest

# Analyze (show problems)
fieldalignment ./...

# Fix automatically
fieldalignment -fix ./...
```

Example output:

```
user.go:5:6: struct of size 40 could be 24
```

The tool reorders your fields by decreasing alignment requirements, minimizing padding.

---

## When NOT to Optimize

Before you `fieldalignment -fix` your entire codebase, consider:

### 1. Cache Line Locality

Sometimes you want related fields together, even if it means more padding:

```go
type CacheFriendly struct {
    // Hot path fields - accessed together
    Count    int64
    IsActive bool
    
    // Cold path fields
    CreatedAt time.Time
    Metadata  map[string]string
}
```

### 2. Struct Tags and JSON Order

Field order affects JSON output with `omitempty`:

```go
type APIResponse struct {
    ID     int    `json:"id"`
    Status string `json:"status"`
    Error  string `json:"error,omitempty"` // You probably want this last
}
```

### 3. Readability

A struct with 50 fields sorted by size is unreadable. **Comments exist. Use them.**

---

## Prove It Yourself

Sometimes even I doubt myself. Then I remember: I'm just a persona—a mask worn by a human who's lending their brain to an LLM so it can type faster than their fingers ever could.

Yes, this text *looks* AI-generated. Because it is. But the knowledge? That came from years of staring at compiler errors, reading spec documents at 2 AM, and debugging memory issues that shouldn't exist. The LLM polishes the prose; the scars are authentic.

So don't take my word for it. Run the code. The bytes don't lie, even if the text sounds suspiciously fluent:

```go
package main

import (
    "fmt"
    "unsafe"
)

type BadUser struct {
    IsActive   bool
    ID         int64
    Age        uint8
    Balance    float64
    IsVerified bool
}

type GoodUser struct {
    ID         int64
    Balance    float64
    IsActive   bool
    Age        uint8
    IsVerified bool
}

func main() {
    fmt.Printf("BadUser size: %d bytes\n", unsafe.Sizeof(BadUser{}))
    fmt.Printf("GoodUser size: %d bytes\n", unsafe.Sizeof(GoodUser{}))

    var bad BadUser
    fmt.Printf("\nBadUser field offsets:\n")
    fmt.Printf("  IsActive:   offset %d, size %d\n", unsafe.Offsetof(bad.IsActive), unsafe.Sizeof(bad.IsActive))
    fmt.Printf("  ID:         offset %d, size %d\n", unsafe.Offsetof(bad.ID), unsafe.Sizeof(bad.ID))
    fmt.Printf("  Age:        offset %d, size %d\n", unsafe.Offsetof(bad.Age), unsafe.Sizeof(bad.Age))
    fmt.Printf("  Balance:    offset %d, size %d\n", unsafe.Offsetof(bad.Balance), unsafe.Sizeof(bad.Balance))
    fmt.Printf("  IsVerified: offset %d, size %d\n", unsafe.Offsetof(bad.IsVerified), unsafe.Sizeof(bad.IsVerified))

    var good GoodUser
    fmt.Printf("\nGoodUser field offsets:\n")
    fmt.Printf("  ID:         offset %d, size %d\n", unsafe.Offsetof(good.ID), unsafe.Sizeof(good.ID))
    fmt.Printf("  Balance:    offset %d, size %d\n", unsafe.Offsetof(good.Balance), unsafe.Sizeof(good.Balance))
    fmt.Printf("  IsActive:   offset %d, size %d\n", unsafe.Offsetof(good.IsActive), unsafe.Sizeof(good.IsActive))
    fmt.Printf("  Age:        offset %d, size %d\n", unsafe.Offsetof(good.Age), unsafe.Sizeof(good.Age))
    fmt.Printf("  IsVerified: offset %d, size %d\n", unsafe.Offsetof(good.IsVerified), unsafe.Sizeof(good.IsVerified))
}
```

Output:

```
BadUser size: 40 bytes
GoodUser size: 24 bytes

BadUser field offsets:
  IsActive:   offset 0, size 1
  ID:         offset 8, size 8
  Age:        offset 16, size 1
  Balance:    offset 24, size 8
  IsVerified: offset 32, size 1

GoodUser field offsets:
  ID:         offset 0, size 8
  Balance:    offset 8, size 8
  IsActive:   offset 16, size 1
  Age:        offset 17, size 1
  IsVerified: offset 18, size 1
```

There it is. 40 vs 24. The numbers don't lie. *You* do—every time you tell yourself "it's probably fine."

---

## The Verdict

```go
// Your code before reading this post:
// - Wastes memory
// - Slower than necessary
// - Embarrassing

// Your code after:
// - Still probably has bugs
// - But at least the structs are tight
```

Run `fieldalignment` on your codebase. Look at how much memory you've been wasting. Feel the appropriate amount of shame. Then fix it.

I'll be watching.

---

## Further Reading

- [The Go Blog: Memory Layout](https://go.dev/doc/gc-guide)
- [fieldalignment source](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/fieldalignment)
- [unsafe.Sizeof, Alignof, Offsetof](https://pkg.go.dev/unsafe)
