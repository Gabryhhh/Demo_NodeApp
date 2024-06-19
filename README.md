# 🚀 Distribuzione di Applicazioni su AWS ECS con Terraform

<p align="center">
  <img src="https://img.shields.io/badge/Docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white" alt="Docker"/>
  <img src="https://img.shields.io/badge/Terraform-%235835CC.svg?style=for-the-badge&logo=terraform&logoColor=white" alt="Terraform"/>
  <img src="https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white" alt="AWS"/>
  <img src="https://img.shields.io/badge/Git-%23F05033.svg?style=for-the-badge&logo=git&logoColor=white" alt="Git"/>
</p>

## 📚 Prerequisiti

Per seguire questo workshop, è necessario preparare il tuo ambiente con le seguenti dipendenze:

- **Terraform**: Assicurati di avere Terraform installato nel tuo sistema. Puoi scaricare e installare Terraform seguendo le istruzioni dalla [pagina ufficiale di Terraform](https://learn.hashicorp.com/terraform).
- **AWS CLI**: Assicurati di avere AWS CLI installato e configurato con le tue credenziali AWS. Puoi installare AWS CLI seguendo la [documentazione ufficiale](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).
- **Docker**: Docker è utilizzato per creare e gestire i container dei microservizi. Puoi scaricare e installare Docker seguendo le istruzioni per il tuo sistema operativo dalla [documentazione di Docker](https://docs.docker.com/get-docker/).
- **Git**: Git è utilizzato per il versioning del codice e il controllo delle modifiche. Se non hai Git installato, puoi scaricarlo e installarlo dalla [pagina ufficiale di Git](https://git-scm.com/downloads).

## 🎯 Obiettivi del Workshop

Questo workshop è progettato per fornire una comprensione pratica dell'utilizzo di Terraform per gestire l'infrastruttura su AWS, implementare microservizi su un cluster ECS e creare pipeline CI/CD automatizzate. Al termine di questo workshop, gli studenti saranno in grado di:

- Creare e configurare un cluster ECS utilizzando Terraform.
- Implementare microservizi Docker su un cluster ECS.
- Configurare e gestire una pipeline CI/CD per il deployment continuo dei microservizi.
- Implementare strumenti di osservabilità per monitorare e tracciare le applicazioni.

## 🛠️ Passaggi

### 1. Configurazione dell'Ambiente

Clona il repository e naviga nella directory principale:

```bash
git clone https://github.com/tuonome/repository.git
cd repository
```

### 2. Configurazione dell'Accesso AWS

Configura l'accesso AWS con le tue credenziali:

```bash
aws configure
```

### 3. Configurazione e Deploy di ECS con Terraform

Naviga nella directory `terraform/` e inizializza Terraform:

```bash
cd terraform
terraform init
```

Modifica il file `main.tf` con le tue configurazioni personalizzate per ECS:

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "5.53.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

resource "aws_ecs_cluster" "ecs_cluster" {
  name = "${var.owner}-my-ecs-cluster"
}

resource "aws_ecs_task_definition" "bikes_task_definition" {
  family                   = "bikes"
  container_definitions    = file("${path.module}/bikes-deployment.yaml")
}

resource "aws_ecs_service" "bikes_service" {
  name            = "bikes-service"
  cluster         = aws_ecs_cluster.ecs_cluster.id
  task_definition = aws_ecs_task_definition.bikes_task_definition.arn
  desired_count   = 2
}

resource "aws_ecs_task_definition" "cars_task_definition" {
  family                   = "cars"
  container_definitions    = file("${path.module}/cars-deployment.yaml")
}

resource "aws_ecs_service" "cars_service" {
  name            = "cars-service"
  cluster         = aws_ecs_cluster.ecs_cluster.id
  task_definition = aws_ecs_task_definition.cars_task_definition.arn
  desired_count   = 2
}

resource "aws_iam_role" "ecs_service_role" {
  name               = "${var.owner}-ecs-service-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = {
        Service = "ecs.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_service_policy" {
  role       = aws_iam_role.ecs_service_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceRole"
}

resource "aws_lb" "application_load_balancer" {
  name               = "my-application-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = var.public_subnet_ids
}

resource "aws_security_group" "alb_sg" {
  name        = "alb_sg"
  description = "Security group for ALB"
  vpc_id      = var.vpc_id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_lb_listener" "alb_listener" {
  load_balancer_arn = aws_lb.application_load_balancer.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "fixed-response"
    status_code      = "200"
    content_type     = "text/plain"
    message_body     = "OK"
  }
}

resource "aws_iam_role" "alb_ingress_controller_role" {
  name               = "alb-ingress-controller-role"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}

resource "aws_iam_policy" "alb_ingress_controller_policy" {
  name = "alb-ingress-controller-policy"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "ec2:DescribeAccountAttributes",
        "ec2:DescribeAddresses",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeRouteTables",
        "ec2:DescribeLoadBalancers",
        "ec2:DescribeTags",
        "ec2:GetCoipPoolUsage",
        "ec2:DescribeCoipPools",
        "ec2:DescribeNatGateways",
        "ec2:DescribeEgressOnlyInternetGateways",
        "ec2:DescribePrefixLists",
        "ec2:DescribeAddresses",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeRouteTables",
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeRouteTables"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "alb_ingress_controller_policy_attachment" {
  policy_arn = aws_iam_policy.alb_ingress_controller_policy.arn
  role      

 = aws_iam_role.alb_ingress_controller_role.name
}

resource "helm_release" "alb_ingress_controller" {
  name       = "alb-ingress-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.3.1"
  values = [
    {
      "clusterName"                    = "${var.owner}-my-ecs-cluster"
      "serviceAccount.create"          = "false"
      "serviceAccount.annotations"     = "eks.amazonaws.com/role-arn: ${aws_iam_role.alb_ingress_controller_role.arn}"
      "serviceAccount.name"            = "aws-load-balancer-controller"
      "image.repository"               = "602401143452.dkr.ecr.us-west-2.amazonaws.com/amazon/aws-load-balancer-controller:v2.2.0"
      "image.tag"                      = "v2.2.0"
      "region"                         = "us-west-2"
      "vpcId"                          = "${var.vpc_id}"
      "subnetIds"                      = "${var.public_subnet_ids}"
      "serviceAccount.externalPolicy"  = "true"
    }
  ]
}
```

### 4. Creazione e Deploy dei Microservizi Docker

Naviga nella directory `docker/` e crea le immagini Docker dei microservizi:

```bash
cd ../docker
docker build -t nome_microservizio .
```

Dopo aver costruito le immagini, esegui il push sul registro Docker di AWS ECR:

```bash
docker tag nome_microservizio:latest aws_account_id.dkr.ecr.region.amazonaws.com/repository:tag
docker push aws_account_id.dkr.ecr.region.amazonaws.com/repository:tag
```

### 5. Configurazione della Pipeline CI/CD

Configura una pipeline CI/CD utilizzando strumenti come AWS CodePipeline e AWS CodeBuild per il deployment continuo dei microservizi su ECS.

### 6. Implementazione degli Strumenti di Osservabilità

Implementa strumenti come Amazon CloudWatch per il monitoraggio e la gestione delle metriche delle applicazioni su ECS.

### 7. Verifica del Deployment

Per verificare che il deployment sia andato a buon fine, puoi accedere alla console AWS ECS e verificare lo stato dei tuoi servizi e dei tuoi task.

### 8. Distruggi tutte le risorse create

Una volta completato il workshop, puoi distruggere tutte le risorse create eseguendo:

```bash
terraform destroy
```

### 9. Aggiungi un Application Load Balancer (ALB)

#### Step 1: Creazione del ruolo IAM per il controller del load balancer

Sostituisci `<tuo-nome>` con il nome del tuo account IAM e `<id-del-tuo-account>` con l'ID del tuo account AWS, quindi esegui il comando:

```bash
eksctl create iamserviceaccount \
  --cluster=${var.owner}-my-ecs-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name ${var.owner}AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::${var.aws_account_id}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

#### Step 2: Installazione dell'AWS Load Balancer Controller utilizzando Helm

Installa Helm eseguendo i seguenti comandi:

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

Aggiungi il repository dei chart Helm di eks:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Infine, installa l'AWS Load Balancer Controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=${var.owner}-my-ecs-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.annotations=eks.amazonaws.com/role-arn: ${aws_iam_role.alb_ingress_controller_role.arn} \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.repository=602401143452.dkr.ecr.us-west-2.amazonaws.com/amazon/aws-load-balancer-controller \
  --set image.tag=v2.2.0 \
  --set region=us-west-2 \
  --set vpcId=${var.vpc_id} \
  --set subnetIds=${var.public_subnet_ids} \
  --set serviceAccount.externalPolicy=true
```

---

## 📚 Risorse Aggiuntive

- [Documentazione AWS ECS](https://docs.aws.amazon.com/ecs/index.html)
- [Documentazione Terraform](https://learn.hashicorp.com/terraform)

## 🤝 Contributi

Siete invitati a contribuire migliorando questo workshop con nuove funzionalità, correzioni di bug o suggerimenti. Create una pull request e sarò felice di esaminarla!

-R3cube Team
