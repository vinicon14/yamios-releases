# Especificação — Login e contas de usuário no YAMI OS 1.0.0

Data: 2026-11-22
Status: aprovada para geração do plano de implementação

## 1. Objetivo

Adicionar uma tela de login dentro do YAMI OS antes da área de trabalho, com contas separadas de dados, preservação da conta existente `yami` e opção de criar e trocar de usuário. A versão 1.0.0 inclui todas as mudanças já prontas da 0.8.0 mais este recurso.

## 2. Contexto

- O YAMI OS 0.8.0 já tem gerenciamento de usuários Linux no painel de configurações, mas não tem tela de login antes da área de trabalho.
- A conta existente é `yami` (usuário live), com dados em `/home/yami`. Sua senha deve ser definida como `vv12122020`.
- A interface do YAMI OS roda como Electron em kiosk e já impede fechamento acidental da janela.
- A versão 1.0.0 deve ser publicada como pacote de atualização para 0.8.0 e como ISO nova para novas instalações.

## 3. Arquitetura

- O shell Electron inicia em uma tela de login antes de criar a janela principal.
- A autenticação usa contas Linux locais (`/etc/passwd`, PAM) e dados separados em `/home/<usuario>`.
- A conta `yami` continua sendo a conta principal e não perde dados.
- A criação de novas contas é restrita ao administrador.
- A troca de usuário encerra a sessão atual e retorna à tela de login.
- O YAMI OS não fecha ao clicar no botão fechar; somente desligamento/reinício encerra a sessão.

## 4. Contas

### 4.1 Conta existente

- Usuário: `yami`
- Senha: `vv12122020`
- Dados preservados: `/home/yami`, configurações do YAMI, licenças e histórico.
- A atualização para 1.0.0 não apaga nem substitui esses dados.

### 4.2 Criação de contas

- Disponível somente para administrador.
- Campos: nome de exibição, nome de usuário e senha.
- Validação: usuário em minúsculas, senha de 8 a 128 caracteres.
- A conta criada recebe diretório `/home/<usuario>` e configurações iniciais.

### 4.3 Troca de usuário

- Disponível no painel de configurações.
- Encerra a sessão atual e mostra a tela de login.
- Dados de cada usuário permanecem separados.

## 5. Interface

- Tela de login com campos de usuário e senha, botão entrar e mensagem de erro.
- Tela de criação de conta acessível apenas ao administrador.
- Painel de configurações com seção "Contas de usuário", lista de contas, criação e troca.
- A interface mantém o padrão visual do YAMI OS.

## 6. Segurança

- Senhas armazenadas com scrypt e verificadas com `timingSafeEqual`.
- A tela de login não expõe se o usuário existe.
- A criação de contas exige privilégio administrativo.
- A troca de usuário não apaga dados.
- O YAMI OS não fecha acidentalmente.

## 7. Verificação

- Teste de login com `yami` e senha `vv12122020`.
- Teste de criação de conta pelo administrador.
- Teste de troca de usuário.
- Teste de preservação de dados após atualização.
- Teste de que o botão fechar não encerra o YAMI OS.

## 8. Fora de escopo

- Login remoto por e-mail.
- Sincronização de contas entre computadores.
- Execução simultânea de duas contas na mesma sessão.
- Recuperação de senha por e-mail.
