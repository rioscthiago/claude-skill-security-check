# /security-check — Skill para Claude Code

> Auditoria de segurança das mudanças pendentes na branch atual, em uma única invocação.

Combina o foco do `/security-review` (built-in do Claude Code) com:

1. **Checklist anti-vibe-coding** — padrões recorrentes de exploração em apps modernos
2. **Práticas validadas do OWASP** (Cheat Sheets + Secure Headers Project)

Cobre 15+ categorias: Access Control (IDOR, privilege escalation), Business Logic (OWASP A04), Auth & Secrets (anti-enumeração, MFA), Input Validation (SQL/NoSQL/XSS), SSRF (com image proxy), Race Conditions, Secrets & Misconfig (A05), Security Headers, File Upload, Webhooks & Supply Chain (SLSA/SBOM), Exception Handling (A10), AI/LLM Security, Logging Estruturado (OWASP vocab), Privacidade LGPD/GDPR, Honey pots.

> **Importante:** a skill **só audita e relata**. Não modifica código, configuração nem infra automaticamente. O relatório é desenhado como **insumo para o planejamento nativo do agente** (Claude Code / Codex) — após você definir escopo, o agente executa com seu próprio planner.

---

## 🇧🇷 Português

### Como instalar

```bash
mkdir -p ~/.claude/skills/security-check
curl -o ~/.claude/skills/security-check/SKILL.md \
  https://raw.githubusercontent.com/rioscthiago/claude-skill-security-check/main/SKILL.md
```

Pronto. A skill aparece na próxima invocação do Claude Code.

### Como usar

Dentro do Claude Code, na raiz de qualquer repo Git:

```
/security-check
```

A skill vai:
1. **Mapear o terreno** — stack, trust boundaries, superfícies críticas, escopo do diff
2. **Auditar contra o checklist** — varre cada arquivo modificado procurando padrões de brecha
3. **Self-check com 12 perguntas** — simula mentalmente cenários de ataque clássicos
4. **Reportar achados** com finding format expandido: severidade (🔴/🟠/🟡/🟢), arquivo:linha, evidência, cenário de ataque, fix sugerido, **risco de regressão**, **dependências entre fixes**, **esforço estimado**, **modulação em PRs**, **análise de regressão agregada** e **tarefas que só humano pode executar**

A skill **não aplica fix sozinha** — devolve um relatório rico que o agente usa como contexto pro planejamento nativo dele, após você decidir escopo (todos / só CRÍTICO / fatiar em PRs).

### Quando usar

- **Antes de qualquer deploy** de feature nova
- Após mexer em: auth, sessão, cookies, upload, query SQL/NoSQL, fetch externo, CORS, operações financeiras, webhooks, integração com LLM
- Quando alguém pedir "review de segurança" ou "checa brechas"

### O que a skill checa (resumo)

| Categoria | Exemplos |
|---|---|
| 🔴 Access Control | IDOR, privilege escalation (vertical/horizontal), path traversal |
| 🔴 Business Logic (A04) | Saldo nunca negativo, cupom uso único, reembolso prazo, quota por plano, ownership |
| 🔴 Auth & Secrets | Fallback em secrets, JWT validation, Argon2id, anti-enumeração, timing-safe response, MFA, refresh token rotation |
| 🔴 Input Validation | SQL/NoSQL injection, XSS via `innerHTML`/`dangerouslySetInnerHTML`, command injection |
| 🔴 SSRF | RFC1918, loopback, link-local, CGNAT, DNS rebinding, image proxy (Camo/go-camo) |
| 🔴 Race Conditions | Atomic UPDATE, SELECT FOR UPDATE, idempotency keys, UNIQUE constraint, likes/follows/recursos únicos/convites |
| 🔴 Secrets & Misconfig (A05) | Stack traces, debug endpoints, directory listing, ambientes não-prod públicos |
| 🟠 Security Headers | CSP, HSTS, X-Frame-Options, Permissions-Policy (configs OWASP) |
| 🟠 File Upload | Magic bytes, allowlist, UUID rename, fora do webroot |
| 🟠 Webhooks & Supply Chain | HMAC validation, lockfile, npm audit, SRI, SLSA/cosign, SBOM, CI/CD como código privilegiado |
| 🟠 Exception Handling (A10) | Global handler, fail closed, sem catches silenciosos, mensagens genéricas |
| 🟠 AI/LLM | Prompt injection, no `bypassPermissions` em prod |
| 🟡 Logging Estruturado (A09) | JSON com vocabulário OWASP, prefixos de evento, anti log injection |
| 🟡 Privacidade (LGPD/GDPR) | Endpoints de data subject rights (acesso/correção/exclusão/portabilidade), retenção em logs+backups |
| 🟢 Honey pots | `/wp-admin`, `/.env`, `/phpmyadmin` |

---

## 🇺🇸 English

### Install

```bash
mkdir -p ~/.claude/skills/security-check
curl -o ~/.claude/skills/security-check/SKILL.md \
  https://raw.githubusercontent.com/rioscthiago/claude-skill-security-check/main/SKILL.md
```

The skill is available on next Claude Code invocation.

### Usage

From inside Claude Code, at the root of any Git repo:

```
/security-check
```

The skill will:
1. **Map the terrain** — stack, trust boundaries, critical surfaces, diff scope
2. **Audit against the checklist** — scans each modified file for vulnerability patterns
3. **12-question self-check** — mentally simulates classic attack scenarios
4. **Report findings** with expanded format: severity (🔴/🟠/🟡/🟢), file:line, evidence, attack scenario, suggested fix, **regression risk**, **dependencies between fixes**, **effort estimate**, **suggested PR modularization**, **aggregated regression analysis**, and **human-only tasks**

The skill **does not apply fixes itself** — it delivers a rich report meant as context for the agent's native planner, after you decide scope (all / 🔴 only / split into PRs).

> **Important:** the skill is **read-only**. It never modifies code, configuration, or infrastructure automatically. The report is designed to feed your agent's native planner (Claude Code / Codex).

### When to use

- **Before any deploy** of new features
- After touching: auth, session, cookies, upload, SQL/NoSQL queries, external fetches, CORS, financial operations, webhooks, LLM integrations
- When asked for a "security review" or "check for vulns"

### Coverage (summary)

Same 12+ categories as above — see the Portuguese table.

---

## Sources

The checklist is grounded on:
- **OWASP Top 10:2021** — [A04 Insecure Design](https://owasp.org/Top10/2021/A04_2021-Insecure_Design/), [A05 Security Misconfiguration](https://owasp.org/Top10/2021/A05_2021-Security_Misconfiguration/), [A08 Software/Data Integrity Failures](https://owasp.org/Top10/2021/A08_2021-Software_and_Data_Integrity_Failures/), [A09 Logging Failures](https://owasp.org/Top10/2021/A09_2021-Security_Logging_and_Monitoring_Failures/)
- **OWASP Cheat Sheets** — [Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html), [Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html), [HTTP Headers](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html), [Forgot Password](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html), [Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html), [Logging Vocabulary](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Vocabulary_Cheat_Sheet.html), [Authentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- Recurring exploitation patterns observed in modern web apps (especially "vibe coded" ones)
- LGPD/GDPR baseline for data subject rights endpoints and retention covering logs+backups

## Caveats

- This is an **assistive** skill — it doesn't replace a real pentest, security engineer, or formal audit
- It scans the **diff/branch only**, not the whole codebase
- "No issues found" doesn't mean the code is secure — it means none of the patterns in the checklist matched
- Some flagged items may be intentional in your context — always review before applying suggested fixes

## Acknowledgments

- Structural ideas (4-phase workflow, 12-question self-check, NoSQL/XSS coverage) were adapted from [vitormiziara/saas-security](https://github.com/vitormiziara/saas-security).
- The expansion into **A04 Insecure Design** (backend-side business rules), **broader race condition scenarios** (likes/follows/unique resources/single-use invites), **A10 Exception Handling**, **A09 structured logging**, **LGPD/GDPR baseline** and **CI/CD integrity** was informed by **Tiago's security checklist** ([@construindoumastartup](https://instagram.com/construindoumastartup)). Items were cross-referenced with OWASP Cheat Sheets before being incorporated.

## License

MIT
