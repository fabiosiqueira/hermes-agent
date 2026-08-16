---
name: hermes-sync
description: Use quando este fork (hermes-engine, upstream NousResearch/hermes-agent) precisa reconciliar — `local/all-fixes` atrás de `upstream/main`, carries acumulados sem rota de volta, destino dos nossos PRs incerto, suspeita de que produção já resolveu nossa proposta, ou nossos PRs abertos apodreceram (conflitados/CI-vermelho). Triggers "/hermes-sync", "sincroniza fork", "reconcilia carries upstream", "audita nossos PRs no hermes-agent", "adere à produção", "apruma nossos PRs". Conduz o loop audita→sync→reconcilia→adere-produção→apruma-PRs. Alias: /hermes-sync
---

# /hermes-sync — Reconciliação fork hermes-engine ↔ upstream

Skill local a este projeto (não global) — opera só sobre este repo.

## Princípio central

Rodamos `local/all-fixes` = `upstream/main` + carries (features/fixes nossos que ainda não landaram upstream). A reconciliação periódica é cara e propensa a erro; esta skill destila o workflow em processo repetível. Seis princípios travados:

1. **Dedup-first.** NUNCA abrir PR sem `gh pr list --search` antes. Achou solução alheia equivalente → **absorve** (depende do PR dela como rota), nunca duplica. Lição #40652: nosso PR duplicou o #23715 alheio = lixo no upstream.
2. **Conflito = integrar nosso carry SOBRE o control-flow do upstream.** Superset limpo, sem código órfão; nunca "escolher um lado". Verificar com testes-alvo + `py_compile`.
3. **Harmonia.** `origin/main` é espelho fast-forward de `upstream/main` — nunca commitar direto nele. Todo carry/PR nosso deve ficar mergeável sobre `local/all-fixes`.
4. **Audita auto, muta sob aprovação.** Fases read-only rodam sozinhas; toda ação que escreve ou é pública (FF, merge, force-push, abrir/fechar PR, dispatch de workflow) pede OK antes.
5. **Aderir à produção.** Onde `upstream/main` já provê a capacidade da nossa proposta — por rota-PR merged OU por refactor próprio deles — **convergir**: tomar a versão de produção, remover órfão residual, fechar nosso PR como superseded — **mesmo que a rota-PR/carry siga OPEN**. Grep a produção (`git show upstream/main:<file>`) antes de assumir que só a rota merged resolve. Rebase de PR antigo: upstream pode ter REMOVIDO feature que a PR carregava como contexto → mantê-la cega ressuscita dead code (NameError); isole adições reais com `git show <commit> | grep '^+'`.
6. **Aprumar nossos PRs abertos.** PR OPEN nosso não é espera passiva: num repo gigante o maintainer só olha *mergeável + verde + issue-linked*. Manter cada um rebasado sobre `upstream/main`, CI verde, e cross-ref a issue existente do maintainer (dedup: nunca abrir issue órfã). Falha de CI pode ser flake alheio — diagnostique o job ANTES de "consertar"; nunca enfraqueça teste de terceiro. Falha só local = deps ausentes reproduzem idêntica em `upstream/main` puro.

**Antes de executar:** leia `learned.md` (conflitos recorrentes, termos de dedup-search que funcionaram, rotas de carry). **Ao fim:** registre gotchas novos lá, e persista o mapa de rotas no auto-memory do projeto (tipo `project`, nome `hermes-carry-pr-routes`) — não em caminho hardcoded.

| Fase | O que faz | Auto / Aprovação |
| --- | --- | --- |
| **0. Preflight** | valida remotes (origin/upstream), branch atual, working tree limpo; `git fetch` | auto (read-only) |
| **1. Auditoria de PRs** | lista PRs nossos no upstream; classifica MERGED / CLOSED (superseded vs rejeitado) / OPEN; p/ cada OPEN checa saúde (`mergeable`/CI/issue-link) **e se produção já supera a proposta** (grep `upstream/main`) | auto (read-only) |
| **2. Sync do fork** | FF `origin/main`←`upstream/main` (só se FF puro); merge `upstream/main`→`local/all-fixes`. Conflito → PARA | aprovação |
| **3. Reconciliação de carries** | lista carries; carrega mapa da memória (se existir; senão reconstrói do zero via `git log` + `gh`); re-valida cada rota; **checa adesão-à-produção (princípio 5) por carry**; árvore de decisão | auto audita / aprovação p/ agir |
| **3b. Aprumar PRs abertos** | p/ cada PR nosso OPEN não-superseded: rebase sobre `upstream/main`, testes-alvo + `py_compile`, force-push, cross-ref issue existente (princípio 6) | auto audita / aprovação p/ agir |
| **4. Memória + relatório** | atualiza o mapa de rotas no auto-memory; imprime tabela-resumo | auto |

## Quando usar

- `local/all-fixes` ficou atrás de `upstream/main` e precisa reconciliar.
- Carries acumularam sem rota de volta a `origin/main` — a branch nunca encolhe.
- Destino dos nossos PRs no upstream (merged/superseded/rejeitado/open) está incerto.
- Suspeita de que produção já resolveu nossa proposta (checar adesão-à-produção, princípio 5).
- Nossos PRs abertos apodreceram (conflitados/CI-vermelho/sem-issue) e precisam ser aprumados (princípio 6).
- Operador diz "sincroniza fork" / "reconcilia carries" / "adere à produção" / "apruma nossos PRs" / "/hermes-sync".

**NÃO usar para:** auto-resolver conflito de merge (sempre julgamento humano) · mexer em PR de terceiros (só lê/absorve) · generalizar para outros forks (hardcode do modelo: remotes `origin`/`upstream`, branch `local/all-fixes`) · disparar build/deploy do artefato (`build-engine.yml` etc. vivem no `defi-project`, fora do escopo desta skill).

### 0. Preflight (AUTO, read-only)

```bash
git remote | grep -qx origin && git remote | grep -qx upstream || { echo "ERRO: remote faltando"; exit 1; }
[ -z "$(git status --porcelain)" ] || { echo "ERRO: working tree sujo"; exit 1; }
git fetch upstream
git fetch origin
```

### 1. Auditoria de PRs nossos no upstream (AUTO, read-only)

```bash
gh pr list --author @me --state all -R NousResearch/hermes-agent \
  --json number,title,state,url
```

`--author @me` é portável (não hardcode login). Fallback se `@me` falhar: `GHLOGIN=$(gh api user --jq .login)` e use `--author "$GHLOGIN"`.

Para cada PR CLOSED, leia o último comment p/ distinguir superseded vs rejeitado:

```bash
gh pr view <n> -R NousResearch/hermes-agent --json closed,state,comments,body
```

### 2. Sync do fork (APROVAÇÃO)

FF de `origin/main` só se for **FF puro** (zero commits próprios à frente):

```bash
git rev-list --count upstream/main..origin/main
```

`0` = FF seguro. Qualquer valor `>0` → origin/main divergiu, **PARA** e investiga.

Mergeabilidade SEM mutar working tree (requer git ≥ 2.38):

```bash
git merge-tree --write-tree --messages upstream/main local/all-fixes
echo "EXIT_CODE:$?"
```

Gate primário = exit code (`0` limpo, `1` conflito). Gate secundário:

```bash
git merge-tree --write-tree --messages upstream/main local/all-fixes 2>&1 | grep -E '^CONFLICT'
```

Conflito → **PARA**, mostra os arquivos ao operador. Resolução é princípio 2, verificada com testes-alvo + `py_compile`. **Só depois** o merge real + commit + push (aprovação). FF, merge, push são todos APROVAÇÃO.

### 3. Reconciliação de carries (AUTO audita / APROVAÇÃO p/ agir)

```bash
git log upstream/main..local/all-fixes --no-merges
```

`--no-merges` count **≠** carries líquidos: pares add+revert se cancelam (cheque com `git diff upstream/main local/all-fixes -- <arquivo>` = 0 linhas → neutralizado). Árvore de decisão por carry:

- **Ainda faz sentido?** Par add+revert se cancela → STALE, neutraliza.
- **Produção já provê a capacidade?** (princípio 5, cheque SEMPRE, independe da rota) — `git show upstream/main:<file> | grep <símbolo/feature>`. Achou equivalente → **CONVERGIR**: `git checkout --theirs <file>` só quando o único delta nosso no arquivo é o carry; remove órfão residual (`grep -rn <helper>` sem caller); testes-alvo + `py_compile`; fecha nosso PR superseded — APROVAÇÃO. Vale **mesmo com a rota-PR OPEN**.
- **Tem rota no mapa (memória)?** Re-valida via `gh`: MERGEOU → CONVERGIR; OPEN → mantém em harmonia (candidato a aprumar, Fase 3b); CLOSED/rejeitada → decide com operador.
- **Sem rota** → **DEDUP-SEARCH ANTES de qualquer PR**:

```bash
gh pr list --search "<termos da mudança>" --state all \
  -R NousResearch/hermes-agent --json number,title,state,author,url
```

  - achou equivalente alheio → **ABSORVE** (registra a rota, ZERO PR nosso).
  - inédito de fato → oferece abrir nosso PR — APROVAÇÃO.

**Escopo de "fechar PR":** fechar PR **NOSSO** superseded/duplicado entra no escopo (com aprovação). **NUNCA** fecha PR de terceiros.

### 3b. Aprumar nossos PRs abertos (AUTO audita / APROVAÇÃO p/ agir)

Princípio 6. Para cada PR **nosso** OPEN não-superseded, audita saúde (read-only):

```bash
gh pr view <n> -R NousResearch/hermes-agent --json isDraft,mergeable,reviewDecision,statusCheckRollup,headRefName
git fetch upstream "pull/<n>/head" && git merge-tree --write-tree upstream/main FETCH_HEAD; echo "exit=$?"
```

`mergeable != MERGEABLE`, CI vermelho, ou conflita com `upstream/main` → **aprumar** (APROVAÇÃO, reporta antes de cada force-push):

1. Rebase em branch de trabalho: `git checkout -B rebase/<slug> origin/<headRefName>` → `git rebase upstream/main`.
2. Resolve conflitos por princípio 2. **Cuidado (princípio 5):** upstream pode ter REMOVIDO feature que a PR carregava como contexto — isole adições reais com `git show <commit> -- <file> | grep '^+'` e dropa o resto; senão ressuscita dead code (NameError).
3. Verifica: testes-alvo da PR + `py_compile`. **CI vermelho pode ser flake alheio** — diagnostica o job (`gh run view <id> --log-failed`), nunca enfraquece teste de terceiro. Falha só local = dep ausente reproduz idêntica em `upstream/main` puro.
4. `git push --force-with-lease origin rebase/<slug>:<headRefName>`.
5. **Issue-âncora (dedup-first):** `gh issue list -R NousResearch/hermes-agent --search "<feature>"`. Achou request existente do maintainer → cross-ref por `gh pr comment`. **NÃO** abre issue órfã de contribuidor externo. Comment termina com footer `— 🤖 <modelo em uso>` (nome do modelo que está escrevendo, nunca hardcoded).
6. Cleanup: volta p/ `local/all-fixes`, `git branch -D rebase/<slug>`.

### 4. Memória + relatório (AUTO)

Reescreve o mapa de rotas no auto-memory do projeto (tipo `project`) com o estado revalidado. Imprime tabela-resumo: destino de cada PR nosso + rota de cada carry.

## Red flags — PARE

- "Abro o PR direto" → sem dedup-search você acabou de criar lixo no upstream. Busque primeiro.
- "Escolho um lado no conflito" → não. Integre nosso carry SOBRE o control-flow do upstream, superset limpo, verificado com testes-alvo + `py_compile`.
- "origin/main está à frente, mas FF mesmo assim" → não é FF puro (`rev-list --count` > 0). PARA e investiga.
- "Conto N commits, logo N carries" → pares add+revert se cancelam. Cheque o diff líquido por arquivo, não a contagem.
- "Esse carry tem rota OPEN, mas vou abrir outro PR" → rota OPEN já é a rota. Não duplique a sua própria.
- "Fecho esse PR de terceiro que duplica o nosso" → NUNCA fecha PR de terceiros. Só os nossos, sob aprovação.
- "A rota-PR não mergeou, então o carry fica" → cheque a PRODUÇÃO (princípio 5): upstream pode ter reimplementado por refactor próprio. Converge e fecha superseded mesmo com a rota OPEN.
- "Rebasei a PR e mantive o bloco em conflito inteiro" → upstream pode ter removido feature que a PR carregava como contexto; isole adições reais.
- "CI vermelho, vou mexer no teste que falhou" → pode ser flake alheio ou dep local ausente. Diagnostica o job e compara com `upstream/main` puro ANTES; nunca enfraquece teste de terceiro.
- "Abro issue pra ancorar meu PR" → dedup primeiro. Issue órfã de externo não é triada; só cross-ref a request EXISTENTE do maintainer.
- "Vou disparar o build/deploy pra fechar o loop" → fora do escopo desta skill. Sync termina em `local/all-fixes` reconciliado + push; build/deploy é decisão separada do operador, feita a partir do `defi-project`.

## Quick reference

| Item | Valor |
| --- | --- |
| Repo local | `hermes-engine` (fork); upstream real chama-se `hermes-agent` — `-R NousResearch/hermes-agent` |
| Remotes | `origin` = fork pessoal · `upstream` = NousResearch · branch `local/all-fixes` |
| Filtro de PRs | `gh pr list --author @me` (portável; fallback `gh api user --jq .login`) |
| FF check | `git rev-list --count upstream/main..origin/main` — `0` = FF puro |
| Merge-tree gate | `git merge-tree --write-tree --messages upstream/main local/all-fixes` — exit `0`/`1` + `grep '^CONFLICT'` |
| Carries | `git log upstream/main..local/all-fixes --no-merges` (cheque pares add+revert) |
| Resolução de conflito | integra carry SOBRE control-flow upstream, superset limpo + testes-alvo + `py_compile` |
| Dedup obrigatório | `gh pr list --search "<termos>" --state all -R NousResearch/hermes-agent` antes de abrir PR |
| Fonte do mapa | auto-memory do projeto (tipo `project`, nome `hermes-carry-pr-routes`) — lê → re-valida → reescreve |
