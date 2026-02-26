# Claude Code — Workflow de Engenharia de Software

## Os Três Modos de Operação

A escolha do modo define quanto controle você mantém vs. quanta autonomia você delega ao Claude.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ESCOLHA DO MODO                              │
│                                                                 │
│  INTERATIVO          SKILLS              AGENTIC LOOP           │
│  ──────────          ──────              ────────────           │
│  Você aprova         Etapas              Task longa:            │
│  cada passo          repetíveis          defina o objetivo      │
│                      automatizadas       e deixe rodar          │
│                                                                 │
│  Exploração          /prd                "Implemente X          │
│  Alto risco          /debug               até os testes         │
│  Ambiguidade         /security-review     passarem"             │
│                      /pre-merge                                 │
└─────────────────────────────────────────────────────────────────┘
```

| Modo | Quando usar | Nível de autonomia |
|------|-------------|-------------------|
| **Interativo** | Exploração, alto risco, task ambígua, primeira vez num módulo | Você aprova cada passo |
| **Skills** | Etapas com padrão definido e repetível | Claude executa o padrão, você revisa o resultado |
| **Agentic Loop** | Task longa e bem definida, critérios de aceite verificáveis | Claude itera autonomamente até concluir |

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

| Etapa | Modelo |
|-------|--------|
| Exploração do codebase | **Opus** |
| Plan Mode | **Opus** — sempre |
| PRD / Spec | **Opus** |
| Agentic Loop (tarefa complexa) | **Opus** |
| Execução simples (CRUD, boilerplate) | **Sonnet** |
| Refatoração crítica / migração | **Opus** |
| Debugging | **Opus** |
| Code Review | **Opus** |
| Tasks repetitivas e bem definidas | **Sonnet** |

**Regra:** Opus para pensar e decidir. Sonnet para executar quando o caminho já está claro.

---

## Modo 1 — Interativo

Fluxo passo a passo onde você aprova cada etapa. Use para tarefas ambíguas, alto risco ou quando está explorando um módulo desconhecido.

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
│  6. AÇÃO             │────▶│  DEBUGGING       │
└──────────┬───────────┘     └────────┬────────┘
           │◀────────────────────────┘
           │
           ▼
┌──────────────────────┐
│  7. ITERAÇÃO         │  → Feedback loop: implementação ↔ spec
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  8. CODE REVIEW      │  → /security-review + /pre-merge
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  9. CI/CD → MERGE    │  → Pipeline verde, PR, documentação
└──────────────────────┘
```

### 0. SETUP

**CLAUDE.md — contexto persistente do projeto:**

```markdown
# CLAUDE.md

## Stack
- Runtime, linguagem, framework, banco, ORM, test runner, linter

## Estrutura
src/
  controllers/   → handlers HTTP, sem lógica de negócio
  services/      → lógica de negócio
  repositories/  → acesso ao banco

## Convenções
- Commits: Conventional Commits (feat, fix, chore, refactor)
- Branches: feature/*, fix/*, chore/*
- Todo endpoint precisa de teste de integração

## Comandos
- npm run dev / test / lint / build

## O que NÃO fazer
- Listar restrições importantes do projeto
```

**Exploração do codebase:**
```
"Explore a arquitetura e me dê um resumo de:
 - estrutura de camadas e responsabilidades
 - padrões e convenções usados
 - áreas de risco para a tarefa X"
```

Use subagents para paralelizar:
```
"Use subagents para explorar em paralelo:
 - como autenticação está implementada
 - quais testes já existem para o módulo Y
 - como erros são tratados na camada de serviço"
```

### 1. PROMPT

Templates por tipo de tarefa:

**Bug Fix:**
```
Contexto: [comportamento atual com exemplo concreto]
Esperado: [comportamento correto]
Reprodução: [passos ou snippet]
Restrição: [o que não pode mudar]
```

**Nova Feature:**
```
Feature: [nome e objetivo em uma linha]
Comportamento: [o que deve acontecer, com exemplos]
Fora do escopo: [o que NÃO implementar nessa iteração]
Dependências: [outros módulos ou serviços envolvidos]
```

**Refatoração:**
```
Alvo: [arquivo ou módulo]
Problema atual: [por que refatorar]
Objetivo: [como deve ficar]
Restrição: [comportamento externo deve ser 100% preservado]
```

### 2. PLAN MODE

```
/model claude-opus-4-6
"Antes de implementar, me mostre o plano completo."
```

O plano deve conter:
1. Arquivos que serão lidos, criados ou modificados
2. Abordagem técnica e alternativas descartadas
3. Decisões arquiteturais e seus trade-offs
4. Ordem de implementação com dependências entre etapas
5. Riscos e pontos de atenção

**Você aprova antes de qualquer edição.**

### 3. PRD / SPEC

```markdown
## Objetivo
O que deve ser feito e por quê.

## Escopo
- IN: o que será implementado
- OUT: o que não será tocado

## Requisitos Funcionais
- [ ] RF1: descrição clara e testável

## Requisitos Não-Funcionais
- Performance, segurança, compatibilidade

## Critérios de Aceite
Como verificar que está pronto e correto.

## Casos de Borda
Comportamentos em situações não-óbvias.

## Arquivos Impactados
Lista dos arquivos que serão criados ou modificados.
```

### 4. BRANCH

```bash
git checkout -b feature/nome-descritivo
git checkout -b fix/descricao-do-bug
git checkout -b refactor/descricao
git checkout -b chore/descricao
```

Regra: um PR por branch, uma feature por PR.

### 5. TESTES — Antes da implementação

**TDD:**
```
1. Escreva o teste que descreve o comportamento esperado (red)
2. Implemente o mínimo para passar (green)
3. Refatore mantendo os testes passando (refactor)
```

```
"Antes de implementar, escreva os testes que cobrem os
 critérios de aceite do PRD. Foque em comportamento, não em implementação."
```

| Tipo | Testes obrigatórios |
|------|---------------------|
| Novo endpoint | Integração: happy path + erros + auth |
| Nova função de negócio | Unitário: happy path + casos de borda |
| Refatoração | Regressão: todos os testes existentes passando |
| Bug fix | Regressão: teste que reproduz o bug antes do fix |

### 6. AÇÃO

- Revise cada diff individualmente
- Commits incrementais por unidade lógica
- Ações destrutivas (delete, migrations, push) sempre pedem confirmação

**Gerenciamento de contexto:**
```bash
/compact    # use quando a conversa ficou muito longa

# Para tasks muito longas, abra sessão nova referenciando o PRD:
"Continuando implementação do PRD em [arquivo]. Já implementei X e Y. Próxima etapa: Z."
```

### 7. DEBUGGING

**Modelo: Opus.**

```
1. REPRODUZIR  → isole o caso mínimo
2. HIPÓTESE    → "liste as 3 causas mais prováveis, ordenadas por probabilidade"
3. CONFIRMAR   → confirme antes de corrigir
4. CORRIGIR    → fix cirúrgico
5. PREVENIR    → escreva o teste de regressão
```

```
"Analise o erro abaixo. Não corrija ainda.
 Explique a causa raiz e proponha um fix cirúrgico.

 Erro: [stack trace]
 Contexto: [últimas mudanças]
 Esperado: [comportamento correto]"
```

Não aceite "pode ser X ou Y" sem confirmar qual é. Se o fix for maior do que esperado, volte ao plan mode.

### 8. ITERAÇÃO

```
Implementação não atende spec
          │
          ▼
   É problema de implementação?
    ├── SIM → corrija sem tocar no PRD
    └── NÃO → spec estava errado?
               ├── SIM → atualize PRD e plano, reimplemente
               └── NÃO → requisito mudou → novo prompt do início
```

Mais de 3 iterações no mesmo ponto = revise o plano.

---

## Modo 2 — Skills

Skills são prompts reutilizáveis invocados via `/nome-da-skill`. Transforme em skill qualquer etapa do workflow que você repete com o mesmo padrão.

**Como criar uma skill:**

Crie o arquivo em `.claude/commands/nome-da-skill.md` com o prompt que o Claude deve executar ao invocar `/nome-da-skill`.

### Skills recomendadas para o workflow

#### `/init-project`
`.claude/commands/init-project.md`:
```markdown
Faça as seguintes ações em sequência:

1. Explore a estrutura do projeto (pastas, arquivos principais, package.json ou equivalente)
2. Identifique: stack, framework, banco, test runner, linter, convenções de commit
3. Mapeie a arquitetura: responsabilidade de cada camada, padrões usados
4. Identifique riscos e áreas de acoplamento
5. Crie ou atualize o CLAUDE.md com tudo que foi descoberto

Não implemente nada. Apenas explore e documente.
```

#### `/prd`
`.claude/commands/prd.md`:
```markdown
Com base na descrição da feature a seguir, gere um PRD completo:

$ARGUMENTS

O PRD deve conter:
- Objetivo (problema de negócio)
- Escopo: IN (o que será feito) e OUT (o que não será tocado)
- Requisitos Funcionais (RF) — cada um verificável e testável
- Requisitos Não-Funcionais (performance, segurança, compatibilidade)
- Critérios de Aceite
- Casos de Borda
- Arquivos que serão impactados

Não implemente. Gere apenas o documento para aprovação.
```

#### `/debug`
`.claude/commands/debug.md`:
```markdown
Execute o fluxo sistemático de debugging:

1. REPRODUZIR: com base no erro fornecido, identifique o caso mínimo de reprodução
2. HIPÓTESE: liste as 3 causas mais prováveis, ordenadas por probabilidade. Para cada uma, indique como confirmar
3. Aguarde confirmação de qual hipótese é a correta antes de prosseguir
4. CORRIGIR: proponha um fix cirúrgico que mexa apenas no necessário
5. PREVENIR: escreva um teste de regressão que falha antes do fix e passa depois

Erro / comportamento a analisar:
$ARGUMENTS
```

#### `/security-review`
`.claude/commands/security-review.md`:
```markdown
Faça um security review do diff atual (git diff main) focando em:

1. Injection: SQL, command, path traversal
2. Autenticação: endpoints desprotegidos, tokens expostos
3. Autorização: usuário pode acessar recursos de outros usuários?
4. Dados sensíveis: senhas, tokens ou PII logados ou retornados desnecessariamente
5. Dependências: vulnerabilidades conhecidas (rode npm audit ou equivalente)
6. Secrets hardcoded: credenciais, API keys no código

Para cada problema: severidade (crítico / alto / médio / baixo) + localização + sugestão de fix.
Não faça mudanças. Retorne apenas o relatório.
```

#### `/pre-merge`
`.claude/commands/pre-merge.md`:
```markdown
Execute o checklist completo pré-merge:

1. Rode os testes (npm test ou equivalente) e reporte resultado
2. Rode o linter (npm run lint ou equivalente) e reporte resultado
3. Verifique se todos os critérios de aceite do PRD foram implementados
4. Verifique se há código morto, console.log de debug ou TODOs não resolvidos
5. Verifique se o CLAUDE.md precisa ser atualizado com novas convenções ou módulos
6. Verifique se a documentação (README, API docs) precisa ser atualizada

Retorne um relatório com o status de cada item e o que precisa ser resolvido antes do merge.
```

#### `/review-feature`
`.claude/commands/review-feature.md`:
```markdown
Faça um code review completo do diff da branch atual (git diff main) focando em:

1. Segurança: use o checklist do /security-review
2. Correção: casos de borda não tratados, condições de corrida, lógica incorreta
3. Performance: queries N+1, operações desnecessárias em loop, alocações excessivas
4. Design: violações de SOLID, acoplamento excessivo, responsabilidades misturadas
5. Cobertura: o que os testes não estão cobrindo

Para cada problema: severidade + localização (arquivo:linha) + sugestão.
Não faça mudanças.
```

---

## Modo 3 — Agentic Loop

Para tasks longas e bem definidas, você delega o ciclo completo ao Claude. Ele itera autonomamente — plan → implement → test → fix — até os critérios de aceite serem satisfeitos.

**Quando usar:**
- Task tem critérios de aceite verificáveis (testes passam, lint limpo, comportamento X)
- Risco de side effects é baixo (sem migrations, sem push, sem deleção)
- Você tem tempo para revisar o resultado, não cada micro-passo

**Quando NÃO usar:**
- Task ainda está ambígua — resolva a ambiguidade no modo interativo primeiro
- Envolve ações destrutivas ou irreversíveis
- Você não conhece bem o módulo — explore antes no modo interativo

### Como estruturar o prompt para o agentic loop

O prompt precisa de três elementos:

```
1. OBJETIVO      → o que deve ser feito
2. CRITÉRIOS     → como saber que está pronto (verificáveis pelo Claude)
3. RESTRIÇÕES    → o que não pode ser tocado ou feito
```

**Exemplo:**
```
Implemente rate limiting nas rotas públicas da API.

Critérios de aceite (não termine até que todos estejam satisfeitos):
- [ ] Testes de integração passando: limite de 100 req/min por IP
- [ ] 429 + header Retry-After retornado ao atingir o limite
- [ ] /health e /metrics não afetados
- [ ] npm run lint sem erros
- [ ] npm run build sem erros

Restrições:
- Não altere a lógica de autenticação existente
- Não faça push nem commits — apenas implemente e teste localmente
- Se encontrar ambiguidade, pare e me pergunte antes de assumir
```

### Comportamento esperado do agentic loop

```
Claude recebe o prompt
      │
      ▼
Plan mode automático (Opus recomendado)
      │
      ▼
Implementa etapa 1
      │
      ▼
Roda os testes / verifica critério
      │
      ├── PASSOU → próxima etapa
      │
      └── FALHOU → analisa, corrige, testa novamente
                         │
                         └── (repete até passar ou pedir ajuda)
      │
      ▼
Todos os critérios satisfeitos → reporta resultado
```

### Quando o Claude deve pausar e pedir ajuda

Instrua explicitamente no prompt:
```
"Se encontrar qualquer um dos casos abaixo, pare e me consulte:
 - Ambiguidade que exige decisão de design
 - Necessidade de alterar arquivos fora do escopo definido
 - Erro que não consegue resolver após 2 tentativas
 - Necessidade de instalar nova dependência"
```

### Revisão pós-loop

Mesmo com o agentic loop, você revisa antes de commitar:

```bash
git diff main          # revise todas as mudanças
npm test               # confirme que os testes passam
/review-feature        # review de qualidade e segurança
/pre-merge             # checklist final
/commit                # commit com mensagem gerada
```

---

## Fluxo de Decisão — Qual modo usar?

```
Nova task chegou
      │
      ▼
Você entende bem o módulo afetado?
  ├── NÃO → Modo Interativo (explore primeiro)
  └── SIM
        │
        ▼
   Os critérios de aceite são verificáveis pelo Claude?
   (testes passam, lint limpo, comportamento concreto)
     ├── NÃO → Modo Interativo
     └── SIM
           │
           ▼
      Envolve ações de alto risco?
      (migrations, push, delete, CI/CD)
        ├── SIM → Modo Interativo
        └── NÃO
              │
              ▼
         Task longa (3+ etapas interdependentes)?
           ├── SIM → Agentic Loop
           └── NÃO → Skill (/prd, /debug, /security-review...)
```

---

## Regras de Ouro

1. **Setup antes de tudo** — CLAUDE.md atualizado e codebase explorado antes de qualquer plano
2. **Opus para pensar, Sonnet para fazer** — Opus em exploração, plan mode, debugging, review e agentic loops complexos
3. **Plan antes de code** — Nunca implemente sem plano aprovado, mesmo no agentic loop
4. **PRD como contrato** — Use `/prd` para gerar, revise antes de autorizar a execução
5. **Skills para o repetível** — Toda etapa que você faz mais de uma vez vira skill
6. **Agentic loop com critérios verificáveis** — Claude só itera sozinho quando sabe como verificar que terminou
7. **Pause points explícitos** — Sempre instrua o Claude sobre quando parar e pedir ajuda
8. **Branch isolada sempre** — Nunca desenvolva na main, mesmo no agentic loop
9. **Testes antes ou junto** — `/prd` inclui critérios de aceite, que viram testes
10. **Debugging sistemático** — Use `/debug`, não "tenta corrigir e vê o que acontece"
11. **Pipeline como gate** — Nunca mergear sem CI verde, nunca usar --no-verify
12. **Revise antes de commitar** — Agentic loop não significa merge automático; você sempre revisa o resultado

---

## Atalhos Úteis

| Comando | O que faz |
|---------|-----------|
| `/model claude-opus-4-6` | Troca para Opus |
| `/model claude-sonnet-4-6` | Volta para Sonnet |
| `/compact` | Comprime histórico da conversa |
| `/init-project` | Setup: CLAUDE.md + exploração do codebase |
| `/prd [descrição]` | Gera PRD completo para revisão |
| `/debug [erro]` | Debugging sistemático |
| `/security-review` | Security review do diff atual |
| `/review-feature` | Code review completo da branch |
| `/pre-merge` | Checklist pré-merge |
| `/commit` | Gera e cria commit com mensagem automática |
| `/review-pr [número]` | Review de Pull Request |

---

## Exemplo Completo — Agentic Loop

```
TAREFA: "Adicionar rate limiting na API"

PRÉ-REQUISITO (interativo, feito uma vez):
  /init-project  → CLAUDE.md criado com stack e convenções
  git checkout -b feature/rate-limiting

GERAÇÃO DO PRD:
  /model claude-opus-4-6
  /prd "Rate limiting nas rotas públicas /auth/* e /api/v1/*.
        100 req/min por IP. Redis como store. 429 + Retry-After.
        /health e /metrics isentos."
  → você revisa e aprova o PRD

AGENTIC LOOP:
  /model claude-opus-4-6
  "Implemente o PRD de rate limiting aprovado.

   Critérios de aceite (não termine até que todos passem):
   - [ ] npm test — testes de integração do rate limiting passando
   - [ ] npm run lint — sem erros
   - [ ] npm run build — sem erros

   Restrições:
   - Não altere autenticação existente
   - Não faça push nem commits
   - Se precisar de nova dependência, me consulte antes

   Se travar em algum ponto após 2 tentativas, pare e me explique o problema."

  → Claude executa: plan → implement → test → fix → iterate
  → Claude reporta: "Todos os critérios satisfeitos"

REVISÃO (interativo):
  git diff main          → você revisa o diff completo
  /review-feature        → review de qualidade e segurança
  /pre-merge             → checklist final
  /commit                → commit gerado automaticamente
  → PR criado, pipeline verde, merge
```
