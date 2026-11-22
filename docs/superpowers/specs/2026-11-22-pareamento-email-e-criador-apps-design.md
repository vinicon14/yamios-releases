# Especificação — Pareamento por e-mail e criador de apps do YAMI OS

Data: 2026-11-22
Status: aprovada para geração do plano de implementação

## 1. Objetivo

Substituir o pareamento por código de 6 caracteres por um fluxo baseado em link por e-mail e implementar uma IDE completa dentro do YAMI OS para criar, testar, empacotar e publicar aplicativos compatíveis com a loja.

## 2. Contexto

- O pareamento atual usa `POST /api/pairing/start`, código de 6 caracteres e `pollPairing`; falhas de digitação e expiração dificultam o uso.
- A loja do YAMI OS já consulta apps publicados e instala pacotes Linux com verificação de sha256.
- A plataforma de criação de aplicativos ainda não existe no sistema.
- O YAMI OS já inclui OpenCode 2.3.2 como assistente de código.

## 3. Pareamento por e-mail

### 3.1 Fluxo

1. O aplicativo chama `POST /api/pairing/start` com o nome do dispositivo.
2. O site gera um token opaco de confirmação, válido por 10 minutos, e envia um link para o e-mail da conta autenticada.
3. O usuário clica no link no mesmo navegador onde está autenticado.
4. O site valida o token, o usuário e a licença, cria ou reutiliza o dispositivo e marca o pareamento como pronto.
5. O aplicativo consulta `GET /api/pairing/poll` e recebe o token de dispositivo.
6. O aplicativo armazena o token com `safeStorage` e passa a consultar conta e apps.

O código de 6 caracteres deixa de ser exibido e digitado. O link é de uso único, expira em 10 minutos e não pode ser reutilizado.

### 3.2 Banco

Adicionar à tabela `pairings`:

- `confirm_token_hash`
- `email_sent_at`
- `confirmed_at`

O token de confirmação é armazenado apenas como hash HMAC-SHA256. O token em claro existe apenas no link enviado por e-mail.

### 3.3 Rotas

- `POST /api/pairing/start` — autenticado por token de dispositivo ou sessão; cria o pareamento e envia o e-mail.
- `GET /api/pairing/confirm?token=...` — valida o token, redireciona para `/entrar?next=/parear` se necessário e confirma após login.
- `GET /api/pairing/poll` — inalterado, devolve `ready` com token de dispositivo.
- `POST /api/pairing/resend` — reenvia o link, com limite de 3 tentativas por hora por conta.

### 3.4 Segurança

- O link só confirma se o navegador estiver autenticado com a conta dona do pareamento.
- O token expira em 10 minutos e é invalidado após uso.
- Rate limit de 3 reenvios por hora por conta.
- O token nunca é gravado em logs, manifesto ou pacote.
- O aplicativo não exibe nem aceita código digitado.

## 4. Criador de apps

### 4.1 Escopo

A IDE completa é entregue em quatro subprojetos independentes:

1. Núcleo de projetos: criar, listar, renomear e excluir projetos.
2. Editor de arquivos: árvore de arquivos, edição, salvamento e visualização.
3. Build e empacotamento: validação, geração de `.tar.zst` ou `.AppImage` e sha256.
4. Publicação: envio para revisão, aprovação admin e entrada na loja.

Cada subprojeto gera software testável por si só.

### 4.2 Interface

- Nova view `Criar apps` no shell, acessível pelo menu e pela barra de tarefas.
- Layout com quatro regiões: barra de projeto, árvore de arquivos, editor e painel de build/publicação.
- O editor usa monoespaçada, tema atual e atalhos padrão (salvar, buscar, desfazer).
- O painel de build mostra validação, progresso, sha256 e link de publicação.

### 4.3 Projeto

- Projeto armazenado em `~/.local/share/yami/apps-dev/<id>/`.
- Estrutura mínima:
  - `yami-app.json`
  - `src/`
  - `assets/`
  - `README.md`
- `yami-app.json` define `id`, `name`, `version`, `description`, `category`, `entry`, `files` e `sha256`.
- O projeto não é executado automaticamente; a execução exige confirmação e sandbox do sistema.

### 4.4 Editor

- Operações: criar arquivo, criar pasta, renomear, excluir, salvar e recarregar.
- O editor salva com `Ctrl+S`; alterações não salvas são indicadas.
- Busca por texto dentro do projeto.
- Preview de imagem e JSON quando aplicável.
- Nenhum arquivo fora do diretório do projeto pode ser lido ou editado.

### 4.5 Build e empacotamento

- Validação de `yami-app.json`, estrutura e arquivos obrigatórios.
- Geração de `.tar.zst` com `yami-app.json`, `root/` e `posinstalar.sh` opcional.
- Cálculo de sha256 e tamanho.
- O pacote segue as mesmas regras de segurança do YAMI OS.

### 4.6 Publicação

- O build gera um pacote pronto para envio.
- O usuário envia o pacote e a descrição; a solicitação é criada com status `pendente`.
- O administrador aprova ou rejeita no painel existente.
- App aprovado entra no manifesto `yami-apps.json` e na loja.

## 5. Segurança

- O pareamento por e-mail não expõe token de dispositivo no navegador nem no aplicativo.
- O criador de apps não executa código automaticamente.
- O editor só acessa arquivos dentro do diretório do projeto.
- O build valida estrutura e arquivos antes de empacotar.
- Publicação exige aprovação administrativa.
- A loja verifica sha256 antes de instalar.

## 6. Verificação

- Teste de pareamento por e-mail: link válido, expirado, usado duas vezes e conta sem licença.
- Teste de permissão: somente licenciados podem enviar apps.
- Teste de segurança: editor não acessa arquivos fora do projeto.
- Teste de build: pacote gerado com sha256 conferido.
- Teste de publicação: app aprovado aparece em `/api/apps`.

## 7. Fora de escopo

- Geração automática de aplicativos por IA.
- Debug remoto de apps.
- Assinatura digital obrigatória de desenvolvedores.
- Execução de apps dentro da IDE.
