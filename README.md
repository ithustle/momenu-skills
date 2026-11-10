# MoMenu Skills

Agent Skills para a [Mom Factura Payment API](https://api.momenu.online/docs) — pagamentos angolanos (Multicaixa Express, Referencia Bancaria) e facturacao com registo na AGT. Contrato da facturacao: https://api.momenu.online/api/invoicing.openapi.json

Segue a especificacao aberta [Agent Skills](https://agentskills.io/specification) e funciona com qualquer LLM agent compativel.

## Skills Disponiveis

| Skill | Descricao |
|-------|-----------|
| **mom-factura-payments** | Integracao completa de pagamentos (MCX, Referencia Bancaria) com factura emitida e registada na AGT |
| **mom-factura-webhooks** | Webhooks para confirmacao de pagamentos por Referencia Bancaria (status polling como fallback) |
| **mom-factura-testing** | Ambiente de testes QA, simulacao de resultados e checklist pre-producao |
| **mom-factura-pools** | Pool de Factura para revendedores — gestao de sub-tenants com NIF proprio, standalone keys e chave FE |
| **mom-factura-invoicing** | Facturacao pela API — FR, FT, recibos (RC), notas de credito (anular/rectificar por linha), pro-formas, PDF A4/C7, email, estado AGT, clientes e produtos |
| **mom-factura-pool-invoices** | Facturacao como cliente de pool (outra entidade fiscal, serie propria) |

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
npx skills add ithustle/momenu-skills --skill mom-factura-pools -y
npx skills add ithustle/momenu-skills --skill mom-factura-invoicing -y
npx skills add ithustle/momenu-skills --skill mom-factura-pool-invoices -y
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
  mom-factura-pools/
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
