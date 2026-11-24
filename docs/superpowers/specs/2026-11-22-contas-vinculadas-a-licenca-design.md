# Especificação — Contas vinculadas à licença no site da YAMI

Data: 2026-11-22
Status: aprovada para geração do plano de implementação

## 1. Objetivo

Vincular a licença da YAMI à conta do site: ao comprar, o sistema cria uma conta com e-mail e senha definidos no checkout. O usuário gerencia sua conta e o administrador gerencia usuários e licenças.

## 2. Contexto

- O site já possui autenticação, pedidos, licenças, dispositivos e painel administrativo.
- O checkout atual cria apenas um pedido; a licença é liberada após pagamento aprovado.
- O painel `/conta` mostra licenças, pedidos, dispositivos e solicitações.
- O painel `/admin` publica instaladores e gerencia solicitações.

## 3. Arquitetura

### 3.1 Checkout cria conta

1. O usuário informa e-mail, nome e senha no checkout.
2. O site cria ou reutiliza a conta associada ao e-mail.
3. O pedido é criado com `user_id` vinculado.
4. Após pagamento aprovado, a licença é criada para o pedido e associada à conta.
5. O usuário entra com e-mail e senha e vê a licença no dashboard.

### 3.2 Gerenciamento

- O usuário pode alterar nome, senha e ver dispositivos.
- O administrador pode listar usuários, bloquear/desbloquear, redefinir senha e ver licenças.
- A licença é exibida no dashboard como ativa, com chave e histórico de pedidos.

### 3.3 Segurança

- Senha armazenada com scrypt e verificada com `timingSafeEqual`.
- O checkout valida e-mail e senha antes de criar a conta.
- O administrador não vê senhas.
- A redefinição de senha exige confirmação e gera auditoria.

## 4. Banco

Adicionar à tabela `users`:

- `status` (`ativo`, `bloqueado`)
- `license_id` (opcional, referência à licença ativa)

Adicionar à tabela `orders`:

- `user_id` (referência ao usuário)

## 5. Rotas

- `POST /api/checkout` — cria conta se não existir e vincula o pedido.
- `GET /api/conta` — retorna conta, licenças e pedidos do usuário autenticado.
- `PATCH /api/conta` — altera nome e senha.
- `GET /api/admin/users` — lista usuários e licenças.
- `PATCH /api/admin/users/:id` — bloqueia/desbloqueia ou redefine senha.

## 6. Interface

- `/conta`: dashboard com licença, pedidos, dispositivos e solicitações.
- `/admin`: painel de usuários com status, licença e ações.

## 7. Verificação

- Teste de criação de conta no checkout.
- Teste de login com a conta criada.
- Teste de visualização da licença no dashboard.
- Teste de bloqueio e redefinição pelo admin.

## 8. Fora de escopo

- Login social.
- Múltiplas licenças por conta.
- Recuperação de senha por e-mail.
- Integração com YAMI OS além da licença existente.
