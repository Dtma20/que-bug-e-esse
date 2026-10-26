# Instruções para agentes

Vale para qualquer agente de código (Claude Code, Codex, opencode) que trabalhe neste repositório.

## Projeto

- Jogo "Que Bug é Esse?": Kotlin/JVM, DDD com portas e adaptadores, pacote raiz `com.quebugesse`.
- Fluxo Spec-Driven Development: princípios em [docs/constitution.md](docs/constitution.md), backlog em [planejamento-sdd.md](planejamento-sdd.md), cada spec em `specs/NNN-nome/` (`spec.md`, `plan.md`, `tasks.md`).
- A constituição prevalece sobre specs e planos. Mudar a constituição exige ADR em `docs/adr/`.
- Mensagem de commit curta, citando a spec: `S01: aplica veredito correto`.

## Ambiente

- JDK Temurin 25 LTS (`JAVA_HOME` do sistema) e Gradle 9.8.2 em `%USERPROFILE%\tools\gradle-9.8.2\bin`, só para gerar o wrapper. O build usa `./gradlew` (no PowerShell, `.\gradlew.bat`).

## Lições aprendidas

- PowerShell: `(Invoke-WebRequest <url>.sha256).Content` vem como `byte[]` quando o servidor não manda tipo de texto, e a comparação de checksum falhou com "System.Byte[]" (2026-10-26). Ler com `[Text.Encoding]::ASCII.GetString(...)` ou `Invoke-RestMethod`, e imprimir os dois hashes antes de comparar.
