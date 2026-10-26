# Diagrama de componentes

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/Diagrama_comp.pdf`](../../diagramas/Diagrama_comp.pdf), versão de 14/10/2026 |
| Estado | Transcrição fiel do PDF. As mudanças já decididas nas specs e ADRs estão em [Pendências por diagrama](README.md#pendências-por-diagrama) |

Círculo ligado por linha cheia = interface fornecida; seta tracejada até o círculo = interface requerida.

```mermaid
flowchart TB
    front["Front-end<br/>(navegador)"]

    subgraph sub["«subsystem» Que Bug é Esse? · com.quebugesse"]
        subgraph interfaces["Interfaces"]
            nh["«adapter»<br/>NarradorHumano"]
            rest["rest"]
            ws["ws"]
        end
        comandos(("comandos do jogo"))
        subgraph aplicacao["Camada de Aplicação"]
            uc["Casos de Uso<br/>IniciarPartida · FazerPergunta · ResponderPergunta<br/>ProporSolucao · GerarEnigmaComIA · JogarSolo"]
        end
        subgraph dominio["Domínio"]
            gs["Partida de Jogo<br/>(GameSession)"]
            en["Catálogo de Enigmas<br/>(Enigma)"]
            pl["Jogador<br/>(Player)"]
            pNarrador(("Narrador"))
            pVerif(("VerificadorDeSolucao<br/>(fornecedor a decidir)"))
            pGerador(("GeradorDeEnigmas"))
            pRepos(("Repositórios"))
            pDifusao(("difusão de<br/>DomainEvent"))
        end
        subgraph infra["Infraestrutura · implementa as portas do domínio"]
            ai["«adapter»<br/>ai (NarradorIA)"]
            persistence["«adapter»<br/>persistence"]
            messaging["«adapter»<br/>messaging"]
        end
    end

    llm(["API de LLM"])
    db[("Banco de Dados<br/>dados")]

    front -->|HTTP| rest
    front -->|WSS| ws
    rest -.-> comandos
    ws -.-> comandos
    comandos --- uc
    uc -.->|«use»| gs
    uc -.->|«use»| en
    uc -.->|«use»| pl
    gs -.-> pNarrador
    gs -.-> pVerif
    en -.-> pGerador
    uc -.-> pRepos
    uc -.-> pDifusao
    nh --- pNarrador
    ai --- pNarrador
    ai --- pGerador
    persistence --- pRepos
    messaging --- pDifusao
    ai <-->|HTTPS / REST| llm
    persistence <-->|JDBC / TCP| db
```

## Notas do PDF

- "NarradorHumano só repassa a resposta digitada pelo narrador humano."
- "A tela recebe só enunciado e estado público: solucaoCompleta não sai do domínio."
- "Adaptadores trocáveis sem tocar no domínio."

Legenda do PDF: ▭ com ícone = componente · contorno laranja = camada · ○ ligado por linha cheia = interface fornecida · seta tracejada até ○ = interface requerida · tracejada «use» = dependência · seta cheia = protocolo de acesso.
