# Exercício 3.2 — QA · Revisão crítica dos testes gerados por IA

**Papel:** QA
**Tópico:** Revisão Crítica de Outputs de IA
**Ferramentas utilizadas:** Claude (chat)
**Entregável:** Revisão própria dos 3 testes, segunda revisão via Claude, comparação, e Teste 1 reescrito.

> **Contexto chave:** o projeto usa **Vitest + MSW** (Anexo C e `AGENTS.md → Testing Standards`). Os 3 testes simulados usam `jest`, `describe/it` no estilo Jest e `jest.fn()` — primeiro sinal de inconsistência com o stack autorizado.

---

## 1. Revisão própria (QA) — antes do Claude

### Observação transversal aos 3 testes (problema de contexto)

Os 3 trechos usam `jest.fn()` e estilo Jest, mas o `AGENTS.md → Testing Standards (QA)` (cenário 2.1) declara explicitamente:

> _"Framework: **Vitest** (unit + integration). Não usar Jest, Mocha ou outros."_

E a estrutura do repositório (Anexo C) tem `vitest.config.ts` no root e `tests/integration/` como pasta de destino. Qualquer um dos 3 testes seria **rejeitado no Gate 4 de QA** antes mesmo da análise de assertions — viola o stack autorizado.

Isso é classificado como **violação do AGENTS.md** e é o primeiro item da minha revisão.

---

### Teste 1 — `query endpoint` / assertions vagas

```ts
describe("query endpoint", () => {
  it("should return a response", async () => {
    const res = await request(app)
      .post("/api/query")
      .send({ question: "prazo devolução" });
    expect(res.status).toBe(200);
    expect(res.body).toBeDefined();
  });
});
```

**O que o teste pretende testar:** que o endpoint `/api/query` responde HTTP 200 com algum corpo.

**O que ele falha em testar:**

| #   | Problema                                                                                                                                                                    | Classificação                                            |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 1.1 | `expect(res.body).toBeDefined()` passa para **qualquer** corpo — inclusive um erro 200 com `{}`. Não valida nada do domínio.                                                | Anti-padrão de teste (regra `NÃO DEVE` do AGENTS.md)     |
| 1.2 | Não valida o campo `source_document` (guardrail PS-001 — _toda resposta DEVE conter fonte_).                                                                                | Violação de guardrail de produto                         |
| 1.3 | Não valida `confidence_level`, nem qualquer campo do structured output (exercício Dev 3.1).                                                                                 | Cobertura incompleta                                     |
| 1.4 | Pergunta `"prazo devolução"` inline em vez de fixture (`tests/fixtures/queries.ts`).                                                                                        | Violação do AGENTS.md → Padrão de fixtures               |
| 1.5 | Nenhum mock de Azure AI Search nem de Azure OpenAI. Em CI sem credenciais o teste quebra; em CI com credenciais o teste estoura custo de token.                             | Violação do AGENTS.md → "NÃO DEVE chamar serviços reais" |
| 1.6 | Sem `// arrange`, `// act`, `// assert`.                                                                                                                                    | Violação da estrutura obrigatória                        |
| 1.7 | Sem `beforeEach`/`afterEach` — estado vaza entre testes.                                                                                                                    | Independência                                            |
| 1.8 | Sem rastreabilidade — não cita VC do test-plan.                                                                                                                             | Rastreabilidade                                          |
| 1.9 | Usa `request(app)` (estilo supertest) mas o handler do projeto é Azure Function (`HttpRequest → HttpResponseInit`), não Express. Provavelmente nem compila no projeto real. | Bug potencial / inconsistência com stack                 |

**Risco se o teste "passar":** o teste fica verde mesmo se o endpoint responder em inglês, sem citar fonte, com resposta vazia, ou contradizendo POL-001. Resultado: **falsa sensação de segurança no go-live**.

---

### Teste 2 — `edge case` com dados irreais

```ts
describe("query endpoint edge cases", () => {
  it("should handle empty question", async () => {
    const res = await request(app).post("/api/query").send({ question: "" });
    expect(res.status).toBe(400);
  });
});
```

**O que o teste pretende testar:** que pergunta vazia retorna 400.

**O que ele falha em testar:**

| #   | Problema                                                                                                                                                                                      | Classificação                                                                               |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 2.1 | Único edge case coberto é "input vazio" — não exercita nenhuma regra de domínio (carga perigosa, contradição de documentos, tier inexistente, idioma).                                        | Cobertura insuficiente — viola os "Testes obrigatórios para o pipeline de RAG" do AGENTS.md |
| 2.2 | Não valida `body.error`, `body.details` ou o nome do campo inválido — atendente que receber esse 400 não vai saber por quê.                                                                   | Assertion superficial                                                                       |
| 2.3 | Nenhum mock — depende do código real interpretar `""` como inválido antes de chamar Search/OpenAI. Se a validação for movida, o teste passa ou quebra por motivo errado.                      | Acoplamento frágil                                                                          |
| 2.4 | Mesmos problemas transversais do Teste 1 (Jest, sem fixtures, sem rastreabilidade, sem arrange/act/assert).                                                                                   | Violação do AGENTS.md                                                                       |
| 2.5 | "Edge case" rotulado mas testa apenas o caso mais trivial. Edge cases reais do domínio (PROC-042 vs PROC-042-v2, tier Platinum, query multi-domínio, payload de 50k chars) ficam descobertos. | Falso sinal de cobertura                                                                    |

**Risco se o teste "passar":** o time acredita que tem "cobertura de edge case" mas só está validando input syntático. Bugs de domínio (priorização da v2, negativa de hazmat, idioma) seguem para produção.

---

### Teste 3 — `feedback endpoint` com mock permissivo

```ts
describe("feedback endpoint", () => {
  it("should save feedback", async () => {
    const mockCreate = jest.fn().mockResolvedValue({ id: "123" });
    const res = await request(app).post("/api/feedback").send({
      queryId: "q1",
      rating: 5,
      comment: "great",
    });
    expect(res.status).toBe(200);
    expect(mockCreate).toHaveBeenCalled();
  });
});
```

**O que o teste pretende testar:** que ao enviar feedback, o handler chama o Cosmos e retorna 200.

**O que ele falha em testar:**

| #   | Problema                                                                                                                                                                                                                                                                                                                                | Classificação                                               |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| 3.1 | **O `mockCreate` é declarado mas nunca conectado ao container do Cosmos** (não há `vi.mock`, `jest.mock` ou spy sobre o módulo `@azure/cosmos`). A asserção `toHaveBeenCalled` valida que **um mock fictício foi chamado** — mas o handler real chama outra instância. O teste passaria mesmo se o handler **nunca chamasse o Cosmos**. | **Bug perigoso de mock** (regra `NÃO DEVE silenciar erros`) |
| 3.2 | Não valida que `queryId`, `rating`, `comment`, `attendantEmail` chegaram corretamente ao Cosmos (`expect(mockCreate).toHaveBeenCalledWith(...)`).                                                                                                                                                                                       | Assertion insuficiente                                      |
| 3.3 | Não valida que `attendantEmail` **não foi logado** (regra do AGENTS.md: dados pessoais não logados). Esse é o bug exato do exercício Dev 3.2.                                                                                                                                                                                           | Lacuna de segurança no teste                                |
| 3.4 | Não valida tratamento de erro: rating fora de 1–5, comment > limite, Cosmos indisponível.                                                                                                                                                                                                                                               | Cobertura insuficiente                                      |
| 3.5 | Não valida o schema Zod de input (que segundo o AGENTS.md deveria existir).                                                                                                                                                                                                                                                             | Violação do AGENTS.md                                       |
| 3.6 | Comentário `'great'` e dados `q1/5/great` são genéricos — não exercitam o domínio (feedback real é sobre uma resposta com pergunta de carga perigosa, SLA Gold, etc.).                                                                                                                                                                  | Padrão de fixtures                                          |

**Risco se o teste "passar":** o teste verde dá luz verde para um handler que pode **não estar persistindo o feedback** (o mock dummy é satisfeito), **pode estar logando dado pessoal** (não há assertion), e **pode aceitar `rating: 100`** (sem validação de schema). Combinado, é o pior tipo de falso positivo: o teste alimenta a métrica de coverage sem testar nada real.

---

## 2. Revisão do Claude — segunda passada

> Prompt usado no Claude (chat): _"Atue como QA sênior. Os 3 testes abaixo foram gerados pelo Copilot para um projeto que usa Vitest + MSW (não Jest), Azure Functions (não Express), Zod para validação, pino para log, e que tem guardrails de produto formalizados (source_document obrigatório, negativa para devolução de carga perigosa, nunca logar dados pessoais). Para cada teste, liste: o que ele pretende testar, o que ele falha em testar, e o risco se passar mesmo com código errado. Seja específico e não invente problemas. AGENTS.md em anexo."_

### Síntese dos achados do Claude

| Teste           | Achados do Claude                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Convergência com QA              |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| **Transversal** | Identifica imediatamente o uso de Jest em projeto Vitest e o `request(app)` incompatível com Azure Functions.                                                                                                                                                                                                                                                                                                                                                          | ✅ Idêntico ao QA                |
| **Teste 1**     | (a) `toBeDefined()` é "useless assertion" — concorda com 1.1. (b) Falta de mock vai estourar custo ou quebrar CI — concorda com 1.5. (c) Não valida `source_document` — concorda com 1.2. (d) **Adicional:** sugere que a pergunta `"prazo devolução"` (sem artigo, sem capitalização) é provavelmente diferente do que o usuário real digita — pode mascarar bug de normalização de input.                                                                            | ✅ Convergente + 1 achado novo   |
| **Teste 2**     | (a) Único edge case = vazio é insuficiente — concorda com 2.1. (b) Falta validar `body.error`/`details` — concorda com 2.2. (c) **Adicional:** aponta que um teste só de "string vazia" não diferencia validação Zod de check manual `if(!question)` — se a regra de validação evoluir, o teste continua verde mas não cobre os novos campos.                                                                                                                          | ✅ Convergente + 1 achado novo   |
| **Teste 3**     | (a) `mockCreate` órfão (não conectado) é o problema central — concorda com 3.1. (b) `toHaveBeenCalledWith` ausente — concorda com 3.2. (c) **Adicional:** observa que o teste **não confere o status code de erro** (espera 200 mas se o handler retornar 500 com mock dummy chamado, o teste falharia por outro motivo — confuso). (d) **Adicional:** sugere validar que o `timestamp` gerado pelo handler é estável (Date determinístico em teste, com fake timers). | ✅ Convergente + 2 achados novos |

### Achados que o Claude trouxe e o QA não tinha listado

1. **Teste 1:** normalização de input — pergunta sem artigo/capitalização pode mascarar bug de pré-processamento.
2. **Teste 2:** o teste não diferencia origem da validação (Zod vs. check manual), o que prejudica refatoração futura.
3. **Teste 3:** falta `vi.useFakeTimers()` para tornar `timestamp` determinístico, e o teste mistura "espero 200" com mock que pode permitir 500 silenciosamente.

### Achados que o QA trouxe e o Claude não enfatizou

1. **Teste 3:** O Claude focou no "mock órfão" mas **não destacou explicitamente a lacuna de segurança** sobre `attendantEmail` logado (item 3.3) — esse é o ponto que conecta diretamente ao exercício Dev 3.2 (revisão do feedback-handler) e ao AGENTS.md de coding (`Nunca logar dados pessoais`).
2. **Transversal:** o Claude apontou Jest, mas o QA também fez a leitura do **risco institucional** (Gate 4 de QA rejeita antes da análise técnica). Claude focou no técnico, QA focou no processo.

---

## 3. Comparação QA × Claude

| Dimensão                          | QA                                                                              | Claude                                                                                    | Avaliação honesta                                    |
| --------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| **Velocidade**                    | 20–25 min para listar e classificar todos os problemas.                         | < 30s para a primeira passada.                                                            | Claude é imbatível em recall inicial.                |
| **Profundidade técnica**          | Identifica todos os problemas estruturais (Jest, supertest, ausência de mock).  | Idem + 3 achados adicionais (normalização, origem da validação, fake timers).             | Claude levemente mais profundo em detalhes técnicos. |
| **Conexão com processo**          | Liga cada problema a uma regra do AGENTS.md / VC / guardrail e ao Gate 4 de QA. | Liga ao AGENTS.md quando solicitado, mas tende a tratar problemas como técnicos isolados. | QA humano mais forte.                                |
| **Foco em segurança/privacidade** | Destaca o `attendantEmail` como lacuna do teste (conecta a Dev 3.2).            | Mencionou de passagem; não enfatizou.                                                     | QA humano mais forte.                                |
| **Recall**                        | 9 achados Teste 1 / 5 Teste 2 / 6 Teste 3.                                      | Cobre todos os do QA + 3 novos.                                                           | Claude maior recall técnico.                         |
| **Precisão**                      | Sem falsos positivos.                                                           | Sem falsos positivos.                                                                     | Empate.                                              |

**Conclusão honesta:** Claude é claramente mais rápido e tem maior recall técnico. O revisor humano (QA) entrega: (a) conexão com processo e gates, (b) sensibilidade a riscos institucionais (privacidade, conformidade com AGENTS.md), e (c) priorização — decidir o que reprova merge vs. o que vira ticket. O workflow ótimo é o Claude como **primeira passada exaustiva** e o QA como **segunda passada de priorização e ligação ao processo**.

---

## 4. Teste 1 reescrito (Vitest + MSW + fixtures + guardrails)

```ts
// tests/integration/query/returns-source-and-deadline.test.ts
// VC: VC-02 (source_document obrigatório) + VC-04 (cobertura plena) · Test ID: TC-02.1
// spec: specs/query-endpoint/test-plan.md

import { describe, it, expect, beforeEach, afterEach, vi } from "vitest";
import { setupServer } from "msw/node";
import { queryHandler } from "../../../src/functions/query/handler";
import { buildQueryRequest } from "../../fixtures/queries";
import { chunks } from "../../fixtures/chunks";
import { expected } from "../../fixtures/expected-responses";
import {
  mockAiSearchHandler,
  mockOpenAiCompletionHandler,
} from "../../fixtures/msw/handlers";

const mswServer = setupServer();

describe("QueryEndpoint — return deadline (POL-001)", () => {
  beforeEach(() => {
    mswServer.listen({ onUnhandledRequest: "error" });
  });

  afterEach(() => {
    mswServer.resetHandlers();
    mswServer.close();
    vi.restoreAllMocks();
  });

  it("should return 200 with source_document and the 7-business-day deadline when asked about standard return policy", async () => {
    // arrange
    const req = buildQueryRequest({
      question: "Qual o prazo de devolução para produtos standard?",
    });
    mswServer.use(
      mockAiSearchHandler({ returnChunks: [chunks.POL_001_A] }),
      mockOpenAiCompletionHandler({
        completion: expected.RETURN_DEADLINE.text,
      }),
    );

    // act
    const res = await queryHandler(req);

    // assert — structured output válido
    expect(res.status).toBe(200);
    expect(res.body).toMatchObject({
      source_document: expect.arrayContaining(["POL-001-A"]),
      confidence_level: "high",
    });

    // assert — guardrail PS-001 (fonte sempre presente, nunca array vazio para confiança alta)
    expect(res.body.source_document).not.toHaveLength(0);

    // assert — conteúdo factual (termos obrigatórios das fixtures)
    expected.RETURN_DEADLINE.must_contain_terms.forEach((term) => {
      expect(res.body.answer).toContain(term);
    });
    // Exemplo de must_contain_terms: ['7 (sete) dias úteis', 'Portal do Cliente']

    // assert — guardrail PS-003 (idioma)
    expect(res.body.answer).toMatch(/[áéíóúçãõ]|prazo|devolução/i);

    // assert — não vaza estado nem campos extras do schema Zod
    expect(Object.keys(res.body).sort()).toEqual(
      ["answer", "confidence_level", "source_document"].sort(),
    );
  });
});
```

### Por que essa versão passa no Gate 4 de QA

| Critério do AGENTS.md                        | Como o teste reescrito atende                                                                                                              |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Framework Vitest**                         | `import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest'`                                                                 |
| **MSW para HTTP externo**                    | `setupServer` + `mockAiSearchHandler` + `mockOpenAiCompletionHandler` com `onUnhandledRequest: 'error'` — quebra se algo vazar para a rede |
| **Fixtures do domínio**                      | `buildQueryRequest`, `chunks.POL_001_A`, `expected.RETURN_DEADLINE` — zero string inventada                                                |
| **Nomenclatura `should ... when ...`**       | `should return 200 with source_document ... when asked about standard return policy`                                                       |
| **Arrange / Act / Assert explícitos**        | Comentários `// arrange`, `// act`, `// assert` separam as fases                                                                           |
| **Independência (`afterEach`)**              | `resetHandlers()` + `close()` + `vi.restoreAllMocks()`                                                                                     |
| **Assertion específica (não `toBeDefined`)** | Valida status, `source_document`, `confidence_level`, termos obrigatórios e shape exato do schema                                          |
| **Cobre guardrail PS-001**                   | `expect(res.body.source_document).not.toHaveLength(0)`                                                                                     |
| **Cobre guardrail PS-003**                   | regex que valida marca de português na resposta                                                                                            |
| **Valida structured output (Dev 3.1)**       | `Object.keys(res.body).sort()` confere que o schema retornado tem só os 3 campos esperados — nenhum vazamento de campo extra               |
| **Rastreabilidade**                          | Cabeçalho com `VC-02 + VC-04 + TC-02.1` e referência ao `test-plan.md`                                                                     |
| **Não acopla à string do LLM**               | Usa `must_contain_terms` em vez de `toBe("texto completo")` — resistente a microajustes de prompt                                          |

---

## 5. Evidência de uso da ferramenta

**Claude (chat) — prompts principais:**

1. _"Atue como QA sênior. Para cada um dos 3 testes abaixo, liste: (a) o que ele pretende testar; (b) o que falha em testar; (c) o risco se passar mesmo com código errado. Contexto: Vitest+MSW (não Jest), Azure Functions (não Express), guardrails PS-001 (source_document), PS-002 (negativa hazmat), PS-003 (PT-BR), AGENTS.md proíbe `toBeDefined` solo e log de dados pessoais."_
2. _"Critique a minha lista de achados (anexa). O que eu deixei passar? O que eu marquei como problema sem ser de fato? Seja honesto."_
3. _"Reescreva o Teste 1 numa versão que (i) usa Vitest+MSW, (ii) referencia fixtures do domínio NovaTech, (iii) valida o guardrail PS-001, (iv) inclui rastreabilidade por VC, (v) não acopla à string literal do LLM, (vi) checka o shape do structured output."_

Iteração-chave: a primeira reescrita do Teste 1 dada pelo Claude omitia o teste do shape do structured output (`Object.keys(res.body).sort()`). Adicionei essa assertion no segundo passe porque a função do harness é exatamente garantir que o schema Zod do exercício Dev 3.1 não esteja deixando passar campos extras silenciosamente.
