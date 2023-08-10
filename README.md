# Arquiteura-TrabalhoFinal-Compass
## Case:
Nós somos da empresa "Fast Engineering S/A"
e gostaríamos de uma solução dos senhores(as),
que fazem parte da empresa terceira "TI
SOLUÇÕES INCRÍVEIS".
Nosso eCommerce está crescendo e a solução
atual não está atendendo mais a alta demanda de
acessos e compras que estamos tendo.
Desde o Início do ano os acessos e compras
estão crescendo 20% a cada mês.
Atualmente usamos:
* 01 servidor para Banco de Dados Mysql;
* 01 servidor para a aplicação utilizando REACT;
* 01 servidor de web Server e que armazena
estáticos como fotos e links.
Nosso pedido é um ORÇAMENTO com:
** Seguir o modelo de apresentação PPT da
Trilha de Estudo Objetivo Instruções
CompassUOL.
ESCOPO;
ARQUITETURA DA NOVA SOLUÇÃO;
VALORES;
PRAZO DE ENTREGA;
CRONOGRAMA MACRO DE ENTREGAS;
Sobre a construção da arquitetura para o futuro
website da nossa empresa, precisamos seguir as
melhores práticas DevOps.

## Descrição da atividade:
Migração de um ambiente On-premisse para a nuvem AWS.
### Requisitos para o ambiente:
- Kubernetes
- Banco de dados PaaS 
- Multi-AZ
- Segurança de backup de dados 
- Resiliência de dados
- Balanceamento de carga com Health Check
- Segurança – liberar o mínimo possível
## Ilustração da arquitetura atual:
![Captura de tela 2023-08-10 143205](https://github.com/JuFick/Arquiteura-TrabalhoFinal-Compass/assets/132408071/f8bceebb-cabd-4f8a-8fc7-647ee95f79a1)

---

## Ilustração da proposta de solução arquitetura: 
![Captura de tela 2023-08-10 154411](https://github.com/JuFick/Arquiteura-TrabalhoFinal-Compass/assets/132408071/fad54f4b-6bad-4f73-a011-5450c17e9591)


## Descrição geral da arquitetura: 
O acesso à aplicação é configurado no domínio do site no **Rota 53, o serviço de DNS da AWS**. O tráfego de entrada passa por um **WAF – Web Application Firewall**, que é uma camada a mais de segurança para o ambiente. O tráfego é direcionado para um **ALB – Application Load Balancer**, que o distribui entre as máquinas do cluster Kubernetes. O cluster Kubernetes é implementado com o serviço de **EKS – Elastic Kubernetes Service**, o qual gerencia o cluster e contém um **AutoScaling** para as máquinas. As máquinas do cluster rodam a aplicação. Elas se conectam ao banco de dados da aplicação, o **Amazon Aurora RDS**. Os snapshots de backup do RDS são armazenados num **Bucket S3 Glacier Instant Retrieval**. Para comunicação externa, as máquinas do cluster contam com o **NAT Gateway**, o elemento que possibilita conexão externa para elas. O ambiente ainda oferece um **EFS – Elastic File System**, o serviço de NFS da Amazon, que é montado nas máquinas armazenando arquivos compartilhados da aplicação. Os estáticos do site são armazenados num **Bucket S3 Standard**.

### Camada de rede/entrada:
- A resolução de DNS do site é feita com o Rota 53;
- O tráfego é filtrado pelo WAF (Web Application Firewall);
- O ALB (Application Load Balancer) distribui o tráfego entre os nodes do cluster Kubernetes;

### Camada de aplicação:
- O cluster Kubernetes é implementado com o EKS (Elastic Kubernetes Service), que gerencia autonomamente os nodes;
- Os nodes do cluster são instâncias EC2 do tipo M6g.medium;
- O cluster EKS contém um AutoScaling, serviço que garante dimensionamento horizontal flexível. Inicialmente, é definida a quantidade de 4 máquinas (2 em cada AZ);

### Camada de dados:
- O Amazon RDS é o banco de dados da aplicação. Provisionado no modo Multi-AZ;
- O Amazon S3 é usado para armazenamento de estáticos do site e de snapshots de backup do RDS;
- Um EFS (Elastic File System) é utilizado para armazenamento de arquivos compartilhados da aplicação;

### Acesso administrativo:
- O ambiente interno da aplicação pode ser acessado através de um *bastion host, que é uma instância EC2, tipo t2.micro*. Este bastion pode ser usado para fins de acesso ao banco de dados, aos arquivos compartilhados (EFS), a ferramentas de monitoramento… Existe um AutoScaling Group configurado para a instância Bastion, definido para as duas Azs, garantindo alta disponibilidade de acesso administrativo.
- São definidas credenciais de acesso temporário, roles (funções de permissionamento) e demais mecanismos de segurança com os serviços de IAM (Identity Access Management);

### Configuração de rede:
- A aplicação é hospedada na região AWS “sa-east-1” (São Paulo) e dividida entre duas Zonas de Disponibilidade, sa-east-1a e sa-east-1b, garantindo alta disponibilidade.
- A rede interna da aplicação é implementada com uma *VPC* (Virtual Private Network), abrangendo as duas AZs (sa-east-1a e sa-east-1b);
- Dentro da VPC existe: uma subnet pública, com rotas para um Internet Gateway. Ela contém o Bastion Host, o Nat gateway e Load Balancer; uma subnet privada, contendo os nodes do cluster EKS; uma subnet privada, contendo o banco de dados RDS.
- O esquema de subnets é espelhado entre as duas AZs. Trata-se portanto de 6 subnets ao todo.

### Banco de Dados:
- O Amazon Aurora é um banco de dados relacional, desenvolvido pela AWS. Ele roda de maneira serverless(AWS cuida da infraestrutura) ou pode escolher uma instância e configurar manualmente.
- Optamos pela escolha de rodá-lo em uma instância (t3.medium) com 300GB. Também adicionamos uma opção de 300GB de espaço adicional para back-up e 1000GB de exportação de snapshot mensal para o nosso Bucket S3 Glacier IA.
- O serviço divide o volume de um banco de dados em blocos de 10 GB, espalhados por diferentes discos. Cada pedaço é replicado de seis maneiras em três zonas de disponibilidade da AWS (AZs)
- Backup O Amazon Aurora realiza back-ups automatizados que podem ser armazenados em buckets S3 ou o espaço para armazenamento pode ser provisionado pela própria AWS. Também temos a opção das snapshots manuais, que são armazenados no Amazon S3, duram lá até que sejam deletados manualmente; o banco de dados pode ser restaurado a partir de um snapshot, que é armazenado com redundância geográfica(custo mais baixo). Quando o banco de dados está funcionando abaixo do limite, ele acumula créditos, os quais são usados durante picos de uso, evitando que chegue no limite armazenamento e que se precise provisionar mais capacidade. Com o Aurora é possível criar Read Replicas do banco de dados: são replicas criadas com snapshots do banco em diferentes regiões, para diminuir risco de desastre e também para distribuir o tráfego de leitura.

## Especificações técnicas
### Instâncias EC2 Worker Nodes:
m6g.medium (1 vCPU; 4GB de memória RAM)
20GB de volume EBS
### Instância EC2 Bastion Host:
t2.micro (1 vCPU; 1GB de memória RAM)
### Banco de dados RDS Aurora:
Aurora Standard
db.t4g.medium (2 vCPU; 4GB de memória RAM;)
300GB de armazenamento
300GB de backup
1000GB de exportação de snapshots por mês

## Custos finais:
![Captura de tela 2023-08-10 143257](https://github.com/JuFick/Arquiteura-TrabalhoFinal-Compass/assets/132408071/f2f0d165-32d3-4e35-ad13-b34273a2f8cc)

#### Link para consulta dos serviços:
https://calculator.aws/#/estimate?id=b3bbc7e219f7d2acde09c698fe53503eba42d336 
