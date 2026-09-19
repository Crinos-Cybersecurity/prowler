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

## Instalação na instância "cloud"

`pip install prowler` (ou imagem Docker oficial, se a instalação via
pip exigir dependências de sistema problemáticas — checar a
documentação de instalação real do fork no momento do deploy, não
assumir). Executado como subprocesso de vida curta pelo worker, nunca
como serviço de longa duração.
