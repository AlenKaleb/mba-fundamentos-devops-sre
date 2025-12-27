# Exportação anotada — `fullcycle-infra-main`

Este documento exporta e comenta tecnicamente o código do diretório
`fullcycle-infra-main/fullcycle-infra-main`, explicando arquitetura,
conceitos, padrões e boas práticas relevantes.

## Visão geral da arquitetura

- **Propósito**: provisionar infraestrutura AWS com Terraform.
- **Padrão**: uso de **módulos Terraform** para encapsular recursos (neste caso, EC2).
- **Persistência do estado**: backend remoto S3 para armazenar o estado do Terraform
  de forma centralizada e segura.

---

## `main.tf`

### Código exportado

```hcl
module "ec2_instance" {
  source = "terraform-aws-modules/ec2-instance/aws"

  name = "single-instance"

  instance_type          = "t2.micro"
  monitoring             = false
  vpc_security_group_ids = ["sg-06f76a53f08a418d4"]
  subnet_id              = "subnet-0bfb6070cab586136"

  tags = {
    Terraform   = "true"
    Environment = "dev"
    Name        = "Teste pipeline"
  }
}

terraform {
  backend "s3" {
    bucket = "teste-repo-fullcycle"
    key    = "teste"
    region = "sa-east-1"
  }
}
```

### Anotações técnicas

- **`module "ec2_instance"`**
  - Aplica o **padrão de reuso com módulos**. Em vez de declarar recursos diretamente,
    este código consome um módulo oficial da comunidade (`terraform-aws-modules/ec2-instance/aws`).
  - Benefício: reduz boilerplate, traz boas práticas testadas e facilita upgrades.

- **`source = "terraform-aws-modules/ec2-instance/aws"`**
  - Referencia o módulo público no Registry do Terraform. Evita duplicar a lógica de criação
    de uma instância EC2.

- **`instance_type = "t2.micro"`**
  - Define o tipo de instância. `t2.micro` é compatível com o free tier da AWS em muitos casos.

- **`monitoring = false`**
  - Desliga o **detailed monitoring**, reduzindo custos. Em ambientes críticos, recomenda-se
    habilitar para granularidade maior nas métricas.

- **`vpc_security_group_ids` / `subnet_id`**
  - O acoplamento direto a IDs indica **dependência externa** (infra já existente).
  - Boa prática: declarar esses valores via variáveis para facilitar reaproveitamento e
    evitar hardcoding.

- **`tags`**
  - Tagging consistente ajuda em governança, custo e auditoria.
  - `Terraform = true` é um padrão comum para identificar recursos gerenciados por IaC.

- **`terraform { backend "s3" { ... } }`**
  - Usa backend remoto em S3 para estado compartilhado.
  - Benefícios: trabalho em equipe, locking (em conjunto com DynamoDB), versionamento do estado.
  - Boa prática adicional: configurar também o `dynamodb_table` para **state locking**.

---

## `settings.yml`

### Código exportado

```yaml
aws-region: 'sa-east-1'
terraform-version: '1.1.7'
account-id: '137068236554'
```

### Anotações técnicas

- **Configurações externas**
  - Este arquivo centraliza parâmetros de execução (ex.: região, versão do Terraform, account ID).
  - É útil para pipelines de CI/CD ou automações que precisam desses valores.

- **Boas práticas**
  - Evitar duplicação de valores no pipeline.
  - Adicionar validação ou documentação das chaves esperadas quando usado por scripts.

---

## Recomendações gerais e boas práticas

- **Variabilização**: mover IDs de VPC/Subnet e tipo de instância para variáveis (`variables.tf`),
  permitindo reaproveitamento em ambientes (`dev`, `staging`, `prod`).
- **State locking**: adicionar `dynamodb_table` ao backend S3 para evitar conflitos de estado.
- **Separação de ambientes**: usar workspaces ou diretórios separados com `terraform.tfvars`.

