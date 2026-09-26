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

A lista de RFCs, agrupada por área, está em [`INDEX.md`](INDEX.md).

## Valores de status

- **Draft** — aberta para discussão, ainda não decidida.
- **Accepted** — decidida; é assim que Paco se comporta, ou vai se comportar.
- **Rejected** — considerada e recusada.
- **Superseded by RFC NNNN** — substituída; leia a RFC mais nova.

## Propondo uma RFC

1. Copie [`0000-template.md`](0000-template.md) para `text/NNNN-nome-curto.md`,
   `NNNN` sendo o próximo número livre.
2. Preencha, adicione-a ao [`INDEX.md`](INDEX.md) na área correspondente e
   abra um pull request.
3. A discussão acontece no PR. Quando resolvida, um mantenedor faz o merge
   com `Status: Accepted`, ou fecha o PR e registra `Status: Rejected`.

Uma RFC que muda outra já aceita pode editá-la no mesmo pull request (para
correções e esclarecimentos) ou substituí-la (para uma decisão diferente).

## Trabalho do dia a dia

Bugs, funcionalidades já cobertas por uma RFC aceita e tarefas de manutenção
são acompanhados como issues no repositório relevante, não aqui.
