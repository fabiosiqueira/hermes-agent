# learned — hermes-sync

> Auto-evolutivo. Leia antes de executar; atualize ao fim de cada sync.
> Aceita: gotchas de conflito recorrente, termos de dedup-search que funcionaram, rotas de carry. Cap 8 entradas, ≤180 chars.
> Restaurada do backup global em 2026-08-16, agora local ao projeto hermes-engine (era `~/.claude/skills/hermes-sync`).

- **`rerere` estava DESLIGADO até 2026-08-16** (ligado com `autoupdate`; 3 resoluções em `.git/rr-cache`, que é local e NÃO versionado — reclonar perde). Sem ele, a mesma resolução trivial era refeita à mão a cada sync.
- **Conflito recorrente aqui é POSICIONAL, não semântico:** upstream empilha kwargs novos (`skip_background_review`) exatamente onde nosso carry inseriu o dele (`skip_memory_provider`, ao lado de `skip_memory`). Cura: parkear knob do fork no FIM da assinatura, fora da zona quente.
- **Contagem bruta ≠ carries líquidos:** pares add+revert se cancelam; cheque diff líquido (`git diff upstream/main local/all-fixes -- <f>`) E se merge anterior já tocou o arquivo — sem a 2ª checagem, `--theirs` descarta resolução humana em silêncio.
- **DEDUP antes de tudo** — abrir PR, abrir issue-âncora E aprumar: #40652 duplicou #23715; #45808 duplicou #27977. Achou alheio → doar o delta lá, nunca competir. `skip_memory_provider` tem 4 PRs alheios OPEN (#9802, #18565, #31661, #7557) — absorver.
- **Convergência sem rota merged:** produção absorve por refactor próprio (memory-bridge → `MemoryManager.notify_memory_tool_write`). Ação: `--theirs` no core (só se o único delta nosso lá for o carry) + `grep -rn <helper>` p/ caçar órfão — carry que auto-mergeia limpo em arquivo NÃO-conflitado vira dead code.
- **Rebase de PR parado: o CONTEXTO apodrece, não só o código.** Upstream pode ter removido feature que a PR carregava (`profile` no cron → NameError), subido default, ou podado teste de propósito. Isole adições reais com `git show <commit> -- <file> | grep '^+'` e `git log -S <símbolo>`.
- **Actions do fork estão DESLIGADAS no nível do repo** (confirmado 2026-08-16, `enabled:false`) — imagem do runtime é buildada pelo `build-engine.yml` do **defi-project**; nossos PRs são testados pela CI do upstream. `gh workflow list` diz `active`; o campo que vale é `actions/permissions`.
- **Aprumar PR ≠ sincronizar a branch — e a branch é a que roda em produção.** Meça `git rev-list --count local/all-fixes..upstream/main` na Fase 0, não no fim; drift de 2351 commits em 10 dias (2026-08-06→16) mostra que o upstream é rápido demais pra assumir "ainda deve estar perto".
