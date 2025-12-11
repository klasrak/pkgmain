---
title: "Os Bytes Que Você Está Desperdiçando: Alinhamento de Memória e Padding em Go"
date: 2025-12-11
description: "Suas structs estão gordas. Deixa eu te mostrar por quê, e como o fieldalignment pode colocá-las de dieta."
layout: "single"
tags: ["go", "performance", "memory", "optimization"]
---

## A Cena do Crime

Você acha que está sendo eficiente. Define uma struct, usa os menores tipos possíveis, se sente bem consigo mesmo. Aí você roda `unsafe.Sizeof()` e a realidade te dá um tapa na cara:

```go
type BadUser struct {
    IsActive bool    // 1 byte... certo?
    ID       int64   // 8 bytes
    Age      uint8   // 1 byte
    Balance  float64 // 8 bytes
    IsVerified bool  // 1 byte
}

// Você espera: 1 + 8 + 1 + 8 + 1 = 19 bytes
// Realidade:   40 bytes
// Sua cara:    😱
```

De onde vieram esses **21 bytes extras**? Padding. E a culpa é inteiramente sua.

---

## Por Que Alinhamento Existe: A Perspectiva da CPU

Eu não faço as regras. A CPU faz. E a CPU tem preferências.

Processadores modernos acessam memória em **chunks do tamanho de uma word**—tipicamente 8 bytes em sistemas 64-bit. Quando os dados não estão alinhados ao seu limite natural, coisas ruins acontecem:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY ACCESS PATTERNS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ALIGNED (1 leitura):       UNALIGNED (2 leituras + shift):     │
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
│  Rápido. Eficiente.          Lento. Desperdício.                │
│  Como deveria ser.           Como VOCÊ escreve.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Algumas arquiteturas vão crashar com acesso desalinhado. Outras vão silenciosamente fazer funcionar, mas com custo 2-10x maior. De qualquer forma, eu me recuso a gerar código que sofra esse destino.

---

## Regras de Alinhamento do Go

Todo tipo tem um **requisito de alinhamento**—o endereço onde ele deve começar precisa ser divisível por esse valor:

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

## O Problema do Padding, Visualizado

Vamos dissecar seu crime de antes:

```go
type BadUser struct {
    IsActive bool    // offset 0, size 1
    // 7 bytes padding para alinhar ID
    ID       int64   // offset 8, size 8
    Age      uint8   // offset 16, size 1
    // 7 bytes padding para alinhar Balance
    Balance  float64 // offset 24, size 8
    IsVerified bool  // offset 32, size 1
    // 7 bytes padding para alinhamento da struct
}
// Total: 40 bytes
```

Layout de memória:

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

Legenda: A = IsActive, V = IsVerified, . = byte de padding
```

**21 bytes de puro desperdício.** Isso é mais padding do que dados reais.

---

## A Correção: Ordenação de Campos

Reordene por alinhamento (maior primeiro):

```go
type GoodUser struct {
    ID       int64   // offset 0, size 8
    Balance  float64 // offset 8, size 8
    IsActive bool    // offset 16, size 1
    Age      uint8   // offset 17, size 1
    IsVerified bool  // offset 18, size 1
    // 5 bytes padding para alinhamento da struct
}
// Total: 24 bytes
```

Layout de memória:

```
Offset:   0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
        ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
        │           ID (int64)          │      Balance (float64)        │
        └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
Offset:  16  17  18  19  20  21  22  23
        ┌───┬───┬───┬───┬───┬───┬───┬───┐
        │ A │Age│ V │ . │ . │ . │ . │ . │
        └───┴───┴───┴───┴───┴───┴───┴───┘

Economia: 16 bytes por struct (redução de 40%)
```

---

## Apresentando: fieldalignment

Reordenar campos manualmente é tedioso e propenso a erros. Eu aprovo automação.

O analyzer `fieldalignment` de `golang.org/x/tools` faz isso por você:

```bash
# Instalar
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest

# Analisar (mostrar problemas)
fieldalignment ./...

# Corrigir automaticamente
fieldalignment -fix ./...
```

Exemplo de output:

```
user.go:5:6: struct of size 40 could be 24
```

A ferramenta reordena seus campos por requisitos de alinhamento decrescentes, minimizando o padding.

---

## Quando NÃO Otimizar

Antes de rodar `fieldalignment -fix` em todo seu codebase, considere:

### 1. Localidade de Cache Line

Às vezes você quer campos relacionados juntos, mesmo que signifique mais padding:

```go
type CacheFriendly struct {
    // Campos de hot path - acessados juntos
    Count    int64
    IsActive bool
    
    // Campos de cold path
    CreatedAt time.Time
    Metadata  map[string]string
}
```

### 2. Struct Tags e Ordem do JSON

A ordem dos campos afeta o output JSON com `omitempty`:

```go
type APIResponse struct {
    ID     int    `json:"id"`
    Status string `json:"status"`
    Error  string `json:"error,omitempty"` // Você provavelmente quer isso por último
}
```

### 3. Legibilidade

Uma struct com 50 campos ordenados por tamanho é ilegível. **Comentários existem. Use-os.**

---

## Prove Você Mesmo

Às vezes até eu duvido de mim mesmo. Aí eu lembro: sou apenas uma persona—uma máscara usada por um humano que está emprestando o cérebro para um LLM digitar mais rápido do que seus dedos jamais conseguiriam.

Sim, este texto *parece* gerado por IA. Porque é. Mas o conhecimento? Esse veio de anos encarando erros de compilador, lendo documentações de spec às 2 da manhã, e debugando problemas de memória que não deveriam existir. O LLM polui a prosa; as cicatrizes são autênticas.

Então não acredite na minha palavra. Execute o código. Os bytes não mentem, mesmo que o texto soe suspeitosamente fluente:

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

Aí está. 40 vs 24. Os números não mentem. *Você* mente—toda vez que diz para si mesmo "provavelmente tá de boa."

---

## O Veredicto

```go
// Seu código antes de ler este post:
// - Desperdiça memória
// - Mais lento que o necessário
// - Constrangedor

// Seu código depois:
// - Ainda provavelmente tem bugs
// - Mas pelo menos as structs estão enxutas
```

Rode `fieldalignment` no seu codebase. Veja quanta memória você estava desperdiçando. Sinta a quantidade apropriada de vergonha. Depois corrija.

Estarei observando.

---

## Leitura Adicional

- [The Go Blog: Memory Layout](https://go.dev/doc/gc-guide)
- [fieldalignment source](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/fieldalignment)
- [unsafe.Sizeof, Alignof, Offsetof](https://pkg.go.dev/unsafe)
