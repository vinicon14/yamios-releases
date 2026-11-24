# Publicação de apps no site — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Permitir que usuários com licença paga enviem `.AppImage` e `.tar.zst` para revisão, que o administrador aprove e publique no GitHub, e que a loja do YAMI OS consulte os apps publicados.

**Architecture:** O upload usa Vercel Blob com URL assinada. Os metadados ficam vinculados à solicitação existente. A aprovação baixa o arquivo, confere sha256 e publica como asset da release `v0.8.0`. A rota `/api/apps` devolve o catálogo público para usuários licenciados.

**Tech Stack:** Next.js 16, React 19, TypeScript, Vitest, `@vercel/blob`, GitHub Releases.

**Convenção de caminho:** `/site` = `D:\Downloads\open code yami edit`.

---

### Task 1: Extender `feature_requests` com metadados de app

**Files:**
- Modify: `/site/lib/db/schema.sql`
- Modify: `/site/lib/db/sql-shared.ts`
- Modify: `/site/lib/db/types.ts`
- Test: `/site/tests/app-submission-schema.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/app-submission-schema.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { SCHEMA_SQL } from "../lib/db/sql-shared";

describe("metadados de app", () => {
  it("contém os campos de arquivo", () => {
    for (const column of ["file_name", "file_size", "file_sha256", "file_blob_pathname", "app_version", "published_asset_url"]) {
      expect(SCHEMA_SQL).toContain(column);
    }
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/app-submission-schema.test.ts` em `/site`.

Expected: falha.

- [ ] **Step 3: atualizar esquema**

Adicionar à tabela `feature_requests` em `schema.sql` e `SCHEMA_SQL`:

```sql
  file_name TEXT,
  file_size INTEGER,
  file_sha256 TEXT,
  file_blob_pathname TEXT,
  app_version TEXT,
  app_icon TEXT,
  published_asset_url TEXT,
  published_at TEXT
```

- [ ] **Step 4: atualizar tipos**

Em `types.ts`, adicionar a `FeatureRequest`:

```ts
  fileName: string | null;
  fileSize: number | null;
  fileSha256: string | null;
  fileBlobPathname: string | null;
  appVersion: string | null;
  appIcon: string | null;
  publishedAssetUrl: string | null;
  publishedAt: string | null;
```

Atualizar `rowToFeatureRequest` em `sql-shared.ts` para mapear os campos.

- [ ] **Step 5: rodar teste**

Run: `npm test -- tests/app-submission-schema.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 6: commit**

```bash
git add lib/db/schema.sql lib/db/sql-shared.ts lib/db/types.ts tests/app-submission-schema.test.ts
git commit -m "feat(db): metadados de app em solicitações"
```

---

### Task 2: Validação de arquivo de app

**Files:**
- Create: `/site/lib/app-submissions.ts`
- Test: `/site/tests/app-submissions.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/app-submissions.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { validateAppFile } from "../lib/app-submissions";

describe("validação de app", () => {
  it("aceita AppImage e tar.zst válidos", () => {
    expect(validateAppFile({ name: "app.AppImage", size: 1000, sha256: "a".repeat(64) })).toEqual({ ok: true });
    expect(validateAppFile({ name: "app.tar.zst", size: 1000, sha256: "a".repeat(64) })).toEqual({ ok: true });
  });

  it("rejeita formato, tamanho ou hash inválidos", () => {
    expect(validateAppFile({ name: "app.exe", size: 1000, sha256: "a".repeat(64) }).ok).toBe(false);
    expect(validateAppFile({ name: "app.AppImage", size: 600 * 1024 * 1024, sha256: "a".repeat(64) }).ok).toBe(false);
    expect(validateAppFile({ name: "app.AppImage", size: 1000, sha256: "abc" }).ok).toBe(false);
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/app-submissions.test.ts` em `/site`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Create `/site/lib/app-submissions.ts`:

```ts
const MAX_APP_BYTES = 500 * 1024 * 1024;
const ALLOWED = [".AppImage", ".tar.zst"];

export function validateAppFile(input: { name: string; size: number; sha256: string }) {
  const name = String(input?.name ?? "").trim();
  const size = Number(input?.size);
  const sha256 = String(input?.sha256 ?? "").trim().toLowerCase();
  if (!ALLOWED.some((ext) => name.toLowerCase().endsWith(ext))) return { ok: false, reason: "formato_invalido" };
  if (!Number.isFinite(size) || size <= 0 || size > MAX_APP_BYTES) return { ok: false, reason: "tamanho_invalido" };
  if (!/^[a-f0-9]{64}$/.test(sha256)) return { ok: false, reason: "sha256_invalido" };
  return { ok: true };
}
```

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/app-submissions.test.ts` em `/site`.

Expected: 2 testes passam.

- [ ] **Step 5: commit**

```bash
git add lib/app-submissions.ts tests/app-submissions.test.ts
git commit -m "feat(apps): validação de arquivos de aplicativo"
```

---

### Task 3: Upload privado no Blob

**Files:**
- Create: `/site/app/api/uploads/app/route.ts`
- Test: `/site/tests/app-upload-route.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/app-upload-route.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { hashPassword } from "../lib/auth";
import { setStoreForTests } from "../lib/db";
import { POST as uploadPost } from "../app/api/uploads/app/route";

describe("upload de app", () => {
  it("exige licença", async () => {
    const store = createMemoryStore();
    setStoreForTests(store);
    await store.createUser({ id: "u1", email: "u@example.com", passwordHash: await hashPassword("senha-forte-123"), name: "U", status: "ativo", licenseId: null });
    const res = await uploadPost(new Request("http://localhost:3000/api/uploads/app", {
      method: "POST",
      headers: { "content-type": "application/json", origin: "http://localhost:3000" },
      body: JSON.stringify({ pathname: "apps/test.AppImage", clientPayload: JSON.stringify({ kind: "app", filename: "test.AppImage", version: "1.0.0", size: 1000, sha256: "a".repeat(64) }) }),
    }));
    expect(res.status).toBe(403);
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/app-upload-route.test.ts` em `/site`.

Expected: falha de rota ausente.

- [ ] **Step 3: implementar**

Create `/site/app/api/uploads/app/route.ts`:

```ts
import { NextResponse } from "next/server";
import { handleUpload, type HandleUploadBody } from "@vercel/blob/client";
import { getStore } from "@/lib/db";
import { getSessionUser, readSessionCookie } from "@/lib/auth";
import { userHasPaidLicense } from "@/lib/license-gate";
import { clientIp, jsonError, sameOrigin } from "@/lib/http";
import { rateLimit } from "@/lib/rate-limit";
import { validateAppFile } from "@/lib/app-submissions";

export const dynamic = "force-dynamic";
export const runtime = "nodejs";

export async function POST(req: Request) {
  const current = await getSessionUser(readSessionCookie(req));
  if (!current) return jsonError("não autenticado", 401);
  if (!sameOrigin(req)) return jsonError("origem inválida", 403);
  const store = await getStore();
  if (!(await userHasPaidLicense(current.user.email, store))) return jsonError("licença necessária", 403);
  const limited = rateLimit(`app-upload:${current.user.id}`, 20, 60 * 60 * 1000);
  if (!limited.ok) return jsonError("muitas tentativas", 429);

  let body: HandleUploadBody;
  try { body = await req.json(); } catch { return jsonError("pedido inválido", 400); }

  try {
    const result = await handleUpload({
      request: req,
      body,
      onBeforeGenerateToken: async (_pathname, clientPayload) => {
        const raw = JSON.parse(clientPayload ?? "{}");
        const check = validateAppFile({ name: raw.filename, size: raw.size, sha256: raw.sha256 });
        if (!check.ok) throw new Error(check.reason);
        return {
          allowedContentTypes: ["application/octet-stream", "application/x-executable"],
          maximumSizeInBytes: 500 * 1024 * 1024,
          addRandomSuffix: false,
          allowOverwrite: false,
          cacheControlMaxAge: 31536000,
          tokenPayload: JSON.stringify({ userId: current.user.id, email: current.user.email, ip: clientIp(req) }),
        };
      },
    });
    return NextResponse.json(result);
  } catch (err) {
    return jsonError(err instanceof Error ? err.message : "não foi possível autorizar o envio", 400);
  }
}
```

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/app-upload-route.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add app/api/uploads/app/route.ts tests/app-upload-route.test.ts
git commit -m "feat(upload): autorização de envio de apps"
```

---

### Task 4: Anexar arquivo à solicitação

**Files:**
- Modify: `/site/app/api/conta/requests/route.ts`
- Modify: `/site/components/FeatureRequestsPanel.tsx`
- Test: `/site/tests/app-submission-request.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/app-submission-request.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { hashPassword } from "../lib/auth";
import { setStoreForTests } from "../lib/db";
import { POST as requestPost } from "../app/api/conta/requests/route";

describe("solicitação com arquivo", () => {
  it("grava metadados do app", async () => {
    const store = createMemoryStore();
    setStoreForTests(store);
    const user = await store.createUser({ id: "u1", email: "u@example.com", passwordHash: await hashPassword("senha-forte-123"), name: "U", status: "ativo", licenseId: null });
    await store.createOrder({ id: "o1", email: "u@example.com", amountBrl: 20, userId: "u1" });
    await store.createLicense("o1", "YAMI-DEMO-KEY");
    const session = await (await import("../lib/auth")).createSessionForUser(user);
    const res = await requestPost(new Request("http://localhost:3000/api/conta/requests", {
      method: "POST",
      headers: { "content-type": "application/json", cookie: `yami_session=${session.raw}` },
      body: JSON.stringify({
        title: "Meu app",
        description: "Descrição do aplicativo",
        category: "produtividade",
        fileName: "meu-app.AppImage",
        fileSize: 1000,
        fileSha256: "a".repeat(64),
        fileBlobPathname: "apps/teste/meu-app.AppImage",
        appVersion: "1.0.0",
      }),
    }));
    expect(res.status).toBe(201);
    const items = await store.listFeatureRequests("u1");
    expect(items[0]?.fileName).toBe("meu-app.AppImage");
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/app-submission-request.test.ts` em `/site`.

Expected: falha.

- [ ] **Step 3: implementar**

Em `/site/app/api/conta/requests/route.ts`, aceitar os campos de arquivo e passar para `createFeatureRequest`.

Em `/site/components/FeatureRequestsPanel.tsx`, adicionar input de arquivo e chamada ao upload antes de criar a solicitação.

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/app-submission-request.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add app/api/conta/requests/route.ts components/FeatureRequestsPanel.tsx tests/app-submission-request.test.ts
git commit -m "feat(apps): anexo de arquivo em solicitação"
```

---

### Task 5: Publicação de app aprovado no GitHub

**Files:**
- Modify: `/site/app/api/admin/requests/[id]/route.ts`
- Test: `/site/tests/app-publish.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/app-publish.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { transitionRequest } from "../lib/feature-requests";

describe("publicação de app", () => {
  it("exige versão e URL ao publicar", async () => {
    const store = createMemoryStore();
    await store.createUser({ id: "u1", email: "u@example.com", passwordHash: "hash", name: "U", status: "ativo", licenseId: null });
    const request = await store.createFeatureRequest({ userId: "u1", title: "App", description: "Descrição do app", category: "geral" });
    await expect(transitionRequest(store, { id: request.id, actorUserId: "admin", actorRole: "admin", toStatus: "publicada" })).rejects.toThrow("versao_publicacao_invalida");
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/app-publish.test.ts` em `/site`.

Expected: falha de função ausente ou comportamento incorreto.

- [ ] **Step 3: implementar**

Adicionar à rota admin a ação `publish`, que:
1. baixa o arquivo do Blob;
2. confere sha256 e tamanho;
3. publica no GitHub como asset;
4. grava `published_asset_url` e `published_at`;
5. chama `transitionRequest(..., "publicada", version)`.

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/app-publish.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add app/api/admin/requests/[id]/route.ts tests/app-publish.test.ts
git commit -m "feat(apps): publicação de app aprovado no GitHub"
```

---

### Task 6: Catálogo público de apps

**Files:**
- Create: `/site/app/api/apps/route.ts`
- Test: `/site/tests/apps-catalog.test.ts`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/apps-catalog.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createMemoryStore } from "../lib/db/memory";
import { setStoreForTests } from "../lib/db";
import { GET as appsGet } from "../app/api/apps/route";

describe("catálogo de apps", () => {
  it("exige licença", async () => {
    const store = createMemoryStore();
    setStoreForTests(store);
    const res = await appsGet(new Request("http://localhost:3000/api/apps"));
    expect(res.status).toBe(401);
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/apps-catalog.test.ts` em `/site`.

Expected: falha de rota ausente.

- [ ] **Step 3: implementar**

Criar rota que lista solicitações publicadas com arquivo, retorna metadados públicos e exige token de dispositivo.

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/apps-catalog.test.ts` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add app/api/apps/route.ts tests/apps-catalog.test.ts
git commit -m "feat(apps): catálogo público de apps publicados"
```

---

### Task 7: Página de instalação do YAMI OS

**Files:**
- Create: `/site/app/(site)/instalar-yami-os/page.tsx`
- Test: `/site/tests/install-page.test.tsx`

- [ ] **Step 1: escrever o teste**

Create `/site/tests/install-page.test.tsx`:

```tsx
import { describe, expect, it } from "vitest";
import { renderInstallCommands } from "../lib/install-page";

describe("página de instalação", () => {
  it("gera comandos para juntar as partes", () => {
    const commands = renderInstallCommands(2);
    expect(commands.join(" ")).toContain("cat parte01 parte02");
  });
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test -- tests/install-page.test.tsx` em `/site`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Criar `lib/install-page.ts` com `renderInstallCommands(partsCount)` e a página `/instalar-yami-os`, lendo `lib/yamios/latest.json` e exibindo links, sha256 e comandos.

- [ ] **Step 4: rodar teste**

Run: `npm test -- tests/install-page.test.tsx` em `/site`.

Expected: 1 teste passa.

- [ ] **Step 5: commit**

```bash
git add "app/(site)/instalar-yami-os/page.tsx" lib/install-page.ts tests/install-page.test.tsx
git commit -m "feat(install): página de instalação do YAMI OS"
```

---

### Task 8: Integração da loja no YAMI OS

**Files:**
- Modify: `/os/src/interface/store.js`
- Modify: `/os/src/account/ipc.cjs`
- Test: `/os/tests/store-catalog.test.mjs`

- [ ] **Step 1: escrever o teste**

Create `/os/tests/store-catalog.test.mjs`:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { normalizeApp } from "../src/interface/store-apps.mjs";

test("normaliza app publicado", () => {
  const app = normalizeApp({ id: "notes", name: "Notas", version: "1.0.0", platform: "linux", sha256: "a".repeat(64) });
  assert.equal(app.id, "notes");
  assert.equal(app.platform, "linux");
});
```

- [ ] **Step 2: executar e confirmar falha**

Run: `npm test` em `/os`.

Expected: falha de módulo ausente.

- [ ] **Step 3: implementar**

Criar `store-apps.mjs` com `normalizeApp`, IPC `yami:apps-catalog`, download com verificação de sha256 e instalação sem sudo.

- [ ] **Step 4: rodar testes**

Run: `npm test` em `/os`.

Expected: todos passam.

- [ ] **Step 5: commit**

Nenhum commit; a área desktop é ignorada.

---

### Task 9: Verificação

- [ ] **Step 1: rodar todos os testes**

Run em `/site`: `npm test`.

Run em `/os`: `npm test`.

Expected: todos passam.

- [ ] **Step 2: verificar segurança**

Confirmar que nenhum arquivo é executado automaticamente e que o sha256 é conferido antes da instalação.

- [ ] **Step 3: registrar evidências**

Atualizar o arquivo de progresso.

---

## Auto-revisão do plano

- Cobertura da spec: upload (Task 3), anexo (Task 4), publicação (Task 5), catálogo (Task 6), página de instalação (Task 7), loja (Task 8).
- Sem placeholders nas etapas de implementação.
- Nomes consistentes: `validateAppFile`, `normalizeApp`, `renderInstallCommands`, `published_asset_url`.
