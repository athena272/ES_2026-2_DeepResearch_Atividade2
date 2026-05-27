# Trecho para o Relatório Técnico (PDF) — Parte 1

> Copie as seções abaixo para o capítulo **Eixo A — GPR (Arqueologia de Issues)** do PDF. Ajuste numeração conforme o template da equipe. Figuras: colocar prints de `assets/parte1/` quando disponíveis.

---

## GPR — Arqueologia de issues: liberação do WebWatcher (Issue #88)

### Resumo executivo

Reconstituímos a evolução da funcionalidade **WebWatcher** no repositório [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) a partir da [Issue #88](https://github.com/Alibaba-NLP/DeepResearch/issues/88), aberta em setembro de 2025 por um usuário externo que não localizava o código prometido na documentação. A linha do tempo liga **dez comentários**, **vários pull requests** (notadamente #74, #78, #80 em agosto e #208 em novembro) e **commits** em `WebAgent/WebWatcher/infer/`, sem que nenhum merge referencie explicitamente `#88`.

O diagnóstico GPR é de **escopo mal comunicado** (“release” em PRs de README ≠ código de inferência), **atraso em relação a promessa pública** (publicação prevista para setembro, núcleo de inferência integrado em novembro) e **rastreabilidade fraca** entre demanda da comunidade e entrega técnica. Isso complementa a Atividade 1: lá, trilhas como #233 mostram ligação issue→PR→arquivo; aqui, o padrão dominante em feature grande é o oposto.

**Risco global:** Alto.

### Figuras sugeridas

| Figura | Arquivo | Legenda sugerida |
|--------|---------|------------------|
| Figura X.1 | `issue-88-abertura.png` | Abertura da Issue #88 — código WebWatcher não encontrado (01/09/2025). |
| Figura X.2 | `issue-88-comentario-set.png` | Mantenedor define escopo como código de inferência com previsão “este mês” (08/09/2025). |
| Figura X.3 | `pr-80-merge.png` | PR #80 mergeado como “release WebWatcher” sem descrição e sem review (15/08/2025). |
| Figura X.4 | `pr-208-merge.png` ou `commit-dc87c85-stat.png` | Integração do código em `WebAgent/WebWatcher/infer/` (nov/2025). |
| Figura X.5 | `issue-88-fechamento.png` | Fechamento da issue por indicação de pasta, sem link automático a PR. |

### Texto analítico (corpo do relatório)

A gestão observável do ciclo de vida do WebWatcher combina **anúncios precoces** (pull requests de agosto de 2025 que atualizam README e roadmap) com **entrega tardia do artefato executável** (scripts de inferência e avaliação sob `WebAgent/WebWatcher/infer/`, integrados em novembro via PR #208). Para o usuário que abriu a Issue #88, essa dissociação gera ambiguidade: o histórico de merges sugere “lançamento” enquanto o problema reportado — ausência de código de inferência — permanece aberto por cerca de setenta dias.

A discussão nos comentários é reveladora do ponto de vista de **gerência de requisitos (GRE)**. Inicialmente, o relato é de localização de código; em seguida, um membro da equipe restringe o escopo ao **código de inferência** e indica prazo mensal. Não há atualização formal da issue com checklist de entregáveis nem vínculo a milestone no GitHub. Vários participantes repetem a mesma pergunta (“同问”), sinal de que o canal de issue funciona como **pressão social**, não como **instrumento de rastreabilidade**.

Do ponto de vista **MPS.BR / GPR (nível G)**, faltam práticas mínimas de monitoramento e controle de mudança: (i) definir o que constitui “release” (documentação, benchmark, inferência); (ii) amarrar issues públicas a pull requests com `Closes #NN`; (iii) publicar release ou tag quando o conjunto estiver utilizável — gap já identificado na Atividade 1 (ausência de releases nomeadas no GitHub).

Recomenda-se ao projeto open source: templates de issue e PR com campos de artefato e caminho no repositório; label ou milestone `webwatcher` agrupando #88, #181 e PRs correlatos; mensagens de commit descritivas em entregas grandes. Para a equipe auditora, a lição é que **rastreabilidade pontual** (como na correção #233 documentada na A1) **não generaliza** para funcionalidades multimodais no monorepo `WebAgent/`.

### Tabela resumo (Evidência → Risco)

| Etapa | Evidência (link) | Diagnóstico em uma linha | Risco |
|-------|------------------|---------------------------|-------|
| Abertura #88 | [Issue #88](https://github.com/Alibaba-NLP/DeepResearch/issues/88) | Demanda de localização de código; escopo implícito | Médio |
| Comentários | [Thread #88](https://github.com/Alibaba-NLP/DeepResearch/issues/88) | Escopo = inferência; prazo não cumprido na percepção dos usuários | Alto |
| PRs ago/2025 | [#74](https://github.com/Alibaba-NLP/DeepResearch/pull/74), [#78](https://github.com/Alibaba-NLP/DeepResearch/pull/78), [#80](https://github.com/Alibaba-NLP/DeepResearch/pull/80) | “Release” só de documentação | Alto |
| Entrega nov/2025 | [PR #208](https://github.com/Alibaba-NLP/DeepResearch/pull/208), commits `6d2eeae`/`dc87c85` | Código real sem link à #88 | Médio |
| Fechamento | Comentários nov/2025 em #88 | Resolução verbal; benchmark ainda corrigido em dez (#226) | Alto |

### Transição para a Parte 2

Os mesmos pull requests (#80, #208) aparecem na amostra da **Parte 2 (ritmo de entrega e qualidade do fluxo em PRs)**, onde se analisa cadência de merges e ausência de revisão comentada — sem repetir a narrativa da Issue #88.

---

*Fonte completa com diagrama e detalhamento: [parte1-arqueologia-issues.md](./parte1-arqueologia-issues.md) no repositório ES_2026-2_DeepResearch_Atividade2.*
