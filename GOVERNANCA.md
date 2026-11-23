# Governanca do GitHub da Globals

Vale para todos os repositorios ativos da organizacao `weareglobals`, sem excecao, a partir de 2026-11-21.
A Globals opera sistemas de missao critica: estas regras sao o minimo, nao o teto.
As regras sao aplicadas pelo proprio GitHub (rulesets da organizacao), nao por combinado.

## Branches

| Branch | Papel | Como entra codigo |
|---|---|---|
| `main` (ou `master`, ate ser renomeada) | Producao. Cada merge vira uma versao (tag + release). | PR vindo **so** de `develop`, `release/*` ou `hotfix/*`, 1 aprovacao, **merge commit** |
| `develop` | Integracao. Branch padrao de todo repo. | PR de `feature/*` ou `bugfix/*`, 1 aprovacao, merge **so por squash** (o titulo do PR vira o commit) |
| `feature/CHAVE-N-descricao` | Trabalho novo. Ex.: `feature/ONE-157-limite-noturno` | Sai da `develop`, volta por PR para a `develop` |
| `bugfix/CHAVE-N-descricao` | Correcao que nao e urgente | Sai da `develop`, volta por PR para a `develop` |
| `release/AAAA-MM-DD` | Opcional: estabilizar uma versao sem travar a `develop` | Sai da `develop`; PR para a `main` e de volta para a `develop` |
| `hotfix/CHAVE-N-descricao` | Correcao urgente de producao | Sai da `main` (o check recusa se trouxer commit da `develop`); PR para a `main` **e** para a `develop` |

Modelo gitflow. O check `governanca / verificar` recusa PR fora desse fluxo e PR de fork.

Nas duas branches protegidas: sem push direto, sem push forcado, sem apagar a branch, aprovacao cai com commit novo,
quem deu o ultimo push nao aprova, conversas resolvidas antes do merge.

### Mesma regra para todos
Todo repositorio segue exatamente as mesmas regras: nao ha nivel "critico" ou "padrao". O que muda entre repos e so
quem e dono do codigo (CODEOWNERS).

**Excecao temporaria: `globals-infra` e `globals-qa-deploys`.** Os robos do GitOps do QA (build do `one`,
`qa-digest-syncer`, `argocd-image-updater`) gravam direto na `master` do `globals-infra` (347 de 429 commits em 60 dias),
hoje com credencial atribuida ao Monan. Aplicar a regra agora para o QA. Criterio de saida: os robos gravando so no
repo `globals-qa-deploys` (so valores de imagem), o Argo lendo dele, e um PR humano de prova no `globals-infra`.
Nesse dia o `globals-infra` entra nas mesmas regras.

### Donos (CODEOWNERS)
- Em todo repo, `/.github/` (workflows, CODEOWNERS) so muda com aprovacao de Monan ou Nikolas. Isso impede que um PR
  altere o proprio check obrigatorio para passar.
- Nos repos de infra e no `.github`, o repo inteiro tem esses donos.

### Break-glass
So admins da organizacao podem passar por cima das regras, e **so via PR** (nunca push direto). Uso exclusivo em
incidente sem segunda pessoa disponivel. Cada uso fica registrado em Insights > Rule insights (o GitHub Team nao manda alerta), e revisado toda semana
por quem nao usou, e tem que ser justificado no ticket do incidente em ate 24 h.

### Por que squash na develop e merge commit na main
- Na `develop` o historico fica linear e cada commit e um PR, com titulo padronizado e o ID do Windshift.
  Ninguem precisa reescrever os proprios commits de trabalho.
- Da `develop` para a `main` o merge e merge commit: squash ou rebase ali reescreveriam os commits e a `main` e a
  `develop` divergiriam a cada release. Na `main`, cada merge commit e uma release.

## Commits e PRs

Padrao [Conventional Commits](https://www.conventionalcommits.org/) com o ID do Windshift:

```
tipo(CHAVE-N): descricao no imperativo
feat(ONE-157): limite noturno do Pix
fix(LINCE-42): saldo nao atualiza apos estorno
```

Tipos: `feat`, `fix`, `perf`, `refactor`, `docs`, `test`, `build`, `ci`, `chore`, `style`, `revert`.
Mudanca incompativel: `feat(ONE-200)!: ...` ou rodape `BREAKING CHANGE: ...`.

O **titulo do PR** segue esse padrao (ele vira o commit no squash). Commits intermediarios da branch sao livres.

O check obrigatorio **governanca / verificar** (so vale vindo do GitHub Actions) barra o PR quando:
- o titulo nao e `tipo(CHAVE-N): descricao`, com o ID do Windshift no escopo;
- a branch esta fora do gitflow (tabela acima) ou vem de fork;
- falta algum dos quatro itens do checklist do template, ou algum esta desmarcado
  (o que nao se aplica: marque e escreva "nao se aplica").

PRs de release (`develop` -> `main`) tem titulo `release: AAAA-MM-DD`; os commits ja foram checados na `develop`.
PRs do Dependabot (branch `dependabot/*`, para a `develop`) sao dispensados so de ID e checklist; o resto vale.

## Revisao e aprovacao

A IA revisa o codigo; o humano aprova a mudanca. Ninguem precisa ler o diff linha a linha, mas quem aprova responde
pelo que entrou: "e isso que queriamos, e o risco e aceitavel?".

1. **Revisao de IA.** O Codex revisa o PR (revisao automatica da organizacao). Revisao feita fora do GitHub (Claude ou
   Codex local) vale se for registrada no PR com `VEREDITO: APROVADO` (ou `REPROVADO`) e o commit revisado.
   Regras especificas de cada repo ficam no `AGENTS.md`, secao `## Review guidelines`.
2. **Conversas resolvidas.** Todo apontamento abre uma conversa, e o merge so sai com todas resolvidas: corrigir, ou
   responder por que nao se aplica e resolver.
3. **Aviso automatico de aprovacao** (`globals-aprovacao.yml`). Quando o PR fica pronto (revisao de IA do commit atual,
   conversas resolvidas e checks verdes), o GitHub avisa no espaco **#code-review** do Google Chat marcando quem pode
   aprovar, com o resumo do PR, e registra o pedido num comentario do PR. Um aviso por commit; commit novo, novo aviso.
   - Dispara com review, resposta em conversa, comentario no PR e commit novo. So clicar em "Resolver" numa conversa nao
     dispara workflow no GitHub: se o PR ficou pronto assim e o aviso nao veio, comente `/aprovacao` no PR.
   - Com a revisao de IA feita, espera ate 15 min pelos checks e conversas que faltam; depois disso, `/aprovacao`.
   - Quem e marcado: os donos dos arquivos alterados no `CODEOWNERS` da branch de destino; sem dono, o aprovador padrao.
     O autor nunca e marcado.
   - O mapa login do GitHub -> pessoa no Chat fica na variavel da organizacao `APROVADORES_CHAT` (fora do repo, sem
     dado pessoal versionado). Pessoa nova: incluir la. O webhook fica no secret `GCHAT_APROVACAO_WEBHOOK`.
4. **1 aprovacao humana** obrigatoria, de alguem que nao deu o ultimo push. A revisao de IA nao conta como aprovacao, e
   o token do Actions nao aprova PR. Quem aprova le o resumo e a revisao no PR, e aprova em Files changed -> Review
   changes -> Approve.

## Versoes, tags e releases

- Cada merge na `main` gera automaticamente a proxima versao semantica `vX.Y.Z`, a tag e a release com notas
  (workflow `globals-release.yml`). Nada e commitado na `main` para isso.
  - `feat` -> minor, `fix` e demais -> patch, `!`/`BREAKING CHANGE` -> major.
  - Primeira release de um repo sem tag: `v1.0.0` (o que ja esta em producao e a linha de base).
- Tag `v*` e criada pelo workflow de release; ninguem cria tag `v*` a mao. Nenhuma tag `v*` pode ser apagada ou reescrita
  (ruleset `tags-de-versao`; o GitHub nao deixa restringir a criacao so ao Actions numa regra da organizacao).

## Arquivos padrao em cada repo

| Arquivo | Origem |
|---|---|
| `.github/workflows/globals-governanca.yml` | `modelos/workflows/` deste repo, fixado num commit deste repo |
| `.github/workflows/globals-release.yml` | `modelos/workflows/` deste repo, fixado num commit deste repo |
| `.github/workflows/globals-aprovacao.yml` | `modelos/workflows/` deste repo, fixado num commit deste repo (so na `develop`) |
| `.github/CODEOWNERS` | `/.github/` com `@weareglobals/aprovadores-infra`; nos repos de infra, o repo inteiro |
| Template de PR | `.github/pull_request_template.md` deste repo vale para todo repo sem template proprio |

Os arquivos ficam na `develop` e na `main`. Repo novo: rodar `gov-bootstrap.sh` nele. As regras da organizacao ja
valem sozinhas para todo repo novo. Mudou o workflow aqui: rodar o bootstrap de novo para atualizar o commit fixado.

## Repos sem atividade

Repos sem push ha mais de 12 meses sao **arquivados** (so leitura, reversivel em Settings > Unarchive).
Em 2026-11-21 foram arquivados 72.

## Seguranca da organizacao

- 2FA obrigatorio para todos os membros e colaboradores externos.
- Membros nao criam repositorio publico.
- Tokens pessoais: so fine-grained, com aprovacao e validade maxima de 90 dias; tokens classic bloqueados.
- GitHub Actions: token do workflow so le por padrao; workflow nao aprova PR.
- Dependabot (alertas e correcoes de seguranca) ligado em todo repo.
- Apps de terceiros so com acesso aos repos que usam. Revisao trimestral de membros, externos, apps e chaves.
- Sem credencial de nuvem no GitHub: deploys e segredos ficam na AWS (assinatura dupla da esteira).

### Compensacoes por estarmos no plano Team (sem Enterprise)
| Recurso Enterprise | Compensacao |
|---|---|
| SSO com o Google | 2FA obrigatorio + saida de pessoa inclui remover da org no mesmo dia + revisao trimestral |
| Envio do audit log | Webhook da organizacao grava os eventos (membros, repos, regras, pushes) no `log-archive` com WORM |
| Workflows obrigatorios da org | CODEOWNERS em `/.github/` + check aceito so do app GitHub Actions + workflow fixado por commit |
| Secret scanning e CodeQL (GHAS) | gitleaks e varredura de dependencias no CI padrao |

## Proximas fases
1. CI padrao `ci` (build, teste, lint, gitleaks, varredura de dependencias), exigido por regra a medida que cada repo
   ficar pronto.
2. Backup diario dos repos na conta `backup` (WORM).
3. Imagens assinadas e atestado de origem do build na esteira do `one`; SBOM por release.
4. Plano de deploy de hotfix com o GitHub fora do ar.

## Mudancas nestas regras
PR neste repo, aprovado por Monan ou Nikolas.
