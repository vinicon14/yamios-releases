# Progresso — Integração YAMI OS 0.8.0

Plano: `docs/superpowers/plans/2026-11-21-integracao-yami-os-0.8.0.md`

Worktrees:
- Releases: `D:\Downloads\yami relise os\.worktrees\feature-yami-os-0.8.0`
- Site: `C:\Users\vinim\.config\superpowers\worktrees\yami-site\feature-yami-site-0.8.0`

| Task | Nome | Status |
|---|---|---|
| 1 | Preparar áreas de trabalho e confirmar base 0.7.0 | concluída |


| 2 | Tornar o checkout rigoroso e idempotente | concluída (site `9735a9c`) |
| 3 | Ampliar esquema, tipos e conversores do banco | concluída (site `181db22`) |
| 4 | Implementar novos métodos nos três stores | concluída (site `e5dbfe7`) |
| 5 | Registrar eventos de pagamento e evitar dupla liberação | concluída (site `04bd864`) |
| 6 | Serviço de solicitações e notificações | concluída (site `6ced557`) |
| 7 | Autenticação de dispositivo | concluída (site `984d294`) |
| 8 | Rotas HTTP para aplicativo e administração | concluída (site `e0b56d7`, 129 testes; minors pendentes: mensagens de erro internas no DELETE/admin, coerção `String()` no PATCH, testes de PATCH não pendente e JSON inválido) |
| 9 | Interfaces de solicitações no site | concluída (site `e306b02`, 132 testes; rota `/api/conta/requests` adicionada para sessão por cookie) |
| 10 | Patch determinístico e cliente HTTPS | concluída na área `desenvolvimento/yami-os` (não versionada); 21 testes desktop no total |
| 11 | Handlers IPC no processo principal | concluída e ligada pelo patch; 21 testes desktop no total |
| 12 | View integrada da conta YAMI | concluída na área desktop; 21 testes desktop no total |
| 13 | Gerar pacote, manifesto e checksum 0.8.0 | concluída; pacote `e434778a…d6`, 223645 bytes |
| 14 | Verificação final antes da publicação | concluída no Linux (zstd/tar/executável/sha); release e deploy publicados |

Ressalvas para integração (Task 8):
- A rota `/api/desktop/login` deve aplicar rate limit e validar origem quando houver `Origin`.
- Administração deve derivar `actorRole: "admin"` de `getAdminFromRequest`, nunca do corpo da requisição.
- `revokeDesktopDevice` deve ser chamado sempre com `userId` validado.
- A rota de login não deve devolver `device.tokenHash`.

Preferência do usuário: usar `opencode-go/claude-haiku-5-5` para os próximos subagentes (implementação e revisão).

Pendências resolvidas em 22/11:
- JSON-LD: `lib/json-ld.ts` escapa `<` e `>`; `app/(site)/page.tsx` usa a função; testes adicionados. Site `aeb9bf6`.
- Lint: `AdminPanel.tsx` corrigido (dependência do `useMemo`); `npx eslint` sem erros (12 warnings pré-existentes).
- Teste final: `tests/final-check.test.mjs` varre segredos e valida o manifesto 0.8.0.
- Deploy: `https://yami-site-pi.vercel.app/api/yamios/latest` devolve 0.8.0 e o mesmo sha256 do pacote.

Pendências que dependem do usuário:
- Revogar o token do GitHub colado no chat: https://github.com/settings/tokens.
- Revogar/gerar novas credenciais do Mercado Pago e configurá-las na Vercel. `/api/status` mostra `salesOpen: false`.
- ISO 0.8.0 não foi gerada; o WSL Ubuntu 24.04 não tem `live-build`/`xorriso` e a unidade F: dos scripts de build está indisponível.

Pendências técnicas não concluídas:
- Teste visual/Electron real: o app foi copiado e aberto, mas o controle do desktop voltou para o usuário; não houve interação com a janela. Continua sem evidência de login e reinício no Electron real.
- Painel admin mostra itens aprovados e em desenvolvimento; confirmar com o usuário se quer ocultar histórico antigo.

Regras de execução:
- Um subagent implementador por tarefa, sequencialmente.
- Revisão de conformidade com a spec, depois revisão de qualidade.
- Nenhum push ou deploy sem autorização explícita.
