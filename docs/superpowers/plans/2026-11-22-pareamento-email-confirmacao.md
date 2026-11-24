# Pareamento por e-mail com confirmação POST — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fazer o pareamento por e-mail entregar o token de dispositivo ao aplicativo, exigir licença na confirmação e evitar consumo acidental por scanners de e-mail.

**Architecture:** O aplicativo pede o e-mail e chama `POST /api/pairing/email/start`. O site cria o pareamento, envia o link e devolve `pollToken`. O link abre `/parear?token=...`; a página valida sem consumir e oferece um botão que envia `POST /api/pairing/confirm`. O aplicativo recebe o token de dispositivo pelo polling.

**Tech Stack:** Next.js 16, React 19, TypeScript, Vitest, Electron, Resend.

---

### Task 1: Corrigir o serviço de pareamento

**Files:**
- Modify: `/site/lib/pairing.ts`
- Test: `/site/tests/pairing-email.test.ts`

- [ ] **Step 1: ajustar `confirmPairingByEmail`**

```ts
export async function confirmPairingByEmail(store: Store, confirmToken: string, userId: string) {
  const row = await store.getPairingByConfirmHash(hashSecret(confirmToken));
  if (!row || row.userId !== userId) throw new Error("token inválido");
  if (isPairingExpired(row)) throw new Error("token expirado");
  if (row.consumedAt) throw new Error("token já usado");
  const user = await store.getUserById(userId);
  if (!user) throw new Error("token inválido");
  if (!(await userHasPaidLicense(user.email, store))) throw new Error("licença necessária");
  const deviceToken = generateToken(32);
  const device = await store.createDevice({
    id: crypto.randomUUID(),
    userId: user.id,
    name: row.deviceName ?? "PC de casa",
    tokenHash: hashSecret(deviceToken),
  });
  const updated = await store.confirmPairing({
    id: row.id,
    userId: user.id,
    deviceId: device.id,
    deviceName: device.name,
    pendingDeviceToken: deviceToken,
  });
  if (!updated) throw new Error("token já usado");
  await store.addAudit({
    id: crypto.randomUUID(),
    userId: user.id,
    deviceId: device.id,
    action: "pairing.confirm_email",
    ip: null,
    meta: device.name,
  });
  return { deviceToken, device };
}
```

- [ ] **Step 2: rodar testes**

Run: `npm test -- tests/pairing-email.test.ts` em `/site`.

Expected: 3 testes passam.

- [ ] **Step 3: commit**

```bash
git add lib/pairing.ts tests/pairing-email.test.ts
git commit -m "fix(pairing): entrega token e exige licença na confirmação"
```

---

### Task 2: Rotas anônima e confirm POST

**Files:**
- Create: `/site/app/api/pairing/email/start/route.ts`
- Modify: `/site/app/api/pairing/confirm/route.ts`
- Create: `/site/app/(site)/parear/page.tsx`
- Test: `/site/tests/pairing-email-confirm.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/pairing-email-confirm.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { hashPassword } from "../lib/auth";
import { setStoreForTests } from "../lib/db";
import { POST as emailStartPost } from "../app/api/pairing/email/start/route";
import { POST as confirmPost } from "../app/api/pairing/confirm/route";
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
  it("não consome o token no GET", async () => {
    await seed();
    const start = await emailStartPost(req("/api/pairing/email/start", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ email: "u@example.com", deviceName: "PC de casa" }) }));
    expect(start.status).toBe(200);
    const { confirmToken } = await start.json();
    const get = await confirmGet(req(`/api/pairing/confirm?token=${confirmToken}`));
    expect(get.status).toBe(200);
    const post = await confirmPost(req("/api/pairing/confirm", { method: "POST", headers: { "content-type": "application/json", origin: "http://localhost:3000" }, body: JSON.stringify({ token: confirmToken }) }));
    expect(post.status).toBe(200);
  });

  it("rejeita token inválido no POST", async () => {
    const res = await confirmPost(req("/api/pairing/confirm", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ token: "invalido" }) }));
    expect(res.status).toBe(400);
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/pairing-email-confirm.test.ts` em `/site`.

Expected: falha de rotas ausentes.

- [ ] **Step 3: implementar as rotas**

`/site/app/api/pairing/email/start/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getStore } from "@/lib/db";
import { startPairingByEmail } from "@/lib/pairing";
import { sendPairingEmail } from "@/lib/email";
import { clientIp, isValidEmail, jsonError } from "@/lib/http";
import { rateLimit } from "@/lib/rate-limit";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function POST(req: Request) {
  let body: Record<string, unknown>;
  try { body = await req.json(); } catch { return jsonError("invalid_request", 400); }
  const email = String(body.email ?? "").trim().toLowerCase();
  const deviceName = String(body.deviceName ?? "PC de casa");
  if (!isValidEmail(email)) return jsonError("invalid_request", 400);
  const limitedIp = rateLimit(`pair-start-ip:${clientIp(req)}`, 10, 15 * 60 * 1000);
  const limitedEmail = rateLimit(`pair-start-email:${email}`, 3, 60 * 60 * 1000);
  if (!limitedIp.ok || !limitedEmail.ok) return jsonError("rate_limited", 429);
  const store = await getStore();
  try {
    const started = await startPairingByEmail(store, email, deviceName);
    await sendPairingEmail(started.email, started.confirmToken);
    return NextResponse.json({ ok: true, pollToken: started.pollToken, expiresAt: started.expiresAt });
  } catch {
    return NextResponse.json({ ok: true, pollToken: crypto.randomUUID(), expiresAt: new Date(Date.now() + 10 * 60 * 1000).toISOString() });
  }
}
```

`/site/app/api/pairing/confirm/route.ts` (manter o POST por código e adicionar GET não consumidor e POST por token):

```ts
export async function GET(req: Request) {
  const token = new URL(req.url).searchParams.get("token") ?? "";
  if (!token) return jsonError("token ausente", 400);
  const current = await getSessionUser(readSessionCookie(req));
  if (!current) return NextResponse.redirect(new URL(`/entrar?next=/parear&token=${encodeURIComponent(token)}`, req.url));
  const store = await getStore();
  const row = await store.getPairingByConfirmHash(hashSecret(token));
  if (!row || isPairingExpired(row) || row.consumedAt) return jsonError("token inválido", 400);
  return NextResponse.json({ ok: true, pending: true });
}

export async function POST(req: Request) {
  const current = await getSessionUser(readSessionCookie(req));
  if (!current) return jsonError("não autenticado", 401);
  const store = await getStore();
  if (!(await userHasPaidLicense(current.user.email, store))) return jsonError("licença necessária", 403);
  let body: Record<string, unknown>;
  try { body = await req.json(); } catch { return jsonError("invalid_request", 400); }
  const token = String(body.token ?? "");
  if (!token) return jsonError("token ausente", 400);
  try {
    const result = await confirmPairingByEmail(store, token, current.user.id);
    return NextResponse.json({ ok: true, device: { id: result.device.id, name: result.device.name } });
  } catch (error) {
    return jsonError(error instanceof Error ? error.message : "token inválido", 400);
  }
}
```

`/site/app/(site)/parear/page.tsx`:

```tsx
"use client";

import { useRouter, useSearchParams } from "next/navigation";
import { useState } from "react";

export default function ParearPage() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const token = searchParams.get("token") ?? "";
  const [message, setMessage] = useState("");
  const [busy, setBusy] = useState(false);

  async function confirm() {
    setBusy(true);
    const res = await fetch("/api/pairing/confirm", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ token }),
    });
    const data = await res.json();
    if (!res.ok) {
      setMessage(data.error ?? "Não foi possível confirmar.");
      setBusy(false);
      return;
    }
    setMessage("Computador ativado. Você já pode fechar esta página.");
    router.refresh();
  }

  return (
    <main className="mx-auto max-w-xl px-5 py-12">
      <h1 className="text-3xl font-semibold">Ativar computador</h1>
      <p className="mt-4 text-sm text-muted">Clique no botão abaixo para confirmar o pareamento deste computador.</p>
      <button className="btn-primary mt-6" disabled={busy || !token} onClick={() => void confirm()}>
        {busy ? "Confirmando…" : "Confirmar pareamento"}
      </button>
      {message ? <p className="mt-4 text-sm" role="status">{message}</p> : null}
    </main>
  );
}
```

- [ ] **Step 4: rodar testes**

Run: `npm test -- tests/pairing-email-confirm.test.ts` em `/site`.

Expected: 2 testes passam.

- [ ] **Step 5: commit**

```bash
git add app/api/pairing app/(site)/parear tests/pairing-email-confirm.test.ts
git commit -m "feat(pairing): confirmação por POST e página /parear"
```

---

### Task 3: Atualizar a interface de conta

**Files:**
- Modify: `/site/components/AccountPanel.tsx`
- Test: `/site/tests/account-pairing.test.tsx`

- [ ] **Step 1: escrever o teste**

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

- [ ] **Step 2: implementar**

Adicionar `pairingButtonText` e substituir o formulário de código por um botão que chama `sendPairingLink`, que pede o e-mail e chama `POST /api/pairing/email/start`.

- [ ] **Step 3: rodar testes**

Run: `npm test -- tests/account-pairing.test.tsx` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 4: commit**

```bash
git add components/AccountPanel.tsx tests/account-pairing.test.tsx
git commit -m "feat(ui): pareamento por link de e-mail"
```

---

### Task 4: Atualizar o aplicativo para pedir e-mail

**Files:**
- Modify: `/os/src/interface/yami-account.js`
- Create: `/os/src/interface/pairing-email.mjs`
- Test: `/os/tests/pairing-email.test.mjs`

- [ ] **Step 1: escrever o teste**

```js
import test from "node:test";
import assert from "node:assert/strict";
import { pairingStatusText } from "../src/interface/pairing-email.mjs";

test("mostra o texto correto", () => {
  assert.equal(pairingStatusText("pending"), "Verifique seu e-mail e clique no link de ativação.");
  assert.equal(pairingStatusText("ready"), "Computador ativado.");
});
```

- [ ] **Step 2: implementar**

`pairing-email.mjs`:
```js
export function pairingStatusText(status) {
  if (status === "ready") return "Computador ativado.";
  if (status === "expired") return "O link expirou. Gere um novo pareamento.";
  return "Verifique seu e-mail e clique no link de ativação.";
}
```

Atualizar `yami-account.js` para pedir o e-mail, chamar `startEmailPairing`, mostrar o texto e continuar consultando `pollPairing`.

- [ ] **Step 3: rodar testes**

Run: `npm test` em `/os`.

Expected: todos passam.

---

### Task 5: Verificação

- [ ] **Step 1: rodar todos os testes**

Run em `/site`: `npm test`.

Run em `/os`: `npm test`.

Expected: todos passam.

- [ ] **Step 2: verificar segurança**

Confirmar que o GET do link não consome o token e que o POST exige sessão e licença.

- [ ] **Step 3: registrar evidências**

Atualizar o arquivo de progresso.

---

## Auto-revisão do plano

- Cobertura da spec: pareamento por e-mail (Tasks 1-5), criador de apps (Tasks 6-9 da spec anterior), segurança e verificação (Task 5).
- Sem placeholders nas etapas de implementação.
- Nomes consistentes: `confirmToken`, `pairingStatusText`, `startEmailPairing`, `pollPairing`.
