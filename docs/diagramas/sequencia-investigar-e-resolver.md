# Diagrama de sequência 2/2: investigar e resolver

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/diagrama_seq_jogar.pdf`](../../diagramas/diagrama_seq_jogar.pdf), versão de 14/10/2026 |
| Estado | Transcrição fiel do PDF. As mudanças já decididas nas specs e ADRs estão em [Pendências por diagrama](README.md#pendências-por-diagrama) |

```mermaid
sequenceDiagram
    actor I as Investigador
    participant FP as «useCase»<br/>FazerPerguntaUseCase
    participant RP as «useCase»<br/>ResponderPerguntaUseCase
    participant PS as «useCase»<br/>ProporSolucaoUseCase
    participant GS as «AggregateRoot»<br/>GameSession
    participant N as «port»<br/>Narrador
    participant V as «port»<br/>VerificadorDeSolucao
    participant ER as «port»<br/>EnigmaRepository

    loop até o investigador propor uma solução
        I->>FP: 01 fazerPergunta(texto)
        FP->>GS: 02 fazerPergunta(autor, texto)
        GS-->>FP: 03 Question (aberta) / PerguntaFeita
        FP-)RP: 04 PerguntaFeita
        RP->>N: 05 responder(pergunta, contexto)
        alt resposta obtida
            N-->>RP: 06 Answer
            RP->>GS: 07 responderPergunta(narrador, id, resposta)
            GS-->>RP: 08 / PerguntaRespondida
            RP-->>I: 09 SIM | NAO | IRRELEVANTE
        else falha do NarradorIA (H4)
            N-->>RP: 10 falha
            RP-->>I: 11 erro — pergunta continua pendente
        end
    end

    I->>PS: 12 proporSolucao(texto)
    PS->>GS: 13 proporSolucao(autor, texto)
    GS-->>PS: 14 / SolucaoProposta
    PS->>ER: 15 buscar(partida.enigmaId)
    ER-->>PS: 16 enigma
    PS->>V: 17 avaliar(texto, enigma.solucaoCompleta)
    V-->>PS: 18 veredito
    PS->>GS: 19 aplicarVeredito(veredito, agora)
    alt veredito.correta
        GS-->>PS: 20 RESOLVIDO, resolvidaEm / PartidaResolvida
        PS-->>I: 21 sucesso + solução completa
    else não correta
        GS-->>PS: 22 EM_ANDAMENTO
        PS-->>I: 23 continuar investigação
    end
```

## Notas do PDF

- "Leitura adotada: o evento PerguntaFeita dispara ResponderPerguntaUseCase, que consulta o port Narrador (humano ou IA). A validar com a equipe."
- "Camada interfaces omitida (só traduz entrada/saída). Cada caso de uso carrega a partida do GameSessionRepository e a salva ao final."
- "Seta aberta (04) = mensagem assíncrona: o evento de domínio."

Legenda do PDF: cabeçalho escuro = «AggregateRoot» · bege = «useCase» · laranja-claro = «port» · seta cheia = chamada · aberta = assíncrona · tracejada = retorno · quadro laranja = fragmento (loop / alt).
