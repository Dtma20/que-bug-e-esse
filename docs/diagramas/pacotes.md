# Diagrama de pacotes

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/diagrama_pacotes.pdf`](../../diagramas/diagrama_pacotes.pdf), versão de 14/10/2026 |
| Estado | Transcrição fiel do PDF. As mudanças já decididas nas specs e ADRs estão em [Pendências por diagrama](README.md#pendências-por-diagrama) |

Raiz: `com.quebugesse`.

```mermaid
flowchart TB
    subgraph interfaces["interfaces"]
        rest["rest"]
        ws["ws"]
    end
    subgraph application["application"]
        usecase["usecase"]
        dto["dto"]
    end
    subgraph infrastructure["infrastructure"]
        persistence["persistence"]
        ai["ai"]
        messaging["messaging"]
    end
    subgraph domain["domain"]
        gamesession["gamesession"]
        enigma["enigma"]
        player["player"]
        shared["shared"]
    end

    usecase -.-> dto
    gamesession -.-> shared
    enigma -.-> shared
    player -.-> shared
    interfaces -.->|«use»| application
    application -.->|«use»| domain
    infrastructure -.->|"«use»<br/>implementa as portas"| domain
```

Legenda do PDF: pasta laranja = camada · pasta clara = pacote Kotlin · tracejada com seta aberta = dependência «use» · raiz: com.quebugesse.
