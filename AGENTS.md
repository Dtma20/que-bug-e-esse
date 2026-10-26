# Instruções para agentes

Vale para qualquer agente de código (Claude Code, Codex, opencode) que trabalhe neste repositório.

## Projeto

- Jogo "Que Bug é Esse?": Kotlin/JVM, DDD com portas e adaptadores, pacote raiz `com.quebugesse`.
- Fluxo Spec-Driven Development: princípios em [docs/constitution.md](docs/constitution.md), backlog em [planejamento-sdd.md](planejamento-sdd.md), cada spec em `specs/NNN-nome/` (`spec.md`, `plan.md`, `tasks.md`).
- A constituição prevalece sobre specs e planos. Mudar a constituição exige ADR em `docs/adr/`.
- Mensagem de commit curta, citando a spec: `S01: aplica veredito correto`.

## Mapa dos documentos

| Caminho | Conteúdo |
|---|---|
| [README.md](README.md) | Visão geral, status e como a equipe decide |
| [planejamento-sdd.md](planejamento-sdd.md) | Backlog S00 a S12, ADRs propostas e melhorias M1 a M25 |
| [docs/constitution.md](docs/constitution.md) | Princípios P1 a P10, fluxo e definição de pronto |
| [docs/adr/](docs/adr/README.md) | ADRs 0001 a 0008 e o fluxo de decisão |
| [docs/pendencias.md](docs/pendencias.md) | O que espera a equipe: ADRs, aprovações, questões Q das specs, tarefas manuais |
| [docs/diagramas/](docs/diagramas/README.md) | Fontes Mermaid (`.md` com bloco `mermaid`) dos oito diagramas e as mudanças pendentes de cada um |
| [diagramas/](diagramas/) | PDFs originais de 14/10/2026, só referência |
| [docs/enigmas/seed.md](docs/enigmas/seed.md) | Enigmas e jogador IA do seed (S02) |
| [docs/modelos/](docs/modelos/) | Modelos de spec e de ADR |
| `specs/000-fundacao/` a `specs/005-cli-local/` | Specs do marco M1; S00 a S02 já têm `plan.md` e `tasks.md` |

Ao fechar uma questão Q, uma ADR ou uma tarefa manual, atualizar também `docs/pendencias.md`. Ao mudar um diagrama, atualizar a tabela "Pendências por diagrama" de `docs/diagramas/README.md`.

## Ambiente

- JDK Temurin 25 LTS (`JAVA_HOME` do sistema) e Gradle 9.8.2 em `%USERPROFILE%\tools\gradle-9.8.2\bin`, só para gerar o wrapper. O build usa `./gradlew` (no PowerShell, `.\gradlew.bat`).

## Lições aprendidas

- PowerShell: `(Invoke-WebRequest <url>.sha256).Content` vem como `byte[]` quando o servidor não manda tipo de texto, e a comparação de checksum falhou com "System.Byte[]" (2026-10-26). Ler com `[Text.Encoding]::ASCII.GetString(...)` ou `Invoke-RestMethod`, e imprimir os dois hashes antes de comparar.
