Firecrawl is a web scraper API. The directory you have access to is a monorepo:
 - `apps/api` has the actual API and worker code
 - `apps/*-sdk` are various SDKs

When making changes to the API, here are the general steps you should take:
1. Write some end-to-end tests that assert your win conditions, if they don't already exist
  - 1 happy path (more is encouraged if there are multiple happy paths with significantly different code paths taken)
  - 1+ failure path(s)
  - Generally, E2E (called `snips` in the API) is always preferred over unit testing.
  - In the API, always use `scrapeTimeout` from `./lib` to set the timeout you use for scrapes.
  - These tests will be ran on a variety of configurations. You should gate tests in the following manner:
    - If it requires fire-engine: `!process.env.TEST_SUITE_SELF_HOSTED`
    - If it requires AI: `!process.env.TEST_SUITE_SELF_HOSTED || process.env.OPENAI_API_KEY || process.env.OLLAMA_BASE_URL`
2. Write code to achieve your win conditions
3. Run your tests using `pnpm harness jest ...`
  - `pnpm harness` is a command that gets the API server and workers up for you to run the tests. Don't try to `pnpm start` manually.
  - The full test suite takes a long time to run, so you should try to only execute the relevant tests locally, and let CI run the full test suite.
4. Push to a branch, open a PR, and let CI run to verify your win condition.
Keep these steps in mind while building your TODO list.

Never bypass `knip` failures (e.g. with `git commit --no-verify`). If the pre-commit `knip` check fails, fix the reported unused exports/files — even if they predate your change — before committing.

---

## FBNet image release flow (este fork → Docker Hub → FBNet)

Este fork publica as imagens Docker que o **FBNet** consome por *pull* (não há build local no FBNet).
Modelo adotado: **release branch** — uma branch canônica detém `:latest`; branches experimentais
publicam tags próprias e **nunca** sobrescrevem `:latest`. Aprovado pelo PO em 2026-06-21.

### Papéis das branches
- **`fbnet-docker-firecrawl-release`** — branch CANÔNICA de produção.
  Build **automático** a cada push (`apps/api/**`, `apps/playwright-service-ts/**`, ou o próprio workflow).
  Publica `fcjbispo/fbnet-firecrawl:latest` + `fcjbispo/fbnet-firecrawl-playwright:latest` e os aliases imutáveis `:sha-<short>`.
- **`fbnet-docker-firecrawl-v*`** (ex.: `v4`) — branches de experimentação.
  Build **manual** via *workflow_dispatch* (GitHub → Actions → "Build & Publish Firecrawl images" → Run workflow → escolher a branch).
  Publicam `:<nome-da-branch>` + `:sha-<short>`. **Nunca** `:latest` → produção fica protegida de merges não validados.

### Imagens e quem consome
| Imagem | Contexto do build | Consumo no FBNet |
|---|---|---|
| `fcjbispo/fbnet-firecrawl:latest` | `apps/api` | serviços `firecrawl` (cmd `dist/src/harness.js`) e `firecrawl-worker` (cmd `dist/src/services/queue-worker.js`) |
| `fcjbispo/fbnet-firecrawl-playwright:latest` | `apps/playwright-service-ts` | serviço `firecrawl-playwright` |

### Pré-requisitos (configurar uma vez)
1. **Secrets do repo** (GitHub → Settings → Secrets and variables → Actions):
   - `DOCKERHUB_USERNAME` = `fcjbispo`
   - `DOCKERHUB_TOKEN` = **Access Token** do Docker Hub com permissão *Read & Write* (não a senha).
2. O workflow `.github/workflows/firecrawl-publish.yml` já está na branch `release`.
3. Habilitar Actions no repo (Settings → Actions → *Allow all actions*).

### Fluxo completo: validar e promover uma nova versão
```bash
# 1. Criar branch experimental a partir da release
git checkout fbnet-docker-firecrawl-release
git pull --ff-only
git checkout -b fbnet-docker-firecrawl-v4
# (desenvolver / sincronizar com upstream mendableai/firecrawl main — ver "Sync upstream" abaixo)

# 2. Push da branch experimental (não dispara build automático — só release dispara)
git push -u origin fbnet-docker-firecrawl-v4

# 3. Build MANUAL: GitHub → Actions → "Build & Publish Firecrawl images"
#    → Run workflow → branch: fbnet-docker-firecrawl-v4 → Run
#    Resultado: publica fcjbispo/fbnet-firecrawl:fbnet-docker-firecrawl-v4 + :sha-<short>

# 4. Testar no FBNet apontando o compose para a tag da branch (temporário):
#    em fbnet/tools/firecrawl/docker-compose.yaml:
#      image: fcjbispo/fbnet-firecrawl:fbnet-docker-firecrawl-v4
#    make start-firecrawl   (validar scrape real + worker)

# 5. Aprovado → PROMOVER via fast-forward (sem merge commit, histórico linear):
git checkout fbnet-docker-firecrawl-release
git merge --ff-only fbnet-docker-firecrawl-v4
git push origin fbnet-docker-firecrawl-release
#    → o push dispara o build AUTOMÁTICO → reescreve :latest (+ novo :sha-<short>)
#    → o FBNet atualiza sozinho (Watchtower faz auto-pull de :latest)

# 6. Reverter o compose do FBNet de volta a :latest (já é o default — só confirmar)
#    e arquivar/excluir a branch experimental se não for mais necessária.
```

### Rollback
- `:latest` é **mutável**. Para voltar a uma versão anterior:
  - **Rápido/temporário:** repointar o compose do FBNet para um `:sha-<short>` imutável (âncora de segurança) e reiniciar o serviço.
  - **Definitivo:** forçar a branch `release` ao commit antigo (`git checkout fbnet-docker-firecrawl-release && git reset --hard <commit> && git push -f`) → o push rebuilda `:latest` para aquele estado.
- As tags `:sha-<short>` são **imutáveis** — existem justamente para rollback e auditoria. Nunca reutilize uma tag `:sha-*`.

### Sincronizar com upstream (`mendableai/firecrawl`)
O Makefile do **FBNet** oferece `make fbnet-firecrawl-sync` (usa `FBNET_FIRECRAWL_UPSTREAM_FOLDER` apontando para este fork +
`FBNET_FIRECRAWL_DEFAULT_BRANCH`), que executa `git fetch upstream && git merge upstream/main` numa branch de **integração**
(não diretamente na `release` — preserve o gate de validação). Após o sync, crie uma branch experimental (`v*`) a partir
do resultado e siga o fluxo de promoção acima (passos 2–6). Em resumo: **upstream → branch de integração → branch experimental validada → release**.

### Notas
- O Dockerfile de `apps/api` usa BuildKit `--mount=type=cache`; por isso o workflow exige `docker/setup-buildx-action` (não funciona com `docker build` puro sem BuildKit).
- O cache GHA usa `scope=api` e `scope=playwright` separados — o GHA não permite duas gravações no mesmo scope.
- Não há `:v3` *rolling*: o `:v3` que existe no Docker Hub é legado e pode ser removido pela UI do Hub. O rolling de produção é `:latest` (na branch `release`).