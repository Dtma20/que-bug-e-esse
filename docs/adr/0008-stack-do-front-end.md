# ADR 0008: Stack do front-end

| Campo | Valor |
|---|---|
| Status | Aceita |
| Proposta em | 26/10/2026 |
| Decidida em | 2026-11-24 |
| Participantes | Diogo (decisão simplificada) |
| Substitui / substituída por | — |

## Contexto

O diagrama de implantação tem um artefato "front-end" carregado no navegador, falando HTTPS e WSS com o servidor. Nenhum diagrama diz como ele é feito. Só a S10 depende disso; os marcos M1 e M2 não.

## Opções

### A. SPA em TypeScript (React, Vue ou Svelte)

- A favor: muito material e componentes prontos; WebSocket no navegador é direto.
- Contra: segunda base de código e segundo build; contrato de mensagens a manter dos dois lados.

### B. Páginas geradas no servidor com htmx

- A favor: uma base de código só, em Kotlin; menos JavaScript.
- Contra: menos comum; interação em tempo real exige extensão de WebSocket do htmx.

### C. Kotlin Multiplatform (Compose para web)

- A favor: Kotlin nos dois lados, tipos compartilhados.
- Contra: tecnologia mais nova, menos material.

## Recomendação

Decidir no começo da S10, pelo que a equipe conhece. Até lá, a S06 documenta as mensagens REST e WebSocket de forma que qualquer opção as consuma.

## Decisão da equipe

**Opção A aceita (enxuta):** SPA mínima em **TypeScript + Vite + React** (Node 24 / pnpm), usando `fetch` e `WebSocket` nativos do navegador e CSS simples, sem meta-framework (sem SSR) nem gerenciador de estado pesado, hospedada no repositório dedicado `Que-Bug-E-Esse/front` (ADR 0009). É a opção mais simples compatível com a separação de repositórios da organização e com a toolchain já instalada.

## Impacto

| Documento | Qualquer opção |
|---|---|
| S10 | Estrutura do projeto, build e testes de interface |
| [Implantação](../diagramas/implantacao.md) | Nomear o artefato front-end |
| [S00](../../specs/000-fundacao/spec.md) | CI ganha o build do front-end (só A e C) |
