# Diagrama de sequência 2/2: investigar e resolver

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/diagrama_seq_jogar.pdf`](../../diagramas/diagrama_seq_jogar.pdf), versão de 14/10/2026 |
| Estado | Atualizado em 2026-11-24 com S01, S03, S04 e ADRs 0001–0002 (sem `NarradorHumano`, com H6 e M3) |

```mermaid
sequenceDiagram
    actor I as Investigador
    actor NH as Narrador humano<br/>(modo GRUPO)
    participant UC as «useCase»<br/>Casos de Uso da Partida
    participant GS as «AggregateRoot»<br/>GameSession
    participant GSR as «port»<br/>GameSessionRepository
    participant PUB as «port»<br/>PublicadorDeEventos
    participant OUV as «ouvinte»<br/>Ouvintes do Solo (S04)
    participant N as «port»<br/>Narrador (só IA)
    participant V as «port»<br/>VerificadorDeSolucao (só solo)

    loop investigação (até resolver, abandonar ou cancelar/retirar · M3)
        I->>UC: 01 FazerPergunta(partida, autor, texto)
        UC->>GS: 02 fazerPergunta(autor, texto, agora)
        GS-->>UC: 03 Question (aberta) / acumula PerguntaFeita
        UC->>GSR: 04 salvar(partida)
        UC->>PUB: 05 publicar([PerguntaFeita]) (H6)

        alt modo SOLO (ADR 0002 A)
            PUB->>OUV: 06 AoFazerPerguntaNoSolo -> ResponderComIA(partida)
            OUV->>N: 07 responder(pergunta, contexto) [HTTP /narrar · S07]
            alt resposta obtida
                N-->>OUV: 08 Answer (SIM | NAO | IRRELEVANTE)
                OUV->>GS: 09 responderPergunta(IA, id, resposta, agora)
                OUV->>GSR: 10 salvar(partida)
                OUV->>PUB: 11 publicar([PerguntaRespondida])
            else falha da IA (H4)
                N-->>OUV: 12 exceção -> IA_INDISPONIVEL (nada salvo; pergunta segue aberta)
            end
        else modo GRUPO (ADR 0002 A · sem porta Narrador)
            NH->>UC: 13 ResponderPergunta(partida, narrador, id, resposta)
            UC->>GS: 14 responderPergunta(narrador, id, resposta, agora)
            UC->>GSR: 15 salvar(partida)
            UC->>PUB: 16 publicar([PerguntaRespondida])
        end

        Note over I,NH: Se pergunta travar (H4 ou narrador ausente · M3),<br/>autor ou narrador executa CancelarPergunta -> PerguntaCancelada.
    end

    I->>UC: 17 ProporSolucao(partida, autor, texto)
    UC->>GS: 18 proporSolucao(autor, texto, agora)
    GS-->>UC: 19 Tentativa (PENDENTE) / acumula SolucaoProposta
    UC->>GSR: 20 salvar(partida)
    UC->>PUB: 21 publicar([SolucaoProposta]) (H6)

    alt modo SOLO (ADR 0001 A)
        PUB->>OUV: 22 AoProporSolucaoNoSolo -> AvaliarComIA(partida)
        OUV->>V: 23 avaliar(proposta, solucaoCompleta, enunciado) [HTTP /verificar · S08]
        alt veredito obtido
            V-->>OUV: 24 Veredito(correta)
            OUV->>GS: 25 aplicarVeredito(IA, veredito, agora)
            OUV->>GSR: 26 salvar(partida)
            OUV->>PUB: 27 publicar([PartidaResolvida | SolucaoRecusada])
        else falha da IA (H4)
            V-->>OUV: 28 exceção -> IA_INDISPONIVEL (nada salvo; tentativa segue pendente)
        end
    else modo GRUPO (ADR 0001 A)
        NH->>UC: 29 JulgarProposta(partida, narrador, correta)
        UC->>GS: 30 aplicarVeredito(narrador, Veredito(correta), agora)
        UC->>GSR: 31 salvar(partida)
        UC->>PUB: 32 publicar([PartidaResolvida | SolucaoRecusada])
    end

    Note over I,NH: Se proposta travar (M3), autor ou narrador executa RetirarProposta -> PropostaRetirada.<br/>Cada caso de uso relê a partida após publicar (EV5) e devolve VisaoInvestigador ou VisaoNarrador (P3).
```

## Notas atualizadas (P9)

- **Decisão validada (ADR 0001 A e ADR 0002 A):** no modo grupo, o narrador humano chama diretamente `ResponderPergunta` e `JulgarProposta` (o adaptador `NarradorHumano` não existe). No modo solo, `PublicadorDeEventos` despacha de forma síncrona (fila FIFO por thread, S04) para `AoFazerPerguntaNoSolo` (`ResponderComIA` → porta `Narrador`) e `AoProporSolucaoNoSolo` (`AvaliarComIA` → porta `VerificadorDeSolucao`).
- **Ordem obrigatória (H6, EV5, S04 D2/D5):** todo caso de uso de comando carrega a partida, executa o comando no agregado, chama `salvar(partida)`, só então chama `puxarEventos()` e `publicar(...)`, e por fim relê a partida do repositório para montar a visão de retorno (assim, no solo, a resposta ou veredito da IA já aparece na visão devolvida).
- **Destravamento (M3, S01 D4):** `CancelarPergunta` e `RetirarProposta` liberam a partida quando a IA falha (H4) ou o narrador humano não responde.
- **Autoria do veredito (S03 D7):** `aplicarVeredito(autor, veredito, agora)` valida `autor = narrador` tanto no grupo quanto no solo (`IA`).
- **Sigilo (P3):** `VisaoInvestigador` só inclui `solucaoCompleta` após a partida atingir `RESOLVIDO` ou `ABANDONADO`.

Legenda: seta cheia = chamada síncrona · tracejada = retorno · quadro = fragmento (`loop` / `alt`).
