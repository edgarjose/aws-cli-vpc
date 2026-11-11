# Criar uma VPC em duas zonas com AWS CLI

Laboratório de rede com duas sub-redes públicas e duas privadas. Usa um NAT Gateway numa das zonas para reduzir a dimensão do exercício; **não oferece saída resiliente a falhas de zona**. O ficheiro `vpc commands` é uma transcrição histórica com IDs da execução original, não um script reutilizável.

## Pré-requisitos

AWS CLI v2, Bash, credenciais numa conta de teste e permissões para gerir VPC, sub-redes, rotas, Internet Gateway, NAT e Elastic IPs. Confirmar que os CIDRs não colidem com redes que pretendas ligar à VPC.

## 1. Selecionar a região e criar a rede

Executar os blocos pela ordem indicada, na mesma sessão Bash. Guardar os IDs devolvidos para limpeza.

```bash
set -euo pipefail
export AWS_DEFAULT_REGION=us-east-1
aws sts get-caller-identity

VPC_ID="$(aws ec2 create-vpc --cidr-block 10.20.0.0/16 --query Vpc.VpcId --output text)"
aws ec2 create-tags --resources "$VPC_ID" --tags Key=Name,Value=practical-vpc-lab
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support

AZ_A="$(aws ec2 describe-availability-zones --filters Name=state,Values=available --query 'AvailabilityZones[0].ZoneName' --output text)"
AZ_B="$(aws ec2 describe-availability-zones --filters Name=state,Values=available --query 'AvailabilityZones[1].ZoneName' --output text)"
PUBLIC_A="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.20.1.0/24 --availability-zone "$AZ_A" --query Subnet.SubnetId --output text)"
PUBLIC_B="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.20.2.0/24 --availability-zone "$AZ_B" --query Subnet.SubnetId --output text)"
PRIVATE_A="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.20.11.0/24 --availability-zone "$AZ_A" --query Subnet.SubnetId --output text)"
PRIVATE_B="$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.20.12.0/24 --availability-zone "$AZ_B" --query Subnet.SubnetId --output text)"
```

São necessárias pelo menos duas zonas disponíveis. Os CIDRs não se sobrepõem. A atribuição automática de IP público não é ativada nas sub-redes privadas.

## 2. Saída pública

```bash
IGW_ID="$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)"
aws ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"
PUBLIC_RT="$(aws ec2 create-route-table --vpc-id "$VPC_ID" --query RouteTable.RouteTableId --output text)"
aws ec2 create-route --route-table-id "$PUBLIC_RT" --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"
PUBLIC_ASSOC_A="$(aws ec2 associate-route-table --route-table-id "$PUBLIC_RT" --subnet-id "$PUBLIC_A" --query AssociationId --output text)"
PUBLIC_ASSOC_B="$(aws ec2 associate-route-table --route-table-id "$PUBLIC_RT" --subnet-id "$PUBLIC_B" --query AssociationId --output text)"
```

## 3. NAT e rotas privadas

```bash
EIP_ALLOC="$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)"
NAT_ID="$(aws ec2 create-nat-gateway --subnet-id "$PUBLIC_A" --allocation-id "$EIP_ALLOC" --query NatGateway.NatGatewayId --output text)"
aws ec2 wait nat-gateway-available --nat-gateway-ids "$NAT_ID"
PRIVATE_RT="$(aws ec2 create-route-table --vpc-id "$VPC_ID" --query RouteTable.RouteTableId --output text)"
aws ec2 create-route --route-table-id "$PRIVATE_RT" --destination-cidr-block 0.0.0.0/0 --nat-gateway-id "$NAT_ID"
PRIVATE_ASSOC_A="$(aws ec2 associate-route-table --route-table-id "$PRIVATE_RT" --subnet-id "$PRIVATE_A" --query AssociationId --output text)"
PRIVATE_ASSOC_B="$(aws ec2 associate-route-table --route-table-id "$PRIVATE_RT" --subnet-id "$PRIVATE_B" --query AssociationId --output text)"
aws ec2 describe-route-tables --route-table-ids "$PUBLIC_RT" "$PRIVATE_RT"
```

Um NAT usa `--nat-gateway-id`; `--gateway-id` identifica um Internet Gateway ou Virtual Private Gateway. O NAT fica na sub-rede pública e depende da rota pública para o IGW.

## 4. Verificar e experimentar

Confirmar as quatro associações de route tables e o NAT em estado available. As duas sub-redes privadas partilham a mesma saída por NAT; tráfego da segunda zona pode gerar transferência entre zonas.

Este guia não lança instâncias por defeito. Para experimentar com uma EC2, usar uma AMI atual da região, o security group correto e um método administrativo restrito. Nunca abrir SSH a `0.0.0.0/0`. Se criares um key pair, o campo de saída é `KeyMaterial` e o ficheiro privado não deve entrar no Git.

## 5. Limpar os recursos do exercício

Terminar primeiro qualquer instância ou interface que tenhas criado nestas sub-redes. Na mesma sessão com os IDs do lab, executar apenas depois de os confirmares:

```bash
aws ec2 delete-nat-gateway --nat-gateway-id "$NAT_ID"
aws ec2 wait nat-gateway-deleted --nat-gateway-ids "$NAT_ID"
aws ec2 release-address --allocation-id "$EIP_ALLOC"
aws ec2 disassociate-route-table --association-id "$PRIVATE_ASSOC_A"
aws ec2 disassociate-route-table --association-id "$PRIVATE_ASSOC_B"
aws ec2 disassociate-route-table --association-id "$PUBLIC_ASSOC_A"
aws ec2 disassociate-route-table --association-id "$PUBLIC_ASSOC_B"
aws ec2 delete-subnet --subnet-id "$PRIVATE_A"
aws ec2 delete-subnet --subnet-id "$PRIVATE_B"
aws ec2 delete-subnet --subnet-id "$PUBLIC_A"
aws ec2 delete-subnet --subnet-id "$PUBLIC_B"
aws ec2 delete-route-table --route-table-id "$PRIVATE_RT"
aws ec2 delete-route-table --route-table-id "$PUBLIC_RT"
aws ec2 detach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"
aws ec2 delete-internet-gateway --internet-gateway-id "$IGW_ID"
aws ec2 delete-vpc --vpc-id "$VPC_ID"
```

Se a sessão tiver sido fechada, recuperar os IDs na consola antes da remoção; não substituir por IDs de outros ambientes. NAT, Elastic IPs e eventuais EC2 geram custos enquanto existirem.

[Guia AWS de NAT Gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) · [Versão com CloudFormation](https://github.com/edgarjose/aws-cloudformation-cross-reference)
