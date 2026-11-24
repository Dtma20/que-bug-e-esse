# Decisões registradas (ADRs)

Toda decisão que outra spec precise assumir (framework, banco, contrato de porta, regra de jogo ambígua) vira um arquivo nesta pasta, conforme o princípio P8 da [constituição](../constitution.md).

## Como a equipe registra uma decisão

1. Abrir o ADR na tabela abaixo e ler "Opções" e "Recomendação".
2. Decidir na reunião ou na issue da ADR (coluna "Issue" do índice), aberta com o modelo "Decisão da equipe". ADR nova ganha a sua issue com o mesmo modelo.
3. No ADR: preencher "Decisão da equipe", trocar o status para `Aceita` (ou `Rejeitada`) e preencher data e participantes.
4. Atualizar a linha do ADR aqui e em [docs/pendencias.md](../pendencias.md).
5. Seguir a seção "Impacto" do ADR, que lista cada spec, CA e diagrama a mudar. Quem não quiser editar esses arquivos pode parar no passo 4: quem pegar a spec afetada depois faz a propagação, guiado pela mesma seção.

Os passos 3 e 4 cabem num PR só, com título `ADR-NNNN: <decisão em poucas palavras>`.

## Status

`Proposta` → `Aceita` ou `Rejeitada`. Uma decisão aceita só muda por uma ADR nova, que leva o status `Aceita` e marca a antiga como `Substituída por NNNN`. ADR não se apaga.

Enquanto um ADR está em `Proposta`, as specs seguem a decisão provisória que cada uma registra (P8). A seção "Impacto" diz o que muda se a equipe escolher outra opção.

## Índice

| ADR | Pergunta | Status | Decisão | Bloqueia | Issue |
|---|---|---|---|---|---|
| [0001](0001-veredito-no-modo-grupo.md) | Quem dá o veredito no modo grupo? | Aceita | Opção A: narrador humano no grupo (`JulgarProposta`), IA só no solo (`AvaliarComIA`) | S03, S08 | [#1](https://github.com/Dtma20/que-bug-e-esse/issues/1) |
| [0002](0002-contrato-da-porta-narrador.md) | A porta `Narrador` é síncrona? | Aceita | Opção A: porta síncrona só para a IA; humano responde direto via `ResponderPergunta` | S03, S04, S07 | [#2](https://github.com/Dtma20/que-bug-e-esse/issues/2) |
| [0003](0003-framework-web.md) | Ktor ou Spring Boot? | Aceita | Opção A: Ktor (REST + WebSocket nativo, composição explícita no `bootstrap`) | S06 | [#3](https://github.com/Dtma20/que-bug-e-esse/issues/3) |
| [0004](0004-banco-de-dados.md) | Qual banco? | Aceita | Opção B: SQLite em arquivo com Flyway, JDBC, modo WAL e bloqueio otimista | S09 | [#4](https://github.com/Dtma20/que-bug-e-esse/issues/4) |
| [0005](0005-provedor-de-llm.md) | Qual provedor de LLM e com que orçamento? | Aceita | Opção A: API comercial com saída estruturada (Gemini `gemini-2.5-flash`), teto US$ 10/mês e 10.000 tokens/partida | S07, S08, S11 | [#5](https://github.com/Dtma20/que-bug-e-esse/issues/5) |
| [0006](0006-modo-offline.md) | O que é o modo "offline"? | Aceita | Opção A: um dispositivo, todos juntos, sem rede ("mesmo dispositivo") | S05, UC1 | [#6](https://github.com/Dtma20/que-bug-e-esse/issues/6) |
| [0007](0007-identidade-e-autenticacao.md) | Como identificar e autenticar jogadores? | Aceita | Opção A: convidado com apelido, código de sala e token assinado por sessão | S06, S12 | [#7](https://github.com/Dtma20/que-bug-e-esse/issues/7) |
| [0008](0008-stack-do-front-end.md) | Qual stack de front-end? | Aceita | Opção A: SPA mínima em TypeScript + Vite + React (`fetch` e `WebSocket` nativos) | S10 | [#8](https://github.com/Dtma20/que-bug-e-esse/issues/8) |
| [0009](0009-divisao-de-repositorios.md) | Como dividir a arquitetura nos três repositórios da organização? | Aceita | Opção A: `back` (núcleo Kotlin/JVM + API + SDD), `ai` (HTTP JSON stateless `/narrar` e `/verificar`), `front` (SPA mínima) | S00, S06, S07, S08, S10 | — |

Modelo para ADR nova: [docs/modelos/adr.md](../modelos/adr.md). Numeração: o próximo número livre, com quatro dígitos.
