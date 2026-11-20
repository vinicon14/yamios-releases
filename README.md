# YAMI OS — atualizações

Repositório público só com os **manifestos e pacotes de atualização** do YAMI OS
(sem código-fonte, sem chaves, sem dados de usuário).

O app **Atualizações YAMI OS** (dentro do sistema) consulta, nesta ordem:

1. `https://yami-site-pi.vercel.app/api/yamios/latest` (Vercel, fonte principal)
2. `https://github.com/vinicon14/yamios-releases/releases/latest/download/latest.json` (este repositório, reserva)

## Formato do `latest.json`

```json
{
  "produto": "YAMI OS",
  "canal": "estavel",
  "versao": "0.5.0",
  "build": "r5",
  "data": "2026-11-20",
  "notas": ["..."],
  "pacotes": [
    { "versao": "0.5.1", "deVersaoMinima": "0.5.0", "url": "https://.../yami-update-0.5.1.tar.zst",
      "sha256": "...", "tamanho": 12345 }
  ],
  "iso": { "url": null, "sha256": "...", "tamanho": 0, "kernel": "6.8.0-149-generic" }
}
```

## Pacote de atualização (`yami-update-X.tar.zst`)

- `yami-pacote.json` — versão e notas
- `root/` — arquivos do sistema que serão copiados por cima (só `usr/`, `opt/`, `etc/`, `var/lib/yami/`)
- `posinstalar.sh` — opcional

O app baixa, confere o **sha256**, pede confirmação e aplica. Nunca mexe em `/home`, contas,
senhas nem nas redes Wi-Fi salvas. A ISO completa (se publicada) é baixada como `.new` na pasta
do YAMI e só troca no próximo boot, mantendo os dados salvos.

## Publicar uma versão

1. Gerar o pacote e o `sha256sum`.
2. Criar a release `vX.Y.Z` aqui com `latest.json` e o pacote como anexos.
3. Atualizar `lib/yamios/latest.json` no `yami-site` e fazer o deploy na Vercel.
