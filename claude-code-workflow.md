# Claude Code — Workflow de Interação

## Visão Geral do Fluxo

```
IDEIA / PROBLEMA
      │
      ▼
┌─────────────────┐
│  1. PROMPT      │  → Descreva o objetivo com contexto claro
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  2. PLAN MODE   │  → Claude explora o codebase e propõe abordagem
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  3. PRD / SPEC  │  → Documento de requisitos validado por você
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  4. AÇÃO        │  → Claude implementa com aprovação por etapa
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  5. REVISÃO     │  → Testes, ajustes, commit
└─────────────────┘
```

---

## Etapas Detalhadas

### 1. PROMPT — Contexto é tudo

**Objetivo:** Comunicar claramente o que você quer.

**Boas práticas:**
- Seja específico sobre o problema, não apenas o sintoma
- Inclua contexto: linguagem, framework, restrições
- Mencione o que NÃO deve ser alterado

**Exemplo ruim:**
> "Adiciona autenticação"

**Exemplo bom:**
> "Adiciona autenticação JWT na API Express (Node.js). O login deve usar email/senha,
> retornar access + refresh token. Não mexa nos controllers existentes, apenas adicione
> middleware e rotas novas em /auth."

---

### 2. PLAN MODE — Antes de codar

**Objetivo:** Alinhar a abordagem antes de qualquer mudança no código.

**Quando usar:**
- Tarefas que tocam 3+ arquivos
- Novas features ou refatorações
- Quando há múltiplas abordagens possíveis

**Como ativar:**
- Claude entra automaticamente em plan mode para tarefas complexas
- Você pode pedir: *"antes de implementar, me mostre o plano"*

**O que acontece:**
1. Claude lê os arquivos relevantes
2. Propõe arquitetura e passos
3. **Você aprova ou ajusta** antes de qualquer edição

**Regra de ouro:** Nunca deixe o Claude codar sem você entender o plano primeiro.

---

## Escolha do Modelo — Opus vs Sonnet

Opus é tecnicamente superior em tudo. A escolha entre os dois é um **trade-off entre qualidade, velocidade e custo**.

| | Opus | Sonnet |
|---|---|---|
| Qualidade | Máxima | Boa o suficiente para a maioria |
| Velocidade | Mais lento | Mais rápido |
| Custo | ~5x mais caro | Mais barato |

```
/model claude-opus-4-6      ← quando precisar do melhor raciocínio
/model claude-sonnet-4-6    ← quando velocidade e custo importam
```

### No Plan Mode

| Situação | Modelo |
|----------|--------|
| Qualquer plan mode | **Opus** — sempre. Planejamento ruim gera retrabalho caro. |

O plano define tudo que vem depois. Economizar aqui é falsa economia.

### Na Execução

| Situação | Modelo |
|----------|--------|
| Plano bem definido e granular | **Sonnet** — execução mecânica, você revisa cada diff |
| Tasks repetitivas (CRUDs, campos, boilerplate) | **Sonnet** |
| Refatoração crítica ou migração de dados | **Opus** — erros custam caro |
| Código muito entrelaçado com decisões em aberto | **Opus** — o plano não cobre tudo |
| Alta complexidade onde um detalhe errado quebra tudo | **Opus** |

**Regra prática:** Sonnet como padrão na execução. Suba para Opus se a task for de alto risco ou se o plano ainda tiver ambiguidades.

---

### 3. PRD / SPEC — Documentar o acordo

**Objetivo:** Cristalizar os requisitos antes da execução.

**Quando criar um PRD:**
- Features novas com múltiplos comportamentos esperados
- Integrações com sistemas externos
- Mudanças que afetam outros times ou usuários

**Template mínimo:**

```markdown
## Objetivo
O que deve ser feito e por quê.

## Escopo
- IN: o que será implementado
- OUT: o que não será tocado

## Requisitos Funcionais
- [ ] RF1: ...
- [ ] RF2: ...

## Requisitos Não-Funcionais
- Performance, segurança, compatibilidade

## Critérios de Aceite
Como saber que está pronto e correto.

## Arquivos Impactados
Lista dos arquivos que serão modificados.
```

**Fluxo prático:**
```
Você descreve → Claude gera rascunho do PRD → Você revisa/ajusta → Aprovação → Implementação
```

---

### 4. AÇÃO — Execução controlada

**Objetivo:** Implementar com visibilidade total.

**Princípios:**
- Aprove cada etapa crítica antes de prosseguir
- Para ações destrutivas (delete, reset, push), Claude deve pedir confirmação
- Use commits incrementais, não um commit gigante no final

**Tipos de ação por risco:**

| Risco | Exemplos | Postura |
|-------|----------|---------|
| Baixo | Editar arquivos locais, rodar testes | Claude executa livremente |
| Médio | Instalar dependências, criar arquivos | Claude informa antes |
| Alto  | Push, delete, migrations de DB | Claude **sempre pede confirmação** |

**Fluxo de execução:**
```
Plano aprovado
      │
      ▼
Claude edita arquivo 1 → você revisa diff
      │
      ▼
Claude edita arquivo 2 → você revisa diff
      │
      ▼
Testes passam → commit
```

---

### 5. REVISÃO — Fechar o ciclo

**Checklist pós-implementação:**

```
□ Ler o diff completo antes de aceitar
□ Rodar testes (unitários + integração)
□ Verificar que nada fora do escopo foi alterado
□ Commitar com mensagem descritiva
□ Atualizar documentação se necessário
```

---

## Padrões de Prompt por Tipo de Tarefa

### Bug Fix
```
Contexto: [descreva o comportamento atual]
Esperado: [descreva o comportamento correto]
Reprodução: [como reproduzir o bug]
Restrição: [o que não pode mudar]
```

### Nova Feature
```
Feature: [nome e objetivo]
Usuário alvo: [quem vai usar]
Comportamento: [o que deve acontecer]
Fora do escopo: [o que não implementar agora]
```

### Refatoração
```
Alvo: [arquivo ou módulo]
Problema atual: [por que refatorar]
Objetivo: [como deve ficar]
Restrição: [comportamento externo deve ser preservado]
```

### Review de Código
```
Por favor, revise [arquivo/PR] focando em:
- Segurança
- Performance
- Manutenibilidade
Não faça mudanças, apenas liste os problemas encontrados.
```

---

## Regras de Ouro

1. **Plan before code** — Sempre alinhe a abordagem antes da execução
2. **Opus sempre no plan mode** — Planejamento ruim gera retrabalho caro. Na execução, Sonnet como padrão — suba para Opus em tasks de alto risco ou alta ambiguidade
3. **Escopo explícito** — Diga o que NÃO deve ser alterado
4. **Revisão incremental** — Leia cada diff, não apenas o resultado final
5. **Commits atômicos** — Um commit por funcionalidade, não um batch gigante
6. **Confirme ações destrutivas** — Push, delete, migrations sempre com confirmação explícita
7. **PRD para features** — Documente antes de implementar o que é grande
8. **Testes antes de fechar** — Nunca marque uma tarefa como pronta sem rodar os testes

---

## Atalhos Úteis

| Comando | O que faz |
|---------|-----------|
| `/plan` | Entra em modo de planejamento |
| `/model claude-opus-4-6` | Troca para Opus (use no plan mode) |
| `/model claude-sonnet-4-6` | Volta para Sonnet (use na execução) |
| `/commit` | Gera e cria commit com mensagem automática |
| `/review-pr` | Faz review de um Pull Request |
| `/help` | Lista todos os comandos disponíveis |

---

## Fluxo Completo — Exemplo Real

```
1. PROMPT
   "Adicionar paginação na listagem de usuários da API REST"

2. PLAN MODE  ← /model claude-opus-4-6
   Claude lê: routes/users.js, controllers/userController.js, models/User.js
   Propõe: adicionar query params ?page e ?limit, retornar metadata de paginação
   Você aprova o plano.

3. PRD (opcional para features maiores)
   - RF1: GET /users?page=1&limit=20
   - RF2: Resposta inclui { data, total, page, totalPages }
   - RF3: Default: page=1, limit=20, max limit=100
   - OUT: Não alterar autenticação nem outros endpoints

4. AÇÃO  ← /model claude-sonnet-4-6
   Claude edita userController.js → você revisa
   Claude edita routes/users.js → você revisa
   Claude sugere teste → você aprova

5. REVISÃO
   Testes rodam → diff aprovado → commit: "feat: add pagination to users endpoint"
```
