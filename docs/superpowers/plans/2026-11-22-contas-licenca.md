# Contas vinculadas à licença — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fazer o checkout criar a conta do usuário, vincular a licença a essa conta e permitir gerenciamento pelo próprio usuário e pelo administrador.

**Architecture:** O checkout passa a exigir senha para contas novas, cria ou reutiliza o usuário pelo e-mail e grava `user_id` no pedido. A liberação da licença passa a atualizar `users.license_id`. Rotas de conta e admin expõem apenas dados autorizados.

**Tech Stack:** Next.js 16, React 19, TypeScript, Vitest, SQLite/Postgres.

**Convenção de caminho:** `/site` = `D:\Downloads\open code yami edit`.

---

### Task 1: Esquema de usuário e pedido vinculados

**Files:**
- Modify: `/site/lib/db/schema.sql`
- Modify: `/site/lib/db/sql-shared.ts`
- Modify: `/site/lib/db/types.ts`
- Test: `/site/tests/license-account-schema.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/license-account-schema.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { SCHEMA_SQL } from "../lib/db/sql-shared";

describe("contas vinculadas à licença", () => {
  it("contém os campos necessários", () => {
    expect(SCHEMA_SQL).toContain("status TEXT NOT NULL DEFAULT 'ativo'");
    expect(SCHEMA_SQL).toContain("license_id TEXT");
    expect(SCHEMA_SQL).toContain("user_id TEXT");
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/license-account-schema.test.ts` em `/site`.

Expected: falha.

- [ ] **Step 3: atualizar esquema**

Em `schema.sql` e `SCHEMA_SQL`, alterar `users` para incluir `status TEXT NOT NULL DEFAULT 'ativo'` e `license_id TEXT`, e `orders` para incluir `user_id TEXT`.

- [ ] **Step 4: atualizar tipos**

Em `types.ts`, acrescentar `status: "ativo" | "bloqueado"` e `licenseId: string | null` em `User`, e `userId: string | null` em `Order`.

Atualizar `rowToUser` e `rowToOrder` em `sql-shared.ts` para mapear os novos campos.

- [ ] **Step 5: rodar teste**

Run: `npm test -- tests/license-account-schema.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 6: commit**

```bash
git add lib/db/schema.sql lib/db/sql-shared.ts lib/db/types.ts tests/license-account-schema.test.ts
git commit -m "feat(db): vincula usuários e licenças"
```

---

### Task 2: Serviço de conta no checkout

**Files:**
- Create: `/site/lib/checkout-account.ts`
- Test: `/site/tests/checkout-account.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/checkout-account.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { verifyPassword } from "../lib/auth";
import { ensureCheckoutAccount } from "../lib/checkout-account";

describe("conta no checkout", () => {
  it("cria conta com senha quando o e-mail é novo", async () => {
    const store = createMemoryStore();
    const user = await ensureCheckoutAccount(store, "novo@example.com", "SenhaTeste123!", "Novo");
    expect(user.email).toBe("novo@example.com");
    expect(await verifyPassword("SenhaTeste123!", user.passwordHash)).toBe(true);
  });

  it("reutiliza conta existente", async () => {
    const store = createMemoryStore();
    const first = await ensureCheckoutAccount(store, "novo@example.com", "SenhaTeste123!", "Novo");
    const second = await ensureCheckoutAccount(store, "novo@example.com", "", "Novo");
    expect(second.id).toBe(first.id);
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/checkout-account.test.ts` em `/site`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Create `/site/lib/checkout-account.ts`:

```ts
import { hashPassword } from "./auth";
import { getStore, type Store, type User } from "./db";

export async function ensureCheckoutAccount(
  store: Store,
  email: string,
  password: string,
  name: string,
): Promise<User> {
  const normalized = email.trim().toLowerCase();
  const existing = await store.getUserByEmail(normalized);
  if (existing) return existing;
  if (password.length < 8 || password.length > 128) {
    throw new Error("senha_invalida");
  }
  const passwordHash = await hashPassword(password);
  return store.createUser({
    id: crypto.randomUUID(),
    email: normalized,
    name: name.trim().slice(0, 80) || "Usuário YAMI",
    passwordHash,
  });
}
```

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/checkout-account.test.ts` em `/site`.

Expected: 2 testes passam.

- [ ] **Step 5: commit**

```bash
git add lib/checkout-account.ts tests/checkout-account.test.ts
git commit -m "feat(checkout): cria conta com senha"
```

---

### Task 3: Checkout vincula pedido à conta

**Files:**
- Modify: `/site/app/api/checkout/route.ts`
- Modify: `/site/components/CheckoutForm.tsx`
- Test: `/site/tests/checkout-linked-order.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/checkout-linked-order.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { setStoreForTests } from "../lib/db";
import { POST as checkoutPost } from "../app/api/checkout/route";
import { ensureCheckoutAccount } from "../lib/checkout-account";

describe("checkout com conta", () => {
  it("cria pedido vinculado à conta", async () => {
    const store = createMemoryStore();
    setStoreForTests(store);
    await ensureCheckoutAccount(store, "cliente@example.com", "SenhaTeste123!", "Cliente");
    const res = await checkoutPost(new Request("http://localhost:3000/api/checkout", {
      method: "POST",
      headers: { "content-type": "application/json", origin: "http://localhost:3000" },
      body: JSON.stringify({ email: "cliente@example.com", password: "", name: "Cliente", accepted: true }),
    }));
    expect(res.status).toBe(200);
    const orders = await store.getOrdersByEmail("cliente@example.com");
    expect(orders[0]?.userId).toBeTruthy();
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/checkout-linked-order.test.ts` em `/site`.

Expected: falha.

- [ ] **Step 3: implementar**

Em `/site/app/api/checkout/route.ts`, importar `ensureCheckoutAccount` e trocar a criação do pedido:

```ts
const user = await ensureCheckoutAccount(store, email, password, name);
const order = await store.createOrder({
  id: crypto.randomUUID(),
  email,
  name: name || null,
  amountBrl: price,
  userId: user.id,
});
```

No corpo, aceitar `password` no JSON e no formulário.

Em `/site/components/CheckoutForm.tsx`, acrescentar campo:

```tsx
<label htmlFor="password" className="block text-sm font-medium">Senha</label>
<input id="password" name="password" type="password" minLength={8} maxLength={128} autoComplete="new-password" className="field mt-1" />
```

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/checkout-linked-order.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add app/api/checkout/route.ts components/CheckoutForm.tsx tests/checkout-linked-order.test.ts
git commit -m "feat(checkout): pedido vinculado à conta"
```

---

### Task 4: Licença vinculada ao usuário

**Files:**
- Modify: `/site/lib/fulfill.ts`
- Test: `/site/tests/license-user-link.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/license-user-link.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { fulfillApprovedPayment } from "../lib/fulfill";

describe("licença vinculada", () => {
  it("grava license_id no usuário", async () => {
    const store = createMemoryStore();
    await store.createUser({ id: "u1", email: "u@example.com", passwordHash: "hash", name: "U", status: "ativo", licenseId: null });
    await store.createOrder({ id: "o1", email: "u@example.com", amountBrl: 20, userId: "u1" });
    const result = await fulfillApprovedPayment({ store, orderId: "o1", paymentId: "p1" });
    const user = await store.getUserById("u1");
    expect(user?.licenseId).toBe(result.license.id);
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/license-user-link.test.ts` em `/site`.

Expected: falha.

- [ ] **Step 3: implementar**

Em `fulfillApprovedPayment`, após garantir a licença:

```ts
if (order.userId) {
  await store.updateUserLicense(order.userId, license.id);
}
```

Adicionar `updateUserLicense(userId, licenseId)` ao `Store` e implementar nos três stores.

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/license-user-link.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add lib/fulfill.ts lib/db/memory.ts lib/db/sqlite.ts lib/db/postgres.ts lib/db/types.ts tests/license-user-link.test.ts
git commit -m "feat(license): licença vinculada ao usuário"
```

---

### Task 5: Gerenciamento da conta

**Files:**
- Create: `/site/app/api/conta/route.ts`
- Modify: `/site/components/AccountPanel.tsx`
- Test: `/site/tests/conta-management.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/conta-management.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { createSessionForUser, hashPassword } from "../lib/auth";
import { setStoreForTests } from "../lib/db";
import { PATCH as contaPatch } from "../app/api/conta/route";

describe("gerenciamento de conta", () => {
  it("altera nome e senha", async () => {
    const store = createMemoryStore();
    setStoreForTests(store);
    const user = await store.createUser({ id: "u1", email: "u@example.com", passwordHash: await hashPassword("senha-antiga"), name: "Antigo", status: "ativo", licenseId: null });
    const { raw } = await createSessionForUser(user);
    const res = await contaPatch(new Request("http://localhost:3000/api/conta", {
      method: "PATCH",
      headers: { "content-type": "application/json", cookie: `yami_session=${raw}` },
      body: JSON.stringify({ name: "Novo", password: "senha-nova-123" }),
    }));
    expect(res.status).toBe(200);
    const updated = await store.getUserById("u1");
    expect(updated?.name).toBe("Novo");
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/conta-management.test.ts` em `/site`.

Expected: falha de rota ausente.

- [ ] **Step 3: implementar**

Create `/site/app/api/conta/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getStore } from "@/lib/db";
import { getSessionUser, hashPassword, readSessionCookie } from "@/lib/auth";
import { jsonError } from "@/lib/http";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function PATCH(req: Request) {
  const current = await getSessionUser(readSessionCookie(req));
  if (!current) return jsonError("não autenticado", 401);
  let body: Record<string, unknown>;
  try { body = await req.json(); } catch { return jsonError("invalid_request", 400); }
  const store = await getStore();
  const name = String(body.name ?? current.user.name).trim().slice(0, 80);
  const password = String(body.password ?? "");
  if (password && (password.length < 8 || password.length > 128)) return jsonError("senha_invalida", 400);
  await store.updateUser(current.user.id, {
    name,
    passwordHash: password ? await hashPassword(password) : current.user.passwordHash,
  });
  return NextResponse.json({ ok: true });
}
```

Adicionar `updateUser(id, input)` ao Store e implementar nos três stores.

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/conta-management.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add app/api/conta/route.ts lib/db tests/conta-management.test.ts
git commit -m "feat(conta): atualização de dados do usuário"
```

---

### Task 6: Painel administrativo de usuários

**Files:**
- Create: `/site/app/api/admin/users/route.ts`
- Create: `/site/components/admin/UsersPanel.tsx`
- Test: `/site/tests/admin-users.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/admin-users.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { createSessionForUser, hashPassword } from "../lib/auth";
import { setStoreForTests } from "../lib/db";
import { GET as adminUsersGet } from "../app/api/admin/users/route";

describe("admin de usuários", () => {
  it("nega acesso a usuário comum e permite admin", async () => {
    const store = createMemoryStore();
    setStoreForTests(store);
    process.env.ADMIN_EMAILS = "admin@example.com";
    const admin = await store.createUser({ id: "a1", email: "admin@example.com", passwordHash: await hashPassword("senha-forte-123"), name: "Admin", status: "ativo", licenseId: null });
    const { raw } = await createSessionForUser(admin);
    const res = await adminUsersGet(new Request("http://localhost:3000/api/admin/users", { headers: { cookie: `yami_session=${raw}` } }));
    expect(res.status).toBe(200);
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/admin-users.test.ts` em `/site`.

Expected: falha de rota ausente.

- [ ] **Step 3: implementar rota e painel**

Criar rota usando `adminFromRequest` de `app/api/desktop/_shared.ts`, listando usuários sem senha.

Criar `UsersPanel.tsx` com tabela de e-mail, status e licença, sem exibir senhas.

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/admin-users.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add app/api/admin/users components/admin/UsersPanel.tsx tests/admin-users.test.ts
git commit -m "feat(admin): gerenciamento de usuários"
```

---

## Auto-revisão do plano

- Cobertura da spec: checkout cria conta (Tasks 2-3), licença vinculada (Task 4), gerenciamento (Task 5), admin (Task 6).
- Sem placeholders nas etapas de implementação.
- Nomes consistentes: `ensureCheckoutAccount`, `userId`, `licenseId`, `updateUserLicense`, `updateUser`.
