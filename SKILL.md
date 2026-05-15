---
name: security-check
description: Faz review de segurança das mudanças pendentes na branch atual aplicando checklist anti-vibe-coding + práticas OWASP. Use quando quiser auditar brechas antes de fazer deploy, ou após implementar feature sensível (auth, upload, query, fetch externo, operação financeira, webhook, LLM).
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash
---

# Security Check — Review de Brechas

Você audita as mudanças pendentes na branch atual buscando vulnerabilidades comuns. Combina o foco do `/security-review` (auditar diff da branch) com:
1. Checklist consolidado de padrões recorrentes de exploração em aplicações web modernas (especialmente "vibe coded")
2. Práticas validadas do **OWASP Cheat Sheets** e **OWASP Secure Headers Project**

## Quando usar

- ANTES de qualquer deploy de feature nova
- Após mexer em: auth, sessão, cookies, upload, query SQL/NoSQL, fetch externo, CORS, operações financeiras/aprovações, webhooks, integração com LLM
- Quando o usuário pedir "review de segurança", "checa brechas", "audita esse código"

## Workflow (4 fases)

### Fase 1 — Reconhecimento

Antes de checar qualquer coisa, mapeie o terreno:
- **Stack**: linguagem, framework, ORM, lib de auth, DB
- **Trust boundaries**: o que vem do usuário? o que vem de serviços internos? o que vem de webhooks?
- **Superfícies críticas**: endpoints de auth, fluxos de pagamento, uploads, rotas admin, chamadas a APIs externas, integração com LLM
- **Escopo do diff**:
  ```bash
  git status
  git diff --stat
  git diff main...HEAD --stat 2>/dev/null || git diff origin/main...HEAD --stat 2>/dev/null
  ```

Se houver mudanças staged/unstaged + commits na branch, audite **tudo junto**. Se a branch tem muitos arquivos, priorize por sensibilidade: auth/session > input handling > fetches externos > resto.

### Fase 2 — Auditar contra o checklist

Para cada arquivo modificado, busque os padrões abaixo. Use `grep` e `Read` — não delegue para subagent (queremos verificação direta no diff).

#### 🔴 Access Control (IDOR + Authz)
- [ ] Em CADA operação de read/edit/delete: o backend verifica que o usuário autenticado é dono ou tem permissão sobre o resource específico (não confia em ID do request)
- [ ] **IDOR test**: mentalmente trocar o ID do request por outro user — o backend rejeita?
- [ ] **Privilege escalation vertical**: usuário comum não consegue chegar em rota admin
- [ ] **Privilege escalation horizontal**: user A não consegue agir como user B
- [ ] **Path traversal**: endpoints de file/diretório bloqueiam `../` e absolute paths
- [ ] Tokens/sessões invalidados **no servidor** no logout (não só no cliente)

#### 🔴 Auth & Secrets
- [ ] `grep -rn "process.env.*||" <arquivos>` — `JWT_SECRET || 'default'` é brecha crítica, deve crashar se não configurado
- [ ] Middleware de auth valida o token de verdade (JWT verify com signature + expiration + audience + issuer), não só checa existência do cookie
- [ ] Cookies: `httpOnly: true`, `secure: true` em prod, `sameSite: 'strict'` ou `lax`
- [ ] **Password hashing**: Argon2id (m=19MiB, t=2, p=1) ou bcrypt 10+ rounds — sinalizar qualquer MD5/SHA-1/SHA-256 puro [OWASP]
- [ ] **Anti-enumeração**: mensagens genéricas em login ("credenciais inválidas"), registro ("se este email não está registrado, você receberá um link") e recuperação de senha ("se este email está registrado, enviaremos instruções")
- [ ] **Rate limiting** em login com lockout progressivo
- [ ] **Access tokens curtos**: 15-30 min (sensíveis: 5-15 min) [OWASP]
- [ ] **Refresh token rotation** implementado (single-use, novo refresh emitido a cada uso) [OWASP]
- [ ] **CSPRNG** para gerar tokens/session IDs/códigos (`crypto.randomBytes`, não `Math.random()`)
- [ ] **Constant-time comparison** para validar tokens (`crypto.timingSafeEqual`, não `===`) — anti timing attack
- [ ] **Session ID regenerado** após login ou mudança de privilégio (anti session fixation)
- [ ] Nunca logar passwords, tokens, keys, dados de cartão

#### 🔴 Input Validation & Injection
- [ ] `grep -rn "sql.raw\|sql\`" <arquivos>` — qualquer interpolação de input em raw SQL é red flag
- [ ] **NoSQL injection**: validar que input não tem operadores como `$where`, `$regex`, `$ne` (MongoDB)
- [ ] Queries LIKE escapam `%`, `_`, `\` antes de usar
- [ ] **XSS**: `grep -rn "innerHTML\|dangerouslySetInnerHTML\|v-html" <arquivos>` — qualquer um desses com dado de usuário é red flag
- [ ] **Command injection**: nenhum `exec()`, `eval()`, `shell_exec()` com input do usuário
- [ ] Uploads validam **Magic Bytes** do arquivo, não só MIME type do header
- [ ] Server tem `bodyLimit` definido (default 1MB para JSON)
- [ ] Validação de comprimento de URLs (max 2048 chars) e campos de texto (max length explícito)
- [ ] **Pagination com max page_size** — nunca permitir requisitar 999999 records
- [ ] Rate limiting granular em rotas sensíveis (login, upload, financeiro)

#### 🔴 SSRF Protection (qualquer fetch que use URL de input)
- [ ] Bloqueia RFC1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- [ ] Bloqueia loopback: `127.0.0.0/8`, `::1`, `localhost`
- [ ] Bloqueia link-local `169.254.0.0/16` (metadata AWS/GCP) e CGNAT `100.64.0.0/10`
- [ ] Timeout curto (10s) em fetches externos
- [ ] Verifica IP **após** DNS resolution (anti DNS rebinding)
- [ ] URLs validadas em protocolo (`https://` only, nunca `file://` ou `gopher://`) e domínio (allowlist quando possível)

#### 🔴 Race Conditions
- [ ] `grep -rn "findOne\|findFirst\|SELECT" <arquivos>` seguido de update — read-then-write em dados concorrentes precisa ser atômico
- [ ] **Atomic UPDATE com WHERE**: `UPDATE wallets SET balance = balance - $1 WHERE user_id = $2 AND balance >= $1` (em vez de SELECT depois UPDATE)
- [ ] **SELECT FOR UPDATE** dentro de transaction quando precisar ler antes de escrever
- [ ] **UNIQUE CONSTRAINT** no DB como camada extra para coupons single-use, likes, slugs únicos
- [ ] **Idempotency keys** em endpoints de pagamento (cliente envia UUID, servidor cacheia resultado por 24h)
- [ ] **Optimistic locking** com coluna `version` para resources não-financeiros (documentos, configs)
- [ ] Postgres: usar `jsonb || jsonb` para merge atômico de metadata, não ler→modificar→escrever

#### 🔴 Secrets & Configuration
- [ ] Nenhum API key, token ou password em código-fonte ou commit history (`git grep -i "password\|secret\|token\|api_key"`)
- [ ] `.env` em `.gitignore`, `.env.example` com valores fictícios
- [ ] Sem credenciais default em qualquer environment
- [ ] Sem stack traces em error responses de prod
- [ ] Endpoints de debug (`/debug`, `/__status`) desabilitados em prod
- [ ] DB roles com mínimo privilégio por serviço (read-only quando não precisa escrever)
- [ ] Container não roda como root

#### 🟠 Security Headers (todos no response, validar com helmet ou equivalente)
Lista validada [OWASP Secure Headers Project]:
- [ ] **Content-Security-Policy** — restritivo: `default-src 'self'; script-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'`
- [ ] **Strict-Transport-Security**: `max-age=63072000; includeSubDomains; preload` (2 anos)
- [ ] **X-Frame-Options**: `DENY` (ou usar `frame-ancestors` no CSP)
- [ ] **X-Content-Type-Options**: `nosniff`
- [ ] **Referrer-Policy**: `strict-origin-when-cross-origin`
- [ ] **Permissions-Policy**: `geolocation=(), camera=(), microphone=(), payment=(), usb=()`
- [ ] CORS com origin explícita, nunca `*` com credentials, sem localhost em prod
- [ ] SSE/EventSource com `withCredentials: true` + `Access-Control-Allow-Credentials: true`
- [ ] TLS 1.2+ only (TLS 1.0/1.1 desabilitados)

#### 🟠 File Upload
- [ ] MIME type validado por **magic bytes**, não só header
- [ ] Tamanho máximo enforced
- [ ] **Allowlist** de tipos permitidos (não blocklist)
- [ ] Filename original descartado, substituído por UUID
- [ ] Armazenado fora do webroot (S3, GCS, etc.)
- [ ] Webserver não pode executar arquivos do upload directory

#### 🟠 Webhooks & Supply Chain
- [ ] **Webhooks validam signature/origin** (HMAC, header `X-Hub-Signature-256` etc.)
- [ ] Lockfile commitado (`package-lock.json`, `yarn.lock`)
- [ ] `npm audit` / `pip-audit` sem HIGH/CRITICAL
- [ ] **SRI hashes** em scripts carregados de CDN (`<script integrity="sha384-...">`)
- [ ] Sem deserialização de dados não-confiáveis sem validação estrita

#### 🟠 AI/LLM Security (se aplicável)
- [ ] Input do usuário **não vai raw para o LLM** — usar delimitadores, escaping, ou filtros de injection
- [ ] LLM tools com mínimo privilégio (não dar acesso a DB write se só precisa ler)
- [ ] Nenhum `bypassPermissions`, `--approval-mode yolo`, `--dangerously-skip-permissions` em produção
- [ ] Output do LLM sanitizado antes de renderizar/executar
- [ ] Interações do LLM com dado de usuário logadas

#### 🟡 Logging & Monitoring
- [ ] Login failures logados com IP + user-agent + timestamp
- [ ] Operações financeiras logadas
- [ ] Access denied events logados
- [ ] Nenhum password/token nos logs
- [ ] Logs em local tamper-resistant

#### 🟢 Defesas Extras
- [ ] **Honey pots**: rotas falsas que logam acessos:
  ```
  /admin, /wp-admin, /phpmyadmin, /.env, /config, /api/v1/internal, /admin-panel
  ```
- [ ] Delay (2-3s) em honey pots para desperdiçar tempo do atacante
- [ ] Logging de IP, user-agent e headers em acessos suspeitos

#### CVEs específicos
- [ ] **Next.js**: Se `package.json` tem next < 14.2.25, sinalizar CVE-2025-29927 (middleware bypass via header `x-middleware-subrequest`)
- [ ] **MCP servers**: validar integridade de tool schemas (tool poisoning, session hijacking)

### Fase 3 — Self-check (12 perguntas, antes de fechar o report)

Para cada feature/endpoint mexido, mentalmente simule:

1. E se eu trocar o ID por outro de um user diferente?
2. E se eu mandar 100 requisições idênticas em paralelo?
3. E se eu mandar um campo com 1 milhão de caracteres?
4. E se eu colocar `<script>alert(1)</script>` em qualquer campo?
5. E se eu mandar `' OR 1=1 --` em qualquer campo?
6. E se eu acessar o endpoint sem estar logado?
7. E se eu forjar/manipular o token?
8. E se eu mandar uma URL externa onde se espera uma interna?
9. E se eu tentar a mesma operação financeira 2x ao mesmo tempo?
10. E se eu acessar/editar/deletar um resource que não é meu?
11. E se eu fizer upload de `.exe` renomeado pra `.jpg`?
12. E se eu inspecionar a response e achar dados de outros users?

Se alguma pergunta não tem resposta clara no código, é um achado.

### Fase 4 — Reportar

Estruture cada finding assim:

```
[CATEGORIA] Severidade: 🔴 CRÍTICO | 🟠 ALTO | 🟡 MÉDIO | 🟢 INFO
Arquivo: <arquivo:linha>
Evidência: <snippet de código>
Por que importa: <impacto em 1 frase — o que o atacante consegue>
Cenário de ataque: <como o atacante explora>
Fix: <mudança concreta de código/config>
```

Report final:

```
## Security Check — <branch>

### 🔴 CRÍTICO (bloqueia deploy)
<findings com formato acima>

### 🟠 ALTO (corrigir antes do próximo release)
...

### 🟡 MÉDIO
...

### 🟢 RECOMENDAÇÕES
...

### ✅ Verificado OK
- <lista do que foi auditado e está correto>
```

Se não achar nada crítico, diga explicitamente: **"Nenhuma brecha crítica encontrada"** e liste o que foi auditado. Não invente problemas para parecer útil.

### Fase 5 — Oferecer fix

Para cada item CRÍTICO ou ALTO, pergunte se o usuário quer que você corrija agora. Não corrija sem confirmação — algumas "brechas" podem ser intencionais no contexto.

## Princípios

- **Defesa em profundidade**: cada camada deve ser independentemente segura. Frontend valida + backend valida + DB tem constraint.
- **Fail closed**: se algo der errado, sistema **nega acesso por default**. Erros não podem abrir brechas.
- **Never trust the frontend**: tudo do cliente é input não-confiável (headers, cookies, body, MIME types, filenames).
- **Verifique no código, não na memória**: o checklist guia o que procurar, mas a vulnerabilidade tem que estar no diff atual.
- **Não cause falso pânico**: se não tem certeza se é exploitable, classifique como 🟡 e explique o cenário de ataque.
- **Foque no que mudou**: não audite a base inteira — só o que está pendente para deploy.
- **Cite linhas específicas**: sempre formato `arquivo.ts:42` para o usuário navegar direto.

## Histórico do checklist

Este checklist combina:
1. **Padrões recorrentes de exploração** em aplicações web modernas — particularmente as construídas com "vibe coding": `JWT_SECRET` com fallback default, middleware que só checa existência de cookie, SSRF sem bloqueio de IPs internos, race conditions em operações financeiras.
2. **OWASP Cheat Sheets** ([Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html), [Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html), [HTTP Headers](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)).
3. **OWASP Secure Headers Project** — lista oficial de headers recomendados.
4. **Padrões consolidados** (timing-safe comparison, idempotency keys, SELECT FOR UPDATE, refresh token rotation).

A regra é: aplicar todos os itens **desde o início**, não deixar para "depois do MVP".
