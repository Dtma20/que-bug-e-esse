# Diagrama de sequência 1/2: iniciar partida

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/diagrama_seq_iniciar.pdf`](../../diagramas/diagrama_seq_iniciar.pdf), versão de 14/10/2026 |
| Estado | Atualizado em 2026-11-24 com S02, S03, S04 e ADRs 0001–0007 |

```mermaid
sequenceDiagram
    actor Q as Quem inicia
    participant T as «interfaces»<br/>CLI / API Ktor
    participant UC as «useCase»<br/>IniciarPartida
    participant PR as «port»<br/>PlayerRepository
    participant ER as «port»<br/>EnigmaRepository
    participant GSR as «port»<br/>GameSessionRepository
    participant PUB as «port»<br/>PublicadorDeEventos

    Q->>T: 01 escolhe modo, categoria? e dificuldade?
    T->>UC: 02 executar(modo, iniciador, investigadores, categoria?, dificuldade?)
    UC->>PR: 03 buscar(iniciador e investigadores)
    PR-->>UC: 04 valida existência e tipo HUMANO (A2)
    UC->>ER: 05 publicados()
    ER-->>UC: 06 candidatos publicados
    UC->>GSR: 07 enigmasJogadosPor(investigadores)
    GSR-->>UC: 08 jaJogados
    UC->>UC: 09 CriterioDeSelecao.elegiveis(...) + sorteador (S02/M18)

    alt modo SOLO
        UC->>UC: 10 narrador = PlayerRef(JOGADOR_IA), investigadores = [iniciador]
    else modo GRUPO
        UC->>UC: 11 narrador = PlayerRef(iniciador)
    end

    create participant GS as «AggregateRoot»<br/>GameSession
    UC-->>GS: 12 «create» criar(id, enigma.id, modo, narrador, investigadores, agora)
    GS-->>UC: 13 AGUARDANDO_INICIO
    UC->>GS: 14 iniciar(agora)
    GS->>GS: 15 valida configuração (H5 estendida)
    GS-->>UC: 16 EM_ANDAMENTO / acumula PartidaIniciada

    UC->>GSR: 17 salvar(partida)
    UC->>GS: 18 puxarEventos()
    UC->>PUB: 19 publicar([PartidaIniciada]) (H6)
    UC->>GSR: 20 buscar(id) (releitura EV5)

    alt modo SOLO
        UC-->>T: 21 VisaoInvestigador (sem solucaoCompleta · P3)
    else modo GRUPO
        UC-->>T: 22 VisaoNarrador (com solucaoCompleta · P3)
    end
    T-->>Q: 23 exibe enunciado e partida iniciada
```

## Notas atualizadas (P9)

- **Sigilo por papel (P3, M1, S03 D4):** `VisaoInvestigador` nunca inclui `solucaoCompleta` antes de um estado terminal; `VisaoNarrador` inclui a solução apenas para o narrador autenticado da partida. Na CLI (S05), a tela compartilhada não exibe a solução automaticamente (exige comando `solucao` com confirmação).
- **Seleção de enigma (S02 D5, M18):** `EnigmaRepository` devolve `publicados()`; o domínio filtra com `CriterioDeSelecao` excluindo `jaJogados` (`GameSessionRepository.enigmasJogadosPor`); a aplicação sorteia via `Sorteador` (determinístico nos testes, P1).
- **Publicação após persistir e releitura (H6, EV5, S04 D7):** `PartidaIniciada` só é publicado após `salvar(partida)` retornar com sucesso; em seguida o caso de uso relê a partida antes de montar a visão de retorno.
- **Sala de espera no grupo online (M11, S06):** na API Ktor (S06), o modo `GRUPO` online separa `CriarPartida` (que cria a partida em `AGUARDANDO_INICIO` com código de sala para os investigadores entrarem via UC12) de `IniciarPartida` (passo 14 em diante). No modo `SOLO` e na CLI local (S05), criação e início ocorrem em sequência como acima.
- **"Quem inicia" (M12):** narrador humano no modo `GRUPO`, ou o próprio investigador no modo `SOLO`.

Legenda: cabeçalho escuro = «AggregateRoot» · bege = «useCase» · laranja-claro = «port» · seta cheia = chamada · tracejada = retorno / «create» · quadro = fragmento (`alt`).
