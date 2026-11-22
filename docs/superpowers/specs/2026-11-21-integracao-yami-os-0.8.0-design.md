# Especificação — Integração YAMI OS 0.8.0 (Marco 1)

Data: 2026-11-21
Status: aprovada para geração do plano de implementação
Versão final planejada do sistema: `0.8.0` (`r8`)

## 1. Objetivo

Implementar a primeira fase da grande atualização do YAMI OS partindo da versão 0.7.0 (`r7`): integração completa entre o sistema e o site oficial da YAMI, abrangendo conta, licença, pagamentos, solicitações de melhoria e acompanhamento no dashboard.

Os demais recursos pedidos — loja, criação de aplicativos, áudio, desempenho, atualizações, backup, personalização, assistente, arquivos, controle rápido e áreas de trabalho — pertencem aos marcos 2 e 3 e terão especificações próprias. A release `v0.8.0` só será publicada depois que todos os marcos passarem nos testes.

## 2. Contexto auditado

- `vinicon14/yamios-releases` contém apenas `latest.json`, `README.md`, manifesto 0.7.0, pacote `yami-os-0.7.0-pacote.tar.zst` e ISO em duas partes. As tags `v0.5.0`, `v0.6.0` e `v0.7.0` apontam para o mesmo commit de manifesto; os metadados e anexos das releases contêm a história real das versões.
- O pacote 0.7.0 instala `app.asar` do shell Electron, tema de ícones `YAMI-Pixel`, script `instalar-opencode.sh` e atalho do OpenCode. Ele não contém o código-fonte do shell nem o mecanismo `aplicar-pacote.sh`.
- O shell é Electron com `contextIsolation`, `sandbox` e CSP restrita. A interface é HTML/CSS/JavaScript em `src/interface`, e o processo principal expõe operações via `ipcMain`/`preload.cjs`.
- A baseLinux usa Debian/Ubuntu com XFCE, NetworkManager (`nmcli`), PipeWire/PulseAudio, `pkexec` e helpers em `/usr/libexec`.
- O site `vinicon14/yami-site` é Next.js 16 com autenticação por cookie `HttpOnly`, scrypt, Postgres/SQLite, pedidos, licenças, dispositivos, pareamento, Checkout Pro do Mercado Pago e webhook.
- O checkout atual usa pagamento único de R$ 20,00, Pix e cartão, `external_reference` com o ID do pedido e validação do pagamento por consulta autenticada à API do Mercado Pago.
- A cópia local do site em `D:\Downloads\open code yami edit` está no branch `main` e sincronizada com `origin/main`.

## 3. Decisões do usuário

1. Executar os marcos de forma sequencial, concluindo a integração antes dos recursos locais do sistema.
2. Manter licença permanente por pagamento único, sem assinatura.
3. Fazer login no aplicativo com e-mail e senha uma única vez; o aplicativo recebe e persiste apenas um token de dispositivo revogável.
4. Usar um aplicativo nativo integrado ao shell, não um site incorporado nem um binário separado.
5. Adotar o layout A aprovado na prévia visual: cartões centrais com abas `Conta`, `Solicitações` e `Avisos`.
6. Publicar somente marcos verificados, com autorização explícita para cada commit, push, deploy e release.

## 4. Escopo deste marco

### 4.1 Incluído

- APIs de desktop no site para autenticação, sessão, licença, pedidos, solicitações e notificações.
- Tabelas e regras de solicitações com aprovação administrativa obrigatória.
- Correções de segurança e idempotência no checkout e webhook do Mercado Pago.
- Nova view integrada `YAMI` no shell da 0.7.0.
- Geração reproduzível do pacote 0.8.0, com `sha256`, manifesto e plano de atualização sem perda de dados.
- Testes automatizados, teste de integração em Linux e verificação de segurança.

### 4.2 Fora deste marco

- Alteração do seletor de áudio e da barra de tarefas além da inclusão do ícone/atalho da view `YAMI`.
- Reestruturação da loja e remoção da instalação automática do Duelverse.
- Plataforma de criação de aplicativos.
- Gerenciadores de desempenho, backup e atualizações, personalização avançada, áreas de trabalho e assistente ampliado.
- Geração da ISO completa 0.8.0; a release do pacote incremental pode ser publicada primeiro, mantendo o padrão de `latest.json` com `iso.url: null` se a ISO não estiver pronta.

## 5. Organização de código e repositórios

- O repositório `yamios-releases` permanece como repositório público de manifestos e pacotes, conforme seu README.
- A especificação, os scripts de construção e o código-fonte desktop trabalhado ficam em `desenvolvimento/`, ignorado pelo Git do repositório de releases por meio de `.git/info/exclude`.
- O código do site é alterado na cópia local de `yami-site`. Cada alteração é commitada e enviada somente após aprovação.
- O shell 0.7.0 é extraído do pacote oficial existente. A modificação é feita por patches determinísticos e versionados sobre essa base, preservando todo o código funcional atual.
- Nenhum segredo, credencial, token de API, senha ou dados pessoais são gravados em arquivos versionados, manifests, logs ou testes.

## 6. Arquitetura do site

### 6.1 APIs de desktop

Todas as rotas retornam JSON, usam `runtime = "nodejs"` e não habilitam CORS. Se uma requisição de navegador trouxer `Origin`, ele deve ser igual à origem configurada do site; requisições nativas sem `Origin` são aceitas.

| Rota | Método | Autenticação | Finalidade |
|---|---|---|---|
| `/api/desktop/login` | POST | e-mail + senha no corpo | Valida credenciais, cria dispositivo e devolve token de dispositivo. |
| `/api/desktop/session` | GET | `Authorization: Bearer <deviceToken>` | Conta, licenças, status formatado, pedidos, dispositivos e contadores de avisos. |
| `/api/desktop/logout` | POST | token de dispositivo | Revoga o dispositivo atual. |
| `/api/desktop/requests` | POST | token de dispositivo | Cria solicitação do usuário autenticado. |
| `/api/desktop/requests` | GET | token de dispositivo | Lista solicitações do usuário com status e histórico resumido. |
| `/api/desktop/requests/:id` | GET | token de dispositivo | Detalha uma solicitação própria. |
| `/api/desktop/requests/:id` | PATCH/DELETE | token de dispositivo | Edita ou retira solicitação somente no estado `pendente`. |
| `/api/desktop/notifications` | GET | token de dispositivo | Lista avisos não expirados, com `unread`. |
| `/api/desktop/notifications/:id` | PATCH | token de dispositivo | Marca um aviso como lido. |

Respostas de erro usam códigos estáveis: `invalid_credentials`, `license_required`, `device_revoked`, `rate_limited`, `invalid_request`, `not_found`, `forbidden` e `service_unavailable`. Mensagens não incluem detalhes internos.

O token de dispositivo é opaco, gerado com 32 bytes aleatórios e armazenado apenas como HMAC-SHA256 no banco, usando o segredo de sessão. Cada login nativo cria um registro com nome derivado do sistema, limitado a 80 caracteres, e cada conta pode ter no máximo 5 dispositivos ativos. O aplicativo reutiliza apenas o dispositivo local já cadastrado; novos dispositivos exigem login e são revogáveis em `/conta`.

A rota de login aplica limite de 10 tentativas por IP e 5 por e-mail em 15 minutos, sempre que o armazenamento de limite permitir. Não define cabeçalho `Access-Control-Allow-Origin`.

### 6.2 Dados

Novas tabelas em `lib/db/schema.sql`, implementadas de forma equivalente nos stores SQLite, Postgres e memória:

- `feature_requests`: `id`, `user_id`, `title`, `description`, `category`, `status`, `created_at`, `updated_at`, `published_version`, `admin_note`.
- `feature_request_events`: `id`, `request_id`, `from_status`, `to_status`, `actor_user_id`, `actor_role`, `note`, `created_at`.
- `user_notifications`: `id`, `user_id`, `kind`, `title`, `body`, `request_id`, `read_at`, `created_at`, `expires_at`.
- `payment_events`: `id`, `order_id`, `payment_id`, `event_type`, `result`, `reason_code`, `created_at`. Nenhum corpo de webhook, token ou dado de pagador é gravado.

Índices: solicitações por usuário e status, eventos por solicitação, avisos por usuário e criação, eventos de pagamento por pedido e pagamento.

Estados de solicitação: `pendente`, `aprovada`, `rejeitada`, `em_desenvolvimento`, `publicada`, `retirada`. O administrador pode transitar para `aprovada`, `rejeitada`, `em_desenvolvimento` e `publicada`; a transição para `publicada` exige `published_version` no formato `X.Y.Z`. Usuário comum só cria, edita e retira enquanto o estado é `pendente`. Toda transição gera evento e, quando relevante, aviso ao solicitante.

### 6.3 Interface administrativa e do usuário

- `/conta` ganha seção `Solicitações`, com formulário (título, categoria, descrição), histórico de status e botão para retirar itens pendentes.
- `/admin` ganha painel de solicitações pendentes com detalhes, nota administrativa e ações `Aprovar`, `Rejeitar`, `Iniciar desenvolvimento` e `Marcar como publicada`.
- A criação de qualquer release continua exclusivamente nos fluxos administrativos já existentes; solicitações nunca publicam arquivos nem alteram manifests.

### 6.4 Correções do Mercado Pago

- `paymentMatchesOrder` exige `external_reference` igual ao `order.id`. Referência ausente ou divergente é rejeitada.
- Confere `status = approved`, moeda `BRL` e valor dentro da tolerância de R$ 0,01 em relação ao pedido e ao preço configurado.
- `markPaid` passa a ser condicional: somente uma transição atômica de `pending` para `paid` associada ao `paymentId` é aceita. Tentativas repetidas do mesmo pagamento retornam o resultado idempotente, e um payment ID associado a outro pedido é rejeitado.
- Todo resultado do webhook grava `payment_events` com código de motivo resumido.
- Em `production`, `chargesEnabled()` exige token de acesso e `MP_WEBHOOK_SECRET`. Sem segredo, o checkout permanece fechado com mensagem de configuração.
- A assinatura `x-signature` é sempre validada quando o segredo existe; a consulta autenticada ao pagamento continua obrigatória.
- O checkout reaproveita pedidos pendentes recentes para evitar clique duplo e mantém o aviso sobre não usar a conta da loja.

## 7. Arquitetura do desktop

### 7.1 Origem e patches

A base é o `app.asar` da release 0.7.0, extraído para `desenvolvimento/yami-os/base/`. Os arquivos editáveis são `src/interface/index.html`, `src/interface/app.js`, `dist/main.js`, `dist/preload.cjs` e novos módulos de interface. Um script Node/Python aplica patches de forma idempotente, falha quando o trecho esperado não existe e verifica o resultado antes de gerar o `app.asar`.

### 7.2 View e interface

- Nova `section.view-container` com `id="view-yami"`, usando as variáveis CSS atuais, tipografia mono e ícones em pixel art do tema `YAMI-Pixel`.
- Botões de acesso no menu Iniciar e na barra de tarefas, com o mesmo padrão de `.pin.nav-btn` e ícone `yami` em pixel art.
- Três abas internas: `Conta`, `Solicitações` e `Avisos`, com contador de não lidos.
- Cartões da aba `Conta`: usuário/conexão, licença e status, pedidos, dispositivos e link para o dashboard no navegador.
- A aba `Solicitações` permite criar e acompanhar itens; a aba `Avisos` mostra aprovações, rejeições, mudanças de status e atualizações publicadas.
- A URL base do site é constante `https://yami-site-pi.vercel.app`. Sobrescrita permitida apenas por `/etc/yami/site.json`, com esquema `https:` ou `http://127.0.0.1` para testes, validada no processo principal.
- `shell.openExternal` continua sendo o único caminho para abrir páginas externas.

### 7.3 IPC e armazenamento

Handlers principais: `yami:account-login`, `yami:account-logout`, `yami:account-session`, `yami:account-requests-list`, `yami:account-request-create`, `yami:account-request-update`, `yami:account-request-withdraw`, `yami:account-notifications`, `yami:account-notification-read`.

O renderer nunca recebe a senha após a chamada de login, não faz `fetch` direto à API e não manipula tokens. O processo principal valida entradas (tamanho, caracteres, tipos), usa `AbortSignal.timeout`, aplica tentativa única em falhas transitórias e sanitiza mensagens de erro.

O token é cifrado com `electron.safeStorage.encryptString` e salvo em `app.getPath("userData")/yami-account.bin` com modo `0600` quando possível. Sem `safeStorage`, nenhum arquivo de sessão é gravado e a interface informa que o login durará apenas até fechar o aplicativo. Nenhuma queda para texto puro é permitida.

### 7.4 Estados e tratamento de falhas

- `401`: limpa a sessão local e mostra formulário de login.
- `403`: mostra `É necessária uma licença YAMI ativa` com link para o site, sem ocultar o restante do sistema.
- `429`: aplica espera progressiva de 1, 2 e 4 minutos e informa o motivo.
- `5xx` ou falha de rede: mantém a última resposta válida em memória com selo `offline/somente leitura`; ações de escrita ficam desabilitadas.
- A sessão e os avisos são atualizados ao abrir a view e a cada 5 minutos enquanto o aplicativo estiver visível. Não há consultas em segundo plano quando a janela está oculta.

## 8. Compatibilidade e empacotamento

- O pacote segue o formato atual: `yami-pacote.json`, `posinstalar.sh` opcional e `root/` contendo apenas `usr/`, `opt/`, `etc/` e `var/lib/yami/`.
- `yami-pacote.json` declara `versao: 0.8.0`, `deVersaoMinima: 0.7.0` e descrição das mudanças.
- `posinstalar.sh` atualiza cache de ícones e recarrega a interface; não apaga pastas de usuário, não reinicia o sistema e não modifica a partição de persistência.
- O script de build gera `sha256`, tamanho, manifesto `latest.json` no padrão atual e resumo de publicação sem caminhos internos.
- Instalações 0.5.0 e 0.6.0 continuam podendo usar os pacotes das versões anteriores. O pacote 0.8.0 destina-se a 0.7.0.
- Se a ISO 0.8.0 não estiver disponível, `iso.url` permanece `null` e `iso.sha256`/partes permanecem com os valores 0.7.0 até uma publicação própria; o app de atualizações continua funcional pelo pacote.

## 9. Testes

### 9.1 Site

- Testes unitários: validação de assinatura, `paymentMatchesOrder`, `external_reference` ausente/diferente, valor divergente, moeda divergente, estados rejeitados, idempotência de webhook duplicado e concorrência de `markPaid`.
- Testes de permissão: usuário não aprova solicitação; administrador vê pendentes; token revogado falha; login com senha errada falha; limites de requisição são aplicados.
- Testes de integração com stores memória e SQLite. Postgres é exercitado quando `DATABASE_URL` de teste estiver disponível; a ausência não invalida a lógica comum.
- Testes de interface com Playwright: login, visualização de licença, criação e acompanhamento de solicitação, notificação de aprovação, revogação de dispositivo e checkout redirecionando corretamente.

### 9.2 Desktop

- Testes unitários das funções puras de normalização de URL, validação de entrada, mapeamento de erros e montagem de solicitações.
- Teste de patch: aplicar duas vezes não altera o resultado; arquivos inesperados na base causam falha.
- Teste de empacotamento: somente diretórios permitidos, permissões `0755` para executáveis, nenhum segredo, `sha256` conferido.
- Teste em Linux isolado: abrir a view, entrar, consultar licença, enviar solicitação, simular aprovação no backend, receber aviso, revogar dispositivo e confirmar que o próximo acesso exige login.
- Teste de atualização: instalar o pacote 0.8.0 sobre uma cópia da 0.7.0 com arquivos de usuário, Wi-Fi salvo e apps instalados; confirmar que permanecem intactos.

### 9.3 Critérios de aceite

1. Nenhuma credencial, token, segredo ou dado de pagador aparece em código, log, teste, manifesto ou resposta de API.
2. Um mesmo pagamento aprovado libera no máximo uma licença e um conjunto de downloads.
3. Solicitação alguma entra em desenvolvimento sem aprovação administrativa registrada.
4. O aplicativo mostra corretamente licença ativa, pedidos e avisos a partir do backend real.
5. A atualização incremental preserva dados pessoais e configurações.
6. Sem backend, o aplicativo sinaliza indisponibilidade em vez de simular sucesso.

## 10. Publicação

1. Rodar lint, typecheck, testes e build do site.
2. Fazer commit do site, com push e deploy apenas após autorização explícita.
3. Validar endpoints em produção com conta de teste e sem expor tokens.
4. Rodar verificações desktop e gerar pacote/checksum.
5. Atualizar `latest.json` de `lib/yamios` no site e o `latest.json` do repositório de releases.
6. Criar release `v0.8.0` somente após os marcos 2 e 3; o pacote 0.8.0 pode ser construído e testado antes disso.
7. Apresentar relatório final com funcionalidades, arquivos alterados, testes, problemas, dependências externas, instruções de atualização e versão publicada.

## 11. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Perda da compatibilidade com pacotes antigos | Manter manifestos e URLs das versões anteriores; não alterar o contrato de `yami-atualizacoes`. |
| Exposição de token no renderer | Manter token apenas no processo principal, cifrado e nunca em logs. |
| Webhook duplicado ou concorrente | Transição atômica de pedido, chave única de pagamento e registros de evento. |
| Aparência desconectada | Reutilizar variáveis CSS, fontes, ícones e padrões de navegação atuais; validar capturas no Linux. |
| Publicação incompleta | Marcos independentes com critérios de aceite e release final após validação conjunta. |
| Indisponibilidade do Mercado Pago | Checkout fecha com mensagem clara em produção sem configuração válida; nenhuma licença é liberada sem confirmação autenticada. |
