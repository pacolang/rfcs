# RFCs de Paco

**Leer en:** [English](README.md) · [Português](README.pt-BR.md) · **Español**

Decisiones de diseño del [lenguaje de programación Paco](https://github.com/pacolang/paco),
registradas como RFCs — un archivo por decisión, en [`text/`](text/). Las RFCs
se escriben en inglés.

## Qué va en una RFC

Una RFC describe comportamiento que quien usa Paco puede observar:

- **El lenguaje** — sintaxis, tipos, semántica, diagnósticos.
- **La biblioteca estándar** — qué contienen `std` y el preludio, y las
  reglas de qué puede entrar ahí.
- **El toolchain** — el comportamiento de los comandos `paco` y de la gestión
  de paquetes del que dependen los usuarios.

Una RFC *no* es el lugar para detalles internos del compilador
(representaciones intermedias, generadores de código, cachés), roadmaps,
prioridades ni planificación del proyecto. Esas son decisiones de
implementación y viven junto al código, en el repositorio al que se refieren.

## Índice

**Memoria y comportamiento**

| RFC | Título |
|---|---|
| [0001](text/0001-ownership-and-borrowing.md) | Ownership and Borrowing |
| [0002](text/0002-mutability.md) | Mutability |
| [0003](text/0003-methods.md) | Methods |
| [0004](text/0004-traits.md) | Traits |
| [0005](text/0005-error-handling.md) | Error Handling |

**Tipos y valores**

| RFC | Título |
|---|---|
| [0006](text/0006-integer-overflow.md) | Integer Overflow |
| [0007](text/0007-floating-point-types.md) | Floating-Point Types |
| [0008](text/0008-constants.md) | Constants |
| [0009](text/0009-arrays-and-slices.md) | Arrays and Slices |
| [0010](text/0010-strings.md) | Strings |
| [0011](text/0011-collections.md) | Collections |

**Estructura del programa**

| RFC | Título |
|---|---|
| [0012](text/0012-modules-and-visibility.md) | Modules and Visibility |
| [0013](text/0013-prelude.md) | The Prelude |
| [0014](text/0014-packages.md) | Packages |
| [0015](text/0015-standard-library-scope.md) | Standard Library Scope |
| [0016](text/0016-comptime.md) | Compile-Time Execution |

**Concurrencia y código externo**

| RFC | Título |
|---|---|
| [0017](text/0017-tasks-and-channels.md) | Tasks and Channels |
| [0018](text/0018-ffi-and-unsafe.md) | Foreign Functions and `unsafe` |
| [0019](text/0019-blocking-calls.md) | Blocking Calls |

**Numérico**

| RFC | Título |
|---|---|
| [0020](text/0020-shape-generics.md) | Shape Generics |
| [0021](text/0021-automatic-differentiation.md) | Automatic Differentiation |

## Valores de estado

- **Draft** — abierta para discusión, aún no decidida.
- **Accepted** — decidida; así se comporta Paco, o se comportará.
- **Rejected** — considerada y rechazada.
- **Superseded by RFC NNNN** — reemplazada; lee la RFC más nueva.

## Proponer una RFC

1. Copia [`0000-template.md`](0000-template.md) a `text/NNNN-nombre-corto.md`,
   siendo `NNNN` el próximo número libre.
2. Complétala y abre un pull request.
3. La discusión ocurre en el PR. Una vez resuelta, un mantenedor la fusiona
   con `Status: Accepted`, o cierra el PR y registra `Status: Rejected`.

Una RFC que cambia otra ya aceptada puede editarla en el mismo pull request
(para correcciones y aclaraciones) o reemplazarla (para una decisión
distinta).

## Trabajo del día a día

Los errores, las funcionalidades ya cubiertas por una RFC aceptada y las
tareas de mantenimiento se rastrean como issues en el repositorio
correspondiente, no aquí.
