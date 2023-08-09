# Arquiteura-TrabalhoFinal-Compass
## Descrição da atividade:
### Escopo:
Escopo da atividade; o que vamos fazer no projeto; as atividades que serão executadas;
Arquitetura da nova solução:
Indicar a arquitetura atual do cliente; mostrar a nova arquitetura, com todo o ambiente planejado: 
Requisitos para o ambiente:
- Kubernetes
- Banco de dados PaaS 
- Multi-AZ
- Segurança de backup de dados 
- Resiliência de dados
- Balanceamento de carga com Health Check
- Segurança – liberar o mínimo possível
(não deixar portas abertas desnecessariamente, dos pods, serviços, instancias...)
*A nossa atuação nesse trabalho é como a de um arquiteto de soluções
🡪 AWS Calculator – tirar print e pegar o link
🡪 Versionamento no Github – simples; somente descrição textual da arquitetura
---
## Ilustração da arquitetura final 
![image](https://github.com/JuFick/Arquiteura-TrabalhoFinal-Compass/assets/132408071/faa3e58b-33b9-439c-8d63-bbe2eeded846)

## Descrição geral da arquitetura: 
O acesso à aplicação é configurado no domínio do site no **Rota 53, o serviço de DNS da AWS**. O tráfego de entrada passa por um **WAF – Web Application Firewall**, que é uma camada a mais de segurança para o ambiente. O tráfego é direcionado para um **ALB – Application Load Balancer**, que o distribui entre as máquinas do cluster Kubernetes. O cluster Kubernetes é implementado com o serviço de **EKS – Elastic Kubernetes Service**, o qual gerencia o cluster e contém um **AutoScaling** para as máquinas. As máquinas do cluster rodam a aplicação. Elas se conectam ao banco de dados da aplicação, que é uma instância **Amazon RDS**. Os snapshots de backup do RDS são armazenados num **Bucket S3 Glacier Instant Retrieval**. Para comunicação externa, as máquinas do cluster contam com o **NAT Gateway**, o elemento que possibilita conexão externa para elas. O ambiente ainda oferece um **EFS – Elastic File System**, o serviço de NFS da Amazon, que é montado nas máquinas armazenando arquivos compartilhados da aplicação. Os estáticos do site são armazenados num **Bucket S3 Standard**.

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

#### RDS
*Multi-AZ 🡺 é mantida uma cópia do banco de dados em outra AZ, é uma outra instância que fica em standby. Quando a instância primária falha, a segunda entra automaticamente em ação, com o mesmo endpoint da primeira.
Burst capability (general purpose SSD – familia t) 🡺 quando o banco de dados está funcionando abaixo do limite, ele acumula créditos, os quais são usados em casos de picos de uso, evitando que se bata no limite e que se precise provisionar mais capacidade.
Backup
O RDS tem dois tipos de backup:
Backups automáticos: com essa configuração é possível restaurar o banco de dados a um “point-in-time” definível de 1 a 35 dias.
Snapshots manuais: são armazenados no Amazon S3; duram lá até que sejam deletados manualmente; o banco de dados pode ser restaurado a partir de um snapshot; é armazenado com redundância geográfica. (custo mais baixo)
Escalabilidade:
É possível escalar a capacidade do RDS sem fazer reboot; Elasticache;
Com o RDS é possível criar Read Replicas do banco de dados: são replicas criadas com snapshots do banco em diferentes regiões, para diminuir risco de desastre e também para distribuir o tráfego de leitura.
Failover de Azs duram em média 90 segundos.*

