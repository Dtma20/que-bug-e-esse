# Diagrama de casos de uso

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/diagrama_casos.pdf`](../../diagramas/diagrama_casos.pdf), versão de 14/10/2026 |
| Estado | Transcrição fiel do PDF. As mudanças já decididas nas specs e ADRs estão em [Pendências por diagrama](README.md#pendências-por-diagrama) |

```mermaid
flowchart LR
    inv["«ator»<br/>Investigador"]
    narr["«ator»<br/>Narrador humano"]
    cur["«ator»<br/>Administrador / Curador"]
    llm["«ator»<br/>Serviço de LLM<br/>(sistema externo)"]

    subgraph sistema["«system» Que Bug é Esse?"]
        UC1(["UC1 Escolher modo de jogo<br/>(solo / grupo online / offline)"])
        UC10(["UC10 Jogar sozinho<br/>(IA assume a narração)"])
        UC4(["UC4 Fazer pergunta"])
        UC6(["UC6 Propor solução final"])
        UC7(["UC7 Consultar histórico<br/>da partida"])
        UC3(["UC3 Iniciar partida"])
        UC5(["UC5 Responder pergunta<br/>(Sim / Não / Irrelevante)"])
        UC9(["UC9 Cadastrar / revisar<br/>enigma manualmente"])
        UC2(["UC2 Selecionar enigma"])
        UC8(["UC8 Gerar enigma via IA"])
    end

    inv --- UC1
    inv --- UC10
    inv --- UC4
    inv --- UC6
    inv --- UC7
    narr --- UC3
    narr --- UC5
    cur --- UC8
    cur --- UC9
    UC10 --- llm
    UC8 --- llm

    UC7 -.->|«extend»| UC6
    UC3 -.->|«include»| UC2
    UC8 -.->|«extend»| UC10

    classDef principal fill:#f6dcb8,stroke:#e08a2c,stroke-width:2px
    classDef externo fill:#ffffff,stroke:#999999,stroke-dasharray:3 3
    class UC4,UC5,UC6 principal
    class llm externo
```

## Notas do PDF

- "No modo solo, UC5 é executado pelo NarradorIA — mesmo caso de uso, outra implementação do port Narrador."
- "Destaque laranja = fluxo principal da investigação (UC4 → UC5 → UC6)."
- "«include» parte do caso base (sempre executa). «extend» parte do caso opcional para o caso que ele estende."

Legenda do PDF: boneco = ator (contorno claro = sistema externo) · elipse = caso de uso · elipse laranja = fluxo principal · linha cheia = associação · tracejada = «include» / «extend» · retângulo = fronteira do sistema.
