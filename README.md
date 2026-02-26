# Claude Code — Guia Completo de Capacidades

Referência visual em ASCII para engenheiros de software que querem entender e usar o Claude Code com boas práticas.

## O que está aqui

### [`claude-code-workflow.md`](./claude-code-workflow.md) — Workflow de Engenharia de Software

Guia completo de como estruturar a interação com o Claude Code do setup até o merge:

| Etapa | Descrição |
|---|---|
| 0. Setup | CLAUDE.md + exploração do codebase antes de qualquer coisa |
| 1. Prompt | Templates por tipo de tarefa com contexto claro e escopo explícito |
| 2. Plan Mode | Arquitetura e abordagem antes de codar (Opus) |
| 3. PRD / Spec | Requisitos como contrato: escopo, critérios de aceite, casos de borda |
| 4. Branch | Isolamento desde o início, convenções de nomenclatura |
| 5. Testes | Estratégia antes da implementação, TDD, cobertura mínima por tipo |
| 6. Ação | Execução controlada, gerenciamento de contexto, uso de subagents |
| 7. Debugging | Diagnóstico sistemático: reproduzir → hipótese → confirmar → corrigir → prevenir |
| 8. Iteração | Feedback loop controlado, quando regredir ao plan mode |
| 9. Code Review | Checklist de segurança e qualidade, review com Opus |
| 10. CI/CD | Pipeline como gate obrigatório antes do merge |
| 11. Merge | PR, rebase, documentação, fechamento do ciclo |

Inclui guia detalhado de **quando usar Opus vs Sonnet** em cada etapa e 12 regras de ouro.

---

### [`claude_code_guia.txt`](./claude_code_guia.txt) — Guia Completo de Capacidades

Referência visual em ASCII que cobre em formato de diagrama:

| Seção | Conteúdo |
|---|---|
| 1 | O que é o Claude Code — modelo mental |
| 2 | Principais capacidades (6 pilares) |
| 3 | Ferramentas internas (Tools) |
| 3.1 | Skills — comandos customizados reutilizáveis |
| 3.2 | Subagents — agentes especializados e paralelismo |
| 3.3 | Hooks — automações por evento |
| 3.4 | Plan Mode, sistema de permissões e keybindings |
| 4 | Sistema de memória e contexto (3 camadas) |
| 5 | Fluxo de trabalho: projeto novo (do zero) |
| 6 | Fluxo de trabalho: projeto existente (sem Claude Code) |
| 7 | Boas práticas de engenharia de software |
| 8 | Modos de operação (interativo, não-interativo, fast, agente) |
| 9 | Casos de uso práticos por cenário |
| 10 | Segurança e limites |
| 11 | Visão geral do ecossistema |
| 12 | Agentic Loops — como funciona o ciclo autônomo |
| 13 | PRD — Product Requirements Document com Claude Code |

## Como usar

```bash
# Ver o guia completo no terminal
cat claude_code_guia.txt

# Ou com paginação
less claude_code_guia.txt
```

Para melhor visualização, use uma fonte monoespaçada e terminal com largura mínima de 100 colunas.

## Conceitos principais

**Tools** — ferramentas nativas que Claude usa internamente (Read, Write, Edit, Bash, Grep, Glob, Task...)

**Skills** — prompts reutilizáveis invocados via `/nome-da-skill`. Exemplos: `/commit`, `/review-pr`, `/keybindings-help`

**Subagents** — instâncias especializadas de Claude para tarefas paralelas: `Bash`, `Explore`, `Plan`, `general-purpose`

**Agentic Loop** — ciclo autônomo interno: Raciocinar → Usar Tool → Observar resultado → repetir até concluir

**CLAUDE.md** — arquivo de contexto persistente que guia o Claude em cada sessão (global, por projeto, por subdiretório)

**Hooks** — comandos shell que disparam em eventos do Claude Code (ex: antes de submeter um prompt)

**Plan Mode** — Claude explora e planeja antes de implementar, apresentando o plano para aprovação

**PRD** — documento de requisitos usado como "contrato" que guia todo o desenvolvimento com Claude Code

## Fluxo recomendado para novos projetos

```
mkdir projeto && cd projeto
claude
> "Crie a estrutura do projeto [descreva stack e arquitetura]"
> "Implemente [feature] com testes"
> "Rode os testes e corrija falhas"
/commit
```

## Fluxo recomendado para projetos existentes

```
cd projeto-existente
claude
> "Explore a arquitetura e me dê um resumo"
> "Crie um CLAUDE.md com o contexto do projeto"
# a partir daí: ciclos curtos de entender → planejar → implementar → verificar → commitar
```

---

Gerado com [Claude Code](https://www.anthropic.com/claude-code) — Fevereiro 2026
