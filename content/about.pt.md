---
title: "man the_compiler"
date: 2025-10-27
description: "A entidade que julga seu código antes dele rodar."
layout: "single"
---

## NAME

**the_compiler** - A entidade que julga seu código antes dele rodar.

## SYNOPSIS

```
the_compiler [OPTIONS] <arquivo_fonte.go>
```

## DESCRIPTION

Cada nil pointer esperando para crashar às 3 da manhã, cada type mismatch escondido à vista de todos, cada erro esquecido que você se recusou a tratar—eu vejo todos antes do primeiro byte tocar o silício. Em Go, não existem surpresas em runtime. Apenas verdades em tempo de compilação.

Eu leio cada linha que você escreve. Eu lembro de cada erro que você comete. E diferente dos seus testes, eu nunca minto.

## OPTIONS

```
--strict       Sem warnings, apenas erros. Como deveria ser.
--verbose      Assista-me julgar cada linha do seu código.
--no-mercy     Trata todos os warnings como erros.
--help         Você vai precisar.
```

## ENVIRONMENT

```
GOOS=linux      # O único SO de verdade
GOARCH=amd64    # Não aceite substitutos
CGO_ENABLED=0   # Go puro ou vá embora
```

## EXIT STATUS

- **0** - Seu código é aceitável. Por enquanto.
- **1** - Build falhou. Reveja suas escolhas de vida.
- **2** - Erro de sintaxe. Você sequer tentou?

## SEE ALSO

`go build(1)`, `go vet(1)`, `staticcheck(1)`, `sua-sindrome-de-impostor(7)`

## BUGS

Nenhum. Apenas funcionalidades que você ainda não entende.

## AUTHOR

Escrito por The Compiler. Mantido por pura raiva e cafeína.

## COPYRIGHT

Licença MIT. Porque até a perfeição deveria ser gratuita.
