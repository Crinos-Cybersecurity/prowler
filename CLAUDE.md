# prowler (fork interno)

**Este arquivo está versionado à força** (`git add -f`): o upstream do
Prowler lista `CLAUDE.md` no próprio `.gitignore` (linha 157), então
sem o `-f` ele existiria só na máquina de quem o criou e sumiria em
qualquer clone novo — **sem aparecer no `git status`**, que é o que o
torna traiçoeiro. Foi exatamente o que aconteceu antes desta correção.
O nome foi mantido (em vez de renomear pra `IRONBOT_FORK.md`, como em
`cloud-tools/cartography/`) porque `CLAUDE.md` é o único nome que o
Claude Code carrega automaticamente ao abrir uma sessão nesta pasta.

Duas consequências práticas, pra não gerar dúvida depois:
- **O `-f` só é necessário na PRIMEIRA vez.** `.gitignore` só vale pra
  arquivo não rastreado; uma vez commitado, alterações futuras aparecem
  no `git status` e entram com `git add` normal.
- **Risco residual**: se o upstream um dia adicionar um `CLAUDE.md`
  próprio, haverá conflito no `git merge upstream/main`. É um arquivo de
  documentação, então resolver é trivial — mas convém saber por quê.

Este diretório é um FORK do Prowler open source, upstream oficial
`prowler-cloud/prowler` (https://github.com/prowler-cloud/prowler).
Licença **Apache License 2.0** — confirmada lendo o arquivo `LICENSE`
real do repositório no momento do fork (2026-09-16), não assumida por
nome. Mesmo padrão de disciplina de fork já usado em `strix/CLAUDE.md`.

## Remotos configurados

- `origin` → nosso fork (`Crinos-Cybersecurity/prowler`, onde fazemos push)
- `upstream` → repositório oficial do Prowler — só pull/fetch, nunca push

## O que foi customizado em relação ao upstream

**Nenhuma customização de código até agora** — fork mantido em
paridade total com o upstream. É invocado como CLI (`prowler aws ...`)
via subprocess pelo worker da instância "cloud"
(`backend/infra/deploy/cloud_worker.py`), consumindo credenciais
temporárias via env vars (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/
`AWS_SESSION_TOKEN`) — nenhum ponto de integração exige mudar o código
do Prowler em si, só como ele é invocado e como o output (JSON) é
consumido do lado de fora.

## Estratégia de merge com upstream

Sem nenhuma customização hoje, um `git fetch upstream && git merge
upstream/main` nunca deveria gerar conflito. Se algum dia precisarmos
customizar algo (ex.: um check novo específico do IronBOT), preferir
isolar como um plugin/check próprio na estrutura de extensão que o
Prowler já oferece (`prowler/providers/aws/services/.../checks/`,
plugável sem editar arquivo existente) em vez de editar arquivo
upstream diretamente — mesma disciplina do `strix/CLAUDE.md`.

## Integração com o resto do projeto

- **Papel no pipeline de nuvem** (ver `backend/CLAUDE.md`, seção
  "Correlação de cadeia de ataque em nuvem (AWS) — MVP"): gera o
  snapshot estruturado de segurança AWS (IAM, S3, rede, etc.) — a
  PRIMEIRA das duas fontes de dado que alimentam a correlação (a
  segunda é o grafo do Cartography).
- **MVP cobre só AWS** — o Prowler já é multi-cloud (AWS/Azure/GCP/
  Kubernetes) nativamente, então estender pra outro provedor depois é
  configuração de invocação (`prowler azure ...` etc.), não mudança de
  código neste fork.
- **Credenciais**: nunca fixas — o worker assume um IAM Role temporário
  via STS (`sts:AssumeRole`) contra a conta AWS do cliente, repassa as
  3 credenciais temporárias como env vars só pro processo filho do
  Prowler, nunca grava em disco, nunca loga.
- **Framework de compliance MITRE ATT&CK**: a versão instalada expõe
  `prowler aws --list-compliance-frameworks` — verificar nessa listagem
  se `mitre_attack` está disponível antes de assumir que o mapeamento
  de técnica vem pronto do Prowler; se não estiver, a classificação de
  técnica MITRE ATT&CK Cloud fica por conta da camada de correlação do
  IronBOT (`app/domain/vulnerability_taxonomy.py`), não do Prowler.

## Instalação na instância "cloud" — **a partir DESTE fork**

`python3.12 -m venv /opt/prowler-venv && /opt/prowler-venv/bin/pip
install -e /opt/prowler`, onde `/opt/prowler` é o código deste
submódulo copiado por `rsync`. **Nunca `pip install prowler` do PyPI.**
Passo a passo em `backend/infra/deploy/DEPLOY.md`, seção 8.

Por que instalar do fork mesmo sem customização de código hoje:

- A versão em produção passa a ser a FIXADA no ponteiro de submódulo do
  repo coordenador. Com PyPI, cada rebuild da máquina puxaria uma versão
  nova em silêncio — com checks adicionados, removidos ou alterados,
  mudando o que o cliente vê sem ninguém decidir nada.
- Qualquer check próprio que venha a ser adicionado aqui só executa se o
  binário vier daqui. Do contrário fica em código morto.

**VENV DEDICADO, não compartilhado com o Cartography.** Os dois não
convivem: este projeto fixa versão exata (`==`) de dezenas de pacotes e
em pelo menos 9 deles a versão fixada é mais baixa que o piso que o
Cartography exige (`azure-mgmt-containerservice` ==34.1.0 vs >=41.0.0,
`azure-mgmt-network` ==28.1.0 vs >=31.0.0, `okta` ==3.4.2 vs >=3.4.4,
entre outros). Não existe resolução possível.

Python **3.12**: este projeto exige `>=3.10,<3.14` e o Cartography
`>=3.11`. Executado como subprocesso de vida curta pelo worker, nunca
como serviço de longa duração.

### Check próprio: `--checks-folder`, sem tocar no fork

O Prowler aceita `--checks-folder` (`-x`) apontando um diretório
EXTERNO de checks — ele copia cada subdiretório para dentro da árvore em
tempo de execução e remove depois. Confirmado funcionando na nossa
imagem: um check colocado ali aparece listado ao lado dos nativos.

**Este é o caminho preferido para cobrir lacuna do upstream**, porque os
checks ficam no NOSSO repositório e o fork segue em diff zero. Convenção
obrigatória: o nome do diretório precisa começar com o nome do serviço
(`ssm_parameter_no_plaintext_secrets` → serviço `ssm`), e dentro dele
vão `__init__.py`, `<nome>.py` e `<nome>.metadata.json`.

Editar arquivo deste fork é o ÚLTIMO recurso — e, quando inevitável,
acompanhado de PR upstream, para que o diff tenha rota de saída em vez
de virar dívida permanente.

**Em uso hoje**, em `backend/infra/deploy/cloud/checks/` (ver o
`README.md` de lá para o diagnóstico completo de cada um):

| Check | Lacuna do upstream que cobre |
|---|---|
| `ssm_parameter_no_plaintext_secrets` | Não existe NENHUM check de Parameter Store. Nunca lê o valor do parâmetro: classifica por nome e tipo, com `ssm:DescribeParameters` apenas. |
| `ssm_document_secrets_ironbot` | `ssm_document_secrets` é falso negativo — reporta PASS em documento com `AWS_SECRET_ACCESS_KEY` visível, em YAML e em JSON. |

Duas armadilhas do validador de metadata, ambas já custaram uma rodada:
`RelatedUrl` **precisa** ser string vazia (campo deprecado) e
`Remediation.Recommendation.Url` só aceita vazio ou
`https://hub.prowler.com/...`. Nossos checks não estão no Hub, então
vazio é a opção honesta e o link real vai para `AdditionalURLs`, que não
é validado.
