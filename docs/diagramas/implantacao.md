# Diagrama de implantação

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/Diagrama_imp.pdf`](../../diagramas/Diagrama_imp.pdf), versão de 14/10/2026 |
| Estado | Atualizado em 2026-11-24 com S00–S10, ADRs 0001–0009 e marco M4 (S11 e S12) |

```mermaid
flowchart TB
    subgraph usuario["Dispositivo do Investigador / Curador «device»<br/>(PC / smartphone)"]
        nav["Navegador Web"]
        fe["«artifact» Que-Bug-E-Esse/front<br/>SPA TypeScript + Vite + React (ADR 0008 · S10/S11)"]
        nav -->|carrega| fe
    end

    subgraph dispNarr["Dispositivo do Narrador «device»<br/>narrador humano (modo grupo online)"]
        navNarr["Navegador Web<br/>SPA Que-Bug-E-Esse/front (tela do narrador)"]
    end

    subgraph localCli["Dispositivo Local «device»<br/>modo mesmo dispositivo (ADR 0006 · S05 · sem SSH)"]
        console["Console / terminal UTF-8<br/>executa Que-Bug-E-Esse/back localmente (interfaces.cli)"]
    end

    proxy["Servidor Web / Proxy Reverso<br/>«Nginx» · HTTPS / WSS (M23)"]

    subgraph servidor["Servidor de Aplicação «device» · Que-Bug-E-Esse/back"]
        subgraph jvm["«executionEnvironment» JVM Temurin 25 LTS · Kotlin · Ktor (ADR 0003)"]
            boot["bootstrap (Main / ServidorHttp / AplicacaoCli)<br/>composição explícita"]
            subgraph interfaces["Interfaces"]
                ws["ws (Ktor WebSocket · S06)"]
                rest["rest (Ktor REST · S06/S11/S12)<br/>partidas · curadoria · perfil · histórico"]
                cli["cli (interfaces.cli · S05)"]
            end
            subgraph aplicacao["Camada de Aplicação"]
                uc["Casos de Uso + Ouvintes (S03/S04/S11/S12)<br/>AoEncerrarPartidaAtualizarEstatisticas (M17)<br/>VisaoInvestigador / VisaoNarrador (P3)<br/>Tokens HMAC e salas (ADR 0007 · S06)"]
            end
            subgraph dominio["Domínio (puro · P1)"]
                dom["GameSession · Enigma · Player<br/>domain.*.port"]
            end
            subgraph infra["Infraestrutura · implementa portas do domínio"]
                pers["persistence<br/>memoria + seed (M1)<br/>sqlite + Flyway + WAL + outbox (S09)"]
                aiHttp["ai (S07/S08/S11)<br/>NarradorIaHttp · VerificadorDeSolucaoHttp<br/>GeradorDeEnigmasHttp"]
                msg["messaging<br/>PublicadorDeEventosEmMemoria (S04)<br/>+ difusão WebSocket (S06)"]
            end
        end
        db[("SQLite em arquivo<br/>WAL + busy_timeout=5000<br/>(ADR 0004)")]
    end

    subgraph servAi["Serviço de IA «container / process» · Que-Bug-E-Esse/ai (rede interna · ADR 0009)"]
        fastapi["«executionEnvironment» Python 3.13 + FastAPI<br/>POST /narrar · POST /verificar · POST /gerar · GET /health<br/>LLM_API_KEY fora do repositório (ADR 0005)"]
    end

    llm(["Google Gemini API<br/>gemini-2.5-flash · JSON Schema<br/>serviço externo (ADR 0005)"])

    usuario <-->|"HTTPS / WSS"| proxy
    dispNarr <-->|"HTTPS / WSS"| proxy
    proxy <-->|"WS"| ws
    proxy <-->|"HTTP (REST)"| rest
    console <-->|"stdin / stdout (mesmo processo)"| cli

    ws -.-> uc
    rest -.-> uc
    cli -.-> uc
    aplicacao -.->|"«use»"| dominio
    infra -.->|"«use»"| dominio
    boot -.-> interfaces
    boot -.-> aplicacao
    boot -.-> infra

    pers <-->|"JDBC local (arquivo)"| db
    aiHttp <-->|"HTTP / JSON (rede interna)"| fastapi
    fastapi <-->|"HTTPS / JSON Schema"| llm
```

## Notas atualizadas (P9)

- **Sem SSH em produção (S05 D1, M15):** o acesso via SSH do PDF original foi removido. A CLI (`interfaces.cli`) roda localmente no mesmo processo JVM para o modo "mesmo dispositivo" (ADR 0006 A) e validação local (M1).
- **Sem `NarradorHumano` (ADR 0002 A):** o narrador humano usa o navegador (`Que-Bug-E-Esse/front`) e envia respostas/julgamentos pela API REST/WebSocket (`ResponderPergunta` e `JulgarProposta`).
- **Banco local SQLite (ADR 0004 B):** o servidor de banco separado via TCP foi substituído por SQLite em arquivo no nó da aplicação com Flyway, modo WAL (`busy_timeout=5000`), bloqueio otimista `versao` e tabela `eventos_outbox` (S09). As estatísticas atualizadas por `AoEncerrarPartidaAtualizarEstatisticas` (S12) são persistidas na tabela `players` e expostas em `GET /api/perfil`.
- **Serviço stateless `Que-Bug-E-Esse/ai` e Gemini (ADR 0005 A, ADR 0009 A, S11):** o `back` nunca fala diretamente com o provedor de LLM nem guarda `LLM_API_KEY`; o serviço `ai` expõe `/narrar`, `/verificar`, `/gerar` e `/health`, isolando prompts defensivos (M7), teto de tokens e saída estruturada do `gemini-2.5-flash`, acessível apenas na rede interna pelo `back` (`NarradorIaHttp`, `VerificadorDeSolucaoHttp` e `GeradorDeEnigmasHttp`).
- **Front-end e acesso único pelo proxy (ADR 0008 A, M23, S10/S11):** a SPA `Que-Bug-E-Esse/front` (TypeScript + Vite + React) comunica-se exclusivamente com `Que-Bug-E-Esse/back` (via proxy HTTPS/WSS em produção ou direto ao Ktor em desenvolvimento), autenticando convidados com token HMAC-SHA256 (ADR 0007 A) e consumindo também as rotas `/api/curadoria/enigmas/*`.
- **Mensageria (S04 D4, S06, S12):** `infrastructure.messaging` contém `PublicadorDeEventosEmMemoria` (despacho pós-persistência H6 para os ouvintes do solo, difusão WebSocket por papel e `AoEncerrarPartidaAtualizarEstatisticas`).

Legenda: retângulo aninhado = nó «device» / ambiente de execução · linha cheia = comunicação (protocolo no rótulo) · tracejada = dependência.
