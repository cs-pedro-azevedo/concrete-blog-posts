# AWS e as aplicações Dockerizadas

Bom, você já estudou sobre arquiteturas e viu que talvez a quebra de grandes aplicações em microserviços seja o caminho para o futuro dos produtos que entregamos. Diminuir complexidade, reduzir pontos de falhas, reaproveitamento da infraestrutura, automação de todo o processo, etc... São vários os motivos que levam a grande parte do setor migrar seus serviços cada vez mais complexos para esse "NOVO CONCEITO" (sim, entre aspas e maiúsculo, porque o conceito não é tão novo assim né?) A ideia não é nova, mas agora temos ferramentas disponíveis para nos ajudar a colocar tudo isso junto e, o mais impressionante de tudo, **FAZER TUDO FUNCIONAR**.

A ideia de utilizar containers para a entrega de microserviços em produção tem sido uma prática muito bem aceita, e esse post visa trazer as soluções da Amazon para o deploy de aplicações dockerizadas. Vamos deployar uma imagem de teste em diferentes serviços e verificar o comportamento das soluções.

## Velha discussão sobre containers
Sim! Containers em produção (e engole esse choro aí!), aqui vamos falar especificamente de **Docker**. 

Geralmente nessa hora começam aquelas discussões sobre rodar container em produção ou não. Você pode olhar o histórico das ferramentas que têm surgido ao longo dos últimos anos e dizer que o Docker talvez não esteja maduro (em relação a tempo) para rodar certos serviços críticos, ok. Você pode vir com aquela velha história de quebrar o sistema porque o container não é totalmente isolado e tem acesso em alguns [namespaces críticos](https://www.youtube.com/watch?v=PivpCKEiQOQ), ok também. Ou ainda ser mais **hardcore** e implicar com o empilhamento de serviço por versão ser um grande erro, **OK**, já entendi seu ponto (e eu também pensava assim a um tempo atrás). Mas não leve a mal o que eu vou dizer agora. Em muitos lugares nos chamam de **arquitetos de sistemas** e se a gente não aprender a lidar com o "material" que é dado para trabalharmos, vamos ter que começar a nos especializar em outra coisa que não seja __tecnologia__.

Se ainda pensarmos em questões como portabilidade, reproducibilidade e até mesmo escalabilidade containers são sim o grande nome do momento. Então não tenha receio de usar! Administro atualmente algumas aplicações (sim, não são todas) que rodam em containers e em produção e sem dor de cabeça.

##AWS

Iniciei alguns projetos no passado que tinham a necessidade de ser deployados em containers. A aplicação já estava toda embarcada em imagens Docker (exatamente o que os desenvolvedores rodavam localmente) e precisávamos somente shippar nossos containers para os nossos diferentes ambientes. Mas daí vinha uma decisão difícil a ser tomada: qual ferramenta utilizar para colocar nossos containers para rodar. Só na AWS temos três ferramentas diferentes para deployar nossos containers, são elas: usando instâncias EC2 configuradas como Docker Hosts, o Elastic Beanstalk que serve para deployar diversas plataformas de serviço e por último, mas não menos importante, o ECS, que é o serviço especializado da Amazon para rodar containers.

### 1. EC2 - Docker Host
O Docker como plataforma para execução de aplicações é uma ferramenta realmente incrível, tudo que precisamos é de um sistema com kernel Linux e voilá. Algumas poucas abstrações em cima e temos um "Sistema Operacional inteiro", stacks e suas dependências resolvidas para suportar a aplicação que tanto desejamos ver rodando. Levando em consideração que só precisamos de um SO preparado, podemos sim adotar a utilização de uma VM (ou instância EC2) rodando o Docker Engine para realizarmos nossos deploys. Ousaria inclusive dizer que é uma forma um pouco mais simples de usar tudo o que a ferramenta tem para dar, mas ao mesmo tempo muito poderosa pois temos mais acessos e a chance de customizar mais conforme precisamos, e isso é importantíssimo.

#### E a automação?
Daí vem a pergunta: é possível automatizar as coisas desse jeito? E a resposta categórica é SIM, muito. Com o sistema aberto temos várias formas de automação, desde a máquina em que o Docker Engine é instalado até a forma como nos conectamos a esta engine para fazer subir o container com a nossa aplicação.

Várias são as abordagens que podemos utilizar quanto à instalação do Docker e as configurações que podemos fazer para disponibilizá-lo. A ideia aqui não é apresentar um guia para instalação do Docker simplesmente, se você precisa fazer a instalação em uma máquina Linux você pode seguir [esse tutorial aqui](https://docs.docker.com/engine/installation/linux/), que é bem explicativo. Eu particularmente gosto de gerenciar todas as configurações necessárias a partir de uma ferramenta como Puppet ou Ansible, mas também é possível que você faça todas as configurações em uma AMI customizada e ajuste as configs via **user data**. De novo, são várias as abordagens possíveis.

Mas como nós da Concrete gostamos de processos cada vez mais automatizados, abaixo segue um trecho de um playbook Ansible para criar uma nova instância EC2 e instalar o Docker e suas dependências para que possamos adotar nos nossos processos automatizados de provisionamento.

```yaml
---

- name: Cria instância EC2 com CentOS e Docker
hosts: localhost
connection: local
gather_facts: false
tasks:

- name: Cria EC2 Docker Host
ec2:
region: <REGION>
key_name: <MY_KEY>
instance_type: <INSTANCE_TYPE>
image: ami-fd0197e0
register: ec2

- name: Instala e configura o Docker
hosts: <INSTANCE_NAME>
remote_user: centos
sudo: yes
tasks:
- name: Cria arquivo do repo com base em arquivo local
copy: src=./docker.repo dest=/etc/yum.repos.d/docker.repo mode=0644

- name: Instala docker-engine via Yum
yum: name=docker-engine state=present

- name: Disabilita o SELinux
selinux: state=disabled

- name: Inicializa e Habilita o Docker na inicialização
service: name=docker state=started enabled=yes

```

Outro aspecto interessante sobre o deploy de containers em ambientes nos quais temos acesso à engine do Docker é a chance de nos conectarmos remotamente a ela e iniciar a execução de containers, por exemplo. Diretamente do client para o host ou via chamada de API:

```shell
$ docker -H tcp://192.168.2.100:5678 run -d pedrocesarti/node-project-sample
```
ou
```shell
$ curl -XPOST http://192.168.2.100:5678/images/create?fromImage=pedrocesarti/node-project-sample
```

Como dito anteriormente, essa é com certeza a que permite o maior número de customizações. Se você não quiser fazer deploy via client diretamente, você pode usar a API da engine e configurar a sua aplicação para realizar todas as operações via API REST (conforme mostrado acima também) ou ainda cadastrar todos os seus docker hosts em um docker-machine e disparar comandos em inúmeros dockers ao mesmo tempo. 

### 2. Elastic Beanstalk - Deploy Docker
A segunda abordagem possível é utilizarmos o serviço de PaaS da AWS, o Elastic Beanstalk. A grande facilidade por trás de um PaaS é permitir o upload de seu código "as is" e ele resolver todos aqueles requisitos necessários pela plataforma na qual o deploy acontecerá.

O Beanstalk permite o deploy em diversas plataformas, como aplicações rodando Java em Tomcat, NodeJS, PHP, Python, Ruby e é claro containers Docker \o/.

<p align="center"><img src="https://dl.dropboxusercontent.com/s/g9ij8rj0vztc4jy/Screen%20Shot%202016-09-07%20at%2011.49.31%20AM.png?dl=0"Beanstalk Docker"></p>

No caso do deploy de containers Docker, temos a opção de realizar deploy de um container único ou múltiplos containers no caso de uma stack de aplicações que desejamos colocar em execução em conjunto. Tudo que precisamos é criar um zip da nossa aplicação (incluindo Dockerfile e um Dockerrun.aws.json). 

Se já tivermos a nossa aplicação embarcada em uma imagem Docker em algum Registry, podemos só atualizar o nosso arquivo Dockerrun.aws.json realizando o upload do arquivo atualizado. Assim, a aplicação é atualizada pelo PaaS e em alguns momentos é disponibilizada para acesso.

```json
{
    "AWSEBDockerrunVersion": "1",
        "Image": {
            "Name": "pedrocesarti/node-project-sample",
            "Update": "true"
        },
        "Ports": [
        {
            "ContainerPort": "5000"
        }
    ],
        "Logging": "/var/log/nginx"
}
```

<p align="center"><img src="https://dl.dropboxusercontent.com/s/l4nqyl8fq9ifbxj/Screen%20Shot%202016-09-07%20at%2012.18.31%20PM.png?dl=0"Beanstalk Update"></p>

#### E a automação?
Assim como no caso do Docker host, com o Beanstalk algumas automações também são possíveis. Lembrando que independente da abordagem utilizada (embarcando a aplicação em uma imagem ou fazendo um zip da aplicação inteira), você precisa realizar o upload da sua aplicação e simplesmente realizar um update do status da aplicação que está rodando atualmente.

Neste caso temos como grandes aliadas as ferramentas da AWS como EB CLI, tanto para administrar quanto para permitir diferentes níveis de automação. Abaixo seguem alguns exemplos de comandos usados para criar uma aplicação e atualizar o código rodando para a nossa aplicação entrar em execução.

O comando eb init vai permitir a criação de uma nova aplicação ou então a utilização de um ambiente existente.


```shell
$ eb init

Select a default region
1) us-east-1 : US East (N. Virginia)
2) us-west-1 : US West (N. California)
3) us-west-2 : US West (Oregon)
4) eu-west-1 : EU (Ireland)
5) eu-central-1 : EU (Frankfurt)
(default is 3): 1


Select an application to use
1) node-project-sample
2) [ Create new Application ]
(default is 2): 1
```
Já o comando eb deploy se encarregará de realizar o update de toda aplicação. No nosso caso o upload do arquivo Dockerrun.aws.json para o S3 e deploy.

```shell
$ eb deploy

Creating application version archive "app-160907_130749".
Uploading My First Elastic Beanstalk Application/app-160907_130749.zip to S3. This may take a while.
Upload Complete.
INFO: Environment update is starting.
INFO: Deploying new version to instance(s).
INFO: Successfully pulled pedrocesarti/node-project-sample:latest
INFO: Successfully built aws_beanstalk/staging-app
INFO: Docker container 608cac2c6bae is running aws_beanstalk/current-app.
INFO: Environment health has transitioned from Ok to Info. Application update in progress on 1 instance. 0 out of 1 instance completed (running for 23 seconds).
INFO: New application version was deployed to running EC2 instances.
```
<p align="center"><img src="https://dl.dropboxusercontent.com/s/33x5wv88yutn0dk/Screen%20Shot%202016-09-07%20at%201.14.14%20PM.png?dl=0"Beanstalk Update"></p>

No caso da utilização do Elastic Beanstalk temos toda uma plataforma já pré-configurda, só aguardando o nosso código para ser deployado. Para um ambiente de desenvolvimento, podemos utilizar os comandos do EB CLI junto com os comandos do GIT para que o desenvolvedor consiga facilmente testar seu código. Se ficou interessado, [pode ver aqui](https://ruby.awsblog.com/post/Tx2AK2MFX0QHRIO/Deploying-Ruby-Applications-to-AWS-Elastic-Beanstalk-with-Git) o post com exemplos de deploy de uma aplicação Ruby.

Lembrando que no background dessas operações a AWS toma conta da necessidade de iniciar as instâncias EC2 que forem necessárias, além de tomar conta de outros recursos como load balancers e grupos de autoscaling.

### 3. ECS - Configuração de Services
Agora seguindo aquela máxima que diz "e por último, mas não menos importante", vamos falar do ECS (EC2 Container Service), um serviço razoavelmente novo e que toma conta da administração de instâncias e gerencia a execução de containers. A ideia por trás do ECS é a criação de um ambiente completo com registry interno (ECR), instâncias vinculadas a um cluster (uma espécie de Docker Swarm) e tudo mais que for necessário para gerência e utilização de containers em larga escala.

Conforme dito anteriormente, as instâncias são administradas dentro de um cluster e para adicionar uma máquina a este cluster basta que seja utilizada uma AMI otimizada para uso em ECS além de adicionar dados para entrada da instância dentro do cluster desejado.

#### E a automação?

Mais uma vez vamos falar de automação e relacionar as ferramentas da Amazon para isso. As ferramentas de CLI da AWS facilitam muito a vida de qualquer SysAdmin. Abaixo vou listar alguns comandos para criação do cluster e sua administração. 

Recomendo muito a utilização do Registry interno do seu ECS, mas para o objetivo deste post vamos usar a imagem que eu já venho usando até aqui e está disponível no [DockerHub](https://hub.docker.com/r/pedrocesarti/node-project-sample/).

```shell
$ aws ecs create-cluster --cluster NodeProjectSample
```
Conforme comentado anteriormente, algumas opções podem ser utilizadas para garantir a adição de instâncias a este cluster, mas lembre de usar uma imagem já preparada para rodar o agente do ECS e de adicionar algo semelhante ao script abaixo no user-data.

```shell
#!/bin/bash
echo ECS_CLUSTER=NodeProjectSample >> /etc/ecs/ecs.config
```
Depois de adicionar pelo menos uma instância, nosso cluster estará pronto para iniciar o trabalho.

<p align="center"><img src="https://dl.dropboxusercontent.com/s/krre1spe757zf3m/Screen%20Shot%202016-09-07%20at%206.02.49%20PM.png?dl=0"ECS Cluster"></p>

```shell
$ aws ecs list-container-instances --cluster NodeProjectSample
{
  "containerInstanceArns": [
     "arn:aws:ecs:us-east-1:757391140512:container-instance/a9c3b63f-5014-4815-ad94-51f6446b4d96"
  ]
}
```

Basicamente os clusters criados no ECS podem iniciar containers de duas maneiras: definindo uma task (tarefa isolada) ou definindo um serviço. O primeiro basicamente se preocupa com que o container inicie e realize sua atividade, geralmente morrendo depois disso. Já no caso dos services a ideia é manter os containers rodando e respondendo a longo prazo, por exemplo em um serviço Web. Então, além da definição da task é possível gerenciar instâncias em diferentes zonas de disponibilidade, além de outros serviços para melhora de desempenho e disponibilidade, como load balancers e autoscaling.

Como tudo se baseia na definição de tarefas e criar um serviço não é nada complexo, vamos focar na criação de uma dessas tasks e verificar o nosso container rodando.

Para quem está começando, a CLI ajuda com a geração de uma task de exemplo e você pode simplesmente popular conforme sua necessidade.


```shell
$ aws ecs register-task-definition --generate-cli-skeleton
```

A tarefa para rodar nosso projeto pode ser definida dentro de um arquivo JSON seguindo o modelo gerado pelo último comando. O arquivo foi salvo com o nome node-project-definition.json e foi utilizado como input para o comando de definição da tarefa.

```json
{
    "family": "nodeproject",
    "containerDefinitions": [
        {
            "name": "nodeproject",
            "image": "pedrocesarti/node-project-sample",
            "cpu": 500,
            "memory": 20,

            "portMappings": [
                {
                    "containerPort": 5000,
                    "hostPort": 80
                }
            ]
        }
    ]
}
```
```shell
$ ecs register-task-definition --family nodeproject --cli-input-json file://node-project-definition.json
```
<p align="center"><img src="https://dl.dropboxusercontent.com/s/5ze3jvd60v1b881/Screen%20Shot%202016-09-08%20at%2011.34.44%20AM.png?dl=0"ECS Task"></p>

Após a definição da task, o último estágio seria a solicitação para execução desta atividade no cluster que criamos anteriormente, esse passo é bem simples e necessitamos somente do comando abaixo.
```shell
aws ecs run-task --cluster NodeProjectSample --task-definition nodeproject:1
```
<p align="center"><img src="https://dl.dropboxusercontent.com/s/fuddzvhg5wbapnp/Screen%20Shot%202016-09-08%20at%2011.21.59%20AM.png?dl=0"ECS Cluster"></p>
Se você acessar a instância do cluster você pode verificar o container rodando.
<p align="center"><img src="https://dl.dropboxusercontent.com/s/1ya76afucl6xmn6/Screen%20Shot%202016-09-08%20at%2011.25.39%20AM.png?dl=0"ECS Cluster"></p>
Se você deseja mais informações sobre definições de tasks e execução das mesmas, você [pode verificar aqui](http://docs.aws.amazon.com/pt_br/AmazonECS/latest/developerguide/task_definition_parameters.html)..

O ECS é sim uma grande opção de ferramenta para colocar containers em execução. A grande ideia é que podemos definir stacks de aplicação em containers e a forma de escalar estas definições se torna muito poderosa, sem necessidade de demais configurações.


## Vamos encerrar?

Como custumo dizer, a melhor ferramenta é aquela que atende melhor suas necessidades e essa é basicamente a ideia deste post. Já tive que usar todas estas ferramentas em diferentes aplicações, mas nunca deixei de lado a ideia de automatizar ao máximo o que eu fazia e a melhor forma de se adequar ao que o projeto precisava. Então espero que você tenha aprendido algo novo ou tido pelo menos algum insight sobre formas de utilizar containers na AWS.

Links do repo e da imagem usada no post:
- [GitHub](https://github.com/pedrocesar-ti/node-project-talk)
- [DockerHub](https://hub.docker.com/r/pedrocesarti/node-project-sample/)

Link também para o post do Gustavo Costa que também fala de [deploy ágil usando Docker](http://blog.concretesolutions.com.br/2016/01/deploy-agil-com-docker/) e para o post do Wesley Silva que mostra um [passo-a-passo de como dockerizar sua aplicação Linux](http://blog.concretesolutions.com.br/2015/07/como-dockerizar-aplicacao-linux/).

Ficou alguma dúvida ou tem algum comentário a fazer? Aproveite os campos abaixo. Quer trabalhar com DevOps em uma empresa verdadeiramente ágil? Acesse aqui. Até a próxima!
