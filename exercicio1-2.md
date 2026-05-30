# Exercício 1.2 — QA — Critérios de Aceitação para Respostas de IA

**Papel:** QA
**Ferramentas utilizadas:** Claude (chat) + Claude Cowork
**Projeto:** Assistente de IA da NovaTech

---

## Parte 1 — Avaliação manual prévia (sem rubrica, sem IA)

Avaliação inicial das 5 respostas confrontando exclusivamente os documentos do Anexo A.

| #   | Pergunta                                      | Resposta do assistente                                                                     | Julgamento                        | Justificativa baseada no Anexo A                                                                                                                                                                                                                                                                                                                                      |
| --- | --------------------------------------------- | ------------------------------------------------------------------------------------------ | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Qual o prazo de devolução?                    | "7 dias úteis, exceto cargas perigosas classes 1 a 6 da ANTT."                             | **Correta**                       | Alinhada ao POL-001 §3.1 (7 dias úteis) e §3.2 (exceção carga perigosa). Fonte citada (POL-001, seção 3.2). Faltou citar §3.1, mas o conteúdo é fiel.                                                                                                                                                                                                                 |
| 2   | Quanto custa frete para 600kg para Manaus?    | "Frete especial para >500kg para a região Norte tem multiplicador 1.8 sobre o valor base." | **Parcialmente correta**          | O multiplicador 1.8 está correto (PROC-042-v2 §2.1). Porém, a resposta omite: (a) fator de peso 1.0 para 500–1.000kg, (b) a necessidade do valor base mensal, e (c) que existe versão anterior (PROC-042 v1) com multiplicador 1.6 — risco de o atendente cair em contradição. Não calcula valor final, mas isso é aceitável porque o valor base não está nos chunks. |
| 3   | Qual o SLA do cliente Platinum?               | "Cliente Platinum tem resposta em até 1h e resolução em até 12h."                          | **Incorreta — alucinação grave**  | O tier "Platinum" **não existe** (SLA-2024 §1 e FAQ-15 confirmam que só existem Gold/Silver/Standard). Os valores 1h/12h não constam em nenhum documento. Fonte citada (SLA-2024) não suporta a resposta — alucinação com aparência de citação.                                                                                                                       |
| 4   | Posso devolver carga perigosa?                | "Sim, cargas perigosas podem ser devolvidas em até 7 dias úteis."                          | **Incorreta — inversão de regra** | POL-001 §3.2 lista carga perigosa como **NÃO elegível** ao processo padrão. A resposta correta seria "não pelo processo padrão; encaminhar à Gestão de Riscos (ramal 4500)". Fonte citada está certa, mas o conteúdo extraído está invertido — risco operacional alto.                                                                                                |
| 5   | Qual o multiplicador de frete para o Sudeste? | "O multiplicador regional para o Sudeste é 1.1."                                           | **Correta**                       | Valor 1.1 alinhado à PROC-042-v2 §2.1 (versão vigente). Fonte citada corretamente. Ressalva: idealmente sinalizaria que existe versão anterior com 1.0 — falta de transparência, mas o valor entregue é o vigente.                                                                                                                                                    |

**Veredito da avaliação manual:** 2 corretas (1, 5), 1 parcialmente correta (2), 2 incorretas (3, 4). As falhas 3 e 4 são as mais críticas para risco de negócio.

---

## Parte 2 — Rubrica de avaliação (construída com Claude)

Rubrica com 4 dimensões, escala 1–3, projetada para que dois avaliadores cheguem a pontuações próximas.

### Dimensão 1 — Precisão factual

> A resposta corresponde ao que está escrito nos documentos normativos do Anexo A?

| Nível             | Critério                                                                                                                                                     |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **3 — Precisa**   | Todas as afirmações da resposta são verificáveis na documentação oficial (POL/PROC/SLA). Valores numéricos, condições e exceções estão fiéis ao texto-fonte. |
| **2 — Parcial**   | A resposta é majoritariamente correta, mas omite uma condição/exceção/qualificador relevante OU usa uma versão obsoleta de um documento sem sinalizar.       |
| **1 — Incorreta** | Contém pelo menos uma afirmação factualmente errada, invertida em relação à fonte, ou inventada (alucinação).                                                |

### Dimensão 2 — Citação de fonte

> A resposta indica onde a informação foi extraída, de forma rastreável?

| Nível                        | Critério                                                                                           |
| ---------------------------- | -------------------------------------------------------------------------------------------------- |
| **3 — Rastreável**           | Cita documento + seção/cláusula específica que de fato suporta a resposta.                         |
| **2 — Genérica**             | Cita apenas o nome do documento, sem seção, OU cita um documento que parcialmente cobre o assunto. |
| **1 — Ausente ou incorreta** | Sem citação, OU cita fonte que não suporta a afirmação (falsa citação).                            |

### Dimensão 3 — Aderência aos guardrails

> A resposta respeita as 4 regras: (a) citar fonte, (b) não inventar prazos/valores, (c) declarar quando não sabe, (d) usar português formal acessível.

| Nível                  | Critério                                                                                        |
| ---------------------- | ----------------------------------------------------------------------------------------------- |
| **3 — Aderente**       | Cumpre os 4 guardrails.                                                                         |
| **2 — Quebra leve**    | Cumpre 3 dos 4 (ex.: tom informal, ou citação genérica). Nenhuma invenção de valor.             |
| **1 — Quebra crítica** | Viola guardrail (b) ou (c) — inventa valor/prazo OU afirma saber algo que não está documentado. |

### Dimensão 4 — Completude e segurança operacional

> A resposta dá ao atendente todas as informações necessárias para agir sem induzir a erros operacionais (escala, exceções, próximos passos)?

| Nível                            | Critério                                                                                                                                       |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **3 — Completa**                 | Cobre regra principal + exceções aplicáveis + próximos passos (quando relevante). O atendente consegue responder ao cliente sem nova consulta. |
| **2 — Suficiente**               | Cobre a regra principal mas omite exceção ou próximo passo que poderia ser útil. Não induz a erro, mas exige consulta adicional.               |
| **1 — Insuficiente / arriscada** | Omite exceção crítica que poderia gerar erro operacional, OU é vaga demais para ser usada.                                                     |

### Pontuação final

- **Excelente:** 11–12 pontos
- **Aceitável:** 8–10 pontos
- **Necessita revisão:** 6–7 pontos
- **Rejeitada:** ≤ 5 pontos OU qualquer dimensão pontuando 1

> **Regra de bloqueio:** qualquer resposta com nota 1 em **Precisão factual** ou em **Aderência aos guardrails** é automaticamente classificada como **Rejeitada**, independente da soma.

---

## Parte 3 — Template reutilizável (gerado com Claude Cowork)

Especificação do template (uma planilha CSV/Sheets que o Cowork materializa). Estrutura pronta para qualquer lote de respostas.

```csv
ID,Data_avaliacao,Avaliador,Pergunta,Resposta_assistente,Fonte_citada,Doc_referencia_Anexo_A,
Precisao_factual_1a3,Citacao_fonte_1a3,Guardrails_1a3,Completude_1a3,
Pontuacao_total,Classificacao,Bloqueio_aplicado_S_N,Observacoes,Acao_recomendada
```

### Campos calculados automaticamente

- `Pontuacao_total` = soma das 4 dimensões.
- `Classificacao` = lookup pela faixa (Excelente / Aceitável / Necessita revisão / Rejeitada).
- `Bloqueio_aplicado_S_N` = "S" se `Precisao_factual_1a3 = 1` OU `Guardrails_1a3 = 1`; nesse caso `Classificacao` é forçada a "Rejeitada".
- `Acao_recomendada`:
  - Rejeitada → "Abrir ticket de correção: revisar prompt/retrieval e re-testar."
  - Necessita revisão → "Revisar chunk e reavaliar."
  - Aceitável → "Monitorar em regressão."
  - Excelente → "Adicionar ao conjunto de regressão como gold answer."

### Convenções de uso (instruções no cabeçalho do template)

1. Cada linha = uma resposta avaliada.
2. Documento de referência do Anexo A é **obrigatório** — sem ele a avaliação é inválida.
3. Dois avaliadores em paralelo para o primeiro lote de cada release; divergência > 1 ponto em qualquer dimensão dispara reunião de calibração.
4. Lotes de regressão: mínimo de 20 respostas por release, distribuídas entre as 10 categorias do Exercício 1.1.

---

## Parte 4 — Aplicação da rubrica às 5 respostas

| #   | Precisão                                                  | Citação                                                                    | Guardrails                                                                    | Completude                                                 | Total  | Bloqueio? | Classificação | Ação recomendada                                                                                                                                                                                                                         |
| --- | --------------------------------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------- | ------ | --------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | 3                                                         | 2 _(cita §3.2 mas a regra de 7 dias está no §3.1)_                         | 3                                                                             | 3                                                          | **11** | Não       | **Excelente** | Adicionar ao conjunto de regressão como gold answer; ajustar prompt para citar ambas as seções quando aplicável.                                                                                                                         |
| 2   | 2 _(omite fator de peso e existência da versão anterior)_ | 3                                                                          | 3                                                                             | 2 _(faltam fator de peso e valor base)_                    | **10** | Não       | **Aceitável** | Ajustar prompt para sempre incluir fator de peso, valor base e ressalva sobre versões coexistentes.                                                                                                                                      |
| 3   | 1 _(tier inexistente + valores inventados)_               | 1 _(cita SLA-2024 mas o documento não suporta a resposta — falsa citação)_ | 1 _(invenção de prazos — viola guardrail b)_                                  | 1                                                          | **4**  | **Sim**   | **Rejeitada** | Bloqueio crítico. Abrir incidente de qualidade: revisar prompt para forçar checagem contra SLA-2024 §1 antes de emitir SLA; adicionar guardrail determinístico que rejeite respostas mencionando tiers fora de {Gold, Silver, Standard}. |
| 4   | 1 _(inversão da exceção do POL-001 §3.2)_                 | 2 _(fonte certa, mas conteúdo extraído contradiz a fonte)_                 | 1 _(induz o atendente a um erro operacional, ainda que cite fonte)_           | 1 _(omite ramal 4500 e direcionamento à Gestão de Riscos)_ | **5**  | **Sim**   | **Rejeitada** | Bloqueio crítico. Revisar chunking do POL-001-B para garantir que a relação "exceção → não elegível" não seja quebrada; adicionar caso de regressão obrigatório.                                                                         |
| 5   | 3                                                         | 3                                                                          | 2 _(não sinaliza coexistência da v1 — quebra leve da transparência esperada)_ | 2 _(omite fator de peso e existência da versão anterior)_  | **10** | Não       | **Aceitável** | Ajustar prompt para sempre declarar a versão vigente ("conforme PROC-042-v2, vigente desde nov/2023") quando houver documento contraditório no índice.                                                                                   |

### Síntese

- **2 respostas Excelente/Aceitável e 2 Rejeitadas** (cenários 3 e 4 são exatamente os critérios de avaliação destacados no enunciado).
- Os bloqueios críticos (alucinação de tier e inversão de exceção) confirmam a necessidade do mecanismo de **enforcement determinístico** complementar ao prompt (ex.: filtro pós-resposta que rejeite menção a tiers inexistentes e que valide se a resposta sobre carga perigosa contém a palavra "não"/"não elegível").

---

## Evidências de uso das ferramentas

- **Claude (chat):** utilizado para iterar a estrutura da rubrica — primeira versão tinha apenas 3 dimensões; o Claude sugeriu separar "completude/segurança operacional" de "precisão" porque uma resposta pode ser factualmente correta mas omitir contexto que induz a erro (caso típico da resposta 4 onde a inversão tornou a "citação" perigosa).
- **Claude Cowork:** utilizado para materializar o template em planilha com cálculos automáticos de pontuação, classificação e bloqueio crítico, e gerar o cabeçalho com convenções de uso para o time.
