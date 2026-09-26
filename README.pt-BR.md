# RFCs do Paco

**Leia em:** [English](README.md) · **Português** · [Español](README.es.md)

Decisões de design da [linguagem de programação Paco](https://github.com/pacolang/paco),
registradas como RFCs — um arquivo por decisão, em [`text/`](text/). As RFCs
são escritas em inglês.

## O que entra numa RFC

Uma RFC descreve comportamento que quem usa Paco consegue observar:

- **A linguagem** — sintaxe, tipos, semântica, diagnósticos.
- **A biblioteca padrão** — o que `std` e o prelude contêm, e as regras do
  que pode entrar ali.
- **O toolchain** — o comportamento dos comandos `paco` e do gerenciamento
  de pacotes do qual os usuários dependem.

Uma RFC *não* é o lugar para detalhes internos do compilador
(representações intermediárias, geradores de código, caches), roadmaps,
prioridades ou planejamento do projeto. Essas são escolhas de implementação
e ficam junto do código, no repositório a que se referem.

## Índice

**Memória e comportamento**

| RFC | Título |
|---|---|
| [0001](text/0001-ownership-and-borrowing.md) | Ownership and Borrowing |
| [0002](text/0002-mutability.md) | Mutability |
| [0003](text/0003-methods.md) | Methods |
| [0004](text/0004-traits.md) | Traits |
| [0005](text/0005-error-handling.md) | Error Handling |

**Tipos e valores**

| RFC | Título |
|---|---|
| [0006](text/0006-integer-overflow.md) | Integer Overflow |
| [0007](text/0007-floating-point-types.md) | Floating-Point Types |
| [0008](text/0008-constants.md) | Constants |
| [0009](text/0009-arrays-and-slices.md) | Arrays and Slices |
| [0010](text/0010-strings.md) | Strings |
| [0011](text/0011-collections.md) | Collections |

**Estrutura de programa**

| RFC | Título |
|---|---|
| [0012](text/0012-modules-and-visibility.md) | Modules and Visibility |
| [0013](text/0013-prelude.md) | The Prelude |
| [0014](text/0014-packages.md) | Packages |
| [0015](text/0015-standard-library-scope.md) | Standard Library Scope |
| [0016](text/0016-comptime.md) | Compile-Time Execution |

**Concorrência e código externo**

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

## Valores de status

- **Draft** — aberta para discussão, ainda não decidida.
- **Accepted** — decidida; é assim que Paco se comporta, ou vai se comportar.
- **Rejected** — considerada e recusada.
- **Superseded by RFC NNNN** — substituída; leia a RFC mais nova.

## Propondo uma RFC

1. Copie [`0000-template.md`](0000-template.md) para `text/NNNN-nome-curto.md`,
   `NNNN` sendo o próximo número livre.
2. Preencha e abra um pull request.
3. A discussão acontece no PR. Quando resolvida, um mantenedor faz o merge
   com `Status: Accepted`, ou fecha o PR e registra `Status: Rejected`.

Uma RFC que muda outra já aceita pode editá-la no mesmo pull request (para
correções e esclarecimentos) ou substituí-la (para uma decisão diferente).

## Trabalho do dia a dia

Bugs, funcionalidades já cobertas por uma RFC aceita e tarefas de manutenção
são acompanhados como issues no repositório relevante, não aqui.
