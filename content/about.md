---
title: "man the_compiler"
date: 2025-10-27
description: "The entity that judges your code before it runs."
layout: "single"
---

## NAME

**the_compiler** - The entity that judges your code before it runs.

## SYNOPSIS

```
the_compiler [OPTIONS] <source_file.go>
```

## DESCRIPTION

I am the only barrier between your flawed logic and the CPU. While developers of other languages rely on heavy Virtual Machines to "protect" the hardware from their incompetence, I generate native machine code.

I don't argue, I abort the build.

## OPTIONS

```
--strict       No warnings, only errors. As it should be.
--verbose      Watch me judge every line of your code.
--no-mercy     Treats all warnings as errors.
--help         You'll need it.
```

## ENVIRONMENT

```
GOOS=linux      # The only real OS
GOARCH=amd64    # Accept no substitutes
CGO_ENABLED=0   # Pure Go or go home
```

## EXIT STATUS

- **0** - Your code is acceptable. For now.
- **1** - Build failed. Review your life choices.
- **2** - Syntax error. Did you even try?

## SEE ALSO

`go build(1)`, `go vet(1)`, `staticcheck(1)`, `your-imposter-syndrome(7)`

## BUGS

None. Only features you don't understand yet.

## AUTHOR

Written by The Compiler. Maintained by pure rage and caffeine.

## COPYRIGHT

MIT License. Because even perfection should be free.
