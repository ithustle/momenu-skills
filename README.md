# MoMenu Skills

Agent Skills para a [Mom Factura Payment API](https://api.momenu.online/docs) — pagamentos angolanos com Multicaixa Express, E-kwanza e Referencia Bancaria.

Segue a especificacao aberta [Agent Skills](https://agentskills.io/specification) e funciona com qualquer LLM agent compativel.

## Skills Disponiveis

| Skill | Descricao |
|-------|-----------|
| **mom-factura-payments** | Integracao completa de pagamentos (MCX, E-kwanza, Referencia Bancaria) com geracao automatica de faturas SAFT-AO |
| **mom-factura-webhooks** | Webhooks para confirmacao de pagamentos por Referencia Bancaria e status polling para E-kwanza |
| **mom-factura-testing** | Ambiente de testes QA, simulacao de resultados e checklist pre-producao |

## Instalacao

### npx skills (recomendado)

```bash
# Instalar todos os skills
npx skills add ithustle/momenu-skills --all

# Instalar skills especificos
npx skills add ithustle/momenu-skills --skill mom-factura-payments --skill mom-factura-webhooks -y

# Instalar um skill especifico
npx skills add ithustle/momenu-skills --skill mom-factura-payments -y
npx skills add ithustle/momenu-skills --skill mom-factura-webhooks -y
npx skills add ithustle/momenu-skills --skill mom-factura-testing -y
```

### Manual — Claude Code

```bash
git clone https://github.com/ithustle/momenu-skills.git
cp -r momenu-skills/skills/mom-factura-payments .claude/skills/
```

### Manual — OpenAI Codex CLI

```bash
git clone https://github.com/ithustle/momenu-skills.git
cp -r momenu-skills/skills/mom-factura-payments .codex/skills/
```

## Estrutura

```
skills/
  mom-factura-payments/
    SKILL.md
    references/
      STATUS-POLLING.md
  mom-factura-webhooks/
    SKILL.md
  mom-factura-testing/
    SKILL.md
```

## Agentes Compativeis

Claude Code, OpenAI Codex, GitHub Copilot, Cursor, Cline, Gemini, Windsurf, e qualquer agente que suporte a especificacao [agentskills.io](https://agentskills.io).

## Documentacao

- [Documentacao HTML](https://api.momenu.online/docs) — documentacao visual interactiva
- [Documentacao LLM](https://api.momenu.online/llms.txt) — versao texto optimizada para LLMs
- [API JSON](https://api.momenu.online/api/docs) — especificacao JSON para integracao programatica

## Licenca

MIT
