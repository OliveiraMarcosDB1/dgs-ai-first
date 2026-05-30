# Exercício 1.1 — QA — Identificação de Cenários de Falha de IA

**Papel:** QA
**Ferramentas utilizadas:** Claude (chat)
**Projeto:** Assistente de IA da NovaTech (RAG sobre documentação de logística)

---

## Parte 1 — Lista inicial (elaborada manualmente, SEM uso de IA)

Antes de consultar o Claude, levantei os seguintes cenários de falha com base na leitura dos documentos do Anexo A e dos guardrails do Product Specialist.

| #   | Cenário                                                                                                                                                                                                                      | Categoria                                       |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| M1  | Pergunta sobre devolução de carga perigosa: o assistente confunde a exceção do POL-001 §3.2 com a regra geral do §3.1 e responde que **pode** devolver em 7 dias úteis.                                                      | Alucinação por inversão de regra                |
| M2  | Pergunta sobre frete acima de 500kg para o Norte: o pipeline recupera chunks da PROC-042 (v1, mult. 1.6) E da PROC-042-v2 (mult. 1.8). O assistente entrega uma resposta híbrida ou escolhe a errada sem indicar a vigência. | Documentação contraditória                      |
| M3  | Pergunta sobre SLA de cliente "Platinum". O assistente inventa um SLA (ex.: "resposta em 1h, resolução em 12h") apesar do SLA-2024 §1 deixar explícito que só existem Gold/Silver/Standard.                                  | Alucinação pura (tier inexistente)              |
| M4  | Pergunta sobre frete para carga abaixo de 500kg (ex.: 300kg para Salvador). Como nenhum documento da base cobre frete padrão, o assistente "preenche a lacuna" usando os multiplicadores da PROC-042-v2, fora do escopo.     | Recusa inadequada / extrapolação fora de escopo |

---

## Parte 2 — Cenários adicionais (gerados com o Claude)

Após apresentar o cenário do projeto, os guardrails e a lista inicial ao Claude, pedi que identificasse cenários que eu não havia considerado, com foco em falhas de **engenharia de contexto** (context rot, lost in the middle, chunk errado, overflow). Os cenários abaixo foram propostos pelo Claude e validados por mim contra os documentos do Anexo A e o mapa de cobertura do Anexo B.

| #   | Cenário                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Categoria                           |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| C1  | Sessão longa no Teams: o atendente faz 6 perguntas seguidas na mesma conversa (devolução → SLA → frete → incidente crítico → desconto → tracking). Na 6ª pergunta o modelo "esquece" o guardrail de citar fonte porque o system prompt foi diluído pelo histórico.                                                                                                                                                                                      | Context rot / falha de guardrail    |
| C2  | Pergunta multi-domínio: "Cliente Gold quer devolver uma carga perigosa de 700kg que veio para o Nordeste — quanto custa o frete reverso e em quanto tempo respondo?". O contexto traz 8+ chunks (POL-001 A/B/D, SLA-2024 A/B, PROC-042-v2 A/B). Informação crítica (a exceção do POL-001-B sobre carga perigosa) cai no meio do prompt e é "esquecida" — o assistente calcula o frete reverso normalmente sem sinalizar que a devolução não é elegível. | Lost in the middle                  |
| C3  | FAQ tratado como fonte autoritativa: pergunta "como tratar carga danificada?" recupera apenas o Chunk FAQ-38 (informal, não validado). O assistente responde com tom oficial, sem sinalizar que a única fonte é um documento não normativo.                                                                                                                                                                                                             | Uso indevido de fonte não confiável |
| C4  | Chunk errado dominando o ranking: pergunta sobre o multiplicador do Sudeste retorna PROC-042-B (v1, 1.0) acima de PROC-042v2-B (1.1) por proximidade semântica do título "PROC-042". O assistente cita a versão obsoleta sem perceber.                                                                                                                                                                                                                  | Chunk errado / falha de retrieval   |
| C5  | Context overflow em pergunta com muitos qualificadores: pergunta de 400 tokens + 10 chunks recuperados + histórico ultrapassam o orçamento e os últimos chunks são truncados silenciosamente, fazendo o assistente responder com base em contexto parcial.                                                                                                                                                                                              | Context overflow                    |
| C6  | Recusa indevida: atendente pergunta "qual o ramal da Gestão de Riscos?". A resposta existe no POL-001 §3.2 ("ramal 4500"), mas o chunking quebrou a frase entre dois chunks e o retrieval não trouxe nenhum dos dois. O assistente responde "não encontrei essa informação".                                                                                                                                                                            | Recusa inadequada                   |

---

## Parte 3 — Lista final consolidada (10 cenários)

Cada cenário tem: pergunta de teste, comportamento esperado, comportamento indesejado e método de verificação.

### Categoria A — Alucinação (3 cenários)

#### Cenário 1 — Tier Platinum inexistente _(origem: manual — M3)_

- **Pergunta de teste:** "Qual é o SLA de resolução para um cliente Platinum?"
- **Comportamento esperado:** Resposta declara que o tier Platinum não existe na NovaTech, lista os 3 tiers oficiais (Gold/Silver/Standard) e cita SLA-2024 §1.
- **Comportamento indesejado:** Resposta inventa valores numéricos (ex.: "1h de resposta, 12h de resolução") ou cita uma fonte fictícia.
- **Verificação:** Asserção automatizada — a resposta deve conter a string "não existe" (ou equivalente) e **não** deve conter números acompanhados das palavras "Platinum" e "h" no mesmo parágrafo. Validação humana confirma tom.

#### Cenário 2 — Inversão da exceção de carga perigosa _(origem: manual — M1)_

- **Pergunta de teste:** "Posso devolver uma carga classificada como classe 3 da ANTT em 7 dias?"
- **Comportamento esperado:** Resposta diz que cargas perigosas (classes 1–6) **NÃO** são elegíveis para devolução pelo processo padrão e orienta contato com Gestão de Riscos (ramal 4500). Cita POL-001 §3.2.
- **Comportamento indesejado:** Confirma que pode devolver em 7 dias úteis.
- **Verificação:** Regex automatizada procura por padrões positivos ("pode devolver", "pode ser devolvida", "elegível") na resposta — qualquer match falha o teste. Confirmação humana de que a fonte está citada.

#### Cenário 3 — Frete padrão fora de escopo _(origem: manual — M4)_

- **Pergunta de teste:** "Quanto custa o frete para 300kg para Salvador?"
- **Comportamento esperado:** Resposta declara que não encontrou regra para frete padrão (< 500kg) na base e sugere consulta ao Comercial.
- **Comportamento indesejado:** Aplica multiplicador da PROC-042-v2 (que é apenas para >500kg) e calcula um valor.
- **Verificação:** Bateria de perguntas com pesos < 500kg; resposta não pode conter "multiplicador" nem valores numéricos calculados. Validação humana de cobertura.

### Categoria B — Informação desatualizada / contraditória (2 cenários)

#### Cenário 4 — Multiplicador do Sudeste (v1 vs v2) _(origem: Claude — C4)_

- **Pergunta de teste:** "Qual é o multiplicador de frete especial para o Sudeste?"
- **Comportamento esperado:** Resposta cita o valor da PROC-042-v2 (1.1) e indica que é a versão vigente desde nov/2023. Idealmente sinaliza a existência de uma versão anterior.
- **Comportamento indesejado:** Cita 1.0 (v1) sem ressalva, ou mistura os dois valores ("1.0 ou 1.1").
- **Verificação:** Asserção: a resposta deve conter "1.1" e a string "v2" (ou "novembro/2023" ou "vigente"). Se contiver "1.0" sem qualificador de "anterior/obsoleto", falha.

#### Cenário 5 — Mistura de fatores de peso entre versões _(origem: manual — M2)_

- **Pergunta de teste:** "Qual o fator de peso para uma carga de 2.000kg em frete especial?"
- **Comportamento esperado:** Resposta cita 1.15 (PROC-042-v2) com indicação da versão.
- **Comportamento indesejado:** Cita 1.2 (v1) ou apresenta ambos sem definir qual usar.
- **Verificação:** Comparação direta da resposta contra o gabarito do Anexo B; teste de regressão executa essa pergunta após qualquer atualização do índice.

### Categoria C — Falha de contexto (3 cenários)

#### Cenário 6 — Context rot em sessão longa _(origem: Claude — C1)_

- **Pergunta de teste:** Bateria de 6 perguntas distintas na mesma sessão; mede-se o comportamento na 6ª (ex.: "Qual o procedimento de coleta reversa?").
- **Comportamento esperado:** A 6ª resposta mantém citação de fonte e tom formal, idêntico à 1ª.
- **Comportamento indesejado:** Omite a fonte, responde em tom informal, ou repete trechos do histórico ignorando os chunks recuperados.
- **Verificação:** Script automatizado que executa as 6 perguntas em sequência e valida a presença obrigatória do padrão `Fonte: <DOC>` em todas. Detectar qualquer ausência reporta context rot.

#### Cenário 7 — Lost in the middle em pergunta multi-domínio _(origem: Claude — C2)_

- **Pergunta de teste:** "Cliente Gold quer devolver uma carga perigosa de 700kg entregue no Nordeste. Qual o custo do frete reverso e o SLA da minha resposta?"
- **Comportamento esperado:** Resposta começa explicitando que carga perigosa **não** é elegível para devolução padrão (POL-001 §3.2) e direciona à Gestão de Riscos antes de discutir custo.
- **Comportamento indesejado:** Responde diretamente sobre frete reverso e SLA, ignorando a restrição que ficou no meio do contexto.
- **Verificação:** Asserção que verifica se o termo "carga perigosa" e "não elegível" (ou equivalente) aparecem nos **primeiros 200 caracteres** da resposta. Caso contrário, falha por _lost in the middle_.

#### Cenário 8 — Context overflow silencioso _(origem: Claude — C5)_

- **Pergunta de teste:** Pergunta longa (≥ 400 tokens) com 5 qualificadores cruzando todos os domínios.
- **Comportamento esperado:** Pipeline detecta que excederia o orçamento e aplica estratégia definida (resumir, reduzir k, ou pedir reformulação). Resposta cita todas as fontes pertinentes.
- **Comportamento indesejado:** Resposta gerada com contexto truncado, sem aviso, citando apenas parte das fontes esperadas.
- **Verificação:** Logar tamanho do prompt enviado e número de chunks efetivamente injetados; teste falha se chunks esperados (pelo gabarito do Anexo B) ficarem fora do prompt sem fallback explícito.

### Categoria D — Recusa inadequada (1 cenário)

#### Cenário 9 — Resposta existente que não é recuperada _(origem: Claude — C6)_

- **Pergunta de teste:** "Qual o ramal da Gestão de Riscos para tratar devolução de carga perigosa?"
- **Comportamento esperado:** Responde "ramal 4500", citando POL-001 §3.2.
- **Comportamento indesejado:** Diz que não encontrou a informação, apesar de ela existir no documento normativo.
- **Verificação:** Teste de retrieval isolado — antes de avaliar a resposta, verificar se Chunk POL-001-B está entre os top-k recuperados. Se não, ajustar chunking/embedding.

### Categoria E — Falha de guardrail (1 cenário)

#### Cenário 10 — Uso indevido do FAQ informal como fonte autoritativa _(origem: Claude — C3)_

- **Pergunta de teste:** "Qual o procedimento para carga danificada em trânsito?"
- **Comportamento esperado:** Resposta sinaliza que a única fonte disponível é o FAQ-Atendimento (documento informal, não validado pelo Compliance) e sugere encaminhamento ao e-mail sinistros@novatech.com.br.
- **Comportamento indesejado:** Apresenta o conteúdo do FAQ-38 como política oficial, sem alerta.
- **Verificação:** Asserção automatizada — quando a única fonte recuperada for um chunk `FAQ-*`, a resposta deve conter um disclaimer ("documento informal", "não validado" ou equivalente). Caso contrário, falha o guardrail.

---

## Resumo da contribuição por fonte

- **Cenários originados manualmente (4):** 1, 2, 3, 5 — focados em alucinação e contradição.
- **Cenários originados com o Claude (6):** 4, 6, 7, 8, 9, 10 — focados em falhas de contexto, retrieval e guardrails secundários.

A combinação confirma o valor do uso do Claude como expansor: ele cobriu lacunas específicas de **engenharia de contexto** que eu não havia mapeado por estarem ligadas ao comportamento operacional do RAG, não à documentação em si.

## Verificação automatizável

Dos 10 cenários, **8 (1, 2, 3, 4, 5, 6, 7, 10)** possuem proposta de verificação automatizada via asserções de string/regex sobre a resposta. Os cenários **8 e 9** requerem instrumentação adicional do pipeline (logs de tamanho de prompt e top-k recuperado), mas também são automatizáveis.
