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

La lista de RFCs, agrupada por área, está en [`INDEX.md`](INDEX.md).

## Valores de estado

- **Draft** — abierta para discusión, aún no decidida.
- **Accepted** — decidida; así se comporta Paco, o se comportará.
- **Rejected** — considerada y rechazada.
- **Superseded by RFC NNNN** — reemplazada; lee la RFC más nueva.

## Proponer una RFC

1. Copia [`0000-template.md`](0000-template.md) a `text/NNNN-nombre-corto.md`,
   siendo `NNNN` el próximo número libre.
2. Complétala, agrégala a [`INDEX.md`](INDEX.md) en el área que corresponda
   y abre un pull request.
3. La discusión ocurre en el PR. Una vez resuelta, un mantenedor la fusiona
   con `Status: Accepted`, o cierra el PR y registra `Status: Rejected`.

Una RFC que cambia otra ya aceptada puede editarla en el mismo pull request
(para correcciones y aclaraciones) o reemplazarla (para una decisión
distinta).

## Trabajo del día a día

Los errores, las funcionalidades ya cubiertas por una RFC aceptada y las
tareas de mantenimiento se rastrean como issues en el repositorio
correspondiente, no aquí.
