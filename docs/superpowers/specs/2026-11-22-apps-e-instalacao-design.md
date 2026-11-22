# Especificação — Publicação de apps e instalação do YAMI OS

Data: 2026-11-22
Status: aprovada para geração do plano de implementação

## 1. Objetivo

Permitir que usuários com licença paga enviem aplicativos para o GitHub por meio do site, com revisão administrativa, e que esses aplicativos possam ser baixados e instalados pela loja do YAMI OS. Também criar uma página pública no site para instalação fácil do YAMI OS a partir da ISO 0.8.0 já publicada.

## 2. Contexto

- O site já possui contas, licenças, solicitações de melhorias com aprovação administrativa, notificações e checkout.
- O YAMI OS 0.8.0 já possui loja com catálogo local, ícones pixel art e instalação de apps pelo processo principal do Electron.
- A release `v0.8.0` do `yamios-releases` contém o pacote incremental e a ISO em duas partes (`aa96926a63c28df2808bd43105776639eac72d653bd8d32bd33266fa63e1b5e3`).
- O repositório de releases continua sem código-fonte; somente manifestos e assets.
- O primeiro app a ser publicado será o YAMI Desktop Windows (2,5 GB), como download externo, com instalação nativa apenas para pacotes Linux.

## 3. Decisões aprovadas

1. Implementar primeiro a publicação de apps e depois a página de instalação.
2. Somente usuários com licença paga podem enviar arquivos.
3. Upload privado no Vercel Blob; após aprovação, arquivo público como asset de release no GitHub.
4. Formatos aceitos: `.AppImage` e `.tar.zst`, até 500 MB.
5. O envio fica vinculado a uma solicitação existente; admin aprova texto e arquivo juntos.
6. App aprovado aparece automaticamente na loja para usuários licenciados.
7. Interface de envio: anexo dentro do formulário de solicitação.
8. Instalação no sistema sem `sudo`, com verificação de sha256 e atalho no menu do usuário.
9. Página pública `/instalar-yami-os` com downloads, comandos copiáveis e sha256.
10. Apps Windows aparecem na loja como downloads externos; não são instalados automaticamente no YAMI OS.

## 4. Backend de envio e publicação

### 4.1 Banco

Adicionar à tabela `feature_requests` (ou tabela de detalhes vinculada):

- `file_name`
- `file_size`
- `file_sha256`
- `file_blob_pathname`
- `app_version`
- `app_icon`
- `published_asset_url`
- `published_at`

A transição para `aprovada` continua pertencente ao administrador. A transição para `publicada` exige `published_version`, `published_asset_url` e `file_sha256`.

### 4.2 Rotas

- `POST /api/uploads/app` — autenticado por cookie ou token de dispositivo; valida licença, extensão e tamanho; devolve URL assinada de upload do Blob e metadados esperados.
- `POST /api/conta/requests` — existente; passa a aceitar `fileName`, `fileSize`, `fileSha256`, `fileBlobPathname`, `appVersion` e `appIcon` junto com título, descrição e categoria.
- `GET /api/admin/requests` — passa a incluir metadados de arquivo e URL assinada de leitura.
- `POST /api/admin/requests/:id` — aprova/rejeita. A aprovação com arquivo baixa o Blob, confere tamanho e sha256, publica no GitHub como asset e grava `published_asset_url`.
- `GET /api/apps` — lista apps publicados para usuários licenciados, com nome, versão, categoria, descrição, ícone, URL do asset, sha256, tamanho, plataforma e data.

O manifesto `yami-apps.json` também é publicado como asset da release `v0.8.0`, contendo os mesmos metadados de `/api/apps`.

### 4.3 Validação

- Extensões permitidas: `.AppImage`, `.tar.zst`.
- Tamanho máximo: 500 MB.
- SHA-256 calculado no navegador e reconferido no servidor no momento da aprovação.
- Nenhum arquivo é executado no servidor.
- O Blob usa caminho privado com identificador aleatório; a URL de leitura é assinada e de curta duração.

## 5. Loja no YAMI OS

- Nova aba “Publicados” na loja, além das categorias atuais.
- Cartões com ícone pixel art, nome, versão, categoria, tamanho, plataforma, selo “Instalado” e botão “Instalar” ou “Atualizar”.
- O cliente HTTPS do processo principal consulta `/api/apps` com o token de dispositivo; o renderer nunca faz download direto.
- Instalação de pacotes Linux:
  1. baixar para `~/.local/share/yami/downloads/`;
  2. conferir sha256 e tamanho;
  3. instalar em `~/.local/share/yami/apps/<id>/`;
  4. criar atalho `~/.local/share/applications/yami-app-<id>.desktop`;
  5. registrar versão em `~/.local/share/yami/apps.json`.
- Apps com `platform: "windows"` são exibidos com botão “Baixar para Windows”; o download usa `published_asset_url` e não tenta instalar no YAMI OS.
- Atualização compara versão instalada com o catálogo; remoção apaga somente arquivos do app e o atalho.
- A confirmação exibe publicador, versão, tamanho e hash. Nenhum app é executado automaticamente.

## 6. Página de instalação do YAMI OS

- Rota pública `/instalar-yami-os`, alimentada por `lib/yamios/latest.json`.
- Mostra versão, data, sha256 da ISO, tamanho e links das duas partes da release.
- Abas Windows, Linux e “Já tenho o YAMI OS”.
- Comandos copiáveis para juntar as partes e verificar o sha256.
- Se `iso.partes` estiver vazio, mostrar “ISO em preparação” em vez de links quebrados.
- Nenhuma instalação é disparada automaticamente pelo navegador.

## 7. Erros

- Hash divergente ao instalar: arquivo baixado é apagado; versão anterior permanece.
- Falha de upload: solicitação permanece pendente e pode ser reenviada.
- Falha ao publicar no GitHub: solicitação fica aprovada, sem `published_asset_url`, e pode ser republicada pelo administrador.
- Sem rede: loja mostra último catálogo em cache como somente leitura.
- Token revogado: aplicativo volta ao formulário de login; nenhuma ação de escrita é permitida.

## 8. Verificação

- Testes de validação de arquivo, permissão de upload, aprovação administrativa e publicação do manifesto.
- Teste de integração: enviar app → aprovar → `/api/apps` → instalar no YAMI com sha256 conferido.
- Verificação da página de instalação contra o manifesto real com duas partes e os links da release.
- Teste de download externo do app Windows sem instalação automática.

## 9. Fora de escopo

- Repositório público com código-fonte dos apps enviados.
- Execução automática de apps após instalação.
- Assinatura digital obrigatória de desenvolvedores.
- Geração de uma nova ISO para cada app publicado.
- Instalação automática de apps Windows no YAMI OS.
