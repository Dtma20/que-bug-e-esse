# Que Bug é Esse?

Jogo de enigmas de pensamento lateral sobre engenharia de software. Os investigadores fazem perguntas de sim ou não, o narrador responde Sim, Não ou Irrelevante, e a partida termina quando alguém propõe a solução certa. No modo grupo o narrador é uma pessoa; no modo solo, uma IA.

Kotlin na JVM, DDD com portas e adaptadores, pacote raiz `com.quebugesse`. O projeto segue Spec-Driven Development: cada funcionalidade nasce como uma spec versionada (spec, plano e tarefas) e só depois vira código, com TDD a partir dos critérios de aceitação.

## Status

Fase de especificação, em 26/10/2026. A constituição e as specs S00 a S05 estão em `Rascunho`, à espera de revisão da equipe, e as oito ADRs estão em `Proposta`. O que falta decidir ou configurar está em [docs/pendencias.md](docs/pendencias.md).

O primeiro marco é o M1: jogar uma partida inteira no terminal, sem rede nem IA (S00 a S05).

## Mapa dos documentos

| Documento | Para que serve |
|---|---|
| [planejamento-sdd.md](planejamento-sdd.md) | Ponto de partida: o que os diagramas decidem, backlog S00 a S12, ADRs e as melhorias M1 a M25 |
| [docs/constitution.md](docs/constitution.md) | Princípios P1 a P10, fluxo de trabalho e definição de pronto. Prevalece sobre specs e planos |
| [docs/adr/](docs/adr/README.md) | Decisões da equipe: índice, status e como decidir |
| [docs/pendencias.md](docs/pendencias.md) | Decisões, aprovações, questões abertas e tarefas manuais |
| [docs/diagramas/](docs/diagramas/README.md) | Fontes Mermaid dos oito diagramas UML e as mudanças à espera de cada um |
| [diagramas/](diagramas/) | PDFs originais da equipe, de 14/10/2026 |
| [docs/enigmas/seed.md](docs/enigmas/seed.md) | Rascunho dos cinco enigmas e do jogador IA que o jogo grava ao subir |
| [docs/modelos/](docs/modelos/) | Modelos de [spec](docs/modelos/spec.md) e de [ADR](docs/modelos/adr.md) |
| [AGENTS.md](AGENTS.md) | Instruções para agentes de código que trabalham no repositório |

### Specs

| Spec | Assunto | Documentos |
|---|---|---|
| S00 | Fundação: módulos Gradle, CI, teste de arquitetura e cobertura | [spec](specs/000-fundacao/spec.md) · [plano](specs/000-fundacao/plan.md) · [tarefas](specs/000-fundacao/tasks.md) |
| S01 | Domínio da partida (`GameSession` e máquina de estados) | [spec](specs/001-dominio-partida/spec.md) · [plano](specs/001-dominio-partida/plan.md) · [tarefas](specs/001-dominio-partida/tasks.md) |
| S02 | Catálogo de enigmas, jogador e seed | [spec](specs/002-catalogo-e-jogador/spec.md) · [plano](specs/002-catalogo-e-jogador/plan.md) · [tarefas](specs/002-catalogo-e-jogador/tasks.md) |
| S03 | Casos de uso da partida | [spec](specs/003-casos-de-uso-da-partida/spec.md) |
| S04 | Eventos depois de persistir | [spec](specs/004-eventos-apos-persistir/spec.md) |
| S05 | CLI local | [spec](specs/005-cli-local/spec.md) |

S06 a S12 estão no backlog (seção 4 do [planejamento](planejamento-sdd.md)).

## Como a equipe decide

Toda decisão que uma spec precise assumir vira uma ADR em [docs/adr/](docs/adr/README.md) (P8 da constituição). O caminho:

1. Ler as opções e a recomendação da ADR.
2. Decidir na reunião ou numa issue aberta com o modelo "Decisão da equipe".
3. Na ADR, preencher "Decisão da equipe", trocar o status para `Aceita` ou `Rejeitada` e anotar data e participantes.
4. Atualizar o índice das ADRs e [docs/pendencias.md](docs/pendencias.md).
5. Seguir a seção "Impacto" da ADR, ou deixar essa propagação para quem pegar a spec afetada.

Os passos 3 e 4 cabem num PR só, com título `ADR-NNNN: <decisão em poucas palavras>`. Enquanto uma ADR está em `Proposta`, as specs seguem a decisão provisória que registram.

Uma spec passa por `Rascunho`, `Aprovada`, `Em implementação` e `Concluída`. O trabalho vai de `spec.md` para `plan.md` e `tasks.md`, e daí para PRs pequenos, cada um listando os CAs que cobre (P5).

## Ambiente

- JDK Temurin 25 LTS, apontado por `JAVA_HOME`.
- Gradle 9.8.2, só para gerar o wrapper na tarefa 1 da S00. Depois disso, o build usa `./gradlew` (no PowerShell, `.\gradlew.bat`).

## Como rodar

Ainda não há código.

## Contribuir

- Uma branch por tarefa ou spec; PR para a `main` usando o modelo do repositório.
- Mensagem de commit curta, no imperativo, citando a spec: `S01: aplica veredito correto`.
- PR que muda modelo, estados, portas ou pacotes atualiza o diagrama em [docs/diagramas/](docs/diagramas/README.md) no mesmo PR (P9).
