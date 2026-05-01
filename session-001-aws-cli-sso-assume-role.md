# Sessão 001 — AWS CLI avançado: SSO, perfis, assume-role e paginação

**Duração estimada:** 60 minutos
**Pré-requisitos:** nenhum

---

## Objetivo

Ao final, você conseguirá configurar perfis nomeados com SSO via `aws configure sso`, realizar `aws sts assume-role` para troca de contexto entre contas, paginar resultados com `--page-size` e `--max-items` sem perder registros, e usar `--query` com JMESPath para filtrar saídas.

---

## Contexto

[FATO] O AWS CLI v2 introduziu suporte nativo ao IAM Identity Center (antigo AWS SSO) com gerenciamento automático de tokens, eliminando a necessidade de ferramentas externas como `aws-vault` para o fluxo básico de autenticação. Antes disso, a única forma de usar credenciais temporárias com o CLI era exportar manualmente as variáveis de ambiente retornadas por `aws sts assume-role`.

[CONSENSO] Para ambientes multi-conta — que é o padrão em organizações com uso maduro de AWS — o fluxo de autenticação recomendado hoje é SSO via IAM Identity Center, com perfis nomeados no `~/.aws/config` para cada combinação de conta+role. O `aws sts assume-role` manual ainda tem uso legítimo em automações e pipelines, mas perdeu espaço como fluxo primário de engenheiros.

Este guia cobre os três pilares que todo engenheiro AWS usa no dia a dia: autenticar via SSO, trocar de contexto entre contas, e extrair dados de APIs sem perder registros ou sobrecarregar a API.

---

## Conceitos principais

### 1. Arquitetura de credenciais do AWS CLI

O CLI resolve credenciais numa cadeia de prioridade bem definida ([FATO], conforme documentação oficial):

```
1. Variáveis de ambiente        AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN
2. Perfil padrão no ~/.aws      [default] em ~/.aws/credentials ou ~/.aws/config
3. Container credentials        (ECS task role via ECS_CONTAINER_METADATA_URI)
4. Instance profile             (EC2 instance role via IMDS)
```

O arquivo `~/.aws/config` é o lugar correto para perfis nomeados — inclusive SSO. O `~/.aws/credentials` é legado e carrega apenas `aws_access_key_id` e `aws_secret_access_key`. [CONSENSO] Em ambientes modernos, o ideal é que `~/.aws/credentials` esteja vazio ou inexistente.

```
~/.aws/
├── config          # perfis, SSO, roles — tudo aqui
└── credentials     # legado; evite usar para novos perfis
```

A distinção entre os dois arquivos importa porque o `--profile` flag do CLI lê ambos, mas o `config` tem suporte a funcionalidades mais ricas (SSO, source_profile, role chaining, MFA serial).

---

### 2. SSO com IAM Identity Center — o modelo `sso-session`

[FATO] O AWS CLI v2.x introduziu o conceito de `sso-session`, que é uma seção separada no `~/.aws/config` responsável por gerenciar o token de acesso SSO de forma centralizada. Antes disso, cada perfil precisava ter as configurações SSO repetidas — e o token não era compartilhado.

**Estrutura atual (v2, recomendada):**

```ini
# ~/.aws/config

[sso-session minha-org]
sso_start_url    = https://minha-org.awsapps.com/start
sso_region       = us-east-1
sso_registration_scopes = sso:account:access

[profile dev]
sso_session      = minha-org
sso_account_id   = 111122223333
sso_role_name    = DeveloperAccess
region           = us-east-1
output           = json

[profile prod-readonly]
sso_session      = minha-org
sso_account_id   = 444455556666
sso_role_name    = ReadOnlyAccess
region           = us-east-1
output           = json
```

O bloco `[sso-session]` é compartilhado entre todos os perfis da mesma org. Um único `aws sso login --sso-session minha-org` autentica todos os perfis simultaneamente.

**Fluxo de autenticação:**

```
aws configure sso
  └── cria a sso-session e o perfil no ~/.aws/config

aws sso login --profile dev
  └── abre o browser → autoriza no Identity Center
  └── salva o token em ~/.aws/sso/cache/<hash>.json

aws s3 ls --profile dev
  └── CLI lê o token do cache
  └── troca por credenciais temporárias via STS
  └── credenciais ficam em ~/.aws/cli/cache/<hash>.json (TTL = duração da role)
```

**Cache de tokens e refresh automático:**

[FATO] O token SSO fica em `~/.aws/sso/cache/`. O CLI v2 verifica o token a cada hora e usa o refresh token para renová-lo automaticamente dentro do período de sessão estendida configurado no Identity Center (padrão: 8h, máximo configurável: 90 dias). Quando o token expira completamente, o comando falha com `Token has expired and refresh failed` — nesse caso, `aws sso login` resolve.

```bash
# Ver o token em cache (útil para debug)
cat ~/.aws/sso/cache/*.json | jq .

# Revogar e reautenticar
aws sso logout
aws sso login --profile dev
```

**Diagrama do fluxo SSO completo:**

```
[Você]
  │
  ├─ aws sso login ──► [Browser: Identity Center]
  │                         │
  │                    Autoriza acesso
  │                         │
  │◄──────── access_token + refresh_token ──────────
  │          (em ~/.aws/sso/cache/)
  │
  ├─ aws ec2 describe-instances --profile dev
  │         │
  │    CLI lê access_token do cache SSO
  │         │
  │    ┌────▼────────────────────┐
  │    │ STS: AssumeRoleWithWebIdentity │
  │    └────┬────────────────────┘
  │         │
  │    credenciais temporárias (15min–12h)
  │    (em ~/.aws/cli/cache/)
  │         │
  │    ┌────▼────────────────┐
  │    │ EC2 API call        │
  │    └─────────────────────┘
```

---

### 3. `aws sts assume-role` — troca de contexto entre contas

`assume-role` é a operação STS que troca credenciais atuais por credenciais temporárias de outra role. É o mecanismo por trás de todo acesso cross-account.

**Uso direto (imperativo — bom para scripts):**

```bash
# Assume a role e extrai as credenciais
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::999988887777:role/DeployRole \
  --role-session-name minha-sessao \
  --duration-seconds 3600 \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | awk '{print $1}')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | awk '{print $2}')
export AWS_SESSION_TOKEN=$(echo $CREDS | awk '{print $3}')

# Verificar que está operando como a role assumida
aws sts get-caller-identity
```

**Uso declarativo via perfil (recomendado para uso interativo):**

```ini
# ~/.aws/config
[profile deploy-prod]
role_arn       = arn:aws:iam::999988887777:role/DeployRole
source_profile = dev          # usa as credenciais do perfil 'dev' para assumir
role_session_name = luiz-cli
duration_seconds = 3600
region         = us-east-1
```

```bash
aws sts get-caller-identity --profile deploy-prod
# O CLI faz o AssumeRole automaticamente, sem nenhum script
```

**Role chaining — e seu limite crítico:**

[FATO] Role chaining é quando você usa as credenciais de uma role assumida para assumir outra role. O limite de duração é **fixo em 1 hora**, independentemente do valor passado em `--duration-seconds`. Isso é uma restrição do STS, não do CLI.

```
Usuário IAM → Assume RoleA (duração: até 12h)    ✅ OK
RoleA       → Assume RoleB (duração: máx 1h)     ⚠️ Sempre 1h
RoleB       → Assume RoleC (duração: máx 1h)     ⚠️ Sempre 1h
```

Se você tentar `--duration-seconds 7200` num chain, o STS retorna erro. O design é intencional — a AWS limita a "profundidade" de delegação para reduzir superfície de ataque.

```ini
# Exemplo de chain: dev → staging → prod
[profile staging]
role_arn       = arn:aws:iam::222233334444:role/StagingAccess
source_profile = dev

[profile prod]
role_arn       = arn:aws:iam::555566667777:role/ProdAccess
source_profile = staging      # chain: usa staging para chegar em prod
```

**MFA num assume-role:**

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::999988887777:role/PrivilegedRole \
  --role-session-name mfa-session \
  --serial-number arn:aws:iam::111122223333:mfa/luiz \
  --token-code 123456
```

```ini
# Ou declarativo no config:
[profile privileged]
role_arn       = arn:aws:iam::999988887777:role/PrivilegedRole
source_profile = default
mfa_serial     = arn:aws:iam::111122223333:mfa/luiz
```

---

### 4. Paginação — `--page-size` vs `--max-items`

Esta é uma das confusões mais comuns no CLI. Os dois parâmetros controlam coisas diferentes.

```
--page-size   = tamanho de cada request para a API AWS
--max-items   = quantidade total de itens no output do CLI
```

**Diagrama do fluxo:**

```
API AWS tem 500 objetos S3

Sem paginação:
  CLI faz 1 request → retorna 500 itens (pode dar timeout)

Com --page-size 50:
  CLI faz 10 requests de 50 itens cada → output: 500 itens
  (protege contra timeout, transparente pro usuário)

Com --max-items 100:
  CLI faz requests até ter 100 itens → para
  output: 100 itens + NextToken para continuar

Com --page-size 50 --max-items 100:
  CLI faz 2 requests de 50 itens → para
  output: 100 itens + NextToken
```

**O bug silencioso:**

[FATO] Se `--max-items` não for múltiplo de `--page-size`, você pode receber itens duplicados ou perder itens. Exemplo clássico: `--page-size 30 --max-items 50` — o CLI pega 2 páginas de 30 (60 itens), retorna os primeiros 50, mas o `NextToken` aponta para o item 31 da segunda página, não para o item 51 do total. Ao usar `--starting-token`, você recomeça do item 31, perdendo o 51–60.

**Recomendação segura:**

```bash
# 1. Para listar tudo sem risco: use apenas --page-size para controlar throughput
aws s3api list-objects-v2 \
  --bucket meu-bucket \
  --page-size 100

# 2. Para paginar manualmente com cursor explícito
aws s3api list-objects-v2 \
  --bucket meu-bucket \
  --page-size 50 \
  --max-items 50

# Pegar o NextToken do output anterior e continuar:
aws s3api list-objects-v2 \
  --bucket meu-bucket \
  --page-size 50 \
  --max-items 50 \
  --starting-token "eyJ..."

# 3. Para evitar timeout em buckets enormes (page menor = requests menores)
aws s3api list-objects-v2 \
  --bucket bucket-com-milhoes-de-objetos \
  --page-size 20
```

---

### 5. `--query` com JMESPath

`--query` aplica uma expressão JMESPath à resposta JSON antes do output. É processado localmente (sem custo adicional de API).

**Sintaxe essencial:**

```bash
# Acesso a campo simples
aws ec2 describe-instances \
  --query 'Reservations[0].Instances[0].InstanceId'

# Projeção: pegar campos específicos de uma lista
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Filtro: apenas instâncias rodando
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?State.Name==`running`].[InstanceId,PrivateIpAddress]' \
  --output text

# Pipe: pegar só o primeiro resultado de uma lista filtrada
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?State.Name==`running`].InstanceId | [0]'

# Aplanar listas aninhadas com []
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].InstanceId'
  # Sem o [] interno você teria [[id1, id2], [id3]] em vez de [id1, id2, id3]
```

**Combinando paginação com query:**

```bash
# Listar todas as funções Lambda com runtime Python
aws lambda list-functions \
  --page-size 50 \
  --query 'Functions[?Runtime==`python3.12`].[FunctionName,CodeSize]' \
  --output table
```

---

## Exemplo prático

**Cenário:** você tem acesso SSO à conta de desenvolvimento e precisa listar todas as instâncias EC2 em produção (outra conta), filtrando só as que estão `running`, sem baixar a lista inteira de uma vez.

```bash
# 1. Autenticar via SSO (uma vez por sessão)
aws sso login --profile dev

# 2. Verificar identidade atual
aws sts get-caller-identity --profile dev

# 3. Assumir a role de prod via perfil encadeado
# (já configurado no ~/.aws/config com source_profile = dev)
aws sts get-caller-identity --profile prod-readonly

# 4. Listar instâncias running em prod, paginando de 20 em 20
aws ec2 describe-instances \
  --profile prod-readonly \
  --region us-east-1 \
  --page-size 20 \
  --query 'Reservations[].Instances[?State.Name==`running`].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# 5. Se o ambiente tem muitas instâncias e você quer paginar manualmente
RESULT=$(aws ec2 describe-instances \
  --profile prod-readonly \
  --page-size 20 \
  --max-items 20 \
  --output json)

echo $RESULT | jq '.NextToken'   # token para a próxima página

aws ec2 describe-instances \
  --profile prod-readonly \
  --page-size 20 \
  --max-items 20 \
  --starting-token "$(echo $RESULT | jq -r '.NextToken')" \
  --output json
```

---

## Armadilhas comuns

**1. `--max-items` sem `--page-size` alinhado — dados faltando silenciosamente**

O erro é usar `--max-items 100` com `--page-size 30` e depois tentar continuar com `--starting-token`. O cursor fica dessincronizado com os limites de página e você perde itens entre as páginas. Solução: use o mesmo valor para ambos, ou use só `--page-size` para listar tudo.

**2. Confundir `~/.aws/credentials` com `~/.aws/config` para perfis SSO**

Perfis SSO **não funcionam** no `~/.aws/credentials`. O `credentials` só entende `aws_access_key_id` e `aws_secret_access_key`. Se você colocar `sso_session` lá, o CLI ignora silenciosamente e cai para o próximo provider na cadeia.

**3. Role chaining com `duration_seconds > 3600`**

Se um perfil usa `source_profile` que aponta para outro perfil (não para credenciais de usuário IAM direto), você está fazendo role chaining. Qualquer `duration_seconds` acima de 3600 vai falhar com `DurationSeconds exceeds the 1 hour session duration limit for roles assumed by role chaining`. O erro não é óbvio se você não sabe que está em chain.

---

## Exercício de reflexão

Você tem três contas AWS: `tools` (onde o IAM Identity Center está), `staging` e `prod`. Um engenheiro configura o seguinte `~/.aws/config`:

```ini
[sso-session empresa]
sso_start_url = https://empresa.awsapps.com/start
sso_region    = us-east-1

[profile staging]
sso_session    = empresa
sso_account_id = 111122223333
sso_role_name  = DevAccess
region         = us-east-1

[profile prod]
role_arn       = arn:aws:iam::444455556666:role/ProdDeploy
source_profile = staging
duration_seconds = 7200
region         = us-east-1
```

Quando ele executa `aws sts get-caller-identity --profile prod`, o comando falha. Por que? Quais são os dois problemas distintos nessa configuração — um de arquitetura, um de limite de serviço — e como você corrigiria cada um? Considere também o impacto na duração da sessão em prod caso a configuração seja corrigida.

---

## Recursos para aprofundar

**Configuração e perfis:**
- [Configuring IAM Identity Center authentication with the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html) — referência primária para o modelo `sso-session`, inclui todos os campos aceitos e exemplos de `aws configure sso` interativo.

**Assume-role e role chaining:**
- [Using an IAM role in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html) — cobre `source_profile`, `credential_source`, MFA e os limites de role chaining com exemplos de config.

**Paginação:**
- [Using the pagination options in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-pagination.html) — documento curto e direto com a distinção entre `--page-size` e `--max-items` e como usar `--starting-token`.

---

