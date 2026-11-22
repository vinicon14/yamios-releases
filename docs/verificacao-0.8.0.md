# Verificação 0.8.0 — pacote desktop (Tasks 13 e 14)

Escopo executado: área desktop (`desenvolvimento/yami-os`) e pacote incremental no worktree de releases. Nada foi commitado, publicado nem criado como release. Nenhuma credencial do Mercado Pago foi usada.

## Build (Task 13)

- `npm install` em `desenvolvimento/yami-os`: instalou 13 pacotes, incluindo `@electron/asar` 3.2.14, com 0 vulnerabilidades.
- Estrutura conferida: `src/package.json`, `src/src/interface/yami-account.js`, `src/dist/main.js`, `src/dist/preload.cjs`.
- `npm run build` (sem `NODE_TEST` definido):
  - `app.asar` gerado a partir de `src/` via API `@electron/asar.createPackage`.
  - Tar escrito em JavaScript (cabeçalhos ustar com modo explícito) e comprimido com zstd do Node 24.
  - Motivo: no Windows, `writeFile` com `mode: 0o755` e `tar --zstd` do bsdtar não preservavam o bit de execução de `posinstalar.sh`.
- Saída do build:
  - arquivo: `yami-os-0.8.0-pacote.tar.zst` na raiz do worktree de releases
  - sha256: `e434778a8b08c9574f7d90c98cc348a00dc19c169c37cbca643995b593efe6d5`
  - tamanho: `223645` bytes
- `build/package-info.json` gravado com o mesmo sha256 e tamanho.
- `node build/update-manifest.mjs` gravou `MANIFEST_WRITTEN` em `latest.json` (versão 0.8.0, `deVersaoMinima` 0.7.0).

## Testes da área desktop

- `npm test` em `desenvolvimento/yami-os`: `tests 21`, `pass 21`, `fail 0`. Inclui `tests/package.test.mjs` (3 testes novos).

## Conteúdo do pacote

- `tar --zstd -tvf` lista:
  - `yami-pacote.json` (-rw-r--r--)
  - `posinstalar.sh` (-rwxr-xr-x)
  - `root/`, `root/opt/`, `root/opt/YamiOS/`, `root/opt/YamiOS/resources/`, `root/opt/YamiOS/resources/app.asar` (-rw-r--r--, 564160 bytes)
  - `root/usr/`, `root/usr/local/`, `root/usr/local/lib/`, `root/usr/local/lib/yami/` (diretórios vazios)
- Nenhum item fora de `yami-pacote.json`, `posinstalar.sh` e `root/...`.
- Mtime das entradas fixado em época zero (determinístico).

## Checksum e manifesto

- sha256 do arquivo (`Get-FileHash`): `e434778a8b08c9574f7d90c98cc348a00dc19c169c37cbca643995b593efe6d5`.
- `latest.json` `pacotes[0].sha256`: `e434778a8b08c9574f7d90c98cc348a00dc19c169c37cbca643995b593efe6d5` (confere).
- `latest.json` `pacotes[0].tamanho`: `223645` (confere com o arquivo).
- `latest.json` `versao`: `0.8.0`; UTF-8 válido (sem U+FFFD).

## Teste de atualização (extração isolada)

- `tar --zstd -xf` em `%LOCALAPPDATA%\Temp\opencode\yami-verify-0.8.0`: código de saída 0.
- Extraídos: `root/opt/YamiOS/resources/app.asar` (564160 bytes, sha256 `0f1b05eb692513fe91c05f1c6bd540d9a24cdaf8dbcddf320bd72da40fd82ceb`), `posinstalar.sh` (276 bytes, com bit de execução), `yami-pacote.json`.
- `app.asar` listado com `@electron/asar.listPackage`: 96 entradas, incluindo `\package.json`, `\dist\main.js`, `\dist\preload.cjs` e `\src\interface\yami-account.js`.
- Observação: a extração foi validada no Windows. A validação em Linux com GNU tar não foi feita; o WSL Ubuntu 24.04 não tem o binário `zstd` instalado.

## Varredura de segredos (Task 14)

- Padrões: `APP_USR-`, `TEST-`, `sk-[A-Za-z0-9]{10,}`, chave privada PEM, `MP_ACCESS_TOKEN=` e `MP_WEBHOOK_SECRET=` com valor.
- Alvos: `latest.json`, `desenvolvimento/yami-os/src`, `posinstalar.sh`, `yami-pacote.json`, árvore extraída do `app.asar` (181 arquivos varridos no total).
- Resultado: 0 ocorrências.

## Não executado nesta rodada

- `npm run patch` e a verificação de idempotência do plano.
- Testes do site (`/site`).
- `tests/final-check.test.mjs` da Task 14: não criado, porque referencia caminhos fora do worktree.
- Validação no WSL com `tar --zstd` (exige `zstd` no Ubuntu).
- Commit, push, tag e release no GitHub.
