# Diagrama de estados da partida

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/diagrama_estados.pdf`](../../diagramas/diagrama_estados.pdf), versão de 14/10/2026 |
| Estado | Transcrição fiel do PDF. As mudanças já decididas nas specs e ADRs estão em [Pendências por diagrama](README.md#pendências-por-diagrama) |

```mermaid
stateDiagram-v2
    [*] --> AGUARDANDO_INICIO : criar(enigmaId, narrador, investigadores)
    AGUARDANDO_INICIO --> EM_ANDAMENTO : iniciar() [configuração válida (H5)] / PartidaIniciada
    AGUARDANDO_INICIO --> ABANDONADO : abandonar(autor) [autorizado (H2)] / PartidaAbandonada

    state EM_ANDAMENTO {
        [*] --> AguardandoPergunta
        AguardandoPergunta --> PerguntaPendente : fazerPergunta(autor, texto) [autor ∈ investigadores] / PerguntaFeita
        PerguntaPendente --> AguardandoPergunta : responderPergunta(autor, id, resposta) [autor = narrador] / PerguntaRespondida
        AguardandoPergunta --> SolucaoEmAvaliacao : proporSolucao(autor, texto) [autor ∈ investigadores] (H3) / SolucaoProposta
        SolucaoEmAvaliacao --> AguardandoPergunta : aplicarVeredito(v, t) [not v.correta]
        PerguntaPendente : PerguntaPendente
        PerguntaPendente : falha do NarradorIA / — (H4)
    }

    SolucaoEmAvaliacao --> RESOLVIDO : aplicarVeredito(v, t) [v.correta] / PartidaResolvida
    EM_ANDAMENTO --> ABANDONADO : abandonar(autor) [autorizado (H2)] / PartidaAbandonada

    RESOLVIDO : RESOLVIDO
    RESOLVIDO : resolvidaEm = t
    ABANDONADO : ABANDONADO
    ABANDONADO : resolvidaEm = null

    RESOLVIDO --> [*]
    ABANDONADO --> [*]

    classDef terminal fill:#3b3029,color:#ffffff
    class RESOLVIDO, ABANDONADO terminal
```

## Regras da partida (quadro do PDF)

| Regra | Texto |
|---|---|
| H1 | Estados terminais não aceitam novos comandos |
| H2 | Só participante autorizado pode abandonar |
| H3 | Proposta só sem pergunta em aberto |
| H4 | Falha da IA não muda estado; a pergunta segue pendente |
| H5 | Narrador definido e ao menos 1 investigador |

## Notas do PDF

- "Uma única pergunta em aberto por vez: a invariante vira estrutura do diagrama, não só uma guarda."
- "Toda transição com guarda falsa é rejeitada sem alterar o estado. Rótulos: evento(args) [guarda] / evento de domínio."
- "H1 — terminais rejeitam perguntas, propostas e novo abandono."

Legenda do PDF: ● estado inicial · ◉ estado final · cantos arredondados = estado · fundo escuro = estado terminal · contorno laranja = estado composto · seta = transição evento [guarda] / efeito.
