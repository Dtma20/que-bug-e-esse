# Decisões registradas (ADRs)

Toda decisão que outra spec precise assumir (framework, banco, contrato de porta, regra de jogo ambígua) vira um arquivo nesta pasta, conforme o princípio P8 da [constituição](../constitution.md).

## Como a equipe registra uma decisão

1. Abrir o ADR na tabela abaixo e ler "Opções" e "Recomendação".
2. Decidir na reunião, ou numa issue aberta com o modelo "Decisão da equipe".
3. No ADR: preencher "Decisão da equipe", trocar o status para `Aceita` (ou `Rejeitada`) e preencher data e participantes.
4. Atualizar a linha do ADR aqui e em [docs/pendencias.md](../pendencias.md).
5. Seguir a seção "Impacto" do ADR, que lista cada spec, CA e diagrama a mudar. Quem não quiser editar esses arquivos pode parar no passo 4: quem pegar a spec afetada depois faz a propagação, guiado pela mesma seção.

Os passos 3 e 4 cabem num PR só, com título `ADR-NNNN: <decisão em poucas palavras>`.

## Status

`Proposta` → `Aceita` ou `Rejeitada`. Uma decisão aceita só muda por uma ADR nova, que leva o status `Aceita` e marca a antiga como `Substituída por NNNN`. ADR não se apaga.

Enquanto um ADR está em `Proposta`, as specs seguem a decisão provisória que cada uma registra (P8). A seção "Impacto" diz o que muda se a equipe escolher outra opção.

## Índice

| ADR | Pergunta | Status | Recomendação | Bloqueia |
|---|---|---|---|---|
| [0001](0001-veredito-no-modo-grupo.md) | Quem dá o veredito no modo grupo? | Proposta | Narrador humano no grupo, IA só no solo | S03, S08 |
| [0002](0002-contrato-da-porta-narrador.md) | A porta `Narrador` é síncrona? | Proposta | Porta só para a IA; o humano responde pelo caso de uso | S03, S04, S07 |
| [0003](0003-framework-web.md) | Ktor ou Spring Boot? | Proposta | Ktor, salvo se a equipe dominar Spring | S06 |
| [0004](0004-banco-de-dados.md) | Qual banco? | Proposta | PostgreSQL com Flyway | S09 |
| [0005](0005-provedor-de-llm.md) | Qual provedor de LLM e com que orçamento? | Proposta | Escolher pela bateria de referência da S07 | S07, S08, S11 |
| [0006](0006-modo-offline.md) | O que é o modo "offline"? | Proposta | Um dispositivo, todos juntos, sem rede | S05, UC1 |
| [0007](0007-identidade-e-autenticacao.md) | Como identificar e autenticar jogadores? | Proposta | Convidado com apelido, código de sala e token por sessão | S06, S12 |
| [0008](0008-stack-do-front-end.md) | Qual stack de front-end? | Proposta | Decidir na S10 | S10 |

Modelo para ADR nova: [docs/modelos/adr.md](../modelos/adr.md). Numeração: o próximo número livre, com quatro dígitos.
