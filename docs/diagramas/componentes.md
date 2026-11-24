# Diagrama de componentes

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/Diagrama_comp.pdf`](../../diagramas/Diagrama_comp.pdf), versão de 14/10/2026 |
| Estado | Atualizado em 2026-11-24 com S00–S05, ADRs 0001–0009 (M1 concluído; S06–S09 especificadas) |

Círculo ligado por linha cheia = interface fornecida; seta tracejada até o círculo = interface requerida.

```mermaid
flowchart TB
    front["Front-end SPA<br/>TypeScript + Vite + React<br/>(Que-Bug-E-Esse/front · ADR 0008)"]
    proxy["Proxy Reverso<br/>Nginx · HTTPS / WSS<br/>(M23)"]
    term["Terminal local<br/>(mesmo dispositivo · ADR 0006)"]

    subgraph back["«subsystem» Que-Bug-E-Esse/back · com.quebugesse"]
        boot["«composition root»<br/>bootstrap (Main)<br/>monta adaptadores, seed, casos de uso e ouvintes"]

        subgraph interfaces["Interfaces · depende só de application (P7)"]
            rest["rest (Ktor REST · S06)<br/>+ validação de token convidado (ADR 0007)"]
            ws["ws (Ktor WebSocket · S06)"]
            cli["cli (interfaces.cli · S05)"]
        end

        comandos(("DTOs e casos de uso<br/>VisaoInvestigador / VisaoNarrador (P3)"))

        subgraph aplicacao["Camada de Aplicação"]
            uc["Casos de Uso (12 em S03 + salas/histórico em S06)<br/>IniciarPartida · FazerPergunta · ResponderPergunta · CancelarPergunta<br/>ResponderComIA · ProporSolucao · RetirarProposta · JulgarProposta<br/>AvaliarComIA · Abandonar · ConsultarPartida · RegistrarJogador"]
            ouv["Ouvintes (application.ouvinte · S04)<br/>AoFazerPerguntaNoSolo · AoProporSolucaoNoSolo"]
            auth["Emissão / verificação de token HMAC<br/>e códigos de sala (ADR 0007 · S06)"]
        end

        subgraph dominio["Domínio · puro (P1)"]
            gs["Partida (GameSession)<br/>domain.gamesession"]
            en["Catálogo (Enigma)<br/>domain.enigma"]
            pl["Jogador (Player)<br/>domain.player"]
            pNarrador(("domain.gamesession.port<br/>Narrador (síncrona, só IA)"))
            pVerif(("domain.gamesession.port<br/>VerificadorDeSolucao (só solo)"))
            pGSR(("domain.gamesession.port<br/>GameSessionRepository<br/>+ enigmasJogadosPor"))
            pER(("domain.enigma.port<br/>EnigmaRepository<br/>GeradorDeEnigmas (S11)"))
            pPR(("domain.player.port<br/>PlayerRepository"))
            pPub(("domain.shared.port<br/>PublicadorDeEventos (H6)"))
        end

        subgraph infra["Infraestrutura · implementa as portas do domínio"]
            aiAdapt["«adapter» infrastructure.ai<br/>NarradorIaHttp · VerificadorDeSolucaoHttp (S07/S08)<br/>(fakes em M1 e na CI · P6)"]
            pers["«adapter» infrastructure.persistence<br/>memoria + seed (M1)<br/>sqlite + Flyway + WAL + outbox (S09 · ADR 0004)"]
            msg["«adapter» infrastructure.messaging<br/>PublicadorDeEventosEmMemoria (S04)<br/>+ difusão WebSocket por papel (S06)"]
        end
    end

    aiSvc["Serviço de IA stateless<br/>Python + FastAPI (Que-Bug-E-Esse/ai · ADR 0009)<br/>POST /narrar · POST /verificar · GET /health"]
    llm(["Google Gemini API<br/>gemini-2.5-flash · JSON Schema<br/>(ADR 0005)"])
    db[("SQLite em arquivo<br/>WAL + Flyway + outbox<br/>(ADR 0004)")]

    front -->|"HTTPS / WSS"| proxy
    proxy -->|"HTTP (REST)"| rest
    proxy -->|"WS"| ws
    term -->|"stdin / stdout UTF-8"| cli

    rest -.-> comandos
    ws -.-> comandos
    cli -.-> comandos
    comandos --- uc
    rest -.-> auth
    ws -.-> auth

    ouv -.-> uc
    uc -.->|"«use»"| gs
    uc -.->|"«use»"| en
    uc -.->|"«use»"| pl
    uc -.-> pNarrador
    uc -.-> pVerif
    uc -.-> pGSR
    uc -.-> pER
    uc -.-> pPR
    uc -.-> pPub

    aiAdapt --- pNarrador
    aiAdapt --- pVerif
    pers --- pGSR
    pers --- pER
    pers --- pPR
    msg --- pPub

    boot -.-> interfaces
    boot -.-> aplicacao
    boot -.-> infra

    aiAdapt <-->|"HTTP / JSON (rede interna)"| aiSvc
    aiSvc <-->|"HTTPS / JSON Schema"| llm
    pers <-->|"JDBC local (WAL)"| db
```

## Notas atualizadas (P9)

- **Sem `NarradorHumano` (ADR 0002 A, M2):** o adaptador `NarradorHumano` foi removido da camada `interfaces`. No modo grupo, o narrador humano envia comandos diretamente para `ResponderPergunta` e `JulgarProposta`. A porta `Narrador` é síncrona e usada apenas pela IA no modo solo (`ResponderComIA`).
- **`VerificadorDeSolucao` só no solo (ADR 0001 A):** removido "(fornecedor a decidir)". No modo solo, `AvaliarComIA` chama `VerificadorDeSolucao` via `VerificadorDeSolucaoHttp`; no grupo, o narrador humano julga via `JulgarProposta`.
- **Sigilo da solução por papel (P3, M1):** substituída a nota antiga "solucaoCompleta não sai do domínio". A aplicação monta `VisaoInvestigador` (sem solução até estado terminal) e `VisaoNarrador` (com solução apenas para o narrador autenticado da partida).
- **Três repositórios (ADR 0009 A, M23):** `Que-Bug-E-Esse/front` acessa exclusivamente `Que-Bug-E-Esse/back` (via proxy HTTPS/WSS em produção ou direto ao Ktor em desenvolvimento), nunca o serviço `Que-Bug-E-Esse/ai` nem o banco.
- **Raiz de composição (M16, S05 D4):** o módulo `bootstrap` é o único que conhece todas as camadas e conecta adaptadores às portas.
- **Portas e eventos (S02–S04):** portas agrupadas em `domain.<agregado>.port` e `domain.shared.port` (`PublicadorDeEventos`); eventos saem apenas depois de `salvar` (H6).

Legenda: ▭ = componente · ○ ligado por linha cheia = interface fornecida · seta tracejada até ○ = interface requerida · tracejada «use» = dependência · seta cheia = protocolo de acesso.
