# Sectigo AWSCM — IaC

Módulo Terraform para criar um conector AWS que utiliza uma função Lambda para emitir e renovar certificados SSL/TLS da Sectigo via protocolo ACME, importando-os automaticamente no **AWS Certificate Manager (ACM)**.

---

## Sumário

- [Visão Geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Estrutura do Repositório](#estrutura-do-repositório)
- [Configuração Inicial](#configuração-inicial)
- [Instalação Passo a Passo](#instalação-passo-a-passo)
- [Uso da API](#uso-da-api)
- [Variáveis Terraform](#variáveis-terraform)
- [Outputs](#outputs)
- [Destruição da Infraestrutura](#destruição-da-infraestrutura)
- [Módulos](#módulos)

---

## Visão Geral

Esta solução provisiona automaticamente uma infraestrutura AWS completa para gerenciar o ciclo de vida de certificados SSL/TLS emitidos pela Sectigo via protocolo **ACME**. O fluxo principal é:

1. Uma chamada REST API dispara o processo via **API Gateway**
2. O API Gateway invoca uma **função Lambda** (Python 3.9)
3. A Lambda lê as credenciais ACME de um **bucket S3**
4. Utiliza o cliente **Certbot** para se comunicar com o endpoint ACME da Sectigo
5. Registra o status da requisição em uma tabela **DynamoDB**
6. Importa o certificado emitido no **AWS Certificate Manager (ACM)**

---

## Arquitetura

```
Usuário
  │
  ▼
API Gateway (REST — REGIONAL)
  │  Autenticação: x-api-key
  │  Endpoint: GET /SectigoAWSCM/queries
  │  Parâmetros: domains, account, action, arn
  ▼
Lambda Function (Python 3.9 — timeout: 600s)
  │
  ├──► S3 Bucket ──────────── lê acme_accounts.yaml
  │
  ├──► DynamoDB ──────────── registra status da requisição
  │
  ├──► Sectigo ACME Endpoint — emite/renova certificado
  │    https://acme.enterprise.sectigo.com
  │
  └──► AWS ACM ─────────────── importa o certificado emitido

CloudWatch Logs (retenção: 14 dias)
```

**Recursos provisionados:**

| Recurso | Nome padrão |
|---|---|
| S3 Bucket (contas ACME) | `sectigo-awscm-ca` |
| IAM Role | `SectigoAWSCM-lambda-role` |
| IAM Policy | `SectigoAWSCM-lambda-policy` |
| Lambda Function | `SectigoAWSCM` |
| DynamoDB Table | `sectigoAWSCM` |
| API Gateway | REST API + Stage + API Key |
| CloudWatch Log Group | `/aws/lambda/SectigoAWSCM` |

---

## Pré-requisitos

- [Terraform](https://www.terraform.io/downloads) >= 1.0 **ou** [OpenTofu](https://opentofu.org/) >= 1.6
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) configurado com credenciais válidas
- Credenciais de acesso à AWS com permissões para criar: IAM, Lambda, S3, DynamoDB, API Gateway, ACM e CloudWatch
- Conta ativa na plataforma **Sectigo Enterprise** com acesso ao endpoint ACME
- **EAB Key** e **EAB HMAC Key** fornecidos pela Sectigo

---

## Estrutura do Repositório

```
sectigo-awscm-iac/
├── main.tf                    # Configuração raiz — backend, provider e módulos
├── variables.tf               # Variáveis de entrada
├── output.tf                  # Outputs da infraestrutura
├── acme_accounts.yaml         # Credenciais ACME da Sectigo (enviado ao S3)
├── install.sh                 # Script de instalação automatizada
├── destroy.sh                 # Script de destruição da infraestrutura
├── files/
│   ├── swagger.json           # Especificação OpenAPI do API Gateway
│   ├── lambda-policy.json     # Política IAM da Lambda (template)
│   └── sectigoAWSCM.zip       # Código-fonte da Lambda (Python)
└── modules/
    ├── s3/                    # Módulo: bucket S3 privado com criptografia
    ├── iam/                   # Módulo: role e policy IAM
    ├── lambda/                # Módulo: função Lambda + CloudWatch Logs
    ├── dynamodb/              # Módulo: tabela DynamoDB
    └── api-gateway/           # Módulo: API Gateway REST + API Key
```

---

## Configuração Inicial

### 1. Clonar o repositório

```bash
git clone https://github.com/Alan-italiano/sectigo-awscm-iac.git
cd sectigo-awscm-iac
```

### 2. Configurar as credenciais ACME

Edite o arquivo `acme_accounts.yaml` com as credenciais fornecidas pela Sectigo:

```yaml
accounts:
  aws:
    - acme-endpoint: https://acme.enterprise.sectigo.com
      eab-hmac-key: SEU_HMAC_KEY_AQUI
      eab-key: SEU_EAB_KEY_AQUI
      email: seu-email@empresa.com
      RenewBeforeDays: 29
      KeyType: RSA
      KeySize: 2048
```

| Campo | Descrição |
|---|---|
| `acme-endpoint` | URL do endpoint ACME da Sectigo Enterprise |
| `eab-hmac-key` | Chave HMAC para External Account Binding (EAB) |
| `eab-key` | Chave EAB fornecida pela Sectigo |
| `email` | E-mail do responsável pela conta |
| `RenewBeforeDays` | Dias antes do vencimento para renovar (padrão: 29) |
| `KeyType` | Tipo de chave (`RSA` ou `EC`) |
| `KeySize` | Tamanho da chave em bits (padrão: 2048) |

### 3. Configurar as credenciais AWS

```bash
export AWS_ACCESS_KEY_ID="sua-access-key"
export AWS_SECRET_ACCESS_KEY="seu-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

---

## Instalação Passo a Passo

### Opção A — Script automatizado (recomendado)

O script `install.sh` realiza toda a instalação de forma automatizada:

```bash
chmod +x install.sh
./install.sh
```

**O que o script faz:**

1. Valida as credenciais AWS (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`)
2. Verifica se a região informada está disponível
3. Cria o bucket S3 para o estado do Terraform (ex.: `sectigo-tf-state-{timestamp}`)
4. Cria o bucket S3 para as contas ACME (ex.: `sectigo-aws-cm-{workspace}-{timestamp}`)
5. Inicializa o Terraform/OpenTofu (`terraform init`)
6. Seleciona ou cria o workspace correspondente à região
7. Executa `terraform plan` e `terraform apply` com aprovação automática
8. Salva os outputs em `install-{workspace}.log`
9. Faz o upload de `acme_accounts.yaml` para o bucket S3
10. Em caso de falha: destrói os recursos criados e encerra

Ao final, o log gerado contém a **URL da API**, a **API Key** e um exemplo de chamada `curl`.

---

### Opção B — Instalação manual

**Passo 1 — Inicializar o backend Terraform**

Crie manualmente o bucket S3 para o estado remoto:

```bash
aws s3api create-bucket \
  --bucket sectigo-aws-cm-tf-states \
  --region us-east-1
```

**Passo 2 — Inicializar o Terraform**

```bash
terraform init
```

**Passo 3 — Selecionar o workspace (região)**

```bash
terraform workspace new us-east-1
# ou, se já existir:
terraform workspace select us-east-1
```

**Passo 4 — Criar o arquivo de variáveis**

Crie um arquivo `terraform.tfvars`:

```hcl
sectigo_bucket     = "sectigo-awscm-ca"
region_name        = "us-east-1"
function_name      = "SectigoAWSCM"
lambda_policy_name = "SectigoAWSCM-lambda-policy"
lambda_role_name   = "SectigoAWSCM-lambda-role"
table_name         = "sectigoAWSCM"
```

**Passo 5 — Planejar e aplicar**

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

**Passo 6 — Fazer upload do arquivo ACME**

```bash
aws s3 cp acme_accounts.yaml s3://$(terraform output -raw s3_bucket_for_accounts)/
```

---

## Uso da API

Após a instalação, utilize o output `invoke-url` para chamar a API:

```bash
curl -X GET \
  "https://{api-id}.execute-api.{region}.amazonaws.com/v1/SectigoAWSCM/queries?action=enroll&domains=exemplo.com.br&account=aws" \
  -H "x-api-key: {sua-api-key}"
```

### Parâmetros da API

| Parâmetro | Obrigatório | Descrição |
|---|---|---|
| `action` | Sim | Ação a executar: `enroll`, `renew`, `list`, etc. |
| `domains` | Não | Domínio(s) para o certificado (ex.: `exemplo.com,*.exemplo.com`) |
| `account` | Não | Nome da conta ACME configurada no YAML (ex.: `aws`) |
| `arn` | Não | ARN do certificado existente no ACM (para renovação) |

### Exemplos de uso

**Emitir um novo certificado:**
```bash
curl -X GET \
  "{invoke-url}?action=enroll&domains=meusite.com.br&account=aws" \
  -H "x-api-key: {api-key}"
```

**Renovar um certificado existente:**
```bash
curl -X GET \
  "{invoke-url}?action=renew&arn=arn:aws:acm:us-east-1:123456789:certificate/xxxxx&account=aws" \
  -H "x-api-key: {api-key}"
```

**Listar certificados:**
```bash
curl -X GET \
  "{invoke-url}?action=list&account=aws" \
  -H "x-api-key: {api-key}"
```

**Invocar diretamente via AWS CLI:**
```bash
aws lambda invoke \
  --function-name SectigoAWSCM \
  --payload '{"action":"list","account":"aws"}' \
  --cli-binary-format raw-in-base64-out \
  response.json
```

---

## Variáveis Terraform

| Variável | Descrição | Padrão |
|---|---|---|
| `sectigo_bucket` | Nome do bucket S3 para contas ACME | `sectigo-awscm-ca` |
| `region_name` | Região AWS | `us-east-1` |
| `function_name` | Nome da função Lambda | `SectigoAWSCM` |
| `lambda_policy_name` | Nome da policy IAM da Lambda | `SectigoAWSCM-lambda-policy` |
| `lambda_role_name` | Nome da role IAM da Lambda | `SectigoAWSCM-lambda-role` |
| `table_name` | Nome da tabela DynamoDB | `sectigoAWSCM` |

---

## Outputs

Após o `terraform apply`, os seguintes valores são exibidos:

| Output | Descrição |
|---|---|
| `invoke-url` | URL completa da API com parâmetros de exemplo |
| `x-api-key` | API Key para autenticação no API Gateway |
| `invoke-command` | Exemplo de comando `curl` pronto para uso |
| `lambda_invoke_command_with_cli` | Comando AWS CLI para invocar a Lambda diretamente |
| `aws_account_id` | ID da conta AWS |
| `lambda_function_name` | Nome da função Lambda criada |
| `s3_bucket_for_accounts` | Nome do bucket S3 para as contas ACME |
| `dynamodb_table_name` | Nome da tabela DynamoDB |

---

## Destruição da Infraestrutura

Para remover todos os recursos provisionados:

### Opção A — Script automatizado

```bash
chmod +x destroy.sh
./destroy.sh
```

O script irá:
1. Validar as credenciais AWS
2. Selecionar o workspace correto
3. Esvaziar e excluir os buckets S3 correspondentes
4. Executar `terraform apply -destroy` com aprovação automática
5. Limpar arquivos locais de log e `.tfvars`

### Opção B — Manual

```bash
# Esvaziar o bucket S3
aws s3 rm s3://{nome-do-bucket} --recursive

# Destruir a infraestrutura
terraform destroy
```

---

## Módulos

### `modules/s3`
Provisiona um bucket S3 privado com:
- Criptografia server-side (AES256)
- Bloqueio total de acesso público

### `modules/iam`
Cria a role e a policy IAM para a Lambda com permissões mínimas para:
- ACM (listar, descrever, importar certificados)
- S3 (leitura do bucket de contas)
- Lambda (auto-invocação)
- DynamoDB (operações na tabela)

### `modules/lambda`
Provisiona a função Lambda com:
- Runtime: Python 3.9
- Timeout: 600 segundos
- Log group no CloudWatch (retenção: 14 dias)
- Variáveis de ambiente: `s3_bucket_yaml`, `lambda_name`, `table_name`

### `modules/dynamodb`
Cria a tabela DynamoDB para rastreamento de requisições:
- Modo: PROVISIONED (5 RCU / 5 WCU)
- Chave primária: `RequestParameters` (String)
- Chave de ordenação: `RequestStatus` (String)

### `modules/api-gateway`
Provisiona a REST API com:
- Endpoint regional
- Autenticação por API Key (`x-api-key`)
- Validação de parâmetros de query
- Integração Lambda via template de mapeamento
- Usage Plan associado à API Key

---

## Multi-região com Workspaces

A infraestrutura suporta implantação em múltiplas regiões AWS através de **Terraform Workspaces**. Cada workspace corresponde a uma região:

```bash
# Criar implantação em outra região
export AWS_DEFAULT_REGION="sa-east-1"
terraform workspace new sa-east-1
terraform apply
```

Todos os recursos são nomeados com o sufixo do workspace para evitar conflitos.

---

## Licença

Distribuído para uso interno. Consulte o responsável pelo repositório para mais informações.
