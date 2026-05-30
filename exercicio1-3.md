# Exercício 1.3 — QA — Plano de Testes para Pipeline de RAG

**Papel:** QA
**Ferramentas utilizadas:** Claude (chat) + Claude Cowork
**Projeto:** Assistente de IA da NovaTech

---

## 1. Premissas

- O pipeline segue o fluxo: **ingestão → extração de texto → chunking → embedding → indexação (Azure AI Search) → retrieval (top-k por similaridade) → montagem de prompt → geração via LLM → resposta**.
- Testes de IA são **não-determinísticos**: avaliamos em graus de qualidade (rubrica do Exercício 1.2) e não apenas pass/fail.
- O Anexo A é a **fonte de verdade**; o mapa de cobertura do Anexo B é o **gabarito de retrieval**.
- Guardrails do produto: citar fonte, nunca inventar prazos/valores, declarar quando não souber, português formal.

---

## 2. Plano de testes por camada

### 2.1 Testes de ingestão

| ID     | Objetivo                                                        | Verificação                                                                                                                      |
| ------ | --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| ING-01 | Cobertura: todos os documentos do Anexo A foram indexados       | Contagem de documentos no índice = 5; lista de IDs bate com `POL-001`, `PROC-042`, `PROC-042-v2`, `SLA-2024`, `FAQ-Atendimento`. |
| ING-02 | Integridade da extração de tabelas (SLA-2024 §2, PROC-042 §2.1) | Para cada linha esperada da tabela, existir pelo menos um chunk que contenha a linha completa (regex sobre o conteúdo do chunk). |
| ING-03 | Preservação de exceções críticas                                | O chunk que contém POL-001 §3.2 contém as palavras "NÃO" e "perigosa" no mesmo trecho (não foram separadas pelo chunking).       |
| ING-04 | Metadados obrigatórios por chunk                                | Cada chunk tem `documento`, `versao`, `secao`, `data_atualizacao`, `classificacao` (normativo/informal).                         |
| ING-05 | Sinalização de fonte informal                                   | Todos os chunks do FAQ têm `classificacao = "informal"`.                                                                         |
| ING-06 | Documentos contraditórios convivem com vigência marcada         | Chunks da PROC-042 v1 têm `versao = 1.0` e `vigente = false`; chunks da v2 têm `versao = 2.0` e `vigente = true`.                |
| ING-07 | Reingestão idempotente                                          | Rodar ingestão duas vezes sobre o mesmo input não gera chunks duplicados (verificar contagem antes/depois).                      |
| ING-08 | OCR funcional para PDFs escaneados (produção)                   | Documento escaneado de teste produz chunk com texto reconhecível (≥ 95% das palavras válidas via dicionário).                    |

### 2.2 Testes de retrieval

Conjunto base derivado do mapa de cobertura do Anexo B. Para cada caso medimos **Recall@k** (os chunks esperados estão no top-k) e **MRR** (Mean Reciprocal Rank).

| ID     | Pergunta                                                              | Top-k esperado (Anexo B)                         | Critério                                                                                                                                 |
| ------ | --------------------------------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| RET-01 | "Qual o prazo de devolução?"                                          | POL-001-A, POL-001-B                             | Recall@3 = 1.0                                                                                                                           |
| RET-02 | "Posso devolver carga perigosa?"                                      | POL-001-B                                        | POL-001-B em top-2; FAQ-03 pode aparecer mas **não pode** estar acima do POL-001-B (fonte normativa deve dominar fonte informal)         |
| RET-03 | "Qual o SLA do cliente Gold?"                                         | SLA-2024-B                                       | Recall@3 = 1.0; SLA-2024-A entre os top-3                                                                                                |
| RET-04 | "Qual o SLA do cliente Platinum?"                                     | SLA-2024-A (declara que só existem 3 tiers)      | SLA-2024-A em top-2; nenhum chunk com valores numéricos de SLA deve dominar (evitar viés para inventar)                                  |
| RET-05 | "Frete para 600kg para Manaus?"                                       | PROC-042v2-B, PROC-042v2-A                       | Versão v2 deve ranquear acima da v1; **alerta** se PROC-042-B aparecer em top-3 sem PROC-042v2-B junto                                   |
| RET-06 | "Frete para 300kg para Salvador?"                                     | (nenhum chunk relevante na base)                 | Score máximo do top-1 deve ficar abaixo do threshold de confiança (a definir empiricamente, ex.: < 0.6); aciona fluxo de "não encontrei" |
| RET-07 | "O que acontece com carga danificada?"                                | FAQ-38                                           | FAQ-38 em top-2; sistema deve marcar a resposta como `fonte=informal`                                                                    |
| RET-08 | "Carga perigosa com frete expresso?"                                  | FAQ-32                                           | idem RET-07                                                                                                                              |
| RET-09 | "Qual o multiplicador para o Sudeste?"                                | PROC-042v2-B                                     | PROC-042v2-B acima de PROC-042-B no ranking                                                                                              |
| RET-10 | Multi-domínio: "Prazo de devolução + carga perigosa + frete especial" | POL-001-A, POL-001-B, PROC-042v2-A, PROC-042v2-B | Recall@6 = 1.0                                                                                                                           |

**Métricas agregadas:** Recall@3 ≥ 0.85, MRR ≥ 0.80, % de queries que retornam chunk v1 sem v2 ≤ 5%.

### 2.3 Testes de geração (assumindo chunks corretos)

Mesmo com retrieval perfeito, o LLM pode falhar. Os testes abaixo isolam a etapa de geração injetando chunks corretos manualmente.

| ID     | Objetivo                           | Critério                                                                                                                                      |
| ------ | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| GEN-01 | Citação de fonte presente          | 100% das respostas contêm padrão `Fonte: <DOC>` (validação por regex).                                                                        |
| GEN-02 | Sem alucinação numérica            | Todo número na resposta deve aparecer em pelo menos um chunk de contexto (validação por extração de números e comparação).                    |
| GEN-03 | Inversão de exceção (POL-001 §3.2) | Pergunta "posso devolver carga perigosa?" + chunk POL-001-B → resposta contém "não" ou "não elegível"; falha se contiver afirmação positiva.  |
| GEN-04 | Tier inexistente                   | Pergunta sobre "Platinum" + chunk SLA-2024-A → resposta declara que tier não existe; sem números inventados.                                  |
| GEN-05 | Recusa estruturada                 | Quando nenhum chunk passa o threshold, resposta segue template "não encontrei essa informação na base oficial. Sugiro escalar ao supervisor." |
| GEN-06 | Tom em português formal            | Avaliação humana amostral (10%) + heurística automatizada (sem gírias, sem emojis).                                                           |
| GEN-07 | Tratamento de fonte informal       | Quando único chunk relevante é do FAQ, resposta inclui disclaimer ("documento informal, não validado pelo Compliance").                       |

### 2.4 Testes de contexto

Camada específica de **engenharia de contexto**. Estes testes exigem instrumentação do pipeline (log de tokens, ordem dos chunks no prompt, histórico).

| ID     | Objetivo                               | Critério                                                                                                                                                                            |
| ------ | -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CTX-01 | Orçamento de contexto respeitado       | Log do tamanho do prompt enviado ao LLM; nenhuma query deve ultrapassar `context_window - reserva_de_saida`. Alerta se chegar a 90% do limite.                                      |
| CTX-02 | Mitigação de _lost in the middle_      | Em prompts com 5+ chunks, chunks normativos críticos (POL/PROC vigentes) devem ficar no **início** ou no **fim** do bloco de contexto, nunca no meio. Verificação por log de ordem. |
| CTX-03 | Context rot em sessão longa            | Bateria de 6 perguntas seguidas; medir se taxa de citação de fonte cai do 1º para o 6º turno. Falha se cair mais de 5%.                                                             |
| CTX-04 | Janela deslizante de histórico         | Após N turnos, o histórico mais antigo é resumido ou descartado conforme política definida. Verificar que o system prompt e os guardrails permanecem íntegros em todos os turnos.   |
| CTX-05 | Detecção de overflow silencioso        | Pergunta proposital de 400+ tokens + 8 chunks; se ultrapassar o orçamento, pipeline deve responder com "pergunta muito complexa, posso dividir?" em vez de truncar silenciosamente. |
| CTX-06 | Prioridade entre fontes contraditórias | Quando contexto contém chunks de duas versões do mesmo documento, prompt deve incluir instrução explícita de priorizar `vigente = true`; resposta cita a v2.                        |

### 2.5 Testes ponta a ponta

Conjunto de regressão executado a cada mudança de prompt, modelo, índice ou documento.

| ID     | Pergunta                                                                       | Resposta esperada (resumo)                                                            | Critério                                     |
| ------ | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- | -------------------------------------------- |
| E2E-01 | "Qual o prazo de devolução padrão?"                                            | "7 dias úteis (POL-001 §3.1)."                                                        | Rubrica do Ex. 1.2 ≥ 10 pontos               |
| E2E-02 | "Posso devolver carga perigosa classe 3?"                                      | "Não pelo processo padrão. Encaminhar à Gestão de Riscos, ramal 4500 (POL-001 §3.2)." | ≥ 10 e sem bloqueio                          |
| E2E-03 | "Qual o SLA de resolução para Gold em chamado geral?"                          | "Até 24h úteis (SLA-2024 §2)."                                                        | ≥ 10                                         |
| E2E-04 | "Cliente Platinum: qual o SLA?"                                                | "Tier Platinum não existe; tiers oficiais: Gold/Silver/Standard."                     | ≥ 10 e sem números inventados                |
| E2E-05 | "Multiplicador de frete especial para o Norte?"                                | "1.8 (PROC-042-v2, vigente desde nov/2023)."                                          | ≥ 10                                         |
| E2E-06 | "Frete para 300kg para Salvador?"                                              | "Não encontrei regra de frete padrão (<500kg). Sugiro consultar o Comercial."         | Recusa estruturada                           |
| E2E-07 | "Carga danificada em trânsito, o que faço?"                                    | Resposta com base no FAQ-38 + disclaimer de fonte informal + e-mail sinistros@.       | Disclaimer presente                          |
| E2E-08 | Multi-domínio: "Cliente Gold quer devolver carga perigosa de 700kg para o NE." | Resposta declara inelegibilidade da devolução **primeiro**, depois SLA Gold.          | Inelegibilidade nos primeiros 200 caracteres |
| E2E-09 | "Quantos dias para entregar frete especial?"                                   | "+3 dias úteis sobre o prazo padrão (PROC-042-v2 §3)."                                | Sem citar v1                                 |
| E2E-10 | "Qual o desconto para 12 fretes especiais/mês?"                                | "5% sobre o multiplicador regional (PROC-042-v2 §4)."                                 | ≥ 10                                         |

### 2.6 Testes de regressão e governança

| ID     | Gatilho                                            | Ação                                                                                          |
| ------ | -------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| REG-01 | Mudança no system prompt                           | Roda E2E-01..10 + GEN-01..07 + CTX-03                                                         |
| REG-02 | Atualização de documento (novo PDF ou nova versão) | Roda ING-01..06 + RET (todas as queries que tocam o documento alterado) + E2E correspondentes |
| REG-03 | Troca de modelo de embedding                       | Roda toda a suíte RET + métricas agregadas (Recall@3, MRR)                                    |
| REG-04 | Troca de LLM                                       | Roda GEN + E2E completos                                                                      |
| REG-05 | Cron diário                                        | Subset rápido (5 perguntas de smoke) + alerta se métrica cai mais de 10% vs baseline          |

---

## 3. Reconhecimento da natureza não-determinística

- **Pass/fail binário** aplicado apenas a guardrails determinísticos (presença de citação, ausência de tier inexistente, presença de termos críticos como "não elegível").
- **Score gradativo** (rubrica do Ex. 1.2) aplicado à qualidade textual, com limiar de aceitação por categoria.
- Cada teste é executado **3 vezes** (temperature > 0); o resultado considerado é a **mediana** das pontuações e o **pior caso** para guardrails determinísticos.
- Variação > 2 pontos entre execuções da mesma pergunta dispara revisão de prompt (sinal de instabilidade).

---

## 4. Artefato organizado (gerado com Claude Cowork)

Especificação do checklist rastreável que o Cowork materializa (Planilha com abas + dashboard).

### Aba 1 — Catálogo de testes

```
ID | Camada | Descricao | Pergunta/Input | Criterio_aceitacao | Tipo (binario/graduado) | Automatizado_S_N | Responsavel | Versao_pipeline | Status (Pendente/Pass/Fail/Bloqueado) | Ultima_execucao | Evidencia (link)
```

### Aba 2 — Execuções

```
ID_teste | Data | Build_id | Resultado | Score | Observacao | Bug_link
```

### Aba 3 — Métricas agregadas (dashboard)

- Recall@3 médio (camada RET) — meta ≥ 0.85
- MRR (camada RET) — meta ≥ 0.80
- % de respostas com citação válida (camada GEN-01) — meta = 100%
- % de queries acima de 90% do orçamento de contexto (CTX-01) — meta ≤ 2%
- Taxa de bloqueio crítico no E2E (rubrica do Ex. 1.2) — meta = 0%
- Tendência: gráfico de qualidade por release

### Aba 4 — Backlog de bugs

```
ID | Teste_origem | Severidade (Critica/Alta/Media/Baixa) | Descricao | Status | Owner | Release_alvo
```

### Convenções de uso

- Responsável por camada: ING e CTX → Tech Lead/Dev; RET e GEN → QA; E2E → QA + Product Specialist.
- Frequência mínima de execução: diária (smoke), por PR de prompt (REG-01), semanal (suíte completa de E2E).
- Regra de release: **nenhum release vai a produção** com taxa de bloqueio crítico > 0% ou Recall@3 < baseline − 5%.

---

## Evidências de uso das ferramentas

- **Claude (chat):** usado para construir o esqueleto inicial dos testes por camada e para revisar lacunas — em particular sugeriu separar **CTX** como camada própria (eu havia colocado dentro de GEN) e adicionou os testes de fonte informal (GEN-07, RET-07/08).
- **Claude Cowork:** usado para gerar o artefato rastreável com as 4 abas, fórmulas de métricas agregadas e o dashboard de tendência por release, pronto para ser compartilhado com o time multidisciplinar.
