# Diagrama de implantação

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/Diagrama_imp.pdf`](../../diagramas/Diagrama_imp.pdf), versão de 14/10/2026 |
| Estado | Transcrição fiel do PDF. As mudanças já decididas nas specs e ADRs estão em [Pendências por diagrama](README.md#pendências-por-diagrama) |

```mermaid
flowchart TB
    subgraph usuario["Dispositivo do Usuário «device»<br/>(PC / smartphone)"]
        nav["Navegador Web"]
        fe["«artifact»<br/>front-end"]
        nav -->|carrega| fe
    end
    subgraph terminal["Terminal do Usuário / CLI «device»<br/>(acesso direto de desenvolvedores ou jogadores avançados)"]
        console["Console / terminal"]
    end
    subgraph dispNarr["Dispositivo do Narrador «device»<br/>narrador humano (modo em grupo)"]
        navNarr["Navegador Web<br/>(tela do narrador)"]
    end

    proxy["Servidor Web / Proxy<br/>«Nginx / Gateway»"]

    subgraph servidor["Servidor de Aplicação «device»"]
        subgraph jvm["«executionEnvironment» JVM · aplicação Kotlin"]
            subgraph interfaces["Interfaces"]
                ws["ws"]
                rest["rest"]
                nh["«adapter»<br/>NarradorHumano"]
            end
            subgraph aplicacao["Camada de Aplicação"]
                uc["Casos de Uso<br/>IniciarPartida · FazerPergunta<br/>ResponderPergunta · ProporSolucao<br/>GerarEnigmaComIA · JogarSolo"]
            end
            subgraph dominio["Domínio"]
                gs["Partida de Jogo"]
                en["Catálogo de Enigmas"]
                pl["Jogador"]
                portas["Portas do domínio"]
            end
            subgraph infra["Infraestrutura · implementa as portas do domínio (○)"]
                pers["Adaptador de persistência"]
                nia["NarradorIA<br/>(ai)"]
                msg["WebSocket / SSE<br/>(messaging)"]
            end
        end
    end

    subgraph servBanco["Servidor de Banco de Dados «device»"]
        db[("Banco de Dados")]
    end

    llm(["API de LLM<br/>serviço externo"])

    usuario <-->|HTTPS| proxy
    usuario <-->|WSS| proxy
    terminal <-->|SSH| proxy
    dispNarr <-->|HTTPS / WSS| proxy
    proxy <-->|WSS| ws
    proxy <-->|"HTTP (REST)"| rest
    ws -.-> uc
    rest -.-> uc
    aplicacao -.->|«use»| dominio
    pers <-->|JDBC / TCP| db
    nia <-->|"HTTPS / REST (Internet)"| llm
```

## Notas do PDF

- "Cliente SSH: jogar via terminal — útil para validar o fluxo manualmente durante o TDD."
- "O NarradorHumano roda no servidor: só repassa a resposta que o narrador digita aqui."
- "Adaptadores trocáveis sem tocar no domínio."
- "Persiste partidas, enigmas e jogadores. Nos testes, os repositórios ficam em memória."

Legenda do PDF: cubo = nó «device» · contorno laranja = camada lógica · ▭ com ícone = componente · ○ = interface fornecida · linha cheia = caminho de comunicação (protocolo no rótulo) · tracejada = dependência.
