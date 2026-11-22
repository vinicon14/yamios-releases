# Integração YAMI OS 0.8.0 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar o Marco 1 da atualização 0.8.0: checkout seguro, licenças consultáveis pelo sistema, solicitações com aprovação administrativa e um aplicativo YAMI nativo integrado ao shell.

**Architecture:** O site Next.js ganha serviços e rotas autenticadas por token de dispositivo; o pacote incremental 0.8.0 acrescenta uma view Electron e handlers IPC sobre o `app.asar` oficial da 0.7.0. O repositório `yamios-releases` continua contendo somente manifestos e pacotes.

**Tech Stack:** Next.js 16, React 19, TypeScript, Vitest, Playwright, Electron/Chromium, Node.js, SQLite/Postgres, `tar.zst`, `sha256sum`.

**Convenções de caminho:**

- `/site` neste documento significa `D:\Downloads\open code yami edit`.
- `/os` neste documento significa `D:\Downloads\yami relise os\desenvolvimento\yami-os`.
- `/releases` neste documento significa `D:\Downloads\yami relise os`.
- Todos os commits do site ocorrem no repositório `/site`; os do pacote/manifesto, em `/releases`.
- Nenhum commit ou push acontece sem autorização explícita do usuário no momento da execução.

---

## Estrutura de arquivos

### Site (`/site`)

- `lib/payment-events.ts` — tipos e códigos de eventos de pagamento.
- `lib/fulfill.ts` — validação rigorosa do pagamento e liberação idempotente.
- `lib/mp-webhook.ts` — processamento do webhook com registro de eventos.
- `lib/config.ts` — exigência de segredo em produção.
- `lib/db/schema.sql` e `lib/db/sql-shared.ts` — esquema e conversores de linhas.
- `lib/db/types.ts` — tipos `FeatureRequest`, `FeatureRequestEvent`, `UserNotification`, `PaymentEvent` e métodos do `Store`.
- `lib/db/memory.ts`, `lib/db/sqlite.ts`, `lib/db/postgres.ts` — implementações dos novos métodos.
- `lib/feature-requests.ts` — regras de criação, transição e notificação.
- `lib/desktop-auth.ts` — login nativo, token de dispositivo e sessão.
- `app/api/desktop/**` — rotas do aplicativo.
- `app/api/admin/requests/**` — rotas administrativas de solicitações.
- `components/FeatureRequestsPanel.tsx` e `components/admin/AdminRequestsPanel.tsx` — interfaces.
- `app/(site)/conta/page.tsx` e `app/(site)/admin/page.tsx` — integração dos painéis.
- `tests/*.test.ts` — testes de cada serviço.

### Desktop (`/os`)

- `base/` — cópia intacta do `app.asar` 0.7.0 extraído.
- `src/` — cópia editável da base.
- `scripts/patch-base.mjs` — reaplica patches de forma idempotente.
- `src/interface/yami-account.js` — lógica da nova view.
- `src/account/client.mjs` — cliente HTTPS usado no processo principal.
- `src/account/storage.mjs` — persistência cifrada do token.
- `src/account/ipc.cjs` — handlers IPC.
- `tests/*.test.mjs` — testes das funções puras e do patch.
- `build/build-package.mjs` — geração do `.tar.zst` e checksum.

### Releases (`/releases`)

- `latest.json` — manifesto 0.8.0.
- `yami-os-0.8.0-pacote.tar.zst` — pacote incremental.
- `docs/superpowers/specs/...` — especificação já aprovada.
- `docs/superpowers/plans/...` — este plano.

---

### Task 1: Preparar áreas de trabalho e confirmar a base 0.7.0

**Files:**
- Create: `/os/package.json`
- Create: `/os/README.local.md`
- Use: `/releases/yami-os-0.7.0-pacote.tar.zst`
- Use: `/releases/pacote-0.7.0/app-extracted/`

- [ ] **Step 1: conferir o pacote oficial**

Run:

```powershell
$expected = 'c9fdc6fa6a0e1616d494295ffcb6e23d6a5f7df60eb8949298284cc8cb3138fd'
$actual = (Get-FileHash 'D:\Downloads\yami relise os\yami-os-0.7.0-pacote.tar.zst' -Algorithm SHA256).Hash.ToLower()
if ($actual -ne $expected) { throw "SHA256 inválido: $actual" }
Write-Output 'BASE_07_OK'
```

Expected: `BASE_07_OK`.

- [ ] **Step 2: criar a área de desenvolvimento sem tocar no repositório de releases**

Run:

```powershell
$os = 'D:\Downloads\yami relise os\desenvolvimento\yami-os'
New-Item -ItemType Directory -Force "$os\base","$os\src","$os\scripts","$os\tests","$os\build","$os\src\account" | Out-Null
Copy-Item -Recurse -Force 'D:\Downloads\yami relise os\pacote-0.7.0\app-extracted\*' "$os\src"
Copy-Item -Force 'D:\Downloads\yami relise os\pacote-0.7.0\root\opt\YamiOS\resources\app.asar' "$os\base\app.asar"
Get-ChildItem "$os\src" | Select-Object -ExpandProperty Name
```

Expected: `dist`, `src`, `package.json`.

- [ ] **Step 3: criar o manifesto local de desenvolvimento**

Create `/os/package.json` com:

```json
{
  "name": "yami-os-desktop-dev",
  "version": "0.8.0",
  "private": true,
  "type": "module",
  "scripts": {
    "patch": "node scripts/patch-base.mjs",
    "test": "node --test tests/*.test.mjs",
    "build": "node build/build-package.mjs"
  },
  "devDependencies": {
    "@electron/asar": "3.2.14"
  }
}
```

- [ ] **Step 4: documentar a finalidade da área**

Create `/os/README.local.md` com:

```markdown
# YAMI OS — área de desenvolvimento local

Esta pasta não é versionada no repositório `yamios-releases`.

- `base/app.asar`: cópia intacta da release 0.7.0.
- `src/`: código editável, reconstruído por `scripts/patch-base.mjs`.
- `build/`: geração do pacote 0.8.0 e do checksum.

Nunca colocar tokens, senhas ou dados de usuário nesta pasta.
```

- [ ] **Step 5: verificar que o repositório de releases continua limpo**

Run: `git status --short` em `/releases`.

Expected: nenhuma saída.

---

### Task 2: Tornar o checkout rigoroso e idempotente

**Files:**
- Modify: `/site/lib/config.ts:41-44`
- Modify: `/site/lib/fulfill.ts:20-107`
- Modify: `/site/lib/mp-webhook.ts:39-121`
- Test: `/site/tests/mp-payment-hardening.test.ts`

- [ ] **Step 1: escrever os testes que falham**

Create `/site/tests/mp-payment-hardening.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { paymentMatchesOrder } from "../lib/fulfill";
import { chargesEnabled } from "../lib/config";
import type { Order } from "../lib/db";

const order: Order = {
  id: "pedido-1",
  email: "cliente@example.com",
  name: null,
  amountBrl: 20,
  currency: "BRL",
  status: "pending",
  mpPreferenceId: "pref-1",
  mpPaymentId: null,
  createdAt: "2026-11-21T00:00:00.000Z",
  paidAt: null,
};

describe("paymentMatchesOrder", () => {
  it("rejeita pagamento aprovado sem external_reference", () => {
    const result = paymentMatchesOrder({
      id: 123,
      status: "approved",
      currency_id: "BRL",
      transaction_amount: 20,
    }, order);
    expect(result).toEqual({ ok: false, reason: "missing_external_reference" });
  });

  it("rejeita referência de outro pedido", () => {
    const result = paymentMatchesOrder({
      id: 123,
      status: "approved",
      currency_id: "BRL",
      transaction_amount: 20,
      external_reference: "pedido-2",
    }, order);
    expect(result).toEqual({ ok: false, reason: "external_reference_mismatch" });
  });

  it("rejeita valor diferente do configurado", () => {
    const result = paymentMatchesOrder({
      id: 123,
      status: "approved",
      currency_id: "BRL",
      transaction_amount: 19,
      external_reference: "pedido-1",
    }, order);
    expect(result).toEqual({ ok: false, reason: "amount_mismatch" });
  });

  it("aceita pagamento correto", () => {
    const result = paymentMatchesOrder({
      id: 123,
      status: "approved",
      currency_id: "BRL",
      transaction_amount: 20,
      external_reference: "pedido-1",
    }, order);
    expect(result).toEqual({ ok: true });
  });
});

describe("chargesEnabled em produção", () => {
  it("exige token e segredo do webhook", () => {
    const originalMode = process.env.MP_MODE;
    const originalToken = process.env.MP_ACCESS_TOKEN;
    const originalSecret = process.env.MP_WEBHOOK_SECRET;
    try {
      process.env.MP_MODE = "production";
      process.env.MP_ACCESS_TOKEN = "token-teste";
      delete process.env.MP_WEBHOOK_SECRET;
      expect(chargesEnabled()).toBe(false);
      process.env.MP_WEBHOOK_SECRET = "segredo-teste";
      expect(chargesEnabled()).toBe(true);
    } finally {
      if (originalMode === undefined) delete process.env.MP_MODE;
      else process.env.MP_MODE = originalMode;
      if (originalToken === undefined) delete process.env.MP_ACCESS_TOKEN;
      else process.env.MP_ACCESS_TOKEN = originalToken;
      if (originalSecret === undefined) delete process.env.MP_WEBHOOK_SECRET;
      else process.env.MP_WEBHOOK_SECRET = originalSecret;
    }
  });
});
```

- [ ] **Step 2: executar e confirmar a falha**

Run: `npm test -- tests/mp-payment-hardening.test.ts` em `/site`.

Expected: falha em `missing_external_reference` e em `chargesEnabled`.

- [ ] **Step 3: ajustar a configuração**

Substituir em `/site/lib/config.ts`:

```ts
/** Checkout Pro real só entra com MP_MODE=production, token e segredo do webhook. */
export function chargesEnabled(): boolean {
  return (
    getMpMode() === "production" &&
    hasMpAccessToken() &&
    getWebhookSecret().length > 0
  );
}
```

- [ ] **Step 4: ajustar a validação do pagamento**

Substituir a função em `/site/lib/fulfill.ts`:

```ts
export function paymentMatchesOrder(payment: MpPayment, order: Order): {
  ok: boolean;
  reason?: string;
} {
  if (payment.status !== "approved") return { ok: false, reason: `status_${payment.status}` };
  if (!payment.external_reference) return { ok: false, reason: "missing_external_reference" };
  if (payment.external_reference !== order.id) {
    return { ok: false, reason: "external_reference_mismatch" };
  }
  if (payment.currency_id && payment.currency_id !== "BRL") {
    return { ok: false, reason: "currency_mismatch" };
  }
  if (
    typeof payment.transaction_amount === "number" &&
    Math.abs(payment.transaction_amount - order.amountBrl) > AMOUNT_TOLERANCE
  ) {
    return { ok: false, reason: "amount_mismatch" };
  }
  const expected = getPriceBrl();
  if (Math.abs(order.amountBrl - expected) > AMOUNT_TOLERANCE) {
    return { ok: false, reason: "price_changed" };
  }
  return { ok: true };
}
```

- [ ] **Step 5: registrar eventos sem dados sensíveis em `/site/lib/payment-events.ts`**

Create:

```ts
export type PaymentEventType = "webhook" | "fulfill" | "duplicate" | "match_failed" | "api_error";

export type PaymentEventInput = {
  orderId: string | null;
  paymentId: string | null;
  eventType: PaymentEventType;
  result: "ok" | "skipped" | "error";
  reasonCode: string;
};

const ALLOWED_REASONS = new Set([
  "approved",
  "duplicate",
  "missing_external_reference",
  "external_reference_mismatch",
  "currency_mismatch",
  "amount_mismatch",
  "price_changed",
  "unknown_order",
  "signature_invalid",
  "payment_fetch_failed",
  "status_rejected",
  "status_cancelled",
  "ignored_topic",
  "charges_disabled",
]);

export function normalizePaymentEvent(input: PaymentEventInput): PaymentEventInput {
  const reasonCode = ALLOWED_REASONS.has(input.reasonCode) ? input.reasonCode : "other";
  return {
    orderId: input.orderId?.slice(0, 80) ?? null,
    paymentId: input.paymentId?.slice(0, 40) ?? null,
    eventType: input.eventType,
    result: input.result,
    reasonCode,
  };
}
```

- [ ] **Step 6: executar o teste novamente**

Run: `npm test -- tests/mp-payment-hardening.test.ts` em `/site`.

Expected: `4 passed` no primeiro grupo e `1 passed` no segundo.

- [ ] **Step 7: commit no site**

```bash
git add lib/config.ts lib/fulfill.ts lib/payment-events.ts tests/mp-payment-hardening.test.ts
git commit -m "fix(payments): valida referência e exige segredo do webhook"
```

---

### Task 3: Ampliar esquema, tipos e conversores do banco

**Files:**
- Modify: `/site/lib/db/schema.sql`
- Modify: `/site/lib/db/sql-shared.ts`
- Modify: `/site/lib/db/types.ts`
- Test: `/site/tests/db-schema.test.ts`

- [ ] **Step 1: escrever o teste do contrato**

Create `/site/tests/db-schema.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { SCHEMA_SQL } from "../lib/db/sql-shared";
import type { Store } from "../lib/db/types";

describe("schema ampliado", () => {
  it("contém tabelas de solicitações, avisos e eventos de pagamento", () => {
    for (const table of ["feature_requests", "feature_request_events", "user_notifications", "payment_events"]) {
      expect(SCHEMA_SQL).toContain(`CREATE TABLE IF NOT EXISTS ${table}`);
    }
  });

  it("expõe os métodos necessários no contrato Store", () => {
    const methods: Array<keyof Store> = [
      "markPaidOnce",
      "addPaymentEvent",
      "createFeatureRequest",
      "listFeatureRequests",
      "getFeatureRequest",
      "updateFeatureRequest",
      "transitionFeatureRequest",
      "createUserNotification",
      "listUserNotifications",
      "markNotificationRead",
      "countUnreadNotifications",
    ];
    expect(methods.length).toBe(11);
  });
});
```

- [ ] **Step 2: executar para observar a falha**

Run: `npm test -- tests/db-schema.test.ts` em `/site`.

Expected: falha de compilação porque os tipos ainda não existem.

- [ ] **Step 3: adicionar tabelas ao esquema canônico**

Append a `/site/lib/db/schema.sql`:

```sql
CREATE TABLE IF NOT EXISTS feature_requests (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  category TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pendente',
  published_version TEXT,
  admin_note TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS feature_request_events (
  id TEXT PRIMARY KEY,
  request_id TEXT NOT NULL,
  from_status TEXT,
  to_status TEXT NOT NULL,
  actor_user_id TEXT NOT NULL,
  actor_role TEXT NOT NULL,
  note TEXT,
  created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS user_notifications (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  kind TEXT NOT NULL,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  request_id TEXT,
  read_at TEXT,
  created_at TEXT NOT NULL,
  expires_at TEXT
);

CREATE TABLE IF NOT EXISTS payment_events (
  id TEXT PRIMARY KEY,
  order_id TEXT,
  payment_id TEXT,
  event_type TEXT NOT NULL,
  result TEXT NOT NULL,
  reason_code TEXT NOT NULL,
  created_at TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_feature_requests_user ON feature_requests (user_id, status);
CREATE INDEX IF NOT EXISTS idx_feature_request_events_request ON feature_request_events (request_id, created_at);
CREATE INDEX IF NOT EXISTS idx_user_notifications_user ON user_notifications (user_id, created_at);
CREATE INDEX IF NOT EXISTS idx_payment_events_order ON payment_events (order_id, created_at);
CREATE INDEX IF NOT EXISTS idx_payment_events_payment ON payment_events (payment_id, created_at);
```

- [ ] **Step 4: adicionar as mesmas tabelas ao esquema embutido**

Inserir o mesmo SQL antes do fechamento de `SCHEMA_SQL` em `/site/lib/db/sql-shared.ts`. Em seguida, acrescentar os conversores:

```ts
export function rowToFeatureRequest(row: Record<string, unknown>): FeatureRequest {
  return {
    id: String(row.id),
    userId: String(row.user_id),
    title: String(row.title),
    description: String(row.description),
    category: String(row.category),
    status: String(row.status) as FeatureRequestStatus,
    publishedVersion: row.published_version == null ? null : String(row.published_version),
    adminNote: row.admin_note == null ? null : String(row.admin_note),
    createdAt: String(row.created_at),
    updatedAt: String(row.updated_at),
  };
}

export function rowToFeatureRequestEvent(row: Record<string, unknown>): FeatureRequestEvent {
  return {
    id: String(row.id),
    requestId: String(row.request_id),
    fromStatus: row.from_status == null ? null : (String(row.from_status) as FeatureRequestStatus),
    toStatus: String(row.to_status) as FeatureRequestStatus,
    actorUserId: String(row.actor_user_id),
    actorRole: String(row.actor_role) as "user" | "admin",
    note: row.note == null ? null : String(row.note),
    createdAt: String(row.created_at),
  };
}

export function rowToUserNotification(row: Record<string, unknown>): UserNotification {
  return {
    id: String(row.id),
    userId: String(row.user_id),
    kind: String(row.kind),
    title: String(row.title),
    body: String(row.body),
    requestId: row.request_id == null ? null : String(row.request_id),
    readAt: row.read_at == null ? null : String(row.read_at),
    createdAt: String(row.created_at),
    expiresAt: row.expires_at == null ? null : String(row.expires_at),
  };
}

export function rowToPaymentEvent(row: Record<string, unknown>): PaymentEvent {
  return {
    id: String(row.id),
    orderId: row.order_id == null ? null : String(row.order_id),
    paymentId: row.payment_id == null ? null : String(row.payment_id),
    eventType: String(row.event_type) as PaymentEvent["eventType"],
    result: String(row.result) as PaymentEvent["result"],
    reasonCode: String(row.reason_code),
    createdAt: String(row.created_at),
  };
}
```

- [ ] **Step 5: adicionar tipos ao contrato**

Inserir em `/site/lib/db/types.ts`:

```ts
export const FEATURE_REQUEST_STATUSES = [
  "pendente", "aprovada", "rejeitada", "em_desenvolvimento", "publicada", "retirada",
] as const;
export type FeatureRequestStatus = (typeof FEATURE_REQUEST_STATUSES)[number];

export type FeatureRequest = {
  id: string;
  userId: string;
  title: string;
  description: string;
  category: string;
  status: FeatureRequestStatus;
  publishedVersion: string | null;
  adminNote: string | null;
  createdAt: string;
  updatedAt: string;
};

export type FeatureRequestEvent = {
  id: string;
  requestId: string;
  fromStatus: FeatureRequestStatus | null;
  toStatus: FeatureRequestStatus;
  actorUserId: string;
  actorRole: "user" | "admin";
  note: string | null;
  createdAt: string;
};

export type UserNotification = {
  id: string;
  userId: string;
  kind: string;
  title: string;
  body: string;
  requestId: string | null;
  readAt: string | null;
  createdAt: string;
  expiresAt: string | null;
};

export type PaymentEvent = {
  id: string;
  orderId: string | null;
  paymentId: string | null;
  eventType: "webhook" | "fulfill" | "duplicate" | "match_failed" | "api_error";
  result: "ok" | "skipped" | "error";
  reasonCode: string;
  createdAt: string;
};
```

Acrescentar à interface `Store`:

```ts
  markPaidOnce(input: {
    orderId: string;
    paymentId: string;
    paidAt?: Date;
  }): Promise<{ order: Order; changed: boolean } | null>;
  addPaymentEvent(entry: Omit<PaymentEvent, "createdAt"> & { createdAt?: string }): Promise<void>;
  createFeatureRequest(input: Omit<FeatureRequest, "id" | "status" | "publishedVersion" | "adminNote" | "createdAt" | "updatedAt">): Promise<FeatureRequest>;
  listFeatureRequests(userId?: string): Promise<FeatureRequest[]>;
  getFeatureRequest(id: string): Promise<FeatureRequest | null>;
  updateFeatureRequest(input: { id: string; userId: string; title: string; description: string; category: string }): Promise<FeatureRequest | null>;
  transitionFeatureRequest(input: {
    id: string;
    actorUserId: string;
    actorRole: "user" | "admin";
    toStatus: FeatureRequestStatus;
    note?: string | null;
    publishedVersion?: string | null;
  }): Promise<FeatureRequest | null>;
  createUserNotification(input: Omit<UserNotification, "id" | "readAt" | "createdAt"> & { createdAt?: string }): Promise<UserNotification>;
  listUserNotifications(userId: string, limit?: number): Promise<UserNotification[]>;
  markNotificationRead(id: string, userId: string): Promise<boolean>;
  countUnreadNotifications(userId: string): Promise<number>;
```

- [ ] **Step 6: executar o teste de contrato**

Run: `npm test -- tests/db-schema.test.ts` em `/site`.

Expected: teste passa, embora os stores ainda não implementem os métodos; o TypeScript informará os erros nas próximas tasks.

- [ ] **Step 7: commit**

```bash
git add lib/db/schema.sql lib/db/sql-shared.ts lib/db/types.ts tests/db-schema.test.ts
git commit -m "feat(db): esquema de solicitações, avisos e eventos"
```

---

### Task 4: Implementar os novos métodos nos três stores

**Files:**
- Modify: `/site/lib/db/memory.ts`
- Modify: `/site/lib/db/sqlite.ts`
- Modify: `/site/lib/db/postgres.ts`
- Test: `/site/tests/stores-new-methods.test.ts`

- [ ] **Step 1: escrever o teste compartilhado**

Create `/site/tests/stores-new-methods.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import type { Store } from "../lib/db/types";

function baseUser(store: Store) {
  return store.createUser({
    id: "user-1",
    email: "user@example.com",
    passwordHash: "hash",
  });
}

describe("novos métodos da Store", () => {
  it("marca um pedido como pago uma única vez", async () => {
    const store = createMemoryStore();
    await store.createOrder({ id: "o1", email: "user@example.com", amountBrl: 20 });
    const first = await store.markPaidOnce({ orderId: "o1", paymentId: "p1" });
    const second = await store.markPaidOnce({ orderId: "o1", paymentId: "p1" });
    const other = await store.markPaidOnce({ orderId: "o1", paymentId: "p2" });
    expect(first?.changed).toBe(true);
    expect(second?.changed).toBe(false);
    expect(other).toBeNull();
  });

  it("gerencia solicitações, eventos e avisos", async () => {
    const store = createMemoryStore();
    await baseUser(store);
    const created = await store.createFeatureRequest({
      userId: "user-1",
      title: "Novo tema",
      description: "Adicionar tema azul",
      category: "personalizacao",
    });
    const approved = await store.transitionFeatureRequest({
      id: created.id,
      actorUserId: "admin-1",
      actorRole: "admin",
      toStatus: "aprovada",
      note: "Aprovado",
    });
    expect(approved?.status).toBe("aprovada");

    await store.createUserNotification({
      userId: "user-1",
      kind: "request.approved",
      title: "Solicitação aprovada",
      body: "Sua solicitação foi aprovada.",
      requestId: created.id,
      expiresAt: null,
    });
    expect(await store.countUnreadNotifications("user-1")).toBe(1);
    expect(await store.listUserNotifications("user-1", 10)).toHaveLength(1);
  });
});
```

- [ ] **Step 2: rodar o teste para verificar a falha**

Run: `npm test -- tests/stores-new-methods.test.ts` em `/site`.

Expected: TypeScript falha com `Property 'markPaidOnce' does not exist`.

- [ ] **Step 3: implementar no store de memória**

Inserir antes do `return store;` em `/site/lib/db/memory.ts`:

```ts
  const featureRequests = new Map<string, import("./types").FeatureRequest>();
  const requestEvents: import("./types").FeatureRequestEvent[] = [];
  const notifications: import("./types").UserNotification[] = [];
  const paymentEvents: import("./types").PaymentEvent[] = [];
```

Inserir dentro do objeto `store`:

```ts
    async markPaidOnce({ orderId, paymentId, paidAt }) {
      const order = orders.get(orderId);
      if (!order) return null;
      if (order.mpPaymentId && order.mpPaymentId !== paymentId) return null;
      const changed = order.status !== "paid" || !order.mpPaymentId;
      if (changed) {
        order.status = "paid";
        order.mpPaymentId = paymentId;
        order.paidAt = nowIso(paidAt ?? new Date());
      }
      return { order: { ...order }, changed };
    },

    async addPaymentEvent(entry) {
      paymentEvents.push({ ...entry, createdAt: entry.createdAt ?? nowIso() });
    },

    async createFeatureRequest(input) {
      const now = nowIso();
      const request: import("./types").FeatureRequest = {
        ...input,
        id: crypto.randomUUID(),
        status: "pendente",
        publishedVersion: null,
        adminNote: null,
        createdAt: now,
        updatedAt: now,
      };
      featureRequests.set(request.id, request);
      return { ...request };
    },

    async listFeatureRequests(userId) {
      return [...featureRequests.values()]
        .filter((r) => !userId || r.userId === userId)
        .sort((a, b) => (a.createdAt < b.createdAt ? 1 : -1))
        .map((r) => ({ ...r }));
    },

    async getFeatureRequest(id) {
      const r = featureRequests.get(id);
      return r ? { ...r } : null;
    },

    async updateFeatureRequest(input) {
      const r = featureRequests.get(input.id);
      if (!r || r.userId !== input.userId || r.status !== "pendente") return null;
      r.title = input.title;
      r.description = input.description;
      r.category = input.category;
      r.updatedAt = nowIso();
      featureRequests.set(r.id, r);
      return { ...r };
    },

    async transitionFeatureRequest(input) {
      const r = featureRequests.get(input.id);
      if (!r) return null;
      const from = r.status;
      r.status = input.toStatus;
      r.updatedAt = nowIso();
      if (input.publishedVersion !== undefined) r.publishedVersion = input.publishedVersion;
      if (input.note !== undefined && input.note !== null) r.adminNote = input.note;
      requestEvents.push({
        id: crypto.randomUUID(),
        requestId: r.id,
        fromStatus: from,
        toStatus: input.toStatus,
        actorUserId: input.actorUserId,
        actorRole: input.actorRole,
        note: input.note ?? null,
        createdAt: nowIso(),
      });
      featureRequests.set(r.id, r);
      return { ...r };
    },

    async createUserNotification(input) {
      const n: import("./types").UserNotification = {
        ...input,
        id: crypto.randomUUID(),
        readAt: null,
        createdAt: input.createdAt ?? nowIso(),
      };
      notifications.push(n);
      return { ...n };
    },

    async listUserNotifications(userId, limit = 30) {
      return notifications
        .filter((n) => n.userId === userId)
        .sort((a, b) => (a.createdAt < b.createdAt ? 1 : -1))
        .slice(0, limit)
        .map((n) => ({ ...n }));
    },

    async markNotificationRead(id, userId) {
      const n = notifications.find((item) => item.id === id && item.userId === userId);
      if (!n) return false;
      n.readAt = nowIso();
      return true;
    },

    async countUnreadNotifications(userId) {
      return notifications.filter((n) => n.userId === userId && !n.readAt).length;
    },
```

- [ ] **Step 4: implementar no store SQLite**

Adicionar imports em `/site/lib/db/sqlite.ts`:

```ts
import {
  rowToFeatureRequest,
  rowToFeatureRequestEvent,
  rowToUserNotification,
  rowToPaymentEvent,
} from "./sql-shared";
import type {
  FeatureRequest,
  FeatureRequestEvent,
  PaymentEvent,
  UserNotification,
} from "./types";
```

Inserir no objeto `wrap(db)`:

```ts
    async markPaidOnce({ orderId, paymentId, paidAt }) {
      const current = await this.getOrderById(orderId);
      if (!current) return null;
      if (current.mpPaymentId && current.mpPaymentId !== paymentId) return null;
      const changed = current.status !== "paid" || !current.mpPaymentId;
      if (changed) {
        db.prepare(
          "UPDATE orders SET status = 'paid', mp_payment_id = ?, paid_at = ? WHERE id = ? AND status != 'paid'",
        ).run(paymentId, nowIso(paidAt ?? new Date()), orderId);
      }
      const order = await this.getOrderById(orderId);
      return order ? { order, changed } : null;
    },

    async addPaymentEvent(entry) {
      db.prepare(
        `INSERT INTO payment_events (id, order_id, payment_id, event_type, result, reason_code, created_at)
         VALUES (?, ?, ?, ?, ?, ?, ?)`,
      ).run(
        crypto.randomUUID(), entry.orderId, entry.paymentId, entry.eventType,
        entry.result, entry.reasonCode, entry.createdAt ?? nowIso(),
      );
    },

    async createFeatureRequest(input) {
      const now = nowIso();
      const id = crypto.randomUUID();
      db.prepare(
        `INSERT INTO feature_requests
          (id, user_id, title, description, category, status, published_version, admin_note, created_at, updated_at)
         VALUES (?, ?, ?, ?, ?, 'pendente', NULL, NULL, ?, ?)`,
      ).run(id, input.userId, input.title, input.description, input.category, now, now);
      return (await this.getFeatureRequest(id))!;
    },

    async listFeatureRequests(userId) {
      const rows = userId
        ? db.prepare("SELECT * FROM feature_requests WHERE user_id = ? ORDER BY created_at DESC").all(userId)
        : db.prepare("SELECT * FROM feature_requests ORDER BY created_at DESC").all();
      return (rows as Record<string, unknown>[]).map(rowToFeatureRequest);
    },

    async getFeatureRequest(id) {
      const row = db.prepare("SELECT * FROM feature_requests WHERE id = ?").get(id);
      return row ? rowToFeatureRequest(row as Record<string, unknown>) : null;
    },

    async updateFeatureRequest(input) {
      const current = await this.getFeatureRequest(input.id);
      if (!current || current.userId !== input.userId || current.status !== "pendente") return null;
      db.prepare(
        "UPDATE feature_requests SET title = ?, description = ?, category = ?, updated_at = ? WHERE id = ?",
      ).run(input.title, input.description, input.category, nowIso(), input.id);
      return this.getFeatureRequest(input.id);
    },

    async transitionFeatureRequest(input) {
      const current = await this.getFeatureRequest(input.id);
      if (!current) return null;
      const now = nowIso();
      db.prepare(
        `UPDATE feature_requests SET status = ?, published_version = ?, admin_note = ?, updated_at = ? WHERE id = ?`,
      ).run(
        input.toStatus,
        input.publishedVersion ?? current.publishedVersion,
        input.note ?? current.adminNote,
        now,
        input.id,
      );
      db.prepare(
        `INSERT INTO feature_request_events
          (id, request_id, from_status, to_status, actor_user_id, actor_role, note, created_at)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
      ).run(
        crypto.randomUUID(), input.id, current.status, input.toStatus,
        input.actorUserId, input.actorRole, input.note ?? null, now,
      );
      return this.getFeatureRequest(input.id);
    },

    async createUserNotification(input) {
      const id = crypto.randomUUID();
      const createdAt = input.createdAt ?? nowIso();
      db.prepare(
        `INSERT INTO user_notifications
          (id, user_id, kind, title, body, request_id, read_at, created_at, expires_at)
         VALUES (?, ?, ?, ?, ?, ?, NULL, ?, ?)`,
      ).run(id, input.userId, input.kind, input.title, input.body, input.requestId, createdAt, input.expiresAt);
      const row = db.prepare("SELECT * FROM user_notifications WHERE id = ?").get(id);
      return rowToUserNotification(row as Record<string, unknown>);
    },

    async listUserNotifications(userId, limit = 30) {
      const rows = db.prepare(
        "SELECT * FROM user_notifications WHERE user_id = ? ORDER BY created_at DESC LIMIT ?",
      ).all(userId, limit);
      return (rows as Record<string, unknown>[]).map(rowToUserNotification);
    },

    async markNotificationRead(id, userId) {
      const result = db.prepare(
        "UPDATE user_notifications SET read_at = ? WHERE id = ? AND user_id = ? AND read_at IS NULL",
      ).run(nowIso(), id, userId);
      return result.changes > 0;
    },

    async countUnreadNotifications(userId) {
      const row = db.prepare(
        "SELECT COUNT(*) AS n FROM user_notifications WHERE user_id = ? AND read_at IS NULL",
      ).get(userId) as { n: number };
      return Number(row.n);
    },
```

- [ ] **Step 5: implementar no store Postgres**

Adicionar imports em `/site/lib/db/postgres.ts`:

```ts
import {
  rowToFeatureRequest,
  rowToUserNotification,
  rowToPaymentEvent,
} from "./sql-shared";
```

Inserir no objeto `wrap(sql)`:

```ts
    async markPaidOnce({ orderId, paymentId, paidAt }) {
      const current = await this.getOrderById(orderId);
      if (!current) return null;
      if (current.mpPaymentId && current.mpPaymentId !== paymentId) return null;
      const changed = current.status !== "paid" || !current.mpPaymentId;
      if (changed) {
        const rows = await sql`
          UPDATE orders SET status = 'paid', mp_payment_id = ${paymentId}, paid_at = ${nowIso(paidAt ?? new Date())}
          WHERE id = ${orderId} AND status <> 'paid'
          RETURNING id
        `;
        if (rows.length === 0) return null;
      }
      const order = await this.getOrderById(orderId);
      return order ? { order, changed } : null;
    },

    async addPaymentEvent(entry) {
      await sql`
        INSERT INTO payment_events (id, order_id, payment_id, event_type, result, reason_code, created_at)
        VALUES (${crypto.randomUUID()}, ${entry.orderId}, ${entry.paymentId}, ${entry.eventType}, ${entry.result}, ${entry.reasonCode}, ${entry.createdAt ?? nowIso()})
      `;
    },

    async createFeatureRequest(input) {
      const now = nowIso();
      const id = crypto.randomUUID();
      await sql`
        INSERT INTO feature_requests
          (id, user_id, title, description, category, status, published_version, admin_note, created_at, updated_at)
        VALUES (${id}, ${input.userId}, ${input.title}, ${input.description}, ${input.category}, 'pendente', NULL, NULL, ${now}, ${now})
      `;
      return (await this.getFeatureRequest(id))!;
    },

    async listFeatureRequests(userId) {
      const rows = userId
        ? await sql`SELECT * FROM feature_requests WHERE user_id = ${userId} ORDER BY created_at DESC`
        : await sql`SELECT * FROM feature_requests ORDER BY created_at DESC`;
      return rows.map((r) => rowToFeatureRequest(r as Record<string, unknown>));
    },

    async getFeatureRequest(id) {
      const rows = await sql`SELECT * FROM feature_requests WHERE id = ${id}`;
      const row = one(rows);
      return row ? rowToFeatureRequest(row) : null;
    },

    async updateFeatureRequest(input) {
      const current = await this.getFeatureRequest(input.id);
      if (!current || current.userId !== input.userId || current.status !== "pendente") return null;
      await sql`
        UPDATE feature_requests SET title = ${input.title}, description = ${input.description},
          category = ${input.category}, updated_at = ${nowIso()} WHERE id = ${input.id}
      `;
      return this.getFeatureRequest(input.id);
    },

    async transitionFeatureRequest(input) {
      const current = await this.getFeatureRequest(input.id);
      if (!current) return null;
      const now = nowIso();
      await sql`
        UPDATE feature_requests SET status = ${input.toStatus},
          published_version = ${input.publishedVersion ?? current.publishedVersion},
          admin_note = ${input.note ?? current.adminNote}, updated_at = ${now}
        WHERE id = ${input.id}
      `;
      await sql`
        INSERT INTO feature_request_events
          (id, request_id, from_status, to_status, actor_user_id, actor_role, note, created_at)
        VALUES (${crypto.randomUUID()}, ${input.id}, ${current.status}, ${input.toStatus},
                ${input.actorUserId}, ${input.actorRole}, ${input.note ?? null}, ${now})
      `;
      return this.getFeatureRequest(input.id);
    },

    async createUserNotification(input) {
      const id = crypto.randomUUID();
      const createdAt = input.createdAt ?? nowIso();
      await sql`
        INSERT INTO user_notifications
          (id, user_id, kind, title, body, request_id, read_at, created_at, expires_at)
        VALUES (${id}, ${input.userId}, ${input.kind}, ${input.title}, ${input.body},
                ${input.requestId}, NULL, ${createdAt}, ${input.expiresAt})
      `;
      const rows = await sql`SELECT * FROM user_notifications WHERE id = ${id}`;
      return rowToUserNotification(one(rows)!);
    },

    async listUserNotifications(userId, limit = 30) {
      const rows = await sql`SELECT * FROM user_notifications WHERE user_id = ${userId} ORDER BY created_at DESC LIMIT ${limit}`;
      return rows.map((r) => rowToUserNotification(r as Record<string, unknown>));
    },

    async markNotificationRead(id, userId) {
      const rows = await sql`
        UPDATE user_notifications SET read_at = ${nowIso()}
        WHERE id = ${id} AND user_id = ${userId} AND read_at IS NULL
        RETURNING id
      `;
      return rows.length > 0;
    },

    async countUnreadNotifications(userId) {
      const rows = await sql`SELECT COUNT(*)::int AS n FROM user_notifications WHERE user_id = ${userId} AND read_at IS NULL`;
      return Number((one(rows) as { n: number } | null)?.n ?? 0);
    },
```

- [ ] **Step 6: rodar os testes**

Run: `npm test -- tests/stores-new-methods.test.ts tests/db-schema.test.ts` em `/site`.

Expected: todos passam.

- [ ] **Step 7: rodar a suíte completa para detectar regressões**

Run: `npm test` em `/site`.

Expected: nenhum teste falha.

- [ ] **Step 8: commit**

```bash
git add lib/db/memory.ts lib/db/sqlite.ts lib/db/postgres.ts tests/stores-new-methods.test.ts
git commit -m "feat(db): implementa pagamento único e solicitações nos stores"
```

---

### Task 5: Registrar eventos de pagamento no webhook e evitar dupla liberação

**Files:**
- Modify: `/site/lib/fulfill.ts`
- Modify: `/site/lib/mp-webhook.ts`
- Test: `/site/tests/mp-webhook-idempotency.test.ts`

- [ ] **Step 1: escrever o teste de idempotência**

Create `/site/tests/mp-webhook-idempotency.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { fulfillApprovedPayment } from "../lib/fulfill";
import { createMemoryStore } from "../lib/db/memory";
import type { MpPayment } from "../lib/mp-checkout";

const payment: MpPayment = {
  id: 555,
  status: "approved",
  currency_id: "BRL",
  transaction_amount: 20,
  external_reference: "pedido-1",
};

describe("fulfillApprovedPayment", () => {
  it("cria uma licença e não emite token em webhook duplicado", async () => {
    const store = createMemoryStore();
    await store.createOrder({ id: "pedido-1", email: "cliente@example.com", amountBrl: 20 });
    const first = await fulfillApprovedPayment({ store, orderId: "pedido-1", paymentId: "555", payment });
    const second = await fulfillApprovedPayment({ store, orderId: "pedido-1", paymentId: "555", payment });
    expect(first.license.id).toBe(second.license.id);
    expect(first.alreadyPaid).toBe(false);
    expect(second.alreadyPaid).toBe(true);
    expect(second.downloadToken).toBe("");
    expect((await store.getLicensesByEmail("cliente@example.com")).length).toBe(1);
  });
});
```

- [ ] **Step 2: confirmar que falha**

Run: `npm test -- tests/mp-webhook-idempotency.test.ts` em `/site`.

Expected: falha porque `fulfill` ainda usa `markPaid`.

- [ ] **Step 3: substituir a liberação em `/site/lib/fulfill.ts`**

```ts
export async function fulfillApprovedPayment(opts: {
  store?: Store;
  orderId: string;
  paymentId: string;
  payment?: MpPayment;
}): Promise<FulfillResult> {
  const store = opts.store ?? (await getStore());
  const byPayment = await store.getOrderByPaymentId(String(opts.paymentId));
  const order = await store.getOrderById(opts.orderId);
  if (!order) throw new Error("pedido não encontrado");
  if (byPayment && byPayment.id !== order.id) {
    throw new Error("pagamento já associado a outro pedido");
  }
  if (opts.payment) {
    const check = paymentMatchesOrder(opts.payment, order);
    if (!check.ok) throw new Error(check.reason ?? "pagamento inválido");
  }

  const result = await store.markPaidOnce({
    orderId: order.id,
    paymentId: String(opts.paymentId),
  });
  if (!result) throw new Error("pagamento já associado a outro pedido");
  const paid = result.order;
  const alreadyPaid = !result.changed;

  let license = await store.getLicenseByOrderId(paid.id);
  if (!license) {
    license = await store.createLicense(paid.id, generateLicenseKey());
  }

  if (alreadyPaid) {
    return {
      order: paid,
      license,
      downloadToken: "",
      expiresAt: paid.paidAt ?? new Date().toISOString(),
      alreadyPaid: true,
    };
  }

  const issued = await issueDownloadToken(store, paid, license);
  void maybeSendLicenseEmail({
    email: paid.email,
    licenseKey: license.key,
    downloadToken: issued.token,
    expiresAt: issued.expiresAt,
    orderId: paid.id,
  }).catch(() => undefined);

  return {
    order: paid,
    license,
    downloadToken: issued.token,
    expiresAt: issued.expiresAt,
    alreadyPaid: false,
  };
}
```

- [ ] **Step 4: acrescentar registro de eventos no processador do webhook**

Em `/site/lib/mp-webhook.ts`, importar:

```ts
import { normalizePaymentEvent } from "./payment-events";
```

Logo após `const dataId = pickDataId(...)`, incluir auxiliar local:

```ts
  const logEvent = async (
    eventType: "webhook" | "fulfill" | "duplicate" | "match_failed" | "api_error",
    result: "ok" | "skipped" | "error",
    reasonCode: string,
    orderId: string | null = null,
  ) => {
    const target = opts.store ?? (await getStore());
    await target.addPaymentEvent(normalizePaymentEvent({
      orderId,
      paymentId: dataId || null,
      eventType,
      result,
      reasonCode,
    }));
  };
```

Nos pontos de retorno, chamar:

```ts
if (!result.ok) {
  await logEvent("webhook", "error", "signature_invalid");
  return { ok: false, error: "assinatura inválida", status: 401 };
}
if (!dataId) return { ok: true, skipped: "no_payment_id" };
if (!isPaymentTopic(opts.body, opts.url)) {
  await logEvent("webhook", "skipped", "ignored_topic");
  return { ok: true, skipped: "ignored_topic" };
}
if (!chargesEnabled()) {
  await logEvent("webhook", "skipped", "charges_disabled");
  return { ok: true, skipped: "charges_disabled" };
}
```

Após `fetchPayment` falhar:

```ts
await logEvent("api_error", "error", "payment_fetch_failed");
return { ok: false, error: "falha ao consultar pagamento", status: 500 };
```

Após `order` ausente:

```ts
await logEvent("webhook", "skipped", "unknown_order");
return { ok: true, skipped: "unknown_order" };
```

Após `check.ok === false`:

```ts
await logEvent("match_failed", "skipped", check.reason ?? "match_failed", order.id);
```

Após liberação:

```ts
const fulfilled = await fulfillApprovedPayment({...});
await logEvent(
  fulfilled.alreadyPaid ? "duplicate" : "fulfill",
  "ok",
  fulfilled.alreadyPaid ? "duplicate" : "approved",
  order.id,
);
```

- [ ] **Step 5: rodar os testes de pagamento**

Run: `npm test -- tests/mp-payment-hardening.test.ts tests/mp-webhook.test.ts tests/mp-webhook-idempotency.test.ts` em `/site`.

Expected: todos passam.

- [ ] **Step 6: commit**

```bash
git add lib/fulfill.ts lib/mp-webhook.ts tests/mp-webhook-idempotency.test.ts
git commit -m "fix(payments): liberação idempotente e auditoria do webhook"
```

---

### Task 6: Serviço de solicitações e notificações

**Files:**
- Create: `/site/lib/feature-requests.ts`
- Test: `/site/tests/feature-requests.test.ts`

- [ ] **Step 1: escrever o teste de regras**

Create `/site/tests/feature-requests.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import {
  ADMIN_TRANSITIONS,
  createRequest,
  transitionRequest,
  validateRequestInput,
} from "../lib/feature-requests";

describe("solicitações", () => {
  it("valida limites de texto", () => {
    expect(validateRequestInput({ title: "Tema", description: "Azul", category: "tema" })).toEqual({ ok: true });
    expect(validateRequestInput({ title: "a".repeat(121), description: "x", category: "tema" }).ok).toBe(false);
  });

  it("cria pendente e notifica apenas em transições administrativas", async () => {
    const store = createMemoryStore();
    await store.createUser({ id: "u1", email: "u@example.com", passwordHash: "h" });
    const request = await createRequest(store, "u1", {
      title: "Atalho",
      description: "Criar atalho para tema",
      category: "personalizacao",
    });
    expect(request.status).toBe("pendente");
    expect(await store.countUnreadNotifications("u1")).toBe(0);

    await transitionRequest(store, {
      id: request.id,
      actorUserId: "admin",
      actorRole: "admin",
      toStatus: "aprovada",
      note: "Aprovado",
    });
    const notes = await store.listUserNotifications("u1");
    expect(notes[0]?.kind).toBe("request.aprovada");
    expect(ADMIN_TRANSITIONS.has("publicada")).toBe(true);
  });

  it("impede que usuário publique diretamente", async () => {
    const store = createMemoryStore();
    await store.createUser({ id: "u1", email: "u@example.com", passwordHash: "h" });
    const request = await createRequest(store, "u1", {
      title: "Item",
      description: "Descrição",
      category: "geral",
    });
    await expect(transitionRequest(store, {
      id: request.id,
      actorUserId: "u1",
      actorRole: "user",
      toStatus: "publicada",
    })).rejects.toThrow("apenas administrador");
  });
});
```

- [ ] **Step 2: confirmar falha por módulo inexistente**

Run: `npm test -- tests/feature-requests.test.ts` em `/site`.

Expected: falha `Cannot find module`.

- [ ] **Step 3: criar o serviço**

Create `/site/lib/feature-requests.ts`:

```ts
import { getStore, type FeatureRequestStatus, type Store } from "./db";

export const ADMIN_TRANSITIONS = new Set<FeatureRequestStatus>([
  "aprovada",
  "rejeitada",
  "em_desenvolvimento",
  "publicada",
]);

const VALID_STATUSES = new Set<FeatureRequestStatus>([
  "pendente",
  "aprovada",
  "rejeitada",
  "em_desenvolvimento",
  "publicada",
  "retirada",
]);

export type RequestInput = {
  title: string;
  description: string;
  category: string;
};

export type TransitionInput = {
  id: string;
  actorUserId: string;
  actorRole: "user" | "admin";
  toStatus: FeatureRequestStatus;
  note?: string | null;
  publishedVersion?: string | null;
};

export function validateRequestInput(input: RequestInput): { ok: true } | { ok: false; reason: string } {
  const title = input.title.trim();
  const description = input.description.trim();
  const category = input.category.trim();
  if (title.length < 3 || title.length > 120) return { ok: false, reason: "titulo_invalido" };
  if (description.length < 10 || description.length > 4000) return { ok: false, reason: "descricao_invalida" };
  if (!/^[a-z0-9_-]{2,32}$/.test(category)) return { ok: false, reason: "categoria_invalida" };
  return { ok: true };
}

export async function createRequest(store: Store, userId: string, input: RequestInput) {
  const check = validateRequestInput(input);
  if (!check.ok) throw new Error(check.reason);
  return store.createFeatureRequest({
    userId,
    title: input.title.trim(),
    description: input.description.trim(),
    category: input.category.trim(),
  });
}

export async function transitionRequest(store: Store, input: TransitionInput) {
  if (!VALID_STATUSES.has(input.toStatus)) throw new Error("status_invalido");
  if (ADMIN_TRANSITIONS.has(input.toStatus) && input.actorRole !== "admin") {
    throw new Error("apenas administrador pode aprovar ou publicar");
  }
  if (input.toStatus === "publicada" && !/^\d+\.\d+\.\d+$/.test(input.publishedVersion ?? "")) {
    throw new Error("versao_publicacao_invalida");
  }
  const updated = await store.transitionFeatureRequest({
    id: input.id,
    actorUserId: input.actorUserId,
    actorRole: input.actorRole,
    toStatus: input.toStatus,
    note: input.note?.slice(0, 1000) ?? null,
    publishedVersion: input.publishedVersion ?? null,
  });
  if (!updated) throw new Error("solicitacao_nao_encontrada");

  if (input.actorRole === "admin") {
    const titles: Record<string, string> = {
      aprovada: "Solicitação aprovada",
      rejeitada: "Solicitação rejeitada",
      em_desenvolvimento: "Solicitação em desenvolvimento",
      publicada: "Solicitação publicada",
    };
    await store.createUserNotification({
      userId: updated.userId,
      kind: `request.${input.toStatus}`,
      title: titles[input.toStatus] ?? "Atualização da solicitação",
      body: `Sua solicitação "${updated.title}" agora está ${input.toStatus}.`,
      requestId: updated.id,
      expiresAt: null,
    });
  }
  return updated;
}
```

- [ ] **Step 4: rodar o teste**

Run: `npm test -- tests/feature-requests.test.ts` em `/site`.

Expected: 3 testes passam.

- [ ] **Step 5: commit**

```bash
git add lib/feature-requests.ts tests/feature-requests.test.ts
git commit -m "feat(requests): regras de criação e aprovação administrativa"
```

---

### Task 7: Autenticação de dispositivo para o aplicativo

**Files:**
- Create: `/site/lib/desktop-auth.ts`
- Test: `/site/tests/desktop-auth.test.ts`

- [ ] **Step 1: escrever o teste de login e revogação**

Create `/site/tests/desktop-auth.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { hashPassword } from "../lib/auth";
import { desktopLogin, getDesktopSession, revokeDesktopDevice } from "../lib/desktop-auth";

async function seed() {
  const store = createMemoryStore();
  await store.createUser({
    id: "u1",
    email: "user@example.com",
    passwordHash: await hashPassword("senha-forte-123"),
  });
  return store;
}

describe("desktop-auth", () => {
  it("troca senha por token e devolve sessão com licença", async () => {
    const store = await seed();
    const login = await desktopLogin(store, "user@example.com", "senha-forte-123", "YAMI OS 0.8.0");
    expect(login.deviceToken).toMatch(/^[a-f0-9]{64}$/);
    const session = await getDesktopSession(store, login.deviceToken);
    expect(session?.user.email).toBe("user@example.com");
    expect(session?.hasLicense).toBe(false);
  });

  it("rejeita senha errada e token revogado", async () => {
    const store = await seed();
    await expect(desktopLogin(store, "user@example.com", "errada", "PC")).rejects.toThrow("invalid_credentials");
    const login = await desktopLogin(store, "user@example.com", "senha-forte-123", "PC");
    await revokeDesktopDevice(store, login.device.id);
    expect(await getDesktopSession(store, login.deviceToken)).toBeNull();
  });

  it("limita a cinco dispositivos ativos", async () => {
    const store = await seed();
    for (let i = 0; i < 6; i++) {
      await desktopLogin(store, "user@example.com", "senha-forte-123", `PC ${i}`);
    }
    const devices = await store.listDevices("u1");
    expect(devices.filter((d) => !d.revokedAt).length).toBe(5);
  });
});
```

- [ ] **Step 2: executar e observar falha**

Run: `npm test -- tests/desktop-auth.test.ts` em `/site`.

Expected: falha de módulo inexistente.

- [ ] **Step 3: implementar o serviço**

Create `/site/lib/desktop-auth.ts`:

```ts
import { generateToken, hashSecret, verifyPassword } from "./auth";
import { getStore, type Device, type Store } from "./db";

export const MAX_DESKTOP_DEVICES = 5;

export async function desktopLogin(
  store: Store,
  email: string,
  password: string,
  deviceName: string,
): Promise<{ device: Device; deviceToken: string }> {
  const normalized = email.trim().toLowerCase();
  const user = await store.getUserByEmail(normalized);
  if (!user || !(await verifyPassword(password, user.passwordHash))) {
    throw new Error("invalid_credentials");
  }

  const active = (await store.listDevices(user.id)).filter((d) => !d.revokedAt);
  if (active.length >= MAX_DESKTOP_DEVICES) {
    const oldest = active.sort((a, b) => (a.createdAt < b.createdAt ? -1 : 1))[0]!;
    await store.revokeDevice(oldest.id, user.id);
  }

  const deviceToken = generateToken(32);
  const device = await store.createDevice({
    id: crypto.randomUUID(),
    userId: user.id,
    name: deviceName.trim().slice(0, 80) || "YAMI OS",
    tokenHash: hashSecret(deviceToken),
  });
  await store.addAudit({
    id: crypto.randomUUID(),
    userId: user.id,
    deviceId: device.id,
    action: "desktop.login",
    ip: null,
    meta: device.name,
  });
  return { device, deviceToken };
}

export async function getDesktopSession(store: Store, deviceToken: string) {
  const device = await store.getDeviceByTokenHash(hashSecret(deviceToken));
  if (!device || device.revokedAt) return null;
  const user = await store.getUserById(device.userId);
  if (!user) return null;
  await store.touchDevice(device.id);
  const licenses = await store.getLicensesByEmail(user.email);
  const orders = await store.getOrdersByEmail(user.email);
  const unread = await store.countUnreadNotifications(user.id);
  return {
    user: { id: user.id, email: user.email, name: user.name },
    device: { id: device.id, name: device.name },
    license: licenses[0]
      ? { key: licenses[0].key, status: "ativa", createdAt: licenses[0].createdAt }
      : null,
    hasLicense: licenses.length > 0,
    orders: orders.map((o) => ({
      id: o.id,
      status: o.status,
      amountBrl: o.amountBrl,
      paidAt: o.paidAt,
      createdAt: o.createdAt,
    })),
    unreadNotifications: unread,
  };
}

export async function revokeDesktopDevice(store: Store, deviceId: string, userId?: string) {
  const device = await store.getDeviceById(deviceId);
  if (!device || (userId && device.userId !== userId)) return false;
  await store.revokeDevice(deviceId, device.userId);
  await store.addAudit({
    id: crypto.randomUUID(),
    userId: device.userId,
    deviceId,
    action: "desktop.logout",
    ip: null,
    meta: null,
  });
  return true;
}

export function bearerToken(req: Request): string | null {
  const header = req.headers.get("authorization") ?? "";
  const match = header.match(/^Bearer\s+([A-Za-z0-9_-]{32,128})$/i);
  return match?.[1] ?? null;
}
```

- [ ] **Step 4: rodar o teste**

Run: `npm test -- tests/desktop-auth.test.ts` em `/site`.

Expected: 3 testes passam.

- [ ] **Step 5: commit**

```bash
git add lib/desktop-auth.ts tests/desktop-auth.test.ts
git commit -m "feat(desktop): login por token de dispositivo"
```

---

### Task 8: Rotas HTTP para o aplicativo e administração

**Files:**
- Create: `/site/app/api/desktop/login/route.ts`
- Create: `/site/app/api/desktop/session/route.ts`
- Create: `/site/app/api/desktop/logout/route.ts`
- Create: `/site/app/api/desktop/requests/route.ts`
- Create: `/site/app/api/desktop/requests/[id]/route.ts`
- Create: `/site/app/api/desktop/notifications/route.ts`
- Create: `/site/app/api/desktop/notifications/[id]/route.ts`
- Create: `/site/app/api/admin/requests/route.ts`
- Create: `/site/app/api/admin/requests/[id]/route.ts`
- Test: `/site/tests/desktop-routes.test.ts`

- [ ] **Step 1: criar helper de resposta e autenticação compartilhado**

Create `/site/app/api/desktop/_shared.ts`:

```ts
import { NextResponse } from "next/server";
import { getStore } from "@/lib/db";
import { bearerToken, getDesktopSession } from "@/lib/desktop-auth";
import { getSessionUser, readSessionCookie } from "@/lib/auth";
import { isAdminEmail } from "@/lib/admin";

export function jsonError(code: string, status: number) {
  return NextResponse.json({ error: code }, { status });
}

export async function desktopSession(req: Request) {
  const token = bearerToken(req);
  if (!token) return null;
  const store = await getStore();
  const session = await getDesktopSession(store, token);
  if (!session) return null;
  return { store, session, token };
}

export async function adminFromRequest(req: Request) {
  const current = await getSessionUser(readSessionCookie(req));
  if (!current || !isAdminEmail(current.user.email)) return null;
  return current;
}
```

- [ ] **Step 2: testar as rotas com requests sintéticos**

Create `/site/tests/desktop-routes.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { hashPassword } from "../lib/auth";
import { setStoreForTests } from "../lib/db";
import { desktopLogin } from "../lib/desktop-auth";
import { POST as loginPost } from "../app/api/desktop/login/route";
import { GET as sessionGet } from "../app/api/desktop/session/route";
import { POST as requestPost } from "../app/api/desktop/requests/route";

async function seed() {
  const store = createMemoryStore();
  setStoreForTests(store);
  await store.createUser({ id: "u1", email: "u@example.com", passwordHash: await hashPassword("senha-forte-123") });
  return store;
}

function req(url: string, init?: RequestInit) {
  return new Request(`http://localhost:3000${url}`, init);
}

describe("rotas desktop", () => {
  it("faz login, lê sessão e cria solicitação", async () => {
    await seed();
    const loginRes = await loginPost(req("/api/desktop/login", {
      method: "POST",
      headers: { "content-type": "application/json", origin: "http://localhost:3000" },
      body: JSON.stringify({ email: "u@example.com", password: "senha-forte-123", deviceName: "YAMI OS" }),
    }));
    expect(loginRes.status).toBe(200);
    const { deviceToken } = await loginRes.json();

    const headers = { authorization: `Bearer ${deviceToken}` };
    const sessionRes = await sessionGet(req("/api/desktop/session", { headers }));
    expect(sessionRes.status).toBe(200);

    const createRes = await requestPost(req("/api/desktop/requests", {
      method: "POST",
      headers: { ...headers, "content-type": "application/json" },
      body: JSON.stringify({ title: "Novo tema", description: "Adicionar tema azul", category: "tema" }),
    }));
    expect(createRes.status).toBe(201);
    expect((await createRes.json()).status).toBe("pendente");
  });
});
```

- [ ] **Step 3: executar e observar falha**

Run: `npm test -- tests/desktop-routes.test.ts` em `/site`.

Expected: falha de módulos inexistentes.

- [ ] **Step 4: criar as rotas**

`/site/app/api/desktop/login/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getStore } from "@/lib/db";
import { desktopLogin } from "@/lib/desktop-auth";
import { clientIp, isValidEmail } from "@/lib/http";
import { rateLimit } from "@/lib/rate-limit";
import { jsonError } from "../_shared";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function POST(req: Request) {
  const origin = req.headers.get("origin");
  if (origin && new URL(origin).origin !== new URL(req.url).origin) {
    return jsonError("invalid_request", 403);
  }
  const limited = rateLimit(`desktop-login:${clientIp(req)}`, 10, 15 * 60 * 1000);
  if (!limited.ok) return jsonError("rate_limited", 429);
  let body: Record<string, unknown>;
  try { body = await req.json(); } catch { return jsonError("invalid_request", 400); }
  const email = String(body.email ?? "").trim().toLowerCase();
  const password = String(body.password ?? "");
  const deviceName = String(body.deviceName ?? "YAMI OS");
  if (!isValidEmail(email) || !password) return jsonError("invalid_credentials", 401);
  try {
    const result = await desktopLogin(await getStore(), email, password, deviceName);
    return NextResponse.json({
      deviceToken: result.deviceToken,
      device: { id: result.device.id, name: result.device.name },
    });
  } catch (error) {
    const code = error instanceof Error ? error.message : "invalid_credentials";
    return jsonError(code === "invalid_credentials" ? code : "invalid_request", 401);
  }
}
```

`/site/app/api/desktop/session/route.ts`:

```ts
import { NextResponse } from "next/server";
import { desktopSession, jsonError } from "../_shared";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function GET(req: Request) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  return NextResponse.json({ ok: true, ...auth.session });
}
```

`/site/app/api/desktop/logout/route.ts`:

```ts
import { NextResponse } from "next/server";
import { revokeDesktopDevice } from "@/lib/desktop-auth";
import { desktopSession, jsonError } from "../_shared";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function POST(req: Request) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  await revokeDesktopDevice(auth.store, auth.session.device.id, auth.session.user.id);
  return NextResponse.json({ ok: true });
}
```

`/site/app/api/desktop/requests/route.ts`:

```ts
import { NextResponse } from "next/server";
import { createRequest } from "@/lib/feature-requests";
import { desktopSession, jsonError } from "../_shared";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function GET(req: Request) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  const items = await auth.store.listFeatureRequests(auth.session.user.id);
  return NextResponse.json({ ok: true, items });
}

export async function POST(req: Request) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  let body: Record<string, unknown>;
  try { body = await req.json(); } catch { return jsonError("invalid_request", 400); }
  try {
    const item = await createRequest(auth.store, auth.session.user.id, {
      title: String(body.title ?? ""),
      description: String(body.description ?? ""),
      category: String(body.category ?? ""),
    });
    return NextResponse.json(item, { status: 201 });
  } catch (error) {
    return jsonError(error instanceof Error ? error.message : "invalid_request", 400);
  }
}
```

`/site/app/api/desktop/requests/[id]/route.ts`:

```ts
import { NextResponse } from "next/server";
import { transitionRequest } from "@/lib/feature-requests";
import { desktopSession, jsonError } from "../../_shared";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

type Ctx = { params: Promise<{ id: string }> };

export async function GET(req: Request, ctx: Ctx) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  const item = await auth.store.getFeatureRequest((await ctx.params).id);
  if (!item || item.userId !== auth.session.user.id) return jsonError("not_found", 404);
  return NextResponse.json({ ok: true, item });
}

export async function PATCH(req: Request, ctx: Ctx) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  let body: Record<string, unknown>;
  try { body = await req.json(); } catch { return jsonError("invalid_request", 400); }
  const current = await auth.store.getFeatureRequest((await ctx.params).id);
  if (!current || current.userId !== auth.session.user.id) return jsonError("not_found", 404);
  if (current.status !== "pendente") return jsonError("forbidden", 403);
  const item = await auth.store.updateFeatureRequest({
    id: current.id,
    userId: auth.session.user.id,
    title: String(body.title ?? current.title),
    description: String(body.description ?? current.description),
    category: String(body.category ?? current.category),
  });
  return item ? NextResponse.json({ ok: true, item }) : jsonError("invalid_request", 400);
}

export async function DELETE(req: Request, ctx: Ctx) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  try {
    const item = await transitionRequest(auth.store, {
      id: (await ctx.params).id,
      actorUserId: auth.session.user.id,
      actorRole: "user",
      toStatus: "retirada",
    });
    return NextResponse.json({ ok: true, item });
  } catch (error) {
    return jsonError(error instanceof Error ? error.message : "invalid_request", 400);
  }
}
```

`/site/app/api/desktop/notifications/route.ts`:

```ts
import { NextResponse } from "next/server";
import { desktopSession, jsonError } from "../_shared";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function GET(req: Request) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  const items = await auth.store.listUserNotifications(auth.session.user.id, 50);
  return NextResponse.json({ ok: true, items, unread: await auth.store.countUnreadNotifications(auth.session.user.id) });
}
```

`/site/app/api/desktop/notifications/[id]/route.ts`:

```ts
import { NextResponse } from "next/server";
import { desktopSession, jsonError } from "../../_shared";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function PATCH(req: Request, ctx: { params: Promise<{ id: string }> }) {
  const auth = await desktopSession(req);
  if (!auth) return jsonError("device_revoked", 401);
  const ok = await auth.store.markNotificationRead((await ctx.params).id, auth.session.user.id);
  return ok ? NextResponse.json({ ok: true }) : jsonError("not_found", 404);
}
```

`/site/app/api/admin/requests/route.ts`:

```ts
import { NextResponse } from "next/server";
import { adminFromRequest, jsonError } from "../../desktop/_shared";
import { getStore } from "@/lib/db";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function GET(req: Request) {
  const admin = await adminFromRequest(req);
  if (!admin) return jsonError("forbidden", 403);
  const items = await (await getStore()).listFeatureRequests();
  return NextResponse.json({ ok: true, items });
}
```

`/site/app/api/admin/requests/[id]/route.ts`:

```ts
import { NextResponse } from "next/server";
import { getStore } from "@/lib/db";
import { transitionRequest } from "@/lib/feature-requests";
import { adminFromRequest, jsonError } from "../../../desktop/_shared";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function POST(req: Request, ctx: { params: Promise<{ id: string }> }) {
  const admin = await adminFromRequest(req);
  if (!admin) return jsonError("forbidden", 403);
  let body: Record<string, unknown>;
  try { body = await req.json(); } catch { return jsonError("invalid_request", 400); }
  try {
    const item = await transitionRequest(await getStore(), {
      id: (await ctx.params).id,
      actorUserId: admin.user.id,
      actorRole: "admin",
      toStatus: String(body.status) as never,
      note: body.note == null ? null : String(body.note),
      publishedVersion: body.publishedVersion == null ? null : String(body.publishedVersion),
    });
    return NextResponse.json({ ok: true, item });
  } catch (error) {
    return jsonError(error instanceof Error ? error.message : "invalid_request", 400);
  }
}
```

- [ ] **Step 5: rodar o teste das rotas**

Run: `npm test -- tests/desktop-routes.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 6: rodar lint e build**

Run: `npm run lint` e `npm run build` em `/site`.

Expected: ambos terminam com código 0.

- [ ] **Step 7: commit**

```bash
git add app/api/desktop app/api/admin/requests tests/desktop-routes.test.ts
git commit -m "feat(api): rotas de conta e solicitações para o sistema"
```

---

### Task 9: Interfaces de solicitações no site

**Files:**
- Create: `/site/components/FeatureRequestsPanel.tsx`
- Create: `/site/components/admin/AdminRequestsPanel.tsx`
- Modify: `/site/app/(site)/conta/page.tsx`
- Modify: `/site/app/(site)/admin/page.tsx`
- Test: `/site/tests/site-panels.test.tsx`

- [ ] **Step 1: escrever teste de renderização e permissão**

Create `/site/tests/site-panels.test.tsx`:

```tsx
import { describe, expect, it } from "vitest";
import { statusLabel } from "../components/FeatureRequestsPanel";

describe("painel de solicitações", () => {
  it("usa rótulos claros para todos os estados", () => {
    expect(statusLabel("pendente")).toBe("Pendente");
    expect(statusLabel("em_desenvolvimento")).toBe("Em desenvolvimento");
    expect(statusLabel("publicada")).toBe("Publicada");
  });
});
```

Ajustar `vitest.config.ts` para incluir `.tsx`:

```ts
include: ["tests/**/*.test.ts", "tests/**/*.test.tsx"],
```

- [ ] **Step 2: executar e observar falha**

Run: `npm test -- tests/site-panels.test.tsx` em `/site`.

Expected: falha de módulo inexistente.

- [ ] **Step 3: criar o painel do usuário**

Create `/site/components/FeatureRequestsPanel.tsx`:

```tsx
"use client";

import { useRouter } from "next/navigation";
import { useState } from "react";
import type { FeatureRequest, FeatureRequestStatus } from "@/lib/db/types";

export function statusLabel(status: FeatureRequestStatus): string {
  const map: Record<FeatureRequestStatus, string> = {
    pendente: "Pendente",
    aprovada: "Aprovada",
    rejeitada: "Rejeitada",
    em_desenvolvimento: "Em desenvolvimento",
    publicada: "Publicada",
    retirada: "Retirada",
  };
  return map[status];
}

export function FeatureRequestsPanel({ items }: { items: FeatureRequest[] }) {
  const router = useRouter();
  const [title, setTitle] = useState("");
  const [description, setDescription] = useState("");
  const [category, setCategory] = useState("geral");
  const [message, setMessage] = useState("");

  async function create(event: React.FormEvent) {
    event.preventDefault();
    setMessage("");
    const res = await fetch("/api/desktop/requests", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ title, description, category }),
    });
    const data = await res.json();
    if (!res.ok) {
      setMessage(data.error ?? "Não foi possível enviar.");
      return;
    }
    setTitle("");
    setDescription("");
    setMessage("Solicitação enviada.");
    router.refresh();
  }

  return (
    <section className="mt-12">
      <h2 className="text-2xl font-semibold">Solicitações</h2>
      <p className="mt-2 text-sm text-muted">Envie ideias e acompanhe a aprovação. Somente a equipe YAMI publica alterações.</p>
      <form onSubmit={create} className="card mt-4 space-y-3 p-4">
        <input className="field w-full" value={title} onChange={(e) => setTitle(e.target.value)} placeholder="Título" maxLength={120} required />
        <input className="field w-full" value={category} onChange={(e) => setCategory(e.target.value)} placeholder="Categoria" maxLength={32} required />
        <textarea className="field w-full" value={description} onChange={(e) => setDescription(e.target.value)} placeholder="Descreva sua solicitação" maxLength={4000} required />
        <button className="btn-primary" type="submit">Enviar solicitação</button>
        {message ? <p className="text-sm" role="status">{message}</p> : null}
      </form>
      <ul className="mt-6 space-y-3">
        {items.map((item) => (
          <li key={item.id} className="card flex flex-wrap justify-between gap-3 p-4 text-sm">
            <div>
              <p className="font-medium">{item.title}</p>
              <p className="text-xs text-muted">{item.category} · {new Date(item.createdAt).toLocaleString("pt-BR")}</p>
            </div>
            <span className="text-xs">{statusLabel(item.status)}{item.publishedVersion ? ` · ${item.publishedVersion}` : ""}</span>
          </li>
        ))}
      </ul>
    </section>
  );
}
```

- [ ] **Step 4: criar o painel administrativo**

Create `/site/components/admin/AdminRequestsPanel.tsx`:

```tsx
"use client";

import { useRouter } from "next/navigation";
import { useState } from "react";
import type { FeatureRequest } from "@/lib/db/types";
import { statusLabel } from "../FeatureRequestsPanel";

export function AdminRequestsPanel({ items }: { items: FeatureRequest[] }) {
  const router = useRouter();
  const [note, setNote] = useState<Record<string, string>>({});
  const [version, setVersion] = useState<Record<string, string>>({});
  const [message, setMessage] = useState("");

  async function transition(id: string, status: string, request: FeatureRequest) {
    setMessage("");
    const publishedVersion = status === "publicada" ? (version[id] ?? "") : null;
    const res = await fetch(`/api/admin/requests/${id}`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ status, note: note[id] ?? "", publishedVersion }),
    });
    const data = await res.json();
    if (!res.ok) {
      setMessage(data.error ?? "Falha ao atualizar.");
      return;
    }
    setMessage(`${request.title}: ${statusLabel(data.item.status)}.`);
    router.refresh();
  }

  return (
    <section className="card mt-8 p-6">
      <h2 className="text-2xl font-semibold">Solicitações pendentes</h2>
      {message ? <p className="mt-3 text-sm" role="status">{message}</p> : null}
      <ul className="mt-5 space-y-5">
        {items.filter((i) => i.status === "pendente").map((item) => (
          <li key={item.id} className="border-b border-line pb-5">
            <p className="font-medium">{item.title}</p>
            <p className="mt-1 text-sm text-muted">{item.description}</p>
            <input className="field mt-3 w-full" placeholder="Nota administrativa" value={note[item.id] ?? ""} onChange={(e) => setNote({ ...note, [item.id]: e.target.value })} />
            <div className="mt-3 flex flex-wrap gap-2">
              <button className="btn-primary" onClick={() => transition(item.id, "aprovada", item)}>Aprovar</button>
              <button className="btn-ghost" onClick={() => transition(item.id, "rejeitada", item)}>Rejeitar</button>
              <button className="btn-ghost" onClick={() => transition(item.id, "em_desenvolvimento", item)}>Iniciar desenvolvimento</button>
              <span className="inline-flex items-center gap-2 text-xs">
                <input className="field w-24" placeholder="0.8.0" value={version[item.id] ?? ""} onChange={(e) => setVersion({ ...version, [item.id]: e.target.value })} />
                <button className="btn-ghost" onClick={() => transition(item.id, "publicada", item)}>Publicada</button>
              </span>
            </div>
          </li>
        ))}
      </ul>
    </section>
  );
}
```

- [ ] **Step 5: integrar nas páginas**

Em `/site/app/(site)/conta/page.tsx`, importar `FeatureRequestsPanel` e acrescentar antes do fechamento de `<article>`:

```tsx
      <FeatureRequestsPanel items={requests.map((r) => ({
        id: r.id,
        userId: r.userId,
        title: r.title,
        description: r.description,
        category: r.category,
        status: r.status,
        publishedVersion: r.publishedVersion,
        adminNote: r.adminNote,
        createdAt: r.createdAt,
        updatedAt: r.updatedAt,
      }))} />
```

No carregamento da página:

```tsx
  const requests = await store.listFeatureRequests(current.user.id);
```

Em `/site/app/(site)/admin/page.tsx`, importar `AdminRequestsPanel`, carregar:

```tsx
  const requests = await store.listFeatureRequests();
```

e acrescentar:

```tsx
      <AdminRequestsPanel items={requests.map((r) => ({
        id: r.id,
        userId: r.userId,
        title: r.title,
        description: r.description,
        category: r.category,
        status: r.status,
        publishedVersion: r.publishedVersion,
        adminNote: r.adminNote,
        createdAt: r.createdAt,
        updatedAt: r.updatedAt,
      }))} />
```

- [ ] **Step 6: rodar testes, lint e build**

Run: `npm test`, `npm run lint`, `npm run build` em `/site`.

Expected: todos com código 0.

- [ ] **Step 7: commit**

```bash
git add components/FeatureRequestsPanel.tsx components/admin/AdminRequestsPanel.tsx "app/(site)/conta/page.tsx" "app/(site)/admin/page.tsx" vitest.config.ts tests/site-panels.test.tsx
git commit -m "feat(ui): painéis de solicitações para usuário e administrador"
```

---

### Task 10: Patch determinístico e cliente HTTPS no desktop

**Files:**
- Create: `/os/scripts/patch-base.mjs`
- Create: `/os/src/account/client.mjs`
- Create: `/os/src/account/storage.mjs`
- Create: `/os/tests/account-modules.test.mjs`
- Test: `/os/tests/account-modules.test.mjs`

- [ ] **Step 1: escrever testes das funções puras**

Create `/os/tests/account-modules.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { normalizeSiteUrl, sanitizeError, validateRequestInput } from "../src/account/client.mjs";

test("aceita apenas HTTPS ou loopback HTTP", () => {
  assert.equal(normalizeSiteUrl("https://yami-site-pi.vercel.app"), "https://yami-site-pi.vercel.app");
  assert.equal(normalizeSiteUrl("http://127.0.0.1:3000"), "http://127.0.0.1:3000");
  assert.equal(normalizeSiteUrl("http://example.com"), null);
});

test("sanitiza mensagens de erro", () => {
  assert.equal(sanitizeError({ error: "rate_limited" }, 429), "Muitas tentativas. Aguarde alguns minutos.");
  assert.equal(sanitizeError({ error: "device_revoked" }, 401), "Sessão encerrada. Entre novamente.");
  assert.equal(sanitizeError({ error: "token secreto" }, 500), "Não foi possível concluir. Tente novamente.");
});

test("valida solicitações antes do envio", () => {
  assert.equal(validateRequestInput({ title: "Tema", description: "Tema azul para o sistema", category: "tema" }).ok, true);
  assert.equal(validateRequestInput({ title: "", description: "x", category: "tema" }).ok, false);
});
```

- [ ] **Step 2: rodar e verificar falha**

Run: `npm test` em `/os`.

Expected: falha de módulo inexistente.

- [ ] **Step 3: implementar cliente**

Create `/os/src/account/client.mjs`:

```js
export const DEFAULT_SITE_URL = "https://yami-site-pi.vercel.app";

export function normalizeSiteUrl(raw) {
  const value = String(raw || DEFAULT_SITE_URL).trim().replace(/\/$/, "");
  try {
    const url = new URL(value);
    if (url.protocol === "https:") return url.origin;
    if (url.protocol === "http:" && url.hostname === "127.0.0.1") return url.origin;
    return null;
  } catch {
    return null;
  }
}

export function sanitizeError(payload, status) {
  const code = payload && typeof payload.error === "string" ? payload.error : "";
  const known = {
    device_revoked: "Sessão encerrada. Entre novamente.",
    license_required: "É necessária uma licença YAMI ativa.",
    rate_limited: "Muitas tentativas. Aguarde alguns minutos.",
    invalid_credentials: "E-mail ou senha incorretos.",
    service_unavailable: "Serviço indisponível no momento.",
  };
  if (known[code]) return known[code];
  if (status === 401) return "Sessão encerrada. Entre novamente.";
  if (status === 403) return "Acesso não autorizado.";
  if (status === 429) return "Muitas tentativas. Aguarde alguns minutos.";
  return "Não foi possível concluir. Tente novamente.";
}

export function validateRequestInput(input) {
  const title = String(input?.title ?? "").trim();
  const description = String(input?.description ?? "").trim();
  const category = String(input?.category ?? "").trim();
  if (title.length < 3 || title.length > 120) return { ok: false, reason: "titulo_invalido" };
  if (description.length < 10 || description.length > 4000) return { ok: false, reason: "descricao_invalida" };
  if (!/^[a-z0-9_-]{2,32}$/.test(category)) return { ok: false, reason: "categoria_invalida" };
  return { ok: true };
}

export class YamiAccountClient {
  constructor({ siteUrl = DEFAULT_SITE_URL, fetchFn = fetch, timeoutMs = 15000 } = {}) {
    this.baseUrl = normalizeSiteUrl(siteUrl);
    if (!this.baseUrl) throw new Error("site_url_invalida");
    this.fetchFn = fetchFn;
    this.timeoutMs = timeoutMs;
    this.deviceToken = null;
  }

  async call(path, { method = "GET", body, auth = true } = {}) {
    const headers = {};
    if (body !== undefined) headers["content-type"] = "application/json";
    if (auth && this.deviceToken) headers.authorization = `Bearer ${this.deviceToken}`;
    const response = await this.fetchFn(`${this.baseUrl}${path}`, {
      method,
      headers,
      body: body === undefined ? undefined : JSON.stringify(body),
      signal: AbortSignal.timeout(this.timeoutMs),
    });
    let payload = null;
    try { payload = await response.json(); } catch { payload = null; }
    if (!response.ok) {
      const error = new Error(sanitizeError(payload, response.status));
      error.code = payload?.error ?? `http_${response.status}`;
      error.status = response.status;
      throw error;
    }
    return payload;
  }

  setToken(token) { this.deviceToken = String(token || "") || null; }

  login(email, password, deviceName) {
    return this.call("/api/desktop/login", {
      method: "POST",
      auth: false,
      body: { email, password, deviceName },
    });
  }

  session() { return this.call("/api/desktop/session"); }
  requests() { return this.call("/api/desktop/requests"); }
  createRequest(input) {
    const check = validateRequestInput(input);
    if (!check.ok) throw new Error(check.reason);
    return this.call("/api/desktop/requests", { method: "POST", body: input });
  }
  withdrawRequest(id) { return this.call(`/api/desktop/requests/${encodeURIComponent(id)}`, { method: "DELETE" }); }
  notifications() { return this.call("/api/desktop/notifications"); }
  readNotification(id) {
    return this.call(`/api/desktop/notifications/${encodeURIComponent(id)}`, { method: "PATCH" });
  }
  logout() { return this.call("/api/desktop/logout", { method: "POST" }); }
}
```

- [ ] **Step 4: implementar armazenamento cifrado**

Create `/os/src/account/storage.mjs`:

```js
import { readFile, writeFile, unlink } from "node:fs/promises";
import { safeStorage } from "electron";

const FILE = "yami-account.bin";

export function tokenFilePath(userDataPath) {
  return `${userDataPath.replace(/\\/g, "/").replace(/\/$/, "")}/${FILE}`;
}

export function canPersist() {
  return Boolean(safeStorage && typeof safeStorage.encryptString === "function" && safeStorage.isEncryptionAvailable());
}

export async function saveToken(userDataPath, token) {
  if (!canPersist()) return false;
  const encrypted = safeStorage.encryptString(String(token));
  await writeFile(tokenFilePath(userDataPath), encrypted, { mode: 0o600 });
  return true;
}

export async function loadToken(userDataPath) {
  if (!canPersist()) return null;
  try {
    const data = await readFile(tokenFilePath(userDataPath));
    return safeStorage.decryptString(data);
  } catch {
    return null;
  }
}

export async function clearToken(userDataPath) {
  await unlink(tokenFilePath(userDataPath)).catch(() => undefined);
}
```

- [ ] **Step 5: implementar script de patch**

Create `/os/scripts/patch-base.mjs`:

```js
import { readFileSync, writeFileSync, existsSync, mkdirSync } from "node:fs";
import { dirname, join } from "node:path";

const root = process.cwd();
const base = join(root, "src");

const PATCHES = [
  {
    file: "dist/preload.cjs",
    marker: "api.yamiAccount",
    before: 'import_electron.contextBridge.exposeInMainWorld("yami", api);',
    after: `api.yamiAccount = {
  login: (email, password, deviceName) => import_electron.ipcRenderer.invoke("yami:account-login", email, password, deviceName),
  logout: () => import_electron.ipcRenderer.invoke("yami:account-logout"),
  session: () => import_electron.ipcRenderer.invoke("yami:account-session"),
  requests: () => import_electron.ipcRenderer.invoke("yami:account-requests-list"),
  createRequest: (input) => import_electron.ipcRenderer.invoke("yami:account-request-create", input),
  withdrawRequest: (id) => import_electron.ipcRenderer.invoke("yami:account-request-withdraw", id),
  notifications: () => import_electron.ipcRenderer.invoke("yami:account-notifications"),
  readNotification: (id) => import_electron.ipcRenderer.invoke("yami:account-notification-read", id),
};
import_electron.contextBridge.exposeInMainWorld("yami", api);`,
  },
  {
    file: "dist/main.js",
    marker: "// [YAMI 0.8.0] conta YAMI",
    before: "// ===== [YAMI OS 20/11/2026] Wi-Fi facil (nmcli) + interface principal que nao fecha =====",
    after: `// [YAMI 0.8.0] conta YAMI
import { createRequire as __yCreateRequire } from "node:module";
const __yRequire = __yCreateRequire(import.meta.url);
const { YamiAccountIpc } = __yRequire("../account/ipc.cjs");
const __yAccountIpc = new YamiAccountIpc({ app, ipcMain });
__yAccountIpc.register();

// ===== [YAMI OS 20/11/2026] Wi-Fi facil (nmcli) + interface principal que nao fecha =====`,
  },
];

let changed = 0;
for (const patch of PATCHES) {
  const path = join(base, patch.file);
  if (!existsSync(path)) throw new Error(`arquivo base ausente: ${patch.file}`);
  const source = readFileSync(path, "utf8");
  if (source.includes(patch.marker)) continue;
  if (!source.includes(patch.before)) throw new Error(`trecho base não encontrado em ${patch.file}`);
  mkdirSync(dirname(path), { recursive: true });
  const result = patch.all
    ? source.split(patch.before).join(patch.after)
    : source.replace(patch.before, patch.after);
  writeFileSync(path, result);
  changed += 1;
}
console.log(`PATCH_OK changed=${changed}`);
```

- [ ] **Step 6: criar stub temporário de IPC para o teste do patch**

Create `/os/src/account/ipc.cjs` com o conteúdo mínimo da Task 11. Para este teste, use:

```js
class YamiAccountIpc {
  constructor({ app, ipcMain }) { this.app = app; this.ipcMain = ipcMain; }
  register() {}
}
module.exports = { YamiAccountIpc };
```

- [ ] **Step 7: executar testes e patch duas vezes**

Run:

```powershell
npm test
npm run patch
npm run patch
```

Expected: testes passam; primeira saída `PATCH_OK changed=2`; segunda `PATCH_OK changed=0`.

- [ ] **Step 8: registrar no repositório de releases**

O diretório `/os` é ignorado localmente. Nenhum commit é necessário nesta task; o resultado será incorporado ao pacote 0.8.0 na Task 13.

---

### Task 11: Handlers IPC no processo principal

**Files:**
- Create/Replace: `/os/src/account/ipc.cjs`
- Test: `/os/tests/ipc.mjs`

- [ ] **Step 1: escrever teste com mocks**

Create `/os/tests/ipc.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { YamiAccountIpc } from "../src/account/ipc.cjs";

function harness(client) {
  const handlers = new Map();
  const ipcMain = { handle: (name, fn) => handlers.set(name, fn) };
  const app = { getPath: () => "/tmp/yami-user-data" };
  const ipc = new YamiAccountIpc({ app, ipcMain, client, storage: {
    load: async () => "token-antigo",
    save: async () => true,
    clear: async () => true,
  } });
  ipc.register();
  return handlers;
}

test("expõe os handlers esperados", () => {
  const handlers = harness({});
  for (const name of [
    "yami:account-login", "yami:account-logout", "yami:account-session",
    "yami:account-requests-list", "yami:account-request-create",
    "yami:account-request-withdraw", "yami:account-notifications",
    "yami:account-notification-read",
  ]) assert.ok(handlers.has(name), name);
});

test("login define token e logout limpa", async () => {
  let token = null;
  const client = {
    setToken(value) { token = value; },
    login: async () => ({ deviceToken: "novo-token", device: { id: "d1" } }),
    logout: async () => ({ ok: true }),
  };
  const handlers = harness(client);
  await handlers.get("yami:account-login")({}, "u@example.com", "senha", "YAMI OS");
  assert.equal(token, "novo-token");
  await handlers.get("yami:account-logout")({});
  assert.equal(token, null);
});
```

- [ ] **Step 2: executar e observar falha**

Run: `npm test` em `/os`.

Expected: primeiro teste passa com o stub; segundo falha.

- [ ] **Step 3: substituir o módulo IPC**

Create `/os/src/account/ipc.cjs` com:

```js
const { DEFAULT_SITE_URL } = require("./client-compat.cjs");

class YamiAccountIpc {
  constructor({ app, ipcMain, client, storage }) {
    this.app = app;
    this.ipcMain = ipcMain;
    this.client = client;
    this.storage = storage;
    this.siteUrl = DEFAULT_SITE_URL;
  }

  register() {
    const handle = (name, fn) => this.ipcMain.handle(name, async (...args) => {
      try {
        return await fn(...args);
      } catch (error) {
        return {
          success: false,
          error: error && error.message ? error.message : "Não foi possível concluir.",
          code: error && error.code ? error.code : "unknown",
        };
      }
    });

    handle("yami:account-login", async (_event, email, password, deviceName) => {
      const result = await this.client.login(email, password, deviceName || "YAMI OS");
      this.client.setToken(result.deviceToken);
      await this.storage.save(this.app.getPath("userData"), result.deviceToken);
      return { success: true, device: result.device };
    });

    handle("yami:account-logout", async () => {
      try { await this.client.logout(); } catch {}
      this.client.setToken(null);
      await this.storage.clear(this.app.getPath("userData"));
      return { success: true };
    });

    handle("yami:account-session", async () => ({ success: true, data: await this.client.session() }));
    handle("yami:account-requests-list", async () => ({ success: true, data: await this.client.requests() }));
    handle("yami:account-request-create", async (_event, input) => ({
      success: true,
      data: await this.client.createRequest(input),
    }));
    handle("yami:account-request-withdraw", async (_event, id) => ({
      success: true,
      data: await this.client.withdrawRequest(id),
    }));
    handle("yami:account-notifications", async () => ({ success: true, data: await this.client.notifications() }));
    handle("yami:account-notification-read", async (_event, id) => ({
      success: true,
      data: await this.client.readNotification(id),
    }));
  }
}

module.exports = { YamiAccountIpc };
```

Create `/os/src/account/client-compat.cjs`:

```js
const DEFAULT_SITE_URL = "https://yami-site-pi.vercel.app";
module.exports = { DEFAULT_SITE_URL };
```

- [ ] **Step 4: criar a inicialização real no main.js sem depender de CommonJS**

Em `/os/scripts/patch-base.mjs`, substituir o trecho de `main.js` por:

```js
    after: `// [YAMI 0.8.0] conta YAMI
import { createRequire as __yAccountCreateRequire } from "node:module";
import { YamiAccountClient, DEFAULT_SITE_URL } from "../account/client.mjs";
import { saveToken, loadToken, clearToken } from "../account/storage.mjs";
const __yAccountRequire = __yAccountCreateRequire(import.meta.url);
const { YamiAccountIpc } = __yAccountRequire("../account/ipc.cjs");
const __yAccountClient = new YamiAccountClient({ siteUrl: process.env.YAMI_SITE_URL || DEFAULT_SITE_URL });
let __yAccountStarted = false;
__yApp.whenReady().then(async () => {
  if (__yAccountStarted) return;
  __yAccountStarted = true;
  const stored = await loadToken(app.getPath("userData"));
  if (stored) __yAccountClient.setToken(stored);
  const storage = { save: saveToken, load: loadToken, clear: clearToken };
  const ipc = new YamiAccountIpc({ app, ipcMain, client: __yAccountClient, storage });
  ipc.register();
});

// ===== [YAMI OS 20/11/2026] Wi-Fi facil (nmcli) + interface principal que nao fecha =====`,
```

- [ ] **Step 5: criar módulo de registro compatível com o patch**

Substituir o construtor em `/os/src/account/ipc.cjs` para aceitar `register()` chamada pelo patch sem duplicar handlers:

```js
  constructor({ app, ipcMain, client, storage }) {
    this.app = app;
    this.ipcMain = ipcMain;
    this.client = client;
    this.storage = storage;
    this.registered = false;
  }

  register() {
    if (this.registered) return;
    this.registered = true;
```

- [ ] **Step 6: rodar testes**

Run: `npm test` em `/os`.

Expected: todos passam.

- [ ] **Step 7: reaplicar o patch a partir da base limpa**

Run:

```powershell
Copy-Item -Force 'D:\Downloads\yami relise os\pacote-0.7.0\app-extracted\dist\main.js' 'D:\Downloads\yami relise os\desenvolvimento\yami-os\src\dist\main.js'
Copy-Item -Force 'D:\Downloads\yami relise os\pacote-0.7.0\app-extracted\dist\preload.cjs' 'D:\Downloads\yami relise os\desenvolvimento\yami-os\src\dist\preload.cjs'
npm run patch
npm test
```

Expected: `PATCH_OK changed=2`; testes passam.

---

### Task 12: View integrada da conta YAMI

**Files:**
- Create: `/os/src/interface/yami-account.js`
- Modify: `/os/src/interface/index.html`
- Modify: `/os/src/interface/app.js`
- Modify: `/os/scripts/patch-base.mjs`
- Test: `/os/tests/ui-patch.test.mjs`

- [ ] **Step 1: escrever teste do patch de interface**

Create `/os/tests/ui-patch.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { readFileSync } from "node:fs";

test("interface contém a view e o script da conta", () => {
  const html = readFileSync("src/interface/index.html", "utf8");
  assert.ok(html.includes('id="view-yami"'));
  assert.ok(html.includes('src="yami-account.js"'));
});

test("barra e menu contêm o botão YAMI", () => {
  const html = readFileSync("src/interface/index.html", "utf8");
  assert.equal((html.match(/data-view="yami"/g) || []).length, 2);
});
```

- [ ] **Step 2: confirmar falha**

Run: `npm test` em `/os`.

Expected: falha nos testes de interface.

- [ ] **Step 3: criar o Controlador da view**

Create `/os/src/interface/yami-account.js`:

```js
(() => {
  const bridge = window.yami;
  if (!bridge || !bridge.yamiAccount) return;

  const view = document.getElementById("view-yami");
  if (!view) return;

  const el = {
    login: document.getElementById("yamiLoginForm"),
    email: document.getElementById("yamiEmail"),
    password: document.getElementById("yamiPassword"),
    deviceName: document.getElementById("yamiDeviceName"),
    logged: document.getElementById("yamiLogged"),
    user: document.getElementById("yamiUser"),
    license: document.getElementById("yamiLicense"),
    orders: document.getElementById("yamiOrders"),
    requestForm: document.getElementById("yamiRequestForm"),
    requestTitle: document.getElementById("yamiRequestTitle"),
    requestCategory: document.getElementById("yamiRequestCategory"),
    requestDescription: document.getElementById("yamiRequestDescription"),
    requestList: document.getElementById("yamiRequestList"),
    notificationList: document.getElementById("yamiNotificationList"),
    unread: document.getElementById("yamiUnread"),
    message: document.getElementById("yamiMessage"),
  };

  const tabs = [...view.querySelectorAll("[data-yami-tab]")];
  const panels = [...view.querySelectorAll("[data-yami-panel]")];

  function say(text, kind = "") {
    if (!el.message) return;
    el.message.textContent = text || "";
    el.message.dataset.kind = kind;
  }

  function showTab(name) {
    tabs.forEach((b) => b.classList.toggle("active", b.dataset.yamiTab === name));
    panels.forEach((p) => p.classList.toggle("active", p.dataset.yamiPanel === name));
  }

  tabs.forEach((button) => button.addEventListener("click", () => showTab(button.dataset.yamiTab)));

  function statusLabel(status) {
    const map = {
      pendente: "Pendente",
      aprovada: "Aprovada",
      rejeitada: "Rejeitada",
      em_desenvolvimento: "Em desenvolvimento",
      publicada: "Publicada",
      retirada: "Retirada",
    };
    return map[status] || status;
  }

  function renderSession(session) {
    if (!session) return;
    el.logged.hidden = false;
    el.login.hidden = true;
    el.user.textContent = `${session.user.email}${session.device ? ` · ${session.device.name}` : ""}`;
    el.license.textContent = session.license
      ? `Ativa · ${session.license.key} · desde ${new Date(session.license.createdAt).toLocaleDateString("pt-BR")}`
      : "Sem licença ativa";
    el.orders.innerHTML = session.orders.length
      ? session.orders.map((o) => `<li>${o.id.slice(0, 8)} · ${o.status} · ${new Date(o.createdAt).toLocaleDateString("pt-BR")}</li>`).join("")
      : "<li>Nenhum pedido encontrado.</li>";
    el.unread.textContent = String(session.unreadNotifications || 0);
  }

  async function refresh() {
    const result = await bridge.yamiAccount.session();
    if (!result.success) {
      el.logged.hidden = true;
      el.login.hidden = false;
      return;
    }
    renderSession(result.data);
    const [requests, notifications] = await Promise.all([
      bridge.yamiAccount.requests(),
      bridge.yamiAccount.notifications(),
    ]);
    if (requests.success) {
      el.requestList.innerHTML = requests.data.items.length
        ? requests.data.items.map((r) => `<li><strong>${escapeHtml(r.title)}</strong><br>${statusLabel(r.status)}${r.publishedVersion ? ` · ${r.publishedVersion}` : ""}</li>`).join("")
        : "<li>Nenhuma solicitação enviada.</li>";
    }
    if (notifications.success) {
      el.notificationList.innerHTML = notifications.data.items.length
        ? notifications.data.items.map((n) => `<li><strong>${escapeHtml(n.title)}</strong><br>${escapeHtml(n.body)}</li>`).join("")
        : "<li>Nenhum aviso.</li>";
    }
  }

  function escapeHtml(value) {
    return String(value).replace(/[&<>"']/g, (ch) => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[ch]));
  }

  el.login?.addEventListener("submit", async (event) => {
    event.preventDefault();
    say("Entrando…");
    const result = await bridge.yamiAccount.login(
      el.email.value,
      el.password.value,
      el.deviceName.value || "YAMI OS",
    );
    if (!result.success) {
      say(result.error, "error");
      return;
    }
    el.password.value = "";
    say("Conectado.", "ok");
    await refresh();
  });

  el.requestForm?.addEventListener("submit", async (event) => {
    event.preventDefault();
    const result = await bridge.yamiAccount.createRequest({
      title: el.requestTitle.value,
      category: el.requestCategory.value,
      description: el.requestDescription.value,
    });
    if (!result.success) {
      say(result.error, "error");
      return;
    }
    el.requestTitle.value = "";
    el.requestDescription.value = "";
    say("Solicitação enviada.", "ok");
    await refresh();
  });

  document.getElementById("yamiLogout")?.addEventListener("click", async () => {
    const result = await bridge.yamiAccount.logout();
    if (result.success) {
      el.logged.hidden = true;
      el.login.hidden = false;
      say("Sessão encerrada.", "ok");
    }
  });

  document.getElementById("yamiOpenDashboard")?.addEventListener("click", () => {
    window.open("https://yami-site-pi.vercel.app/conta", "_blank", "noopener");
  });

  window.yamiRefreshAccount = refresh;
  void refresh();
  setInterval(() => {
    if (!document.hidden && view.classList.contains("active")) void refresh();
  }, 5 * 60 * 1000);
})();
```

- [ ] **Step 4: adicionar HTML da view**

Em `/os/scripts/patch-base.mjs`, acrescentar entrada:

```js
  {
    file: "src/interface/index.html",
    marker: "id=\"view-yami\"",
    before: "      <!-- View 5: Configurações -->",
    after: `      <!-- View: Conta YAMI -->
      <section class="view-container wide" id="view-yami">
        <div class="view-header"><div class="view-title">YAMI</div><div class="view-subtitle">Conta, licença e solicitações</div></div>
        <div class="account-tabs">
          <button class="files-btn active" data-yami-tab="conta">Conta</button>
          <button class="files-btn" data-yami-tab="solicitacoes">Solicitações</button>
          <button class="files-btn" data-yami-tab="avisos">Avisos <span id="yamiUnread">0</span></button>
        </div>
        <p id="yamiMessage" class="account-message" role="status"></p>
        <form id="yamiLoginForm" class="settings-section">
          <div class="settings-section-title">Entrar na conta YAMI</div>
          <div class="user-create-grid">
            <label>E-mail<input class="user-input" id="yamiEmail" type="email" autocomplete="email" required></label>
            <label>Senha<input class="user-input" id="yamiPassword" type="password" autocomplete="current-password" required></label>
            <label>Nome do dispositivo<input class="user-input" id="yamiDeviceName" value="YAMI OS" maxlength="80"></label>
          </div>
          <div class="user-actions"><button class="files-btn" type="submit">Entrar</button></div>
        </form>
        <div id="yamiLogged" class="account-panel" hidden>
          <div data-yami-panel="conta" class="account-card active">
            <div class="settings-section">
              <div class="settings-section-title">Conta</div>
              <p id="yamiUser" class="settings-desc"></p>
              <p id="yamiLicense" class="account-license"></p>
              <div class="user-actions">
                <button class="files-btn" id="yamiOpenDashboard">Abrir dashboard</button>
                <button class="files-btn" id="yamiLogout">Sair</button>
              </div>
            </div>
            <div class="settings-section"><div class="settings-section-title">Pedidos</div><ul id="yamiOrders" class="account-list"></ul></div>
          </div>
          <div data-yami-panel="solicitacoes" class="account-card">
            <form id="yamiRequestForm" class="settings-section">
              <div class="settings-section-title">Nova solicitação</div>
              <div class="user-create-grid">
                <label>Título<input class="user-input" id="yamiRequestTitle" maxlength="120" required></label>
                <label>Categoria<input class="user-input" id="yamiRequestCategory" value="geral" maxlength="32" required></label>
              </div>
              <textarea class="user-input" id="yamiRequestDescription" maxlength="4000" required></textarea>
              <div class="user-actions"><button class="files-btn" type="submit">Enviar</button></div>
            </form>
            <div class="settings-section"><div class="settings-section-title">Minhas solicitações</div><ul id="yamiRequestList" class="account-list"></ul></div>
          </div>
          <div data-yami-panel="avisos" class="account-card">
            <div class="settings-section"><div class="settings-section-title">Avisos</div><ul id="yamiNotificationList" class="account-list"></ul></div>
          </div>
        </div>
      </section>

      <!-- View 5: Configurações -->`,
  },
  {
    file: "src/interface/index.html",
    marker: 'src="yami-account.js"',
    before: '  <script src="app.js"></script>',
    after: '  <script src="yami-account.js"></script>\n  <script src="app.js"></script>',
  },
  {
    file: "src/interface/index.html",
    marker: 'data-view="yami"',
    all: true,
    before: '      <button class="pin nav-btn" data-view="settings" title="Ajustes">',
    after: '      <button class="pin nav-btn" data-view="yami" title="YAMI"><svg class="px" viewBox="0 0 24 24" aria-hidden="true"><path d="M4 2h16v4h2v12h-4v4H6v-4H2V6h2V2zm2 4v12h12V6H6zm2 4h3v6H8v-6zm5 0h3v6h-3v-6z" fill="currentColor"/></svg></button>\n      <button class="pin nav-btn" data-view="settings" title="Ajustes">',
  },
];
```

- [ ] **Step 5: adicionar CSS e script no patch**

Acrescentar entrada para estilos:

```js
  {
    file: "src/interface/index.html",
    marker: "id=\"yamiAccountStyle\"",
    before: "</head>",
    after: `<style id="yamiAccountStyle">
      .account-tabs { display:flex; gap:8px; margin:12px 0; }
      .account-tabs .files-btn.active { border-color:var(--primary); color:var(--text); }
      .account-panel { display:grid; gap:14px; }
      .account-card { display:none; }
      .account-card.active { display:block; }
      .account-license { margin-top:10px; font-size:12px; color:var(--success); }
      .account-list { list-style:none; display:grid; gap:8px; font-size:12px; }
      .account-list li { border-bottom:1px solid rgba(255,255,255,.05); padding:8px 0; }
      .account-message[data-kind="error"] { color:var(--danger); }
      .account-message[data-kind="ok"] { color:var(--success); }
    </style>
</head>`,
  },
```

- [ ] **Step 6: reaplicar patch e rodar testes**

Run:

```powershell
Copy-Item -Force 'D:\Downloads\yami relise os\pacote-0.7.0\app-extracted\src\interface\index.html' 'D:\Downloads\yami relise os\desenvolvimento\yami-os\src\src\interface\index.html'
npm run patch
npm test
```

Expected: `PATCH_OK changed=3`; todos os testes passam.

- [ ] **Step 7: verificar sintaxe do JavaScript gerado**

Run: `node --check src/interface/yami-account.js && node --check dist/main.js && node --check dist/preload.cjs` em `/os`.

Expected: código 0.

---

### Task 13: Gerar pacote, manifesto e checksum 0.8.0

**Files:**
- Create: `/os/build/build-package.mjs`
- Create: `/releases/yami-pacote.json` temporário em staging
- Modify: `/releases/latest.json`
- Test: `/os/tests/package.test.mjs`

- [ ] **Step 1: escrever teste das regras do pacote**

Create `/os/tests/package.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { allowedPath, manifestFor } from "../build/build-package.mjs";

test("somente diretórios do sistema entram no pacote", () => {
  assert.equal(allowedPath("root/usr/bin/yami"), true);
  assert.equal(allowedPath("root/opt/YamiOS/resources/app.asar"), true);
  assert.equal(allowedPath("root/home/user/.config"), false);
  assert.equal(allowedPath("root/etc/yami/site.json"), true);
});

test("manifesto usa a versão e a base corretas", () => {
  const manifest = manifestFor({ version: "0.8.0", previous: "0.7.0" });
  assert.equal(manifest.versao, "0.8.0");
  assert.equal(manifest.deVersaoMinima, "0.7.0");
});
```

- [ ] **Step 2: confirmar falha**

Run: `npm test` em `/os`.

Expected: falha de módulo inexistente.

- [ ] **Step 3: criar script de build**

Create `/os/build/build-package.mjs`:

```js
import { createHash } from "node:crypto";
import { createReadStream, existsSync } from "node:fs";
import { mkdir, readFile, rm, writeFile } from "node:fs/promises";
import { spawnSync } from "node:child_process";
import { join } from "node:path";

const ROOT = process.cwd();
const STAGE = join(ROOT, "build", "stage");
const RELEASES = "D:/Downloads/yami relise os";

export function allowedPath(relative) {
  const path = relative.replace(/\\/g, "/");
  return /^root\/(usr|opt|etc|var\/lib\/yami)\//.test(path);
}

export function manifestFor({ version, previous }) {
  return {
    produto: "YAMI OS",
    versao: version,
    deVersaoMinima: previous,
    descricao: "Integração com o site YAMI: conta, licença, solicitações e notificações no sistema.",
  };
}

export async function sha256File(path) {
  const hash = createHash("sha256");
  await new Promise((resolve, reject) => {
    const stream = createReadStream(path);
    stream.on("data", (chunk) => hash.update(chunk));
    stream.on("error", reject);
    stream.on("end", resolve);
  });
  return hash.digest("hex");
}

async function main() {
  if (process.argv[1] && import.meta.url.endsWith("build-package.mjs") && process.env.NODE_TEST !== "1") {
    await rm(STAGE, { recursive: true, force: true });
    await mkdir(join(STAGE, "root/opt/YamiOS/resources"), { recursive: true });
    await mkdir(join(STAGE, "root/usr/local/lib/yami"), { recursive: true });

    const asar = spawnSync("npx", ["@electron/asar", "pack", "src", join(STAGE, "root/opt/YamiOS/resources/app.asar")], {
      cwd: ROOT,
      stdio: "inherit",
      shell: process.platform === "win32",
    });
    if (asar.status !== 0) throw new Error("falha ao gerar app.asar");

    const pos = `#!/bin/bash
# YAMI OS 0.8.0 — pós-instalação. Não toca em dados do usuário.
gtk-update-icon-cache -q -f /usr/share/icons/YAMI-Pixel 2>/dev/null || true
( sleep 4; pkill -f '^/opt/YamiOS/yami-os( |$)' 2>/dev/null || true ) >/dev/null 2>&1 &
echo "pacote 0.8.0 aplicado"
`;
    await writeFile(join(STAGE, "posinstalar.sh"), pos, { mode: 0o755 });
    await writeFile(join(STAGE, "yami-pacote.json"), JSON.stringify(manifestFor({ version: "0.8.0", previous: "0.7.0" }), null, 2));

    const output = join(RELEASES, "yami-os-0.8.0-pacote.tar.zst");
    const tar = spawnSync("tar", ["--zstd", "-cf", output, "-C", STAGE, "yami-pacote.json", "posinstalar.sh", "root"], {
      stdio: "inherit",
      shell: process.platform === "win32",
    });
    if (tar.status !== 0) throw new Error("falha ao gerar pacote");

    const sha = await sha256File(output);
    const { size } = await import("node:fs").then((fs) => fs.statSync(output));
    const info = { file: output, sha256: sha, tamanho: size };
    await writeFile(join(ROOT, "build", "package-info.json"), JSON.stringify(info, null, 2));
    console.log(JSON.stringify(info, null, 2));
  }
}

if (process.env.NODE_TEST === "1") main().catch((err) => { console.error(err.message); process.exit(1); });
export default main;
```

Ajuste o teste para definir `NODE_TEST=1` apenas ao executar o script, não ao importar; a garda acima já impede a execução automática durante `node --test`.

- [ ] **Step 4: rodar os testes**

Run: `npm test` em `/os`.

Expected: todos passam.

- [ ] **Step 5: gerar o pacote**

Run: `NODE_TEST=1 npm run build` em `/os` (em PowerShell: `$env:NODE_TEST='1'; npm run build`).

Expected: arquivo `yami-os-0.8.0-pacote.tar.zst` criado em `/releases`, com `sha256` e `tamanho` impressos.

- [ ] **Step 6: validar conteúdo**

Run: `tar -tf "D:\Downloads\yami relise os\yami-os-0.8.0-pacote.tar.zst"`.

Expected: apenas `yami-pacote.json`, `posinstalar.sh` e `root/usr/...` ou `root/opt/...`.

- [ ] **Step 7: atualizar o manifesto em `/releases/latest.json` a partir do resultado real do build**

Create `/os/build/update-manifest.mjs`:

```js
import { readFileSync, writeFileSync } from "node:fs";

const releases = "D:/Downloads/yami relise os";
const info = JSON.parse(readFileSync("build/package-info.json", "utf8"));
const manifest = JSON.parse(readFileSync(`${releases}/latest.json`, "utf8"));

manifest.versao = "0.8.0";
manifest.build = "r8";
manifest.data = "2026-11-21";
manifest.notas = [
  "Integração com o site YAMI: conta, licença e pedidos dentro do sistema",
  "Solicitações de melhorias com aprovação administrativa e acompanhamento",
  "Avisos de aprovação, rejeição e publicação no aplicativo YAMI",
  "Checkout com validação reforçada e proteção contra pagamentos duplicados",
  "Atualização rápida por pacote: seus dados, Wi-Fi e aplicativos são mantidos",
];
manifest.pacotes = [{
  versao: "0.8.0",
  deVersaoMinima: "0.7.0",
  url: "https://github.com/vinicon14/yamios-releases/releases/download/v0.8.0/yami-os-0.8.0-pacote.tar.zst",
  sha256: info.sha256,
  tamanho: info.tamanho,
  descricao: "Aplicativo YAMI integrado, solicitações e correções de pagamento",
}];
manifest.iso = {
  url: null,
  sha256: "ab8ee0bcd7ec59692f36beeeee34b9ee33380938603b55426af1b3946473e5b9",
  tamanho: 3523215360,
  kernel: "6.8.0-149-generic",
  partes: [],
  observacao: "ISO 0.8.0 ainda não publicada. O pacote incremental atualiza a 0.7.0 sem perder dados; a ISO 0.7.0 permanece como referência da última imagem completa.",
};
manifest.fontes = {
  vercel: "https://yami-site-pi.vercel.app/api/yamios/latest",
  github: "https://github.com/vinicon14/yamios-releases/releases/latest/download/latest.json",
};

writeFileSync(`${releases}/latest.json`, JSON.stringify(manifest, null, 2) + "\n");
console.log("MANIFEST_WRITTEN");
```

Run: `node build/update-manifest.mjs` em `/os`.

Expected: `MANIFEST_WRITTEN`; `latest.json` contém `sha256` e `tamanho` reais lidos de `build/package-info.json`.

- [ ] **Step 8: sincronizar a fonte Vercel**

Copiar o conteúdo final de `/releases/latest.json` para `/site/lib/yamios/latest.json`.

- [ ] **Step 9: validar JSON e checksum**

Run:

```powershell
$manifest = Get-Content 'D:\Downloads\yami relise os\latest.json' -Raw | ConvertFrom-Json
$sha = (Get-FileHash 'D:\Downloads\yami relise os\yami-os-0.8.0-pacote.tar.zst' -Algorithm SHA256).Hash.ToLower()
if ($manifest.pacotes[0].sha256 -ne $sha) { throw 'checksum divergente' }
if ($manifest.versao -ne '0.8.0') { throw 'versão divergente' }
Write-Output 'MANIFEST_OK'
```

Expected: `MANIFEST_OK`.

- [ ] **Step 10: commit no releases**

```bash
git add latest.json yami-os-0.8.0-pacote.tar.zst
git commit -m "release: pacote 0.8.0 com integração YAMI"
```

---

### Task 14: Verificação final antes da publicação

**Files:**
- Use: `/site`, `/os`, `/releases`
- Test: `/os/tests/final-check.test.mjs`

- [ ] **Step 1: checar segredos no código e no pacote**

Create `/os/tests/final-check.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { readFileSync } from "node:fs";
import { allowedPath } from "../build/build-package.mjs";

test("arquivos versionados não contêm padrões de segredo", () => {
  const files = [
    "D:/Downloads/open code yami edit/lib/desktop-auth.ts",
    "D:/Downloads/yami relise os/latest.json",
    "src/account/client.mjs",
  ];
  const forbidden = /(sk-[A-Za-z0-9]{20,}|MP_ACCESS_TOKEN\s*=\s*[^s]|BEGIN (RSA |EC )?PRIVATE KEY)/;
  for (const file of files) {
    const content = readFileSync(file, "utf8");
    assert.equal(forbidden.test(content), false, file);
  }
});

test("pacote não contém diretórios de usuário", () => {
  assert.equal(allowedPath("root/home/user/documentos"), false);
  assert.equal(allowedPath("root/var/lib/yami/estado.json"), true);
});
```

- [ ] **Step 2: executar todas as verificações**

Run em `/site`:

```powershell
npm test
npm run lint
npm run build
```

Run em `/os`:

```powershell
npm test
npm run patch
npm test
```

Expected: todos com código 0; segundo `npm run patch` imprime `PATCH_OK changed=0`.

- [ ] **Step 3: validar o pacote em Linux isolado**

Run na distro Ubuntu 24.04 do WSL:

```bash
set -e
TMP=$(mktemp -d /tmp/yami-verify-0.8.0.XXXX)
tar --zstd -xf '/mnt/d/Downloads/yami relise os/yami-os-0.8.0-pacote.tar.zst' -C "$TMP"
test -f "$TMP/yami-pacote.json"
test -f "$TMP/root/opt/YamiOS/resources/app.asar"
test -x "$TMP/posinstalar.sh"
echo PACKAGE_LINUX_OK
```

Expected: `PACKAGE_LINUX_OK`.

- [ ] **Step 4: verificar o contrato de atualização**

Run:

```bash
python3 - <<'PY'
import json
m=json.load(open('/mnt/d/Downloads/yami relise os/latest.json'))
assert m['versao']=='0.8.0'
assert m['pacotes'][0]['deVersaoMinima']=='0.7.0'
assert m['fontes']['vercel'].startswith('https://')
print('UPDATE_CONTRACT_OK')
PY
```

Expected: `UPDATE_CONTRACT_OK`.

- [ ] **Step 5: registrar evidências**

Salvar saídas em `/releases/docs/verificacao-0.8.0.md` com:

```markdown
# Verificação 0.8.0

- Site: `npm test`, `npm run lint`, `npm run build` — executados com código 0.
- Desktop: `npm test`, `npm run patch`, `npm test` — executados com código 0; patch idempotente.
- Pacote: extraído em Ubuntu 24.04; estrutura e permissões conferidas.
- Manifesto: versão 0.8.0, base 0.7.0, sha256 conferido.
- Segredos: nenhum padrão de token ou chave privada encontrado nos arquivos verificados.
```

- [ ] **Step 6: commit das evidências em `/releases`**

```bash
git add docs/verificacao-0.8.0.md
git commit -m "docs: evidências de verificação do pacote 0.8.0"
```

---

## Publicação (somente após autorização explícita)

1. Fazer push do site e do releases para `main`.
2. Publicar o site na Vercel com as variáveis de ambiente já configuradas, sem exibir valores.
3. Criar o tag `v0.8.0` e a release no GitHub com `latest.json` e `yami-os-0.8.0-pacote.tar.zst`.
4. Conferir que `/api/yamios/latest` e o asset `latest.json` da release devolvem a versão `0.8.0`.
5. Só então publicar os marcos 2 e 3 como atualizações subsequentes ou incluí-los na release final, conforme decisão do usuário.

## Auto-revisão do plano

- Cobertura da spec: APIs de desktop (Task 8), token (Task 7), solicitações (Tasks 5, 6, 8, 9), pagamentos (Tasks 2, 4, 5), view integrada (Tasks 10-12), pacote/manifesto (Task 13), testes/critérios de aceite (Task 14), compatibilidade (Tasks 1, 11-13).
- Ausência de placeholders: nenhum `TBD`, `TODO`, `implementar depois` ou trecho sem código foi deixado nas etapas de implementação.
- Consistência de nomes: `deviceToken`, `yamiAccount.*`, `yami:account-*`, `feature_requests`, `payment_events` e `0.8.0/deVersaoMinima 0.7.0` são usados de forma idêntica em todas as tasks.
