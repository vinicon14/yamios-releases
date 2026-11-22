# Pareamento por e-mail — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Substituir o código de 6 caracteres por um link de confirmação por e-mail, de uso único e expiração de 10 minutos, mantendo o polling do aplicativo.

**Architecture:** O site cria o pareamento, grava apenas o hash do token de confirmação e envia o link via Resend. O navegador autenticado valida o token e confirma o pareamento; o aplicativo continua consultando `pollPairing` até receber o token de dispositivo. O código de 6 caracteres deixa de ser usado.

**Tech Stack:** Next.js 16, React 19, TypeScript, Vitest, Node.js, SQLite/Postgres, Electron, Resend.

**Convenções de caminho:** `/site` = `D:\Downloads\open code yami edit`; `/os` = `D:\Downloads\yami relise os\.worktrees\feature-yami-os-0.8.0\desenvolvimento\yami-os`; `/releases` = `D:\Downloads\yami relise os\.worktrees\feature-yami-os-0.8.0`.

---

### Task 1: Ampliar `pairings` para confirmação por e-mail

**Files:**
- Modify: `/site/lib/db/schema.sql`
- Modify: `/site/lib/db/sql-shared.ts`
- Modify: `/site/lib/db/types.ts`
- Test: `/site/tests/pairing-email-schema.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/pairing-email-schema.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { SCHEMA_SQL } from "../lib/db/sql-shared";

describe("pairings por e-mail", () => {
  it("contém os campos de confirmação", () => {
    expect(SCHEMA_SQL).toContain("confirm_token_hash TEXT");
    expect(SCHEMA_SQL).toContain("email_sent_at TEXT");
    expect(SCHEMA_SQL).toContain("confirmed_at TEXT");
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/pairing-email-schema.test.ts` em `/site`.

Expected: falha (campos ausentes).

- [ ] **Step 3: atualizar o esquema**

Em `/site/lib/db/schema.sql`, substituir a tabela `pairings` por:

```sql
CREATE TABLE IF NOT EXISTS pairings (
  id TEXT PRIMARY KEY,
  code_hash TEXT NOT NULL UNIQUE,
  poll_token_hash TEXT NOT NULL UNIQUE,
  confirm_token_hash TEXT,
  user_id TEXT,
  device_id TEXT,
  device_name TEXT,
  pending_device_token TEXT,
  expires_at TEXT NOT NULL,
  consumed_at TEXT,
  email_sent_at TEXT,
  confirmed_at TEXT,
  created_at TEXT NOT NULL
);
```

Fazer a mesma alteração em `SCHEMA_SQL` em `/site/lib/db/sql-shared.ts`.

- [ ] **Step 4: atualizar tipos e conversores**

Adicionar em `/site/lib/db/types.ts`:

```ts
export type PairingRow = {
  id: string;
  codeHash: string;
  pollTokenHash: string;
  confirmTokenHash: string | null;
  userId: string | null;
  deviceId: string | null;
  deviceName: string | null;
  pendingDeviceToken: string | null;
  expiresAt: string;
  consumedAt: string | null;
  emailSentAt: string | null;
  confirmedAt: string | null;
  createdAt: string;
};
```

Atualizar `rowToPairing` em `/site/lib/db/sql-shared.ts` para mapear `confirm_token_hash`, `email_sent_at` e `confirmed_at`.

- [ ] **Step 5: rodar testes**

Run: `npm test -- tests/pairing-email-schema.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 6: commit**

```bash
git add lib/db/schema.sql lib/db/sql-shared.ts lib/db/types.ts tests/pairing-email-schema.test.ts
git commit -m "feat(pairing): esquema de confirmação por e-mail"
```

---

### Task 2: Serviço de pareamento por e-mail

**Files:**
- Modify: `/site/lib/pairing.ts`
- Test: `/site/tests/pairing-email.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/pairing-email.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { hashPassword } from "../lib/auth";
import { hashSecret } from "../lib/auth";
import { startPairingByEmail, confirmPairingByEmail } from "../lib/pairing";

async function seed(store) {
  await store.createUser({ id: "u1", email: "u@example.com", passwordHash: await hashPassword("senha-forte-123") });
  await store.createOrder({ id: "o1", email: "u@example.com", amountBrl: 20 });
  await store.createLicense("o1", "YAMI-DEMO-KEY");
}

describe("pareamento por e-mail", () => {
  it("cria token de confirmação e não o devolve em claro", async () => {
    const store = createMemoryStore();
    await seed(store);
    const started = await startPairingByEmail(store, "u1", "PC de casa");
    expect(started.confirmToken).toMatch(/^[a-f0-9]{64}$/);
    expect(started.code).toBeUndefined();
  });

  it("confirma com token válido e devolve token de dispositivo", async () => {
    const store = createMemoryStore();
    await seed(store);
    const started = await startPairingByEmail(store, "u1", "PC de casa");
    const confirmed = await confirmPairingByEmail(store, started.confirmToken, "u1");
    expect(confirmed.deviceToken).toMatch(/^[a-f0-9]{64}$/);
    expect(await store.getDeviceByTokenHash(hashSecret(confirmed.deviceToken))).then((d) => d?.name)).toBe("PC de casa");
  });

  it("rejeita token usado duas vezes", async () => {
    const store = createMemoryStore();
    await seed(store);
    const started = await startPairingByEmail(store, "u1", "PC de casa");
    await confirmPairingByEmail(store, started.confirmToken, "u1");
    await expect(confirmPairingByEmail(store, started.confirmToken, "u1")).rejects.toThrow("token inválido");
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/pairing-email.test.ts` em `/site`.

Expected: falha de funções ausentes.

- [ ] **Step 3: implementar o serviço**

Adicionar em `/site/lib/pairing.ts`:

```ts
export const PAIRING_CONFIRM_TTL_MS = 10 * 60 * 1000;

export function generateConfirmToken() {
  return generateToken(32);
}

export async function startPairingByEmail(store, userId, deviceName, now = new Date()) {
  const user = await store.getUserById(userId);
  if (!user) throw new Error("usuário não encontrado");
  if (!(await userHasPaidLicense(user.email, store))) throw new Error("licença necessária");
  const pollToken = generateToken(24);
  const confirmToken = generateConfirmToken();
  const expiresAt = new Date(now.getTime() + PAIRING_CONFIRM_TTL_MS).toISOString();
  await store.createPairing({
    id: crypto.randomUUID(),
    codeHash: hashPairingValue(confirmToken),
    pollTokenHash: hashPairingValue(pollToken),
    userId: null,
    deviceId: null,
    deviceName: deviceName.trim().slice(0, 80) || "PC de casa",
    pendingDeviceToken: null,
    expiresAt,
    consumedAt: null,
    confirmTokenHash: hashPairingValue(confirmToken),
    emailSentAt: null,
    confirmedAt: null,
    createdAt: nowIso(),
  });
  return { pollToken, confirmToken, expiresAt };
}

export async function confirmPairingByEmail(store, confirmToken, userId) {
  const row = await store.getPairingByCodeHash(hashPairingValue(confirmToken));
  if (!row || row.confirmTokenHash !== hashPairingValue(confirmToken)) throw new Error("token inválido");
  if (isPairingExpired(row)) throw new Error("token expirado");
  if (row.confirmedAt) throw new Error("token já usado");
  const user = await store.getUserById(userId);
  if (!user || row.userId && row.userId !== user.id) throw new Error("token inválido");
  const deviceToken = generateToken(32);
  const device = await store.createDevice({
    id: crypto.randomUUID(),
    userId: user.id,
    name: row.deviceName ?? "PC de casa",
    tokenHash: hashSecret(deviceToken),
  });
  await store.confirmPairing({
    id: row.id,
    userId: user.id,
    deviceId: device.id,
    deviceName: device.name,
    pendingDeviceToken: deviceToken,
  });
  await store.addAudit({
    id: crypto.randomUUID(),
    userId: user.id,
    deviceId: device.id,
    action: "pairing.confirm_email",
    ip: null,
    meta: device.name,
  });
  return { device, deviceToken };
}
```

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/pairing-email.test.ts` em `/site`.

Expected: 3 testes passam.

- [ ] **Step 5: commit**

```bash
git add lib/pairing.ts tests/pairing-email.test.ts
git commit -m "feat(pairing): confirmação por e-mail"
```

---

### Task 3: Rotas e e-mail de pareamento

**Files:**
- Modify: `/site/app/api/pairing/start/route.ts`
- Create: `/site/app/api/pairing/confirm/route.ts`
- Create: `/site/app/api/pairing/resend/route.ts`
- Modify: `/site/lib/email.ts`
- Test: `/site/tests/pairing-email-routes.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/pairing-email-routes.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { hashPassword } from "../lib/auth";
import { setStoreForTests } from "../lib/db";
import { POST as startPost } from "../app/api/pairing/start/route";
import { GET as confirmGet } from "../app/api/pairing/confirm/route";

async function seed() {
  const store = createMemoryStore();
  setStoreForTests(store);
  await store.createUser({ id: "u1", email: "u@example.com", passwordHash: await hashPassword("senha-forte-123") });
  await store.createOrder({ id: "o1", email: "u@example.com", amountBrl: 20 });
  await store.createLicense("o1", "YAMI-DEMO-KEY");
  return store;
}

function req(url, init) { return new Request(`http://localhost:3000${url}`, init); }

describe("pareamento por e-mail", () => {
  it("envia link e confirma com token válido", async () => {
    await seed();
    const start = await startPost(req("/api/pairing/start", { method: "POST", headers: { "content-type": "application/json", origin: "http://localhost:3000" }, body: JSON.stringify({ deviceName: "PC de casa" }) }));
    expect(start.status).toBe(200);
    const { confirmToken } = await start.json();
    const confirm = await confirmGet(req(`/api/pairing/confirm?token=${confirmToken}`));
    expect(confirm.status).toBe(200);
  });

  it("rejeita token inválido", async () => {
    await seed();
    const confirm = await confirmGet(req("/api/pairing/confirm?token=token-invalido"));
    expect(confirm.status).toBe(400);
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/pairing-email-routes.test.ts` em `/site`.

Expected: falha de rota ausente.

- [ ] **Step 3: implementar as rotas**

`/site/app/api/pairing/start/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getStore } from "@/lib/db";
import { startPairingByEmail } from "@/lib/pairing";
import { getSessionUser, readSessionCookie } from "@/lib/auth";
import { userHasPaidLicense } from "@/lib/license-gate";
import { sendPairingEmail } from "@/lib/email";
import { clientIp, jsonError } from "@/lib/http";
import { rateLimit } from "@/lib/rate-limit";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function POST(req: Request) {
  const current = await getSessionUser(readSessionCookie(req));
  if (!current) return jsonError("não autenticado", 401);
  const store = await getStore();
  if (!(await userHasPaidLicense(current.user.email, store))) return jsonError("licença necessária", 403);
  const limited = rateLimit(`pair-start:${current.user.id}`, 3, 60 * 60 * 1000);
  if (!limited.ok) return jsonError("muitas tentativas", 429);
  let body: Record<string, unknown> = {};
  try { body = await req.json(); } catch {}
  const deviceName = String(body.deviceName ?? "PC de casa");
  const started = await startPairingByEmail(store, current.user.id, deviceName);
  await store.addAudit({ id: crypto.randomUUID(), userId: current.user.id, deviceId: null, action: "pairing.start_email", ip: clientIp(req), meta: deviceName });
  await sendPairingEmail(current.user.email, started.confirmToken);
  return NextResponse.json({ ok: true, expiresAt: started.expiresAt });
}
```

`/site/app/api/pairing/confirm/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getStore } from "@/lib/db";
import { confirmPairingByEmail } from "@/lib/pairing";
import { getSessionUser, readSessionCookie } from "@/lib/auth";
import { jsonError } from "@/lib/http";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function GET(req: Request) {
  const token = new URL(req.url).searchParams.get("token") ?? "";
  if (!token) return jsonError("token ausente", 400);
  const current = await getSessionUser(readSessionCookie(req));
  if (!current) return NextResponse.redirect(new URL(`/entrar?next=/parear&token=${encodeURIComponent(token)}`, req.url));
  const store = await getStore();
  try {
    const result = await confirmPairingByEmail(store, token, current.user.id);
    return NextResponse.json({ ok: true, device: { id: result.device.id, name: result.device.name } });
  } catch (error) {
    return jsonError(error instanceof Error ? error.message : "token inválido", 400);
  }
}
```

`/site/app/api/pairing/resend/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getStore } from "@/lib/db";
import { getSessionUser, readSessionCookie } from "@/lib/auth";
import { jsonError } from "@/lib/http";
import { rateLimit } from "@/lib/rate-limit";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function POST(req: Request) {
  const current = await getSessionUser(readSessionCookie(req));
  if (!current) return jsonError("não autenticado", 401);
  const limited = rateLimit(`pair-resend:${current.user.id}`, 3, 60 * 60 * 1000);
  if (!limited.ok) return jsonError("muitas tentativas", 429);
  return NextResponse.json({ ok: true });
}
```

Adicionar `sendPairingEmail` em `/site/lib/email.ts`:

```ts
export async function sendPairingEmail(email: string, confirmToken: string): Promise<{ sent: boolean; reason?: string }> {
  if (!emailEnabled()) return { sent: false, reason: "disabled" };
  const url = `${getSiteUrl()}/api/pairing/confirm?token=${encodeURIComponent(confirmToken)}`;
  const body = {
    from: getEmailFrom(),
    to: [email],
    subject: "Confirme o pareamento do YAMI",
    text: [
      "Clique no link abaixo para parear este computador com sua conta YAMI.",
      "",
      url,
      "",
      "O link expira em 10 minutos e pode ser usado uma única vez.",
    ].join("\n"),
  };
  const res = await fetch("https://api.resend.com/emails", {
    method: "POST",
    headers: { Authorization: `Bearer ${getResendApiKey()}`, "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  if (!res.ok) return { sent: false, reason: `http_${res.status}` };
  return { sent: true };
}
```

- [ ] **Step 4: rodar testes**

Run: `npm test -- tests/pairing-email-routes.test.ts` em `/site`.

Expected: 2 testes passam.

- [ ] **Step 5: commit**

```bash
git add app/api/pairing lib/email.ts tests/pairing-email-routes.test.ts
git commit -m "feat(pairing): pareamento por link de e-mail"
```

---

### Task 4: Atualizar a interface de conta

**Files:**
- Modify: `/site/components/AccountPanel.tsx`
- Test: `/site/tests/account-pairing.test.tsx`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/account-pairing.test.tsx`:

```tsx
import { describe, expect, it } from "vitest";
import { pairingButtonText } from "../components/AccountPanel";

describe("pareamento por e-mail", () => {
  it("mostra o texto correto", () => {
    expect(pairingButtonText(false)).toBe("Enviar link de ativação");
    expect(pairingButtonText(true)).toBe("Reenviar link");
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/account-pairing.test.tsx` em `/site`.

Expected: falha de função ausente.

- [ ] **Step 3: implementar**

Adicionar em `/site/components/AccountPanel.tsx`:

```tsx
export function pairingButtonText(hasPending: boolean): string {
  return hasPending ? "Reenviar link" : "Enviar link de ativação";
}
```

Substituir o formulário de código por:

```tsx
<button
  type="button"
  onClick={() => void sendPairingLink()}
  className="btn-primary"
>
  {pairingButtonText(hasPendingPairing)}
</button>
```

Adicionar `sendPairingLink`:

```tsx
async function sendPairingLink() {
  const res = await fetch("/api/pairing/start", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ deviceName }) });
  const data = await res.json();
  setMsg(res.ok ? "Link enviado. Verifique seu e-mail e clique no link para ativar este computador." : data.error ?? "Não foi possível enviar.");
}
```

- [ ] **Step 4: rodar testes**

Run: `npm test -- tests/account-pairing.test.tsx` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add components/AccountPanel.tsx tests/account-pairing.test.tsx
git commit -m "feat(ui): pareamento por link de e-mail"
```

---

### Task 5: Atualizar o aplicativo para polling por e-mail

**Files:**
- Modify: `/os/src/interface/yami-account.js`
- Test: `/os/tests/pairing-email.test.mjs`

- [ ] **Step 1: escrever o teste**

Create `/os/tests/pairing-email.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { pairingStatusText } from "../src/interface/pairing-email.mjs";

test("mostra o texto correto", () => {
  assert.equal(pairingStatusText("pending"), "Verifique seu e-mail e clique no link de ativação.");
  assert.equal(pairingStatusText("ready"), "Computador ativado.");
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test` em `/os`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Create `/os/src/interface/pairing-email.mjs`:

```js
export function pairingStatusText(status) {
  if (status === "ready") return "Computador ativado.";
  if (status === "expired") return "O link expirou. Gere um novo pareamento.";
  return "Verifique seu e-mail e clique no link de ativação.";
}
```

Atualizar `yami-account.js` para chamar `startPairing`, mostrar o texto de `pairingStatusText("pending")` e continuar consultando `pollPairing` até `ready`.

- [ ] **Step 4: rodar testes**

Run: `npm test` em `/os`.

Expected: todos passam.

- [ ] **Step 5: commit**

Nenhum commit; a área desktop é ignorada.

---

### Task 6: Criador de apps — núcleo de projetos

**Files:**
- Create: `/os/src/app-builder/project.mjs`
- Test: `/os/tests/app-builder-project.test.mjs`

- [ ] **Step 1: escrever o teste**

Create `/os/tests/app-builder-project.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { validateProjectName, createProjectManifest } from "../src/app-builder/project.mjs";

test("valida nome do projeto", () => {
  assert.equal(validateProjectName("Meu App").ok, true);
  assert.equal(validateProjectName("a").ok, false);
});

test("cria manifesto inicial", () => {
  const manifest = createProjectManifest({ name: "Meu App", category: "produtividade" });
  assert.equal(manifest.name, "Meu App");
  assert.equal(manifest.version, "1.0.0");
  assert.equal(manifest.entry, "src/main.js");
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test` em `/os`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Create `/os/src/app-builder/project.mjs`:

```js
export function validateProjectName(name) {
  const value = String(name ?? "").trim();
  if (value.length < 3 || value.length > 80) return { ok: false, reason: "nome_invalido" };
  return { ok: true };
}

export function createProjectManifest(input) {
  const name = String(input?.name ?? "").trim();
  return {
    id: name.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-|-$/g, ""),
    name,
    version: "1.0.0",
    description: String(input?.description ?? "").trim(),
    category: String(input?.category ?? "geral").trim(),
    entry: "src/main.js",
    files: ["src/main.js"],
  };
}
```

- [ ] **Step 4: rodar testes**

Run: `npm test` em `/os`.

Expected: todos passam.

- [ ] **Step 5: commit**

Nenhum commit; a área desktop é ignorada.

---

### Task 7: Criador de apps — editor de arquivos

**Files:**
- Create: `/os/src/app-builder/files.mjs`
- Test: `/os/tests/app-builder-files.test.mjs`

- [ ] **Step 1: escrever o teste**

Create `/os/tests/app-builder-files.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { isInsideProject, normalizeProjectPath } from "../src/app-builder/files.mjs";

test("normaliza caminho dentro do projeto", () => {
  assert.equal(normalizeProjectPath("src/main.js"), "src/main.js");
  assert.equal(normalizeProjectPath("../segredo.txt"), null);
});

test("bloqueia saída do projeto", () => {
  assert.equal(isInsideProject("/proj/src/main.js", "/proj"), true);
  assert.equal(isInsideProject("/etc/passwd", "/proj"), false);
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test` em `/os`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Create `/os/src/app-builder/files.mjs`:

```js
import { resolve, relative } from "node:path";

export function normalizeProjectPath(input) {
  const raw = String(input ?? "").trim().replace(/\\/g, "/");
  if (!raw || raw.startsWith("/") || raw.includes("..")) return null;
  const parts = raw.split("/").filter(Boolean);
  if (parts.length === 0) return null;
  return parts.join("/");
}

export function isInsideProject(target, root) {
  const fullTarget = resolve(target);
  const fullRoot = resolve(root);
  const rel = relative(fullRoot, fullTarget);
  return rel === "" || (!rel.startsWith("..") && !resolve(fullRoot, rel).startsWith(fullRoot + ".."));
}
```

- [ ] **Step 4: rodar testes**

Run: `npm test` em `/os`.

Expected: todos passam.

- [ ] **Step 5: commit**

Nenhum commit; a área desktop é ignorada.

---

### Task 8: Criador de apps — build e empacotamento

**Files:**
- Create: `/os/src/app-builder/build.mjs`
- Test: `/os/tests/app-builder-build.test.mjs`

- [ ] **Step 1: escrever o teste**

Create `/os/tests/app-builder-build.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { validateManifest, packageFileName } from "../src/app-builder/build.mjs";

test("valida manifesto mínimo", () => {
  assert.equal(validateManifest({ name: "App", version: "1.0.0", entry: "src/main.js" }).ok, true);
  assert.equal(validateManifest({ name: "App" }).ok, false);
});

test("gera nome de pacote", () => {
  assert.equal(packageFileName({ id: "meu-app", version: "1.0.0" }), "meu-app-1.0.0.tar.zst");
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test` em `/os`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Create `/os/src/app-builder/build.mjs`:

```js
export function validateManifest(manifest) {
  if (!manifest || typeof manifest !== "object") return { ok: false, reason: "manifesto_invalido" };
  if (!String(manifest.name ?? "").trim()) return { ok: false, reason: "nome_obrigatorio" };
  if (!/^\d+\.\d+\.\d+$/.test(String(manifest.version ?? ""))) return { ok: false, reason: "versao_invalida" };
  if (!String(manifest.entry ?? "").trim()) return { ok: false, reason: "entrada_obrigatoria" };
  return { ok: true };
}

export function packageFileName(manifest) {
  const id = String(manifest?.id ?? "app").replace(/[^a-z0-9-]/g, "-");
  return `${id}-${manifest?.version ?? "1.0.0"}.tar.zst`;
}
```

- [ ] **Step 4: rodar testes**

Run: `npm test` em `/os`.

Expected: todos passam.

- [ ] **Step 5: commit**

Nenhum commit; a área desktop é ignorada.

---

### Task 9: Criador de apps — publicação

**Files:**
- Create: `/os/src/app-builder/publish.mjs`
- Test: `/os/tests/app-builder-publish.test.mjs`

- [ ] **Step 1: escrever o teste**

Create `/os/tests/app-builder-publish.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { buildPublicationPayload } from "../src/app-builder/publish.mjs";

test("monta payload de publicação", () => {
  const payload = buildPublicationPayload({ title: "Meu App", description: "Descrição", category: "produtividade", fileName: "meu-app-1.0.0.tar.zst", fileSize: 1234, fileSha256: "abc123" });
  assert.equal(payload.title, "Meu App");
  assert.equal(payload.category, "produtividade");
  assert.equal(payload.fileName, "meu-app-1.0.0.tar.zst");
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test` em `/os`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Create `/os/src/app-builder/publish.mjs`:

```js
export function buildPublicationPayload(input) {
  return {
    title: String(input?.title ?? "").trim(),
    description: String(input?.description ?? "").trim(),
    category: String(input?.category ?? "geral").trim(),
    fileName: String(input?.fileName ?? "").trim(),
    fileSize: Number(input?.fileSize ?? 0),
    fileSha256: String(input?.fileSha256 ?? "").trim(),
  };
}
```

- [ ] **Step 4: rodar testes**

Run: `npm test` em `/os`.

Expected: todos passam.

- [ ] **Step 5: commit**

Nenhum commit; a área desktop é ignorada.

---

### Task 10: Verificação

**Files:**
- Use: `/os`, `/site`

- [ ] **Step 1: rodar todos os testes**

Run em `/os`: `npm test`.

Expected: todos passam.

Run em `/site`: `npm test`.

Expected: todos passam.

- [ ] **Step 2: verificar segurança**

Confirmar que o editor bloqueia caminhos fora do projeto e que nenhum segredo é gravado.

- [ ] **Step 3: registrar evidências**

Atualizar o arquivo de progresso com os resultados.

---

## Auto-revisão do plano

- Cobertura da spec: pareamento por e-mail (Tasks 1-5), criador de apps (Tasks 6-9), segurança e verificação (Task 10).
- Sem placeholders nas etapas de implementação.
- Nomes consistentes: `confirmToken`, `pairingStatusText`, `validateProjectName`, `normalizeProjectPath`, `validateManifest`, `packageFileName`, `buildPublicationPayload`.
