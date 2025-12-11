---
title: "Los Bytes Que Estás Desperdiciando: Alineamiento de Memoria y Padding en Go"
date: 2025-12-11
description: "Tus structs están gordas. Déjame mostrarte por qué, y cómo fieldalignment puede ponerlas a dieta."
layout: "single"
tags: ["go", "performance", "memory", "optimization"]
---

## La Escena del Crimen

Crees que estás siendo eficiente. Defines una struct, usas los tipos más pequeños posibles, te sientes bien contigo mismo. Luego ejecutas `unsafe.Sizeof()` y la realidad te da una bofetada:

```go
type BadUser struct {
    IsActive bool    // 1 byte... ¿verdad?
    ID       int64   // 8 bytes
    Age      uint8   // 1 byte
    Balance  float64 // 8 bytes
    IsVerified bool  // 1 byte
}

// Esperas: 1 + 8 + 1 + 8 + 1 = 19 bytes
// Realidad: 40 bytes
// Tu cara:  😱
```

¿De dónde salieron esos **21 bytes extra**? Padding. Y es enteramente tu culpa.

---

## Por Qué Existe el Alineamiento: La Perspectiva de la CPU

Yo no hago las reglas. La CPU las hace. Y la CPU tiene preferencias.

Los procesadores modernos acceden a la memoria en **chunks del tamaño de una word**—típicamente 8 bytes en sistemas de 64 bits. Cuando los datos no están alineados a su límite natural, pasan cosas malas:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY ACCESS PATTERNS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ALIGNED (1 lectura):       UNALIGNED (2 lecturas + shift):     │
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
│  Rápido. Eficiente.          Lento. Desperdicio.                │
│  Como debe ser.              Como TÚ lo escribes.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Algunas arquitecturas crashearán con acceso no alineado. Otras lo harán funcionar silenciosamente, pero con un costo 2-10x mayor. De cualquier manera, me niego a generar código que sufra este destino.

---

## Reglas de Alineamiento de Go

Cada tipo tiene un **requisito de alineamiento**—la dirección donde debe comenzar debe ser divisible por este valor:

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

## El Problema del Padding, Visualizado

Diseccionemos tu crimen de antes:

```go
type BadUser struct {
    IsActive bool    // offset 0, size 1
    // 7 bytes padding para alinear ID
    ID       int64   // offset 8, size 8
    Age      uint8   // offset 16, size 1
    // 7 bytes padding para alinear Balance
    Balance  float64 // offset 24, size 8
    IsVerified bool  // offset 32, size 1
    // 7 bytes padding para alineamiento de struct
}
// Total: 40 bytes
```

Layout de memoria:

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

Leyenda: A = IsActive, V = IsVerified, . = byte de padding
```

**21 bytes de puro desperdicio.** Eso es más padding que datos reales.

---

## La Solución: Ordenamiento de Campos

Reordena por alineamiento (el más grande primero):

```go
type GoodUser struct {
    ID       int64   // offset 0, size 8
    Balance  float64 // offset 8, size 8
    IsActive bool    // offset 16, size 1
    Age      uint8   // offset 17, size 1
    IsVerified bool  // offset 18, size 1
    // 5 bytes padding para alineamiento de struct
}
// Total: 24 bytes
```

Layout de memoria:

```
Offset:   0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
        ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
        │           ID (int64)          │      Balance (float64)        │
        └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
Offset:  16  17  18  19  20  21  22  23
        ┌───┬───┬───┬───┬───┬───┬───┬───┐
        │ A │Age│ V │ . │ . │ . │ . │ . │
        └───┴───┴───┴───┴───┴───┴───┴───┘

Ahorro: 16 bytes por struct (reducción del 40%)
```

---

## Presentando: fieldalignment

Reordenar campos manualmente es tedioso y propenso a errores. Apruebo la automatización.

El analyzer `fieldalignment` de `golang.org/x/tools` hace esto por ti:

```bash
# Instalar
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest

# Analizar (mostrar problemas)
fieldalignment ./...

# Corregir automáticamente
fieldalignment -fix ./...
```

Ejemplo de output:

```
user.go:5:6: struct of size 40 could be 24
```

La herramienta reordena tus campos por requisitos de alineamiento decrecientes, minimizando el padding.

---

## Cuándo NO Optimizar

Antes de ejecutar `fieldalignment -fix` en todo tu codebase, considera:

### 1. Localidad de Cache Line

A veces quieres campos relacionados juntos, aunque signifique más padding:

```go
type CacheFriendly struct {
    // Campos de hot path - accedidos juntos
    Count    int64
    IsActive bool
    
    // Campos de cold path
    CreatedAt time.Time
    Metadata  map[string]string
}
```

### 2. Struct Tags y Orden de JSON

El orden de los campos afecta el output JSON con `omitempty`:

```go
type APIResponse struct {
    ID     int    `json:"id"`
    Status string `json:"status"`
    Error  string `json:"error,omitempty"` // Probablemente quieras esto al final
}
```

### 3. Legibilidad

Una struct con 50 campos ordenados por tamaño es ilegible. **Los comentarios existen. Úsalos.**

---

## Pruébalo Tú Mismo

A veces hasta yo dudo de mí mismo. Luego recuerdo: soy solo una persona—una máscara usada por un humano que está prestando su cerebro a un LLM para que pueda escribir más rápido de lo que sus dedos jamás podrían.

Sí, este texto *parece* generado por IA. Porque lo es. Pero el conocimiento? Ese vino de años mirando errores de compilador, leyendo documentos de spec a las 2 AM, y debugeando problemas de memoria que no deberían existir. El LLM pule la prosa; las cicatrices son auténticas.

Así que no creas en mi palabra. Ejecuta el código. Los bytes no mienten, aunque el texto suene sospechosamente fluido:

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

Ahí está. 40 vs 24. Los números no mienten. *Tú* mientes—cada vez que te dices "probablemente está bien."

---

## El Veredicto

```go
// Tu código antes de leer este post:
// - Desperdicia memoria
// - Más lento de lo necesario
// - Vergonzoso

// Tu código después:
// - Probablemente todavía tiene bugs
// - Pero al menos las structs están ajustadas
```

Ejecuta `fieldalignment` en tu codebase. Mira cuánta memoria estabas desperdiciando. Siente la cantidad apropiada de vergüenza. Luego corrígelo.

Estaré observando.

---

## Lectura Adicional

- [The Go Blog: Memory Layout](https://go.dev/doc/gc-guide)
- [fieldalignment source](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/fieldalignment)
- [unsafe.Sizeof, Alignof, Offsetof](https://pkg.go.dev/unsafe)
