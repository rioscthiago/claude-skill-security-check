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

## ⛔ Regra inegociável: ZERO mudanças automáticas

Esta skill **NUNCA** modifica código, configuração, infra, env vars ou dependências. O escopo é **só auditar + reportar**. A skill entrega um **relatório rico** (findings + risco de regressão + ordem sugerida + dependências + modulação) que serve como **insumo para o planejamento nativo do agente** (Claude Code / Codex / outro harness) — a skill não constrói plano de execução próprio, pois isso conflita com o planner do harness.

Fluxo:

1. **Auditar** o diff (apenas leitura: `Read`, `Grep`, `Glob`, `Bash` de inspeção)
2. **Entregar o relatório** estruturado (Fase 4)
3. **Aguardar decisão do usuário** sobre quais findings endereçar (todos / só CRÍTICO / fatiar em PRs)
4. A execução em si fica fora da skill — o agente usa o planejamento normal dele com o relatório como contexto

Se o usuário disser "corrige" no meio do relatório, confirmar uma vez: "Quer endereçar todos os CRÍTICOS de uma vez, ou prefere fatiar (ex: auth primeiro, depois input validation)?" — nunca presumir escopo.

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

#### 🔴 Business Logic / Insecure Design (OWASP A04)
Toda regra de negócio com dinheiro, permissão ou acesso a conteúdo **precisa de validação server-side explícita** — nunca depender só de o frontend esconder um botão. Para CADA feature nova, perguntar: "o que um usuário mal-intencionado tentaria explorar aqui?"
- [ ] **Saldo/estoque nunca negativo**: `UPDATE ... SET saldo = saldo - $1 WHERE user_id = $2 AND saldo >= $1` retornando rows affected
- [ ] **Cupom/código uso único por usuário**: `UNIQUE(user_id, coupon_id)` + check antes de aplicar
- [ ] **Reembolso** dentro do prazo + uma única vez por compra (status enum + janela temporal validada)
- [ ] **Acesso a conteúdo pago**: verificar transação concluída no banco, não só `user.is_premium` cacheado
- [ ] **Ação só pelo dono**: `WHERE resource.owner_id = current_user.id` em toda mutation
- [ ] **Quota/limite por plano**: count atual + max do plano, com lock pra não exceder por race
- [ ] **Toggle de estado** (publicar/arquivar/banir) valida estado anterior (não pode "republicar" o que já está publicado)
- [ ] **Workflow de aprovação**: cada transição checa que o ator tem papel certo E o objeto está no estado anterior esperado

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
- [ ] **Resposta timing-safe** em login/registro/recuperação de senha — mesma duração para usuário existente vs inexistente (usar processamento assíncrono ou delay artificial). Diferente de constant-time compare: aqui é tempo total da rota, anti-enumeração via side channel
- [ ] **MFA** disponível para ações sensíveis (acesso admin, mudança de senha/email, operações financeiras) — TOTP/WebAuthn quando viável; sinalizar como 🟡 quando ausente
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
- [ ] **URLs de imagem fornecidas pelo usuário** (avatar externo, embed, link preview, OG image) servidas via **proxy próprio de imagem** (Camo, go-camo ou equivalente) que: (a) anonimiza IP do usuário final, (b) faz validação SSRF no fetch, (c) limita número de redirects, (d) força HTTPS, (e) define tamanho máximo. Nunca embedar URL externa diretamente no `<img src>`.

#### 🔴 Race Conditions
- [ ] `grep -rn "findOne\|findFirst\|SELECT" <arquivos>` seguido de update — read-then-write em dados concorrentes precisa ser atômico
- [ ] **Atomic UPDATE com WHERE**: `UPDATE wallets SET balance = balance - $1 WHERE user_id = $2 AND balance >= $1` (em vez de SELECT depois UPDATE)
- [ ] **SELECT FOR UPDATE** dentro de transaction quando precisar ler antes de escrever
- [ ] **UNIQUE CONSTRAINT** no DB como camada extra para coupons single-use, likes, slugs únicos
- [ ] **Idempotency keys** em endpoints de pagamento (cliente envia UUID, servidor cacheia resultado por 24h)
- [ ] **Optimistic locking** com coluna `version` para resources não-financeiros (documentos, configs)
- [ ] Postgres: usar `jsonb || jsonb` para merge atômico de metadata, não ler→modificar→escrever

**Cenários que DEVEM ter proteção contra check-then-act** (não só financeiro):
- Compras, pagamentos, transferências, saques
- Cupons/descontos/promoções de uso único
- Reembolsos e estornos
- **Curtidas, votos, follows, toggles** (like/unlike, follow/unfollow) — sem UNIQUE pode duplicar
- **Criação de recursos únicos** (username, slug, email, handle) — `UNIQUE CONSTRAINT` obrigatório no DB
- **Upgrades/downgrades de plano** (prorate + ativação atômica)
- **Links de convite, tokens de reset, magic links** de uso único (marcar consumido na mesma TX que aceita)
- **Reservas/agendamentos** que dependem de slot disponível

#### 🔴 Secrets & Configuration
- [ ] Nenhum API key, token ou password em código-fonte ou commit history (`git grep -i "password\|secret\|token\|api_key"`)
- [ ] `.env` em `.gitignore`, `.env.example` com valores fictícios
- [ ] Sem credenciais default em qualquer environment
- [ ] Sem stack traces em error responses de prod
- [ ] Endpoints de debug (`/debug`, `/__status`) desabilitados em prod
- [ ] **Directory listing desabilitado** no webserver (Apache `Options -Indexes`; nginx é default-off — confirmar reverse proxy / static server / S3 bucket)
- [ ] **Ambientes não-prod (dev/staging/preview) não acessíveis publicamente**: auth básica, IP allowlist, VPN, ou domínio sem indexação
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
- [ ] **CI/CD tratado como código privilegiado**: workflows em `.github/workflows/*` (ou equivalente) passam por code review como qualquer outro código
- [ ] **Secrets no pipeline em store dedicado** (GitHub Secrets, Vault, KMS) — nunca em variável de pipeline editável por contribuidores
- [ ] **Segregação de permissões** entre estágios de build/test/deploy (build não pode fazer deploy; PR de fork não tem secrets)
- [ ] **Artefatos assinados** quando viável (cosign/sigstore, SLSA L2+) — anti SolarWinds-style supply chain attack
- [ ] **SBOM gerado** no build (CycloneDX/SPDX) para rastreabilidade de dependências

#### 🟠 Exception Handling & Fail Closed (OWASP A10)
- [ ] **Global exception handler** ativo — toda exceção não capturada vira `5xx` genérico, **nunca expõe** stack trace, SQL query, path de arquivo ou nome de tabela
- [ ] **Fail closed em auth/authz**: exceção dentro de check de permissão → negar acesso, nunca conceder por default. Verificar todos `try/catch` em rotas autenticadas
- [ ] Sem `catch (_) {}` / `except: pass` engolindo exceções silenciosamente em rotas de produção
- [ ] Mensagens de erro pro cliente são genéricas (`"Erro ao processar"`); detalhes vão pro log estruturado, não pro response
- [ ] Comportamento testado com: payload malformado, body vazio, encoding inválido, serviço dependente fora do ar, timeout de DB, payload acima do limit
- [ ] Erros em batch/job background não silenciam falhas — alertam ou marcam item como `failed`, nunca como `success`

#### 🟠 AI/LLM Security (se aplicável)
- [ ] Input do usuário **não vai raw para o LLM** — usar delimitadores, escaping, ou filtros de injection
- [ ] LLM tools com mínimo privilégio (não dar acesso a DB write se só precisa ler)
- [ ] Nenhum `bypassPermissions`, `--approval-mode yolo`, `--dangerously-skip-permissions` em produção
- [ ] Output do LLM sanitizado antes de renderizar/executar
- [ ] Interações do LLM com dado de usuário logadas

#### 🟡 Logging & Monitoring
- [ ] **Formato estruturado JSON** com schema consistente — facilita SIEM/ingest, reduz log injection. Fields recomendados pelo OWASP Logging Vocabulary: `datetime`, `level`, `event`, `appid`, `userId`, `sourceIP`, `userAgent`, `action`, `resource`, `result`, `requestId`
- [ ] **Prefixos de evento consistentes**: `authn:login_success`, `authn:login_failure`, `authz:denied`, `input:validation_failed`, `excess:rate_limit_hit`, `txn:payment_attempt`
- [ ] **Sanitizar input antes de logar** (anti log injection): escapar `\n`, `\r`, control chars que poluem multilinhas
- [ ] Login failures logados com IP + user-agent + timestamp
- [ ] Operações financeiras logadas com `userId + action + amount + resourceId + result`
- [ ] Access denied events logados (tentativas de IDOR/escalação são sinal de ataque)
- [ ] Mudanças de permissão/papel logadas
- [ ] Nenhum password/token/PII sensível nos logs
- [ ] Logs em local tamper-resistant (append-only, WORM storage, ou SIEM com retenção)

#### 🟡 Privacidade & Dados Pessoais (LGPD/GDPR — condicional)
Aplicar quando a aplicação coleta/processa dados pessoais (qualquer app com cadastro de usuário). Validar contra o stack atual:
- [ ] **Minimização de coleta**: cada campo armazenado tem uso explícito numa feature — se não, remover
- [ ] **Endpoint "ver meus dados"** (auth obrigatória, agrega de todas as tabelas que referenciam o user)
- [ ] **Endpoint correção** dos próprios dados
- [ ] **Endpoint exclusão/anonimização** com tratamento de dependências (FK orphans, cascade vs anonimize, manter dados agregados sem PII)
- [ ] **Endpoint exportação portável** em formato comum (JSON/CSV/XML) — GDPR Art. 20; prazo 1 mês (GDPR) / 15 dias (LGPD)
- [ ] **Endpoint revogar consentimento** (cookies, marketing, share com terceiros)
- [ ] **Logs e backups na política de retenção**: deletar do banco principal não basta — definir TTL em logs/backups que contêm PII
- [ ] **Dados sensíveis** (saúde, biometria, religião, orientação, dados financeiros, documento oficial) → criptografia em repouso adicional + acesso auditado
- [ ] **Notificação de incidente** documentada (quem, quando, autoridades; LGPD/ANPD e GDPR/DPA)

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

### Fase 4 — Reportar (insumo para o planner do agente)

**A skill encerra aqui.** O relatório precisa ser rico o suficiente para que o usuário decida o escopo e o agente use seu planejamento nativo (Claude Code / Codex) para executar — não duplicar o planner do harness.

Cada finding deve trazer **fix + risco de regressão + dependências + esforço + tarefas humanas associadas**:

```
[ID] [CATEGORIA] Severidade: 🔴 CRÍTICO | 🟠 ALTO | 🟡 MÉDIO | 🟢 INFO
Arquivo: <arquivo:linha>
Evidência: <snippet de código>
Por que importa: <impacto em 1 frase — o que o atacante consegue>
Cenário de ataque: <como o atacante explora>
Fix sugerido: <mudança concreta de código/config — ou "tarefa humana" se não for editável por agente>
Risco de regressão: <o que pode quebrar ao aplicar: contrato de API alterado, migration necessária, testes que vão exigir update, header novo que pode quebrar consumidor X, comportamento que clientes assumem>
Dependências: <IDs de findings que precisam vir antes, OU "nenhuma">
Esforço: S (≤1h) | M (≤4h) | L (>4h, exige refactor)
```

Estrutura do report final:

```
## Security Check — <branch>

### Sumário
- N findings: X 🔴 / Y 🟠 / Z 🟡 / W 🟢
- Arquivos auditados: <count>
- Áreas de maior risco: <auth | input | race | etc>

### 🔴 CRÍTICO (bloqueia deploy)
<findings com formato acima>

### 🟠 ALTO (corrigir antes do próximo release)
...

### 🟡 MÉDIO
...

### 🟢 RECOMENDAÇÕES
...

### 📦 Modulação sugerida (fases independentes para corrigir em PRs separados)

Quando há muitos findings, agrupar em fases lógicas que possam ser mergeadas separadamente. Princípio: dentro de uma fase, mudanças são coesas e baixo acoplamento; entre fases, respeitar dependências.

Exemplo de agrupamento (ajustar ao stack real):
- **Fase A — Segredos & Config**: não-funcional, baixo risco, deploy isolado → [F1, F2, F5]
- **Fase B — Headers & TLS**: incrementável, sem refactor → [F3, F8]
- **Fase C — Auth & Sessão**: alto impacto, exige testes de regressão → [F4, F7, F9]
- **Fase D — Input validation & Race conditions**: pode mudar contrato → [F6, F10, F11]
- **Fase E — Privacidade & Compliance**: endpoints novos, sem refactor → [F12, F13]

Ordem sugerida de execução: A → B → C → D → E (justificar quando divergir).

### ⚠️ Análise de regressão agregada
Lista de superfícies que podem quebrar e exigem atenção redobrada em testes:
- Endpoints com mudança de contrato (body/response/status)
- Migrations de schema necessárias (com plano de backfill se aplicável)
- Comportamentos que clientes assumem (TTL de token, rate limit, formato de erro)
- Headers novos que podem quebrar consumidores específicos (CSP em SPA com inline scripts, etc.)
- Testes existentes que provavelmente vão falhar e precisam atualizar

### 🧑 Tarefas que SOMENTE o humano pode fazer
(O agente pode até preparar PR, mas não consegue executar essas — listar explicitamente para não passarem despercebidas.)
- [ ] Rotacionar secrets vazados (KMS/Vault/CI)
- [ ] Configurar/atualizar WAF / CDN rules
- [ ] Criar role de DB com least-privilege e migrar serviços
- [ ] Provisionar provider de MFA (Auth0/Clerk/TOTP server)
- [ ] Habilitar assinatura de artefatos no CI (cosign/sigstore)
- [ ] Aprovar política de retenção de logs/backups (compliance/legal)
- [ ] Comunicar usuários afetados se houve vazamento
- [ ] Notificar autoridade (ANPD/DPA) se incidente reportável
- [ ] Atualizar Política de Privacidade / Termos
- [ ] Configurar alertas no SIEM para novos eventos logados

### ✅ Verificado OK
- <lista explícita do que foi auditado e está correto — evita sensação de "achou tudo">

### 👉 Próximo passo (decisão do usuário)
Pergunte ao usuário:
1. Endereçar **todos os findings**, só os 🔴, ou um conjunto específico?
2. Aplicar **tudo num PR** ou seguir a **modulação sugerida** (fases A→E)?
3. Alguma "brecha" listada é **intencional no contexto** e deve ser ignorada?

A partir das respostas, o agente usa seu planejamento normal para executar — a skill **não constrói plano próprio**, não chama TaskCreate para fases de implementação, não edita código. Apenas relatou.
```

Se não achar nada crítico, diga explicitamente: **"Nenhuma brecha crítica encontrada"** e preencha a seção "Verificado OK". Não invente problemas para parecer útil.

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
2. **OWASP Top 10:2021** ([A04 Insecure Design](https://owasp.org/Top10/2021/A04_2021-Insecure_Design/), [A05 Security Misconfiguration](https://owasp.org/Top10/2021/A05_2021-Security_Misconfiguration/), [A08 Software/Data Integrity Failures](https://owasp.org/Top10/2021/A08_2021-Software_and_Data_Integrity_Failures/), [A09 Logging Failures](https://owasp.org/Top10/2021/A09_2021-Security_Logging_and_Monitoring_Failures/)).
3. **OWASP Cheat Sheets** ([Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html), [Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html), [HTTP Headers](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html), [Forgot Password](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html), [Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html), [Logging Vocabulary](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Vocabulary_Cheat_Sheet.html), [Authentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)).
4. **OWASP Secure Headers Project** — lista oficial de headers recomendados.
5. **Padrões consolidados** (timing-safe comparison, timing-safe response anti-enumeração, idempotency keys, SELECT FOR UPDATE, refresh token rotation, SLSA/SBOM para supply chain, image proxy estilo Camo).
6. **LGPD/GDPR baseline** — endpoints de data subject rights (acesso, correção, portabilidade, exclusão, revogação) e retenção cobrindo logs/backups.
7. **Checklist de segurança do Tiago ([@construindoumastartup](https://instagram.com/construindoumastartup))** — referência usada para validar e expandir a skill nas seções de **A04 Insecure Design** (catálogo de regras de negócio backend-side), **cenários ampliados de race conditions** (likes/follows/recursos únicos/convites), **A10 Exception Handling** explícito, **A09 logging estruturado** com vocabulário consistente, **baseline LGPD/GDPR** e **CI/CD integrity**. Os pontos do checklist do Tiago foram cruzados com OWASP Cheat Sheets antes de entrarem na skill.

A regra é: aplicar todos os itens **desde o início**, não deixar para "depois do MVP". E o output desta skill é **sempre só relatório** — execução vem do planner do agente após o usuário definir escopo.
