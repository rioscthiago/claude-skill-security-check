# /security-check — Skill para Claude Code

> Auditoria de segurança das mudanças pendentes na branch atual, em uma única invocação.

Combina o foco do `/security-review` (built-in do Claude Code) com:

1. **Checklist anti-vibe-coding** — padrões recorrentes de exploração em apps modernos
2. **Práticas validadas do OWASP** (Cheat Sheets + Secure Headers Project)

Cobre 12+ categorias: Access Control (IDOR, privilege escalation), Auth & Secrets, Input Validation (SQL/NoSQL/XSS), SSRF, Race Conditions, Security Headers, File Upload, Webhooks, AI/LLM Security, e mais.

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
4. **Reportar achados** — cada um com severidade (🔴/🟠/🟡/🟢), arquivo:linha, evidência, cenário de ataque e fix sugerido
5. **Oferecer correção** — pergunta se quer que aplique cada fix

### Quando usar

- **Antes de qualquer deploy** de feature nova
- Após mexer em: auth, sessão, cookies, upload, query SQL/NoSQL, fetch externo, CORS, operações financeiras, webhooks, integração com LLM
- Quando alguém pedir "review de segurança" ou "checa brechas"

### O que a skill checa (resumo)

| Categoria | Exemplos |
|---|---|
| 🔴 Access Control | IDOR, privilege escalation (vertical/horizontal), path traversal |
| 🔴 Auth & Secrets | Fallback em secrets, JWT validation, Argon2id, anti-enumeração, refresh token rotation |
| 🔴 Input Validation | SQL/NoSQL injection, XSS via `innerHTML`/`dangerouslySetInnerHTML`, command injection |
| 🔴 SSRF | RFC1918, loopback, link-local, CGNAT, DNS rebinding |
| 🔴 Race Conditions | Atomic UPDATE, SELECT FOR UPDATE, idempotency keys, UNIQUE constraint |
| 🟠 Security Headers | CSP, HSTS, X-Frame-Options, Permissions-Policy (configs OWASP) |
| 🟠 File Upload | Magic bytes, allowlist, UUID rename, fora do webroot |
| 🟠 Webhooks & Supply Chain | HMAC validation, lockfile, npm audit, SRI |
| 🟠 AI/LLM | Prompt injection, no `bypassPermissions` em prod |
| 🟡 Logging | IP/UA em failures, sem secrets nos logs |
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
4. **Report findings** — each with severity (🔴/🟠/🟡/🟢), file:line, evidence, attack scenario, suggested fix
5. **Offer fixes** — asks before applying each one

### When to use

- **Before any deploy** of new features
- After touching: auth, session, cookies, upload, SQL/NoSQL queries, external fetches, CORS, financial operations, webhooks, LLM integrations
- When asked for a "security review" or "check for vulns"

### Coverage (summary)

Same 12+ categories as above — see the Portuguese table.

---

## Sources

The checklist is grounded on:
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- Recurring exploitation patterns observed in modern web apps (especially "vibe coded" ones)

## Caveats

- This is an **assistive** skill — it doesn't replace a real pentest, security engineer, or formal audit
- It scans the **diff/branch only**, not the whole codebase
- "No issues found" doesn't mean the code is secure — it means none of the patterns in the checklist matched
- Some flagged items may be intentional in your context — always review before applying suggested fixes

## License

MIT
