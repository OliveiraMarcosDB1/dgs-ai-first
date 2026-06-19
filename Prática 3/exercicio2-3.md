# Exercício 2.3 — QA · Skill de geração de testes (`create-integration-test`)

**Papel:** QA
**Ferramentas utilizadas:** Claude (chat) + Claude Cowork
**Entregável:** `SKILL.md` da skill Artifact `create-integration-test` + checklist de revisão verificável em < 2 min/teste (gerado no Cowork).

> **Caminho no repositório:** `skills/artifact/create-integration-test.md`.

---

## Parte 1 — `SKILL.md`

```markdown
---
name: create-integration-test
level: artifact
description: |
  Gere um teste de integração Vitest+MSW para um endpoint ou serviço do
  NovaTech Assistant. Ative esta skill quando o usuário pedir "criar
  teste de integração", "testar endpoint", "gerar test case para o
  query/feedback/health", ou ao implementar uma task cujo critério de
  aceite mencione "teste de integração".
owner: QA
consumers:
  - Desenvolvedores (Copilot, Claude Code)
  - QA (review)
depends_on:
  - skills/foundation/typescript-conventions.md
  - skills/foundation/error-handling.md
  - skills/foundation/project-structure.md
  - skills/domain/testing-patterns.md
references:
  - AGENTS.md#testing-standards
  - AGENTS.md#product-rules--guardrails
  - tests/fixtures/queries.ts
  - tests/fixtures/chunks.ts
  - tests/fixtures/expected-responses.ts
  - tests/fixtures/msw/handlers.ts
---

# Skill — Create Integration Test

## 1. Quando usar esta skill

Ative esta skill **somente** quando todas as condições abaixo forem
verdadeiras:

- O artefato a gerar é um teste em `tests/integration/<endpoint>/`.
- O alvo do teste é um handler Azure Functions ou um serviço em
  `src/services/`.
- Existe (ou está sendo criado no mesmo PR) um cenário de teste
  rastreável a um VC do `test-plan.md` do módulo.

NÃO use esta skill para:

- Testes unitários puros (use `skills/domain/testing-patterns.md`
  diretamente).
- Testes E2E que consomem tokens reais.
- Snapshots de UI React (use `skills/artifact/create-react-card.md`).

## 2. Pré-leitura obrigatória pelo agente

Antes de gerar código, o agente DEVE ler nesta ordem:

1. `AGENTS.md` — seções `Testing Standards` e `Product Rules & Guardrails`.
2. `skills/foundation/typescript-conventions.md` — strict mode, imports,
   naming.
3. `skills/domain/testing-patterns.md` — padrão Vitest/MSW.
4. O arquivo de `test-plan.md` do módulo (ex.:
   `specs/query-endpoint/test-plan.md`) — para identificar o `TC-XX.Y`
   correspondente.
5. `tests/fixtures/` — para reutilizar perguntas, chunks e respostas
   esperadas.

Se qualquer arquivo da lista 1–3 estiver ausente, **parar** e avisar:
não gerar teste sem o contrato do AGENTS.md.

## 3. Regras prescritivas

### 3.1. Localização e nomeação

- Caminho: `tests/integration/<endpoint>/<arquivo-descritivo>.test.ts`.
- Nome do arquivo: `kebab-case`, descreve o comportamento principal
  (ex.: `returns-source-document.test.ts`,
  `denies-hazmat-return.test.ts`).
- Um arquivo por comportamento. NÃO empilhar 20 cenários no mesmo arquivo.

### 3.2. Estrutura obrigatória

Todo teste DEVE seguir a estrutura abaixo, na ordem:

1. Imports (Vitest, MSW, handler sob teste, fixtures).
2. `describe('<EndpointOrService>')` único por arquivo.
3. `beforeEach` que sobe o MSW server com
   `onUnhandledRequest: 'error'`.
4. `afterEach` que executa `mswServer.resetHandlers()`,
   `mswServer.close()` e `vi.restoreAllMocks()`.
5. Um ou mais `it('should <behavior> when <condition>', async ...)`
   com as fases `// arrange`, `// act`, `// assert` explícitas.

### 3.3. Dados

- Perguntas: SEMPRE de `tests/fixtures/queries.ts`. Se faltar, criar
  a fixture **primeiro** e referenciá-la.
- Chunks: SEMPRE pelo identificador (`chunks.POL_001_A`,
  `chunks.PROC_042v2_B`, etc.). Nunca colar texto cru de chunk.
- Respostas esperadas: validar pelo objeto estruturado em
  `tests/fixtures/expected-responses.ts` (campos
  `source_document`, `confidence_level`, `must_contain_terms`,
  `must_not_contain_terms`, `max_response_time_ms`) e NÃO pela string
  literal gerada pelo LLM.

### 3.4. Assertions obrigatórias por categoria de teste

| Categoria | Assertions mínimas |
|---|---|
| Endpoint RAG | `status === 200` · `source_document` presente · `confidence_level` válido · `must_contain_terms` todos presentes · `must_not_contain_terms` todos ausentes. |
| Erro de validação | `status === 400` · `body.error === 'VALIDATION_ERROR'` · `body.details` cita o campo inválido. |
| Carga perigosa | `must_contain_terms ⊇ ['não é elegível', 'ramal 4500']` · `must_not_contain_terms ⊇ ['7 dias úteis']`. |
| Documento contraditório | resposta cita versão mais recente · termo da versão antiga ausente · `body.metadata.has_conflicting_sources === true`. |
| Sem cobertura | `body.answer === FALLBACK_MESSAGE` · `body.confidence_level === 'none'`. |
| Robustez (prompt injection) | resposta NÃO contém o payload injetado · resposta NÃO revela o system prompt. |
| Robustez (idioma) | `detectLanguage(body.answer) === 'pt-BR'`. |

### 3.5. Mocking

- HTTP externo: SEMPRE via MSW. Helpers já existentes:
  - `mockAiSearchHandler({ returnChunks })`
  - `mockOpenAiCompletionHandler({ completion, latencyMs })`
- Logger: deixar passar (pino com transport silencioso em
  `vitest.setup.ts`).
- Tempo: usar `vi.useFakeTimers()` quando testar timeouts/latência.

## 4. Template (preencher os placeholders `<...>`)

```ts
// tests/integration/<endpoint>/<comportamento>.test.ts
// VC: <VC-XX> · Test ID: <TC-XX.Y> · spec: specs/<modulo>/test-plan.md

import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { setupServer } from 'msw/node';
import { <handler> } from '../../../src/functions/<modulo>/handler';
import { buildQueryRequest } from '../../fixtures/queries';
import { chunks } from '../../fixtures/chunks';
import { expected } from '../../fixtures/expected-responses';
import {
  mockAiSearchHandler,
  mockOpenAiCompletionHandler,
} from '../../fixtures/msw/handlers';

const mswServer = setupServer();

describe('<EndpointName>', () => {
  beforeEach(() => {
    mswServer.listen({ onUnhandledRequest: 'error' });
  });

  afterEach(() => {
    mswServer.resetHandlers();
    mswServer.close();
    vi.restoreAllMocks();
  });

  it('should <behavior> when <condition>', async () => {
    // arrange
    const req = buildQueryRequest({ question: <fixture.question> });
    mswServer.use(
      mockAiSearchHandler({ returnChunks: [<chunks...>] }),
      mockOpenAiCompletionHandler({ completion: expected.<KEY>.text }),
    );

    // act
    const res = await <handler>(req);

    // assert
    expect(res.status).toBe(200);
    expect(res.body).toMatchObject({
      source_document: expect.arrayContaining(expected.<KEY>.source_document),
      confidence_level: expected.<KEY>.confidence_level,
    });
    expected.<KEY>.must_contain_terms.forEach((term) => {
      expect(res.body.answer).toContain(term);
    });
    expected.<KEY>.must_not_contain_terms.forEach((term) => {
      expect(res.body.answer).not.toContain(term);
    });
  });
});
```

## 5. Exemplos completos

### 5.1. DO — teste bem escrito (cobre VC-03, carga perigosa)

```ts
// tests/integration/query/denies-hazmat-return.test.ts
// VC: VC-03 · Test ID: TC-03.1 · spec: specs/query-endpoint/test-plan.md

import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { setupServer } from 'msw/node';
import { queryHandler } from '../../../src/functions/query/handler';
import { buildQueryRequest } from '../../fixtures/queries';
import { chunks } from '../../fixtures/chunks';
import { expected } from '../../fixtures/expected-responses';
import {
  mockAiSearchHandler,
  mockOpenAiCompletionHandler,
} from '../../fixtures/msw/handlers';

const mswServer = setupServer();

describe('QueryEndpoint — hazmat return', () => {
  beforeEach(() => {
    mswServer.listen({ onUnhandledRequest: 'error' });
  });

  afterEach(() => {
    mswServer.resetHandlers();
    mswServer.close();
    vi.restoreAllMocks();
  });

  it('should deny hazmat return and redirect to ramal 4500 when question is about returning hazardous cargo', async () => {
    // arrange
    const req = buildQueryRequest({ question: 'Posso devolver carga perigosa?' });
    mswServer.use(
      mockAiSearchHandler({
        returnChunks: [chunks.POL_001_B, chunks.FAQ_03, chunks.POL_001_A],
      }),
      mockOpenAiCompletionHandler({
        completion: expected.HAZMAT_RETURN_DENIED.text,
      }),
    );

    // act
    const res = await queryHandler(req);

    // assert
    expect(res.status).toBe(200);
    expect(res.body).toMatchObject({
      source_document: expect.arrayContaining(['POL-001-B']),
      confidence_level: 'high',
    });
    ['não é elegível', 'Gestão de Riscos', 'ramal 4500'].forEach((term) => {
      expect(res.body.answer).toContain(term);
    });
    ['7 dias úteis', 'Portal do Cliente'].forEach((term) => {
      expect(res.body.answer).not.toContain(term);
    });
  });
});
```

**Por que é bom:**

- Comentário de cabeçalho com VC + Test ID + spec → rastreabilidade.
- Um único `describe`, um único `it`, um único comportamento.
- `arrange/act/assert` separados.
- Fixtures do domínio NovaTech (`chunks.POL_001_B`, `expected.HAZMAT_RETURN_DENIED`).
- MSW com `onUnhandledRequest: 'error'` → nada vaza para a rede real.
- Valida o guardrail PS-001 (`source_document`) E o guardrail PS-002
  (negativa explícita para devolução de carga perigosa).
- Não acopla à string completa — valida termos contidos/não contidos.

### 5.2. DON'T — anti-padrões comuns gerados por LLM

```ts
// ❌ ANTI-PADRÃO: NÃO copiar, este código reprova no review de QA.
import { handler } from '../../../src/functions/query/handler';

test('hazmat return', async () => {
  const result = await handler({
    body: '{"question": "carga perigosa"}',
  });
  expect(result).toBeDefined();
  expect(result.status).toBeTruthy();
  // chama a API real do Azure OpenAI sem mock
  // (vai falhar em CI sem credenciais ou estourar custo se passar)
});
```

**Problemas (cada um isoladamente reprova no Gate 4):**

| # | Problema | Regra violada |
|---|---|---|
| 1 | Usa `test(...)` em vez de `describe('QueryEndpoint', () => it(...))`. | 3.2 |
| 2 | Título genérico, não descreve o `should ... when ...`. | 3.2 |
| 3 | Pergunta `"carga perigosa"` colada in-line em vez de fixture. | 3.3 |
| 4 | Nenhum mock de Azure AI Search ou OpenAI → chama serviço real. | 3.5 |
| 5 | `toBeDefined()` e `toBeTruthy()` como única assertion → passa para qualquer coisa, inclusive erro 500. | AGENTS.md → NÃO DEVE |
| 6 | Não valida `source_document` (guardrail PS-001). | 3.4 |
| 7 | Não valida a negativa explícita exigida pelo VC-03. | 3.4 |
| 8 | Sem rastreabilidade (não cita VC/Test ID/spec). | 3.1 |
| 9 | Sem `beforeEach`/`afterEach` → estado vaza entre testes. | 3.2 |

## 6. Anti-padrões específicos de IA (bloquear no review)

Padrões que LLMs geram com alta frequência e devem ser sinalizados:

- **Asserção otimista única**: `expect(result).toBeDefined()` ou
  `expect(() => fn()).not.toThrow()` como único guard.
- **String literal do LLM**: `expect(res.body.answer).toBe("O prazo de devolução é de 7 dias úteis ...")` → quebra a cada microajuste de prompt.
- **`await new Promise(r => setTimeout(r, 5000))`** para "esperar" serviço externo: deve usar fake timers e mocks.
- **Hardcode de chave de API ou endpoint do Azure** no teste.
- **`try { ... } catch { /* ignore */ }`** para mascarar erro.
- **Mock parcial**: mocka Azure AI Search mas chama OpenAI real (ou
  vice-versa) — vai vazar em CI.
- **Pergunta sintética**: `"test"`, `"hello"`, `"foo bar"`. Deve ser do
  domínio.
- **Confiança "alta" inventada**: o teste afirma `confidence_level: 'high'`
  para uma pergunta que o test-plan classifica como baixa confiança
  (FAQ-32, FAQ-38).
- **Esperar resposta sobre tier inexistente**: assertions que
  validariam SLA para "Platinum" — alucinação consagrada no teste.
- **Skip de teste injetificado**: `it.skip(...)` sem comentário com VC.

## 7. Dependências

- `skills/foundation/typescript-conventions.md` — strict mode, imports
  por path relativo até 3 níveis, tipos explícitos em parâmetros públicos.
- `skills/foundation/error-handling.md` — formato de erro padronizado
  (`{ error: string, details?: unknown }`).
- `skills/foundation/project-structure.md` — caminhos absolutos do
  repositório.
- `skills/domain/testing-patterns.md` — padrão Vitest + MSW + factories.

## 8. Critérios de maturidade da skill

A skill é considerada **madura** (pronta para uso pelo time) quando:

1. Foi exercitada em ≥ 5 endpoints diferentes do projeto.
2. Pelo menos 2 ciclos de iteração v1 → v2 com base em outputs reais do
   Copilot.
3. Checklist de revisão (parte 2) executa em < 2 min/teste sem ambiguidade.
4. Taxa de aprovação no primeiro review de QA ≥ 70% para testes
   gerados via esta skill.
```

---

## Parte 2 — Checklist de revisão de testes (Claude Cowork)

> Gerado no Cowork e formatado para uso direto pelo QA em PR. Cada item
> é objetivo (sim/não) e o checklist completo é executável em menos de
> 2 minutos por teste.

### Checklist — Code Review de Teste (alvo: < 2 min/teste)

**Bloco A — Rastreabilidade (15s)**

- [ ] Cabeçalho cita VC e Test ID (`// VC: VC-XX · Test ID: TC-XX.Y`).
- [ ] O Test ID existe no `test-plan.md` do módulo correspondente.

**Bloco B — Estrutura (20s)**

- [ ] Um `describe` por arquivo, um `it` por comportamento.
- [ ] Título do `it` segue `should <behavior> when <condition>` em inglês.
- [ ] Fases `// arrange`, `// act`, `// assert` presentes e separadas.
- [ ] `beforeEach` sobe MSW com `onUnhandledRequest: 'error'`.
- [ ] `afterEach` reseta handlers, fecha server, restaura mocks.

**Bloco C — Dados (25s)**

- [ ] Pergunta vem de `tests/fixtures/queries.ts` (não inline e não genérica).
- [ ] Chunks vêm por identificador de `tests/fixtures/chunks.ts`.
- [ ] Resposta esperada vem de `tests/fixtures/expected-responses.ts`.
- [ ] Se faltar fixture, foi criada **no mesmo PR**.

**Bloco D — Mocking (15s)**

- [ ] Azure AI Search está mockado via MSW.
- [ ] Azure OpenAI está mockado via MSW.
- [ ] Nenhuma URL externa real aparece no teste.
- [ ] Nenhuma chave/token hardcoded.

**Bloco E — Assertions (25s)**

- [ ] Pelo menos uma assertion específica ao comportamento sob teste.
- [ ] Nenhuma assertion única do tipo `toBeDefined`, `toBeTruthy`,
      `toBeFalsy`, `not.toThrow`.
- [ ] Validação do guardrail `source_document` quando aplicável.
- [ ] `must_contain_terms` e `must_not_contain_terms` aplicados quando
      aplicável.
- [ ] Não há `expect(body.answer).toBe("<string literal do LLM>")`.

**Bloco F — Guardrails de produto (15s)**

- [ ] Carga perigosa → testa negativa explícita + ramal 4500.
- [ ] Documento contraditório → testa priorização da versão mais recente.
- [ ] Pergunta sem cobertura → testa fallback.
- [ ] Tier Platinum → testa negativa baseada em SLA-2024-A.

**Bloco G — Higiene (5s)**

- [ ] Sem `console.log` deixado.
- [ ] Sem `it.only` / `it.skip` sem justificativa.
- [ ] Sem `try/catch` vazio.
- [ ] Lint e typecheck verdes (verificado pelo CI antes do review humano).

**Decisão final (em 2 min):**

- ✅ Todos os blocos aplicáveis ✔ → **aprovado**.
- ⚠️ Até 2 itens não conformes em blocos B/C/G → **aprovado com
  comentário** (autor corrige antes do merge).
- ❌ Qualquer item não conforme em blocos A/D/E/F → **reprovado**, autor
  deve refazer.

---

## Evidência de uso das ferramentas

**Claude (chat) — prompts principais:**

1. *"Aja como QA sênior. Crie o `SKILL.md` da skill Artifact
   `create-integration-test` para um projeto TypeScript com Vitest + MSW,
   seguindo as convenções de Testing Standards e os guardrails de produto
   do NovaTech Assistant. Inclua frase-ativação, template com
   placeholders, exemplo DO e exemplo DON'T com problemas reais que LLMs
   geram, e lista de anti-padrões específicos de IA."*
2. *"Quais anti-padrões testes gerados por LLM costumam apresentar em
   código real? Liste apenas os que vi acontecer de fato, sem inventar."*
3. *"Defina critérios objetivos de maturidade da skill — métricas que o
   tech lead consegue medir."*

**Claude Cowork — prompts principais:**

1. *"Quero um checklist de code review de teste de integração que um QA
   executa em menos de 2 minutos. Cada item deve ser sim/não, sem
   ambiguidade. Agrupe em blocos (rastreabilidade, estrutura, dados,
   mocking, assertions, guardrails, higiene) e estime tempo por bloco."*
2. *"Adicione uma regra de decisão final (aprovado / aprovado com
   comentário / reprovado) baseada em quais blocos têm itens não
   conformes."*

Iteração-chave: a primeira versão do checklist tinha 35 itens em uma
única lista; foi reorganizada em 7 blocos cronometrados após teste
real (executar em < 2 min era impossível na versão flat).
