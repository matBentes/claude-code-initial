# Claude Code — Workflow de Engenharia de Software

## Visão Geral do Fluxo

```
IDEIA / PROBLEMA
      │
      ▼
┌──────────────────────┐
│  0. SETUP            │  → CLAUDE.md + exploração do codebase
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  1. PROMPT           │  → Contexto claro, escopo explícito
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  2. PLAN MODE (Opus) │  → Arquitetura, abordagem, arquivos impactados
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  3. PRD / SPEC       │  → Requisitos, critérios de aceite, escopo
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  4. BRANCH           │  → Feature branch isolada
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  5. TESTES           │  → Escrever testes antes (TDD) ou definir estratégia
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐     ┌─────────────────┐
│  6. AÇÃO             │────▶│  DEBUGGING       │  → Quando algo quebra
└──────────┬───────────┘     └────────┬────────┘
           │                          │
           │◀─────────────────────────┘
           │
           ▼
┌──────────────────────┐
│  7. ITERAÇÃO         │  → Feedback loop: implementação ↔ spec
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  8. CODE REVIEW      │  → Segurança, qualidade, cobertura
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  9. CI/CD            │  → Pipeline deve passar antes do merge
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  10. MERGE           │  → PR aprovado, branch deletada, changelog
└──────────────────────┘
```

---

## Escolha do Modelo — Opus vs Sonnet

Opus é tecnicamente superior em tudo. A escolha é um **trade-off entre qualidade, velocidade e custo**.

| | Opus | Sonnet |
|---|---|---|
| Qualidade | Máxima | Boa o suficiente para a maioria |
| Velocidade | Mais lento | Mais rápido |
| Custo | ~5x mais caro | Mais barato |

```
/model claude-opus-4-6      ← melhor raciocínio, decisões arquiteturais
/model claude-sonnet-4-6    ← execução mecânica, velocidade, custo
```

| Etapa | Modelo | Motivo |
|-------|--------|--------|
| Exploração do codebase | **Opus** | Entender sistemas complexos exige raciocínio profundo |
| Plan Mode | **Opus** — sempre | Planejamento ruim gera retrabalho caro |
| PRD / Spec | **Opus** | Decisões de escopo têm alto impacto |
| Execução simples (CRUD, boilerplate) | **Sonnet** | Plano claro, execução mecânica |
| Execução com decisões em aberto | **Opus** | Ambiguidade no plano exige julgamento |
| Refatoração crítica / migração de dados | **Opus** | Erros custam caro |
| Debugging | **Opus** | Diagnóstico de causa raiz é raciocínio complexo |
| Code Review | **Opus** | Detectar problemas sutis de segurança e design |
| Tasks repetitivas e bem definidas | **Sonnet** | Velocidade e custo justificados |

**Regra prática:** Opus para pensar e decidir. Sonnet para executar quando o caminho já está claro.

---

## Etapas Detalhadas

### 0. SETUP — Antes de qualquer coisa

**Objetivo:** Dar ao Claude o contexto persistente do projeto para que ele não tome decisões cegas.

#### 0.1 CLAUDE.md — Contexto do projeto

O `CLAUDE.md` é lido automaticamente pelo Claude em cada sessão. É o contrato entre você e o Claude sobre como o projeto funciona.

**Crie ou atualize antes de começar qualquer tarefa:**

```markdown
# CLAUDE.md

## Stack
- Runtime: Node.js 20, TypeScript 5
- Framework: Express 4
- Banco: PostgreSQL 15 com Prisma ORM
- Testes: Vitest + Supertest
- Linter: ESLint + Prettier

## Estrutura
src/
  controllers/   → handlers HTTP, sem lógica de negócio
  services/      → lógica de negócio
  repositories/  → acesso ao banco
  routes/        → definição de rotas
  middlewares/   → auth, validação, error handling

## Convenções
- Commits: Conventional Commits (feat, fix, chore, refactor)
- Branches: feature/*, fix/*, chore/*
- Sem any no TypeScript
- Todo endpoint precisa de teste de integração

## Comandos
- npm run dev       → desenvolvimento
- npm run test      → testes
- npm run lint      → lint
- npm run build     → build de produção

## O que NÃO fazer
- Não usar callbacks, apenas async/await
- Não acessar o banco diretamente nos controllers
- Não commitar sem os testes passando
```

#### 0.2 Exploração do codebase

Para projetos existentes, antes de planejar qualquer coisa:

```
"Explore a arquitetura do projeto e me dê um resumo de:
 - estrutura de pastas e responsabilidade de cada camada
 - padrões usados (design patterns, convenções)
 - pontos de acoplamento e dependências críticas
 - possíveis áreas de risco para a tarefa X"
```

**Use subagents para paralelizar a exploração:**

```
"Use subagents para explorar em paralelo:
 - como autenticação está implementada
 - como erros são tratados
 - quais testes já existem para o módulo Y"
```

---

### 1. PROMPT — Contexto é tudo

**Objetivo:** Comunicar claramente o que você quer, sem deixar espaço para suposições.

**Boas práticas:**
- Seja específico sobre o problema, não apenas o sintoma
- Inclua contexto: linguagem, framework, restrições
- Mencione explicitamente o que NÃO deve ser alterado
- Referencie arquivos ou funções específicas quando possível

**Templates por tipo de tarefa:**

**Bug Fix:**
```
Contexto: [comportamento atual com exemplo concreto]
Esperado: [comportamento correto]
Reprodução: [passos ou snippet para reproduzir]
Restrição: [o que não pode mudar]
Arquivos suspeitos: [se souber]
```

**Nova Feature:**
```
Feature: [nome e objetivo em uma linha]
Usuário alvo: [quem vai usar e como]
Comportamento: [o que deve acontecer, com exemplos]
Fora do escopo: [o que NÃO implementar nessa iteração]
Dependências: [outros módulos ou serviços envolvidos]
```

**Refatoração:**
```
Alvo: [arquivo ou módulo]
Problema atual: [por que refatorar — acoplamento, perf, legibilidade]
Objetivo: [como deve ficar após a refatoração]
Restrição: [comportamento externo deve ser 100% preservado]
Cobertura de testes: [quais testes garantem que não quebramos nada]
```

**Review de código:**
```
Revise [arquivo/PR/função] focando em:
- Segurança (injection, autenticação, autorização)
- Performance (N+1, queries sem índice, loops desnecessários)
- Manutenibilidade (coesão, acoplamento, nomes)
- Cobertura de casos de borda
Não faça mudanças. Retorne uma lista priorizada de problemas.
```

---

### 2. PLAN MODE — Antes de codar

**Objetivo:** Alinhar a abordagem antes de qualquer mudança no código.

**Modelo: Opus — sempre.**

**Quando usar:**
- Tarefas que tocam 3+ arquivos
- Novas features ou refatorações
- Quando há múltiplas abordagens possíveis
- Quando você não conhece bem o módulo afetado

**Como ativar:**
```
/model claude-opus-4-6
"Antes de implementar, me mostre o plano completo."
```

**O que o plano deve conter:**
1. Arquivos que serão lidos, criados ou modificados
2. Abordagem técnica escolhida e alternativas descartadas
3. Decisões arquiteturais e seus trade-offs
4. Ordem de implementação e dependências entre etapas
5. Riscos e pontos de atenção

**Você aprova ou ajusta antes de qualquer edição.**

**Regra de ouro:** Um plano mal feito é mais caro do que nenhum plano. Se o plano não convencer, refaça antes de avançar.

---

### 3. PRD / SPEC — Documentar o acordo

**Objetivo:** Cristalizar os requisitos como contrato entre você e o Claude antes da execução.

**Quando criar:**
- Features novas com múltiplos comportamentos
- Integrações com sistemas externos
- Mudanças que afetam API pública ou outros times

**Template:**

```markdown
## Objetivo
O que deve ser feito e por quê (problema de negócio).

## Escopo
- IN: o que será implementado nessa iteração
- OUT: o que não será tocado (explícito evita scope creep)

## Requisitos Funcionais
- [ ] RF1: descrição clara e testável
- [ ] RF2: ...

## Requisitos Não-Funcionais
- Performance: ex. resposta < 200ms p95
- Segurança: ex. apenas usuários autenticados
- Compatibilidade: ex. não quebrar clientes existentes

## Critérios de Aceite
Como saber que está pronto e correto. Deve ser verificável.

## Casos de Borda
Comportamentos esperados em situações não-óbvias.

## Arquivos Impactados
Lista dos arquivos que serão criados ou modificados.

## Dependências
Serviços externos, variáveis de ambiente, migrações necessárias.
```

**Fluxo:**
```
Você descreve → Claude gera rascunho → você revisa/ajusta → aprovação → implementação
```

---

### 4. BRANCH — Isolamento desde o início

**Objetivo:** Nunca desenvolver direto na branch principal.

```bash
git checkout -b feature/nome-descritivo
# ou
git checkout -b fix/descricao-do-bug
# ou
git checkout -b chore/descricao-da-tarefa
```

**Convenções:**
| Prefixo | Quando usar |
|---------|-------------|
| `feature/` | Nova funcionalidade |
| `fix/` | Correção de bug |
| `refactor/` | Refatoração sem mudança de comportamento |
| `chore/` | Configuração, dependências, CI |

**Regra:** Um PR por branch. Uma feature por PR. PRs pequenos são revisados melhor e mergeados mais rápido.

---

### 5. TESTES — Estratégia antes da implementação

**Objetivo:** Definir como a correção/feature será verificada antes de escrever uma linha de código.

#### TDD (quando aplicável)

```
1. Escreva o teste que descreve o comportamento esperado
2. Rode — deve falhar (red)
3. Implemente o mínimo para passar (green)
4. Refatore mantendo os testes passando (refactor)
```

**Peça ao Claude para começar pelos testes:**
```
"Antes de implementar, escreva os testes que cobrem os critérios de aceite do PRD.
 Use [Vitest/Jest/pytest]. Foque em comportamento, não em implementação."
```

#### Estratégia mínima por tipo de mudança

| Tipo | Testes obrigatórios |
|------|---------------------|
| Novo endpoint | Integração: happy path + erros + auth |
| Nova função de negócio | Unitário: happy path + casos de borda |
| Refatoração | Regressão: todos os testes existentes devem continuar passando |
| Bug fix | Regressão: teste que reproduz o bug antes do fix |
| Migração de dados | Validação: antes e depois com dados reais anonimizados |

---

### 6. AÇÃO — Execução controlada

**Objetivo:** Implementar com visibilidade total, sem surpresas.

**Modelo:** Sonnet como padrão. Opus para alta complexidade ou ambiguidade.

**Princípios:**
- Aprove cada etapa crítica antes de prosseguir
- Revise cada diff — não apenas o resultado final
- Commits incrementais: um commit por unidade lógica, não um batch gigante

**Tipos de ação por risco:**

| Risco | Exemplos | Postura |
|-------|----------|---------|
| Baixo | Editar arquivos locais, rodar testes | Claude executa livremente |
| Médio | Instalar dependências, criar arquivos, variáveis de ambiente | Claude informa antes |
| Alto | Push, delete de arquivos, migrations de DB, alteração de CI/CD | Claude **sempre pede confirmação explícita** |

**Gerenciamento de contexto durante a execução:**

À medida que a conversa cresce, o contexto pode se degradar. Sinais de alerta:
- Claude começa a repetir código já feito
- Claude ignora restrições que você mencionou antes
- Respostas ficam genéricas ou inconsistentes

```bash
/compact    # comprime o histórico preservando o essencial
            # use quando a conversa ficou muito longa

# Para tasks muito longas, abra uma sessão nova e referencie o PRD:
"Continuando a implementação do PRD em [arquivo]. Já implementei X e Y. Próxima etapa: Z."
```

**Uso de subagents para paralelizar:**
```
"Use subagents para implementar em paralelo:
 - subagent 1: escrever os testes de integração
 - subagent 2: implementar o controller
 Depois consolide os resultados."
```

---

### 7. DEBUGGING — Quando algo quebra

**Objetivo:** Diagnóstico sistemático, não tentativa e erro.

**Modelo: Opus** — causa raiz é raciocínio complexo.

**Processo:**

```
1. REPRODUZIR
   - Isolate o caso mínimo que reproduz o problema
   - Documente: input → output atual → output esperado

2. HIPÓTESE
   "Analise o erro abaixo e liste as 3 causas mais prováveis,
    ordenadas por probabilidade. Para cada uma, indique como confirmar."

3. CONFIRMAR
   - Confirme a causa antes de corrigir
   - Nunca corrija sem entender o porquê

4. CORRIGIR
   - Fix cirúrgico: mexa apenas no necessário
   - Se o fix for maior do que esperado, volte ao plan mode

5. PREVENIR
   - Escreva um teste que reproduz o bug antes do fix
   - O teste deve falhar antes e passar depois
```

**Prompt de debugging:**
```
"Analise o seguinte erro:
 [stack trace / comportamento]

 Contexto:
 - O que eu esperava: [...]
 - O que aconteceu: [...]
 - Últimas mudanças antes do problema: [...]

 Não corrija ainda. Primeiro me explique a causa raiz e proponha um fix cirúrgico."
```

**Não faça:**
- Não peça ao Claude para "tentar" fixes aleatórios
- Não aceite "pode ser X ou Y" sem confirmar qual é
- Não corrija sem escrever o teste de regressão

---

### 8. ITERAÇÃO — Feedback loop

**Objetivo:** Ajustar a implementação sem perder coerência com o plano original.

**Quando iterar:**
- Implementação não atende um critério de aceite do PRD
- Testes revelam um caso de borda não considerado
- Review identifica problema de design

**Fluxo de iteração:**

```
Implementação não atende spec
          │
          ▼
   É problema de implementação?
    ├── SIM → corrija sem tocar no PRD
    └── NÃO → o spec estava errado?
               ├── SIM → atualize o PRD e o plano, depois reimplemente
               └── NÃO → requisito mudou → volte ao início com novo prompt
```

**Regra:** Se você iterou mais de 3 vezes no mesmo ponto, pare e revise o plano. Iteração excessiva é sinal de spec ambíguo ou plano inadequado.

---

### 9. CODE REVIEW — Antes do merge

**Objetivo:** Garantir qualidade, segurança e manutenibilidade antes de chegar na main.

**Modelo: Opus.**

#### Review com Claude

```
"Faça um code review do diff abaixo focando em:
 1. Segurança: injection, autenticação/autorização, dados sensíveis expostos
 2. Correção: casos de borda não tratados, condições de corrida
 3. Performance: queries N+1, operações desnecessárias em loop
 4. Design: violações de SOLID, acoplamento excessivo, nomes confusos
 5. Cobertura: o que os testes não estão cobrindo

 Para cada problema encontrado: severidade (crítico/alto/médio/baixo) + sugestão de fix."
```

#### Checklist de segurança mínimo

```
□ Inputs validados e sanitizados na entrada do sistema
□ Autenticação verificada em todos os endpoints protegidos
□ Autorização: usuário pode acessar apenas seus próprios recursos
□ Dados sensíveis não logados (senhas, tokens, PII)
□ Queries parametrizadas (sem concatenação de SQL)
□ Dependências sem vulnerabilidades conhecidas (npm audit / pip audit)
□ Variáveis de ambiente para segredos, nunca hardcoded
```

#### Checklist de qualidade

```
□ Código faz o que o PRD especifica
□ Casos de borda cobertos por testes
□ Sem código morto ou comentado
□ Nomes de variáveis e funções autoexplicativos
□ Sem duplicação desnecessária (DRY onde faz sentido)
□ Tratamento de erros adequado
```

---

### 10. CI/CD — Pipeline como gate obrigatório

**Objetivo:** Nunca mergear sem pipeline verde.

**O pipeline deve verificar:**
```
□ Lint (estilo e formatação)
□ Type check (TypeScript / mypy)
□ Testes unitários
□ Testes de integração
□ Build de produção
□ Análise de segurança (dependências, SAST)
□ Cobertura de testes (não regredir)
```

**Se o pipeline quebrar:**
```
"O CI falhou com o seguinte erro:
 [output do pipeline]

 Contexto: [o que foi alterado]
 Analise a causa e proponha o fix sem mexer no comportamento da feature."
```

**Regra:** Pipeline quebrado = nenhum merge, sem exceções. Nunca use `--no-verify` para contornar.

---

### 11. MERGE — Fechar o ciclo

**Objetivo:** Integrar o trabalho de forma limpa e documentada.

```bash
# Rebase na main antes de mergear (histórico limpo)
git rebase main

# PR via Claude Code
/review-pr [número]

# Commit de merge
/commit
```

**Checklist final:**
```
□ Todos os critérios de aceite do PRD verificados
□ Pipeline verde
□ Review aprovado
□ CLAUDE.md atualizado se o projeto mudou (nova convenção, novo módulo)
□ Documentação atualizada (README, API docs, ADRs)
□ Branch deletada após merge
□ Ticket/issue fechado com referência ao PR
```

---

## Regras de Ouro

1. **Setup antes de tudo** — CLAUDE.md atualizado e codebase explorado antes de qualquer plano
2. **Opus para pensar, Sonnet para fazer** — Opus em exploração, plan mode, debugging e review; Sonnet em execução clara
3. **Plan antes de code** — Nunca implemente sem um plano aprovado
4. **PRD como contrato** — Requisitos documentados evitam retrabalho e scope creep
5. **Branch isolada sempre** — Nunca desenvolva na main
6. **Testes antes ou junto** — Defina como testar antes de implementar
7. **Commits atômicos** — Um commit por unidade lógica, mensagem descritiva
8. **Debugging sistemático** — Entenda a causa antes de corrigir, sempre escreva o teste de regressão
9. **Iteração controlada** — Mais de 3 iterações no mesmo ponto = revise o plano
10. **Pipeline como gate** — Nunca mergear sem CI verde, nunca usar --no-verify
11. **Gerenciar contexto** — Use /compact em conversas longas; sessão nova com referência ao PRD para tasks muito extensas
12. **Confirme ações destrutivas** — Push, delete, migrations, CI/CD sempre com confirmação explícita

---

## Atalhos Úteis

| Comando | O que faz |
|---------|-----------|
| `/model claude-opus-4-6` | Troca para Opus (plan, debug, review) |
| `/model claude-sonnet-4-6` | Volta para Sonnet (execução) |
| `/compact` | Comprime histórico da conversa |
| `/commit` | Gera e cria commit com mensagem automática |
| `/review-pr [número]` | Faz review de um Pull Request |
| `/plan` | Entra em modo de planejamento |
| `/help` | Lista todos os comandos disponíveis |

---

## Fluxo Completo — Exemplo Real

```
TAREFA: "Adicionar rate limiting na API para evitar abuso"

0. SETUP
   CLAUDE.md já tem: stack Express + Redis disponível
   Exploração: "Como middleware está estruturado? Quais rotas precisam de proteção?"

1. PROMPT
   "Implementar rate limiting nas rotas públicas da API (/auth/*, /api/v1/*).
    Usar Redis para estado compartilhado entre instâncias.
    Limite: 100 req/min por IP. Não afetar rotas internas (/health, /metrics).
    Retornar 429 com header Retry-After."

2. PLAN MODE  ← /model claude-opus-4-6
   Claude propõe: express-rate-limit + rate-limit-redis
   Arquivos: middleware/rateLimiter.ts (novo), app.ts (modificar), .env (nova var)
   Você aprova.

3. PRD
   RF1: 100 req/min por IP nas rotas públicas
   RF2: Redis como store (compartilhado entre instâncias)
   RF3: 429 + header Retry-After no limite
   RF4: /health e /metrics isentos
   OUT: não alterar lógica de autenticação existente

4. BRANCH
   git checkout -b feature/rate-limiting

5. TESTES
   Claude escreve testes de integração:
   - happy path: req dentro do limite passa
   - limite atingido: 429 + Retry-After
   - rota interna: não afetada pelo rate limit

6. AÇÃO  ← /model claude-sonnet-4-6
   Claude cria middleware/rateLimiter.ts → você revisa diff
   Claude modifica app.ts → você revisa diff
   Claude atualiza .env.example → você revisa

7. DEBUGGING (se necessário)
   "Redis connection failing em test: [erro]. Analise a causa."

8. ITERAÇÃO
   Teste revelou: IPs de proxy não capturados corretamente
   Fix: adicionar trust proxy config → PRD atualizado com novo caso de borda

9. CODE REVIEW  ← /model claude-opus-4-6
   "Revise focando em segurança: bypass de rate limit possível? IP spoofing?"
   Fix: adicionar validação de X-Forwarded-For

10. CI/CD
    Pipeline verde: lint ✓ types ✓ testes ✓ build ✓

11. MERGE
    /review-pr → aprovado → merge → branch deletada
    CLAUDE.md atualizado: "Rate limiting via express-rate-limit + Redis"
    Ticket fechado com link para o PR
```
