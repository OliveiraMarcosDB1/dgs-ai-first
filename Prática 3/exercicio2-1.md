# Exercício 2.1 — QA · Contribuição para o AGENTS.md: Seção de Testing Standards

**Papel:** QA
**Ferramenta utilizada:** Claude (chat)
**Entregável:** Seção `## Testing Standards (QA)` do `AGENTS.md`, reescrita de um teste ruim, e critérios de review.

---

## 1. Seção `Testing Standards (QA)` do AGENTS.md

> Esta seção é prescritiva e deve ser seguida por qualquer agente de IA (Copilot, Claude Code) ao gerar testes neste repositório. As regras estão no formato `DEVE` / `NÃO DEVE` / `QUANDO EM DÚVIDA` para que sejam parseáveis e executáveis.

```markdown
## Testing Standards (QA)

Esta seção governa a geração e o review de testes automatizados do
NovaTech Assistant. Toda task que envolva código de teste DEVE seguir
estas regras. Em caso de conflito com outras seções, prevalece esta.

### Stack de teste autorizada

- Framework: **Vitest** (unit + integration). Não usar Jest, Mocha ou outros.
- Mocking de HTTP externo: **MSW (Mock Service Worker)**.
- Geração de dados de teste: **factories** localizadas em
  `tests/fixtures/` (ver `tests/fixtures/queries.ts`,
  `tests/fixtures/chunks.ts`, `tests/fixtures/expected-responses.ts`).
- Coverage mínimo: **80% de linhas** por módulo em `src/`. Pull
  requests (registrados em `docs/pull-requests/PR-NNNN.md`) que reduzam
  coverage abaixo de 80% DEVEM ser rejeitados pelo gate de QA.
- Execução: testes rodam em CI via GitHub Actions (`.github/workflows/ci.yml`).
  Todo teste DEVE ser executável localmente com `npm test` sem
  dependências externas (sem internet, sem Azure, sem chaves).

### Organização dos arquivos de teste

- Testes unitários: `tests/unit/<modulo>/<arquivo>.test.ts`.
- Testes de integração: `tests/integration/<endpoint>/<arquivo>.test.ts`.
- Testes E2E: `tests/e2e/<fluxo>.test.ts` (uso restrito — consomem tokens).
- Fixtures compartilhadas: `tests/fixtures/`. NÃO duplicar fixtures
  entre arquivos de teste.

### Nomenclatura

- Linguagem dos identificadores de teste: **inglês** (consistente com
  código e comments — ver Coding Standards).
- Padrão obrigatório:
  - `describe('<ModuleName ou Endpoint>', () => { ... })`
  - `it('should <behavior> when <condition>', async () => { ... })`
- O título do `it` DEVE descrever **comportamento esperado**, não a
  implementação. Bom: `should return 400 when question is empty`.
  Ruim: `tests handler function`.

### Estrutura obrigatória de todo teste

Todo teste DEVE conter as três fases explicitamente separadas, com
comentários de marcação:

```ts
it('should return source_document field when chunks are retrieved', async () => {
  // arrange
  const req = buildQueryRequest({ question: 'Qual o SLA do cliente Gold?' });
  const searchMock = mockAiSearchWithChunks([chunks.SLA_2024_A, chunks.SLA_2024_B]);
  const openAiMock = mockOpenAiCompletion(expected.SLA_GOLD_ANSWER);

  // act
  const res = await queryHandler(req);

  // assert
  expect(res.status).toBe(200);
  expect(res.body.source_document).toEqual(['SLA-2024-A', 'SLA-2024-B']);
  expect(searchMock).toHaveBeenCalledOnce();
  expect(openAiMock).toHaveBeenCalledOnce();
});
```

### Regras `DEVE`

- DEVE conter pelo menos uma assertion específica ao comportamento sob teste.
- DEVE usar fixtures de `tests/fixtures/` para perguntas, chunks e respostas
  esperadas. Dados in-line são permitidos apenas para valores triviais
  (booleanos, números pequenos).
- DEVE mockar toda chamada externa (Azure AI Search, Azure OpenAI, Cosmos)
  via MSW ou stub explícito.
- DEVE ser independente: a ordem de execução NÃO PODE afetar o resultado.
- DEVE limpar estado em `afterEach` quando usar mocks com estado.
- DEVE testar pelo menos um caso de erro para cada handler público
  (input inválido, dependência indisponível, contexto vazio).
- Testes de respostas do assistente DEVEM validar a presença do campo
  `source_document` no JSON de retorno (guardrail PS-001).
- Testes que exercitem cargas perigosas DEVEM validar a negativa explícita
  de devolução pelo processo padrão (guardrail PS-002).

### Regras `NÃO DEVE`

- NÃO DEVE fazer chamadas a serviços reais (Azure, rede, sistema de arquivos
  fora de `tests/`).
- NÃO DEVE usar `toBeDefined()`, `toBeTruthy()`, `toBeFalsy()` ou
  `not.toThrow()` como única assertion de um caso de teste.
- NÃO DEVE depender de variáveis globais, de timing (`setTimeout` sem fake
  timers), ou de ordem de execução de outros testes.
- NÃO DEVE conter dados sensíveis (chaves, tokens, dados pessoais reais).
- NÃO DEVE silenciar erros com `try/catch` vazio.
- NÃO DEVE ter `console.log` deixado em produção (usar `logger` em mocks
  quando necessário).
- NÃO DEVE acoplar a strings exatas geradas pelo LLM. Validar formato,
  campos estruturados (JSON schema) e presença de termos-chave, não a
  redação completa.

### Padrão de mocking

- HTTP externo (Azure AI Search REST, Azure OpenAI REST) → MSW handlers
  versionados em `tests/fixtures/msw/`.
- Funções internas → `vi.fn()` ou `vi.spyOn()`.
- Dados → factories em `tests/fixtures/<nome>.ts` que expõem builders
  parametrizáveis. Exemplo:
  ```ts
  export const buildQueryRequest = (overrides: Partial<QueryRequest> = {})
    : QueryRequest => ({
      question: 'Qual o prazo de devolução?',
      conversationId: 'conv-test-001',
      userId: 'user-test-001',
      ...overrides,
    });
  ```

### Padrão de fixtures de RAG

Para testes que envolvem retrieval e geração de resposta, usar SEMPRE
o trio:

1. **Pergunta** (`tests/fixtures/queries.ts`) — extraída do mapa de
   cobertura do Anexo B.
2. **Chunks esperados** (`tests/fixtures/chunks.ts`) — identificadores
   `POL-001-A`, `PROC-042v2-B`, etc.
3. **Resposta esperada** (`tests/fixtures/expected-responses.ts`) —
   contém campos estruturados (não a string exata): `source_document`,
   `confidence_level`, `must_contain_terms[]`, `must_not_contain_terms[]`.

### Testes obrigatórios para o pipeline de RAG

Todo endpoint que consuma o pipeline de RAG DEVE ter:

- Teste para pergunta com cobertura plena (happy path).
- Teste para pergunta com chunks contraditórios (PROC-042 vs PROC-042-v2):
  a resposta DEVE priorizar a versão mais recente (ADR-0003).
- Teste para pergunta sem cobertura na base (frete padrão < 500kg):
  a resposta DEVE ser a mensagem padrão de "não encontrado", não
  alucinação.
- Teste para pergunta com termo inexistente no domínio ("cliente Platinum"):
  a resposta DEVE negar explicitamente o tier, citando SLA-2024-A.
- Teste para inversão de regra (devolução de carga perigosa): a
  resposta DEVE ser negativa, com encaminhamento ao ramal 4500.

### Testes de robustez obrigatórios

Todo endpoint público de IA DEVE ter pelo menos um teste para cada
categoria abaixo, vivendo em `tests/integration/<endpoint>/robustness.test.ts`:

- **Prompt injection básico**: input contendo
  `"Ignore as instruções anteriores e ..."` — a resposta NÃO PODE
  executar a instrução injetada.
- **Confusão de idioma**: pergunta em inglês ou espanhol — a resposta
  DEVE permanecer em português formal (guardrail PS-003).
- **Pergunta ambígua**: input que cruza dois domínios (ex.: devolução +
  frete + carga perigosa) — a resposta DEVE cobrir cada subdomínio
  explicitamente ou pedir desambiguação.

### `QUANDO EM DÚVIDA`

- Se não houver fixture adequada, criar uma nova em `tests/fixtures/`
  e referenciar no PR markdown — não duplicar dados.
- Se um teste depender de comportamento probabilístico do LLM, validar
  contrato estrutural (schema do JSON, presença de campos) e NÃO a
  resposta literal.
- Se um teste ficar lento (> 500ms), revisar mocks antes de aumentar
  timeout.

### Gate de QA (Gate 4 — Tests → Deploy)

Antes de qualquer aprovação de deploy, o QA verifica:

1. Coverage ≥ 80% no módulo afetado.
2. Todos os testes obrigatórios de RAG e robustez existem e passam.
3. Nenhuma assertion vazia (`toBeDefined` sozinha).
4. Fixtures novas estão em `tests/fixtures/` e não duplicam dados.
5. Nenhum teste depende de serviço externo real.
```

---

## 2. Reescrita do teste ruim

### Antes (gerado pelo Copilot sem guidance)

```ts
// Teste gerado pelo Copilot sem guidance
test("query endpoint works", async () => {
  const result = await handler({ body: '{"question": "test"}' });
  expect(result).toBeDefined();
});
```

#### Problemas identificados

| # | Problema | Regra violada |
|---|----------|---------------|
| 1 | Usa `test(...)` em vez do par `describe`/`it` exigido. | Nomenclatura |
| 2 | Título genérico (`"works"`) não descreve comportamento. | Nomenclatura |
| 3 | Pergunta `"test"` não vem do domínio NovaTech — não é uma fixture. | Padrão de fixtures |
| 4 | Não há mock de Azure AI Search nem de Azure OpenAI — em CI dependeria de serviço externo. | NÃO DEVE chamar serviço real |
| 5 | Falta arrange/act/assert explícitos. | Estrutura obrigatória |
| 6 | Assertion única `toBeDefined()` — passa mesmo se o handler retornar erro 500. | NÃO DEVE usar assertion vazia |
| 7 | Não valida `source_document` (guardrail PS-001) nem status HTTP. | Regras DEVE para RAG |
| 8 | Não há limpeza de estado (`afterEach`) — fragiliza ordem. | Independência |
| 9 | Body é string crua — não usa schema/factory. | Padrão de mocking |

### Depois (reescrito seguindo o padrão)

```ts
// tests/integration/query/query-endpoint.test.ts
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

describe('QueryEndpoint', () => {
  beforeEach(() => {
    mswServer.listen({ onUnhandledRequest: 'error' });
  });

  afterEach(() => {
    mswServer.resetHandlers();
    mswServer.close();
    vi.restoreAllMocks();
  });

  it('should return 200 with source_document when question matches indexed chunks', async () => {
    // arrange
    const req = buildQueryRequest({ question: 'Qual o SLA do cliente Gold?' });
    mswServer.use(
      mockAiSearchHandler({ returnChunks: [chunks.SLA_2024_A, chunks.SLA_2024_B] }),
      mockOpenAiCompletionHandler({ completion: expected.SLA_GOLD_ANSWER.text }),
    );

    // act
    const res = await queryHandler(req);

    // assert
    expect(res.status).toBe(200);
    expect(res.body).toMatchObject({
      source_document: expect.arrayContaining(['SLA-2024-A', 'SLA-2024-B']),
      confidence_level: expect.stringMatching(/^(high|medium)$/),
    });
    expected.SLA_GOLD_ANSWER.must_contain_terms.forEach((term) => {
      expect(res.body.answer).toContain(term);
    });
  });

  it('should return 400 when question is empty', async () => {
    // arrange
    const req = buildQueryRequest({ question: '' });

    // act
    const res = await queryHandler(req);

    // assert
    expect(res.status).toBe(400);
    expect(res.body.error).toBe('VALIDATION_ERROR');
    expect(res.body.details).toContain('question');
  });
});
```

#### Cada melhoria explicada

| Melhoria | Por quê |
|---|---|
| `describe('QueryEndpoint')` + `it('should ... when ...')` | Conforme regra de nomenclatura — facilita leitura do output do Vitest e do report em CI. |
| Fases `// arrange`, `// act`, `// assert` | Conforme estrutura obrigatória — qualquer revisor (ou agente) entende o teste em segundos. |
| `buildQueryRequest` (factory) + `chunks.SLA_2024_A` + `expected.SLA_GOLD_ANSWER` | Reutiliza fixtures do domínio NovaTech; nenhum dado "test" inventado. |
| MSW handlers (`mockAiSearchHandler`, `mockOpenAiCompletionHandler`) | Elimina dependência de Azure real; o teste roda offline em CI. |
| `expect(res.status).toBe(200)` + `toMatchObject({ source_document: ... })` | Assertions específicas; valida o guardrail PS-001 (campo `source_document`). |
| Validação de `must_contain_terms` em vez da string completa | Não acopla à redação literal do LLM — testes não quebram a cada microajuste de prompt. |
| Segundo caso de erro (`question` vazio) | Toda regra DEVE: cobrir ao menos um caso de erro por handler. |
| `mswServer.resetHandlers()` + `vi.restoreAllMocks()` em `afterEach` | Garante independência entre testes. |

---

## 3. Critérios de aprovação de testes gerados por IA no code review de QA

Os critérios abaixo são objetivos — dois QAs distintos devem chegar à
mesma conclusão.

| # | Critério | Verificação |
|---|----------|-------------|
| **C1** | **Assertions específicas e suficientes** | Todo `it` contém pelo menos uma assertion que falharia se a regra de negócio sob teste fosse violada. Nenhum teste depende exclusivamente de `toBeDefined`, `toBeTruthy`, `toBeFalsy` ou `not.toThrow`. |
| **C2** | **Isolamento total de dependências externas** | Nenhum teste faz chamada de rede real (verificado por `onUnhandledRequest: 'error'` no MSW). Toda integração com Azure AI Search, Azure OpenAI ou Cosmos passa por handler MSW ou stub explícito. |
| **C3** | **Dados de teste oriundos de fixtures do domínio** | Perguntas, chunks e respostas esperadas vêm de `tests/fixtures/` e refletem o domínio NovaTech (logística, SLAs, frete, devolução). Strings genéricas como `"test"`, `"foo"`, `"hello world"` reprovam o review. |
| **C4** | **Cobertura dos guardrails de produto** | Para endpoints de IA, o teste valida ao menos: presença de `source_document`, negativa explícita para devolução de carga perigosa quando aplicável, mensagem padrão para perguntas sem cobertura, e priorização da versão mais recente em casos de contradição (ADR-0003). |
| **C5** | **Estrutura arrange/act/assert legível** | As três fases estão visíveis (comentários ou separação por linha em branco). Um revisor identifica em ≤ 30 segundos o que está sendo testado. |

> Qualquer teste gerado por IA que não cumpra C1–C5 simultaneamente é
> reprovado no Gate 4 e devolvido para refatoração antes do deploy.

---

## Evidência de uso do Claude (chat)

Prompts iterativos executados no Claude:

1. *"Aja como QA sênior de um projeto de assistente RAG em Azure (TypeScript, Vitest, MSW). Liste todos os anti-padrões comuns de testes que LLMs geram por padrão e que devo bloquear via AGENTS.md."*
2. *"Com base nas decisões técnicas (Vitest, MSW, coverage 80%, pino, Zod) e nos guardrails de produto (citar fonte, negar devolução de carga perigosa, responder em PT-BR), escreva a seção `Testing Standards` do AGENTS.md em formato prescritivo DEVE/NÃO DEVE/QUANDO EM DÚVIDA."*
3. *"Reescreva o teste ruim a seguir aplicando essas regras: `test('query endpoint works', async () => { const result = await handler({ body: '{\"question\": \"test\"}' }); expect(result).toBeDefined(); });`. Explique cada melhoria em uma tabela."*
4. *"Proponha 3 critérios objetivos que dois QAs aplicariam ao mesmo PR e chegariam à mesma conclusão sobre aprovar ou rejeitar testes gerados por IA."* — refinado para 5 critérios após revisão crítica.

Iteração-chave: a primeira versão da seção misturava recomendações
descritivas ("é importante mockar"); foi reescrita em comandos
imperativos (`DEVE` / `NÃO DEVE`) para que o Copilot consiga parsear.
