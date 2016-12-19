Clusterizando RabbitMQ com Docker Compose
=========================================
A ideia
-------
Olá, vamos falar sobre [RabbitMQ](http://www.rabbitmq.com/)? O RabbitMQ é um servidor de mensagens feito especialmente para trabalhar com [Advanced Message Queuing Protocol(AMQP)](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol), um protocolo de comunicação em rede que permite que aplicações se comuniquem. Aqui no blog da Concrete já falamos um pouco sobre conceitos RabbitMQ, [em um post que você pode ver aqui](http://www.concretesolutions.com.br/2012/01/26/rabbitmq-conceitos-basicos/).
Com a utilização de um middleware para controle de mensageria a integração de sistemas fica geralmente mais barata e menos complexa. O AMQP torna a integração entre vários sistemas diferentes possível e também permite que diferentes produtos, que implementam este mesmo protocolo, possam trocar informações, o que torna o AMQP um dos pioneiros na evolução dos sistemas de troca de mensagem.

Aqui vamos utilizar [Docker-Compose](https://docs.docker.com/compose/) para "subir" quatro instâncias do RabbitMQ junto com o [HAProxy](http://www.haproxy.org/), um [Load Balancer](https://www.nginx.com/resources/glossary/load-balancing/) para distribuir, eficientemente, o tráfego entre os nós do RabbitMQ.

![Rabbit_cluster_setup](imagens/rabbit-cluster-setup.png)

A ideia é que o HAProxy receba uma publicação de alguma fila, de alguma exchange e redirecione para um dos nós do RabbitMQ. O cluster estaria então replicando os dados em todos os nós e caso algum deles falhe, temos os dados nos demais e o próprio RabbitMQ escolhe outro nó para ser o novo "master".

Mão na massa
------------

Vamos utilizar Docker para subir nossas instâncias de RabbitMQ, lembra? Para que funcione é preciso ter o Docker já instalado. [Aqui está o tutorial](https://docs.docker.com/engine/installation/), verifique qual SO está usando e seja feliz. :smile:

É necessária a instalação do Docker-Compose também. Para instalar é um pouco mais simples que o Docker em si. Nesse caso, você pode seguir [esse tutorial](https://docs.docker.com/compose/install/).

Com tudo instalado, vamos começar a destruir o teclado.  :musical_keyboard:

Vamos começar criando nosso script de subida dos nossos docker's. Crie um arquivo chamado docker-compose.yml: `vim docker-compose.yml`
Agora copie e cole o código abaixo no seu arquivo:

```
version: '2'
services:
  rabbitmq1:
    image: rabbitmq:3-management
    container_name: rabbitmq1
    hostname: rabbitmq1
    ports:
      - "8080:15672"
    network_mode: "bridge"
    volumes:
     - $PWD/storage/rabbitmq1:/var/lib/rabbitmq
    environment:
     - RABBITMQ_ERLANG_COOKIE=This_is_my_secret_phrase
     - RABBITMQ_DEFAULT_USER=mqadmin
     - RABBITMQ_DEFAULT_PASS=Admin123XX_
     - CLUSTERED=true

  rabbitmq2:
    image: rabbitmq:3-management
    container_name: rabbitmq2
    hostname: rabbitmq2
    ports:
      - "8081:15672"
    network_mode: "bridge"
    volumes:
     - $PWD/storage/rabbitmq2:/var/lib/rabbitmq
    environment:
     - RABBITMQ_ERLANG_COOKIE=This_is_my_secret_phrase
     - RABBITMQ_DEFAULT_USER=mqadmin
     - RABBITMQ_DEFAULT_PASS=Admin123XX_
     - CLUSTERED=true
    links:
     - rabbitmq1:rabbitmq1

  rabbitmq3:
    image: rabbitmq:3-management
    container_name: rabbitmq3
    hostname: rabbitmq3
    ports:
      - "8082:15672"
    network_mode: "bridge"
    volumes:
      - $PWD/storage/rabbitmq3:/var/lib/rabbitmq
    environment:
     - RABBITMQ_ERLANG_COOKIE=This_is_my_secret_phrase
     - RABBITMQ_DEFAULT_USER=mqadmin
     - RABBITMQ_DEFAULT_PASS=Admin123XX_
     - CLUSTERED=true
    links:
     - rabbitmq1:rabbitmq1
     - rabbitmq2:rabbitmq2

  rabbitmq4:
    image: rabbitmq:3-management
    container_name: rabbitmq4
    hostname: rabbitmq4
    ports:
      - "8083:15672"
    network_mode: "bridge"
    volumes:
      - $PWD/storage/rabbitmq4:/var/lib/rabbitmq
    environment:
     - RABBITMQ_ERLANG_COOKIE=This_is_my_secret_phrase
     - RABBITMQ_DEFAULT_USER=mqadmin
     - RABBITMQ_DEFAULT_PASS=Admin123XX_
     - CLUSTERED=true
    links:
     - rabbitmq1:rabbitmq1
     - rabbitmq2:rabbitmq2
     - rabbitmq3:rabbitmq3

  haproxy:
    image: haproxy:1.6
    container_name: haproxy
    hostname: haproxy
    ports:
      - 5672:5672
      - 20000:20000
    network_mode: "bridge"
    links:
      - rabbitmq1:rabbitmq1
      - rabbitmq2:rabbitmq2
      - rabbitmq3:rabbitmq3
      - rabbitmq4:rabbitmq4
```
Parece complicado, mas eu juro, não é! Vamos dar uma olhada em cada um dos parâmetros.

* **version: '2'**  :arrow_right: Com isso já trabalharemos com as funcionalidades da versão mais recente do docker-compose.

* **services:** :arrow_right: Aqui vamos listar todos os serviços que vamos utilizar, todos os docker que vão subir para que nossa aplicação funcione.

* **rabbitmq1:** :arrow_right: Nome que daremos para os nossos serviços do [RabbitMQ](http://www.rabbitmq.com/).

* **image: rabbitmq:3-management** :arrow_right: Nome da imagem que usaremos para nosso [RabbitMQ](http://www.rabbitmq.com/).

* **container_name: rabbitmq1** :arrow_right: Vamos dar nomes para nossos containers para ficar mais fácil administrá-los.

* **hostname: rabbitmq1** :arrow_right: Vamos colocar o nome dos nossos hosts, isso facilitará e muito a nossa clusterização, pois sempre que utilizar um `docker exec` os nomes internos dos conteiners serão os mesmos.

* **ports: - "8080:15672"** :arrow_right: Com isto poderemos verificar, por meio do nosso browser, se nosso [RabbitMQ](http://www.rabbitmq.com/) está funcionando: `localhost:8080`.

* **network_mode: "bridge"** :arrow_right: Para que nosso cluster funcione, precisamos que todos os nós estejam na mesma "rede", neste caso na `bridge`.

* **volumes: - $PWD/storage/rabbitmq1:/var/lib/rabbitmq** :arrow_right: Criação dos volumes que iremos utilizar.

* **environment:** :arrow_right: Aqui vamos definir nossas variáveis de ambiente utilizadas por cada um dos serviços.

* **- RABBITMQ_ERLANG_COOKIE=This_is_my_secret_phrase** :arrow_right: Para utilizar o [RabbitMQ](http://www.rabbitmq.com/), precisamos definir um Erlang Cookie para cada nó se comunicar entre si.

* **- RABBITMQ_DEFAULT_USER=mqadmin** :arrow_right: Definimos um usuário para nosso nó.

* **- RABBITMQ_DEFAULT_PASS=Admin123XX_** :arrow_right: Definimos uma senha para nosso usuário.

* **- CLUSTERED=true** :arrow_right: Isto é apenas para definir que rodaremos em modo cluster.

* **- links:** :arrow_right: Com este parâmetro definimos todos os links dos nossos serviços com demais serviços da stack.

:point_right: As configurações do HAProxy são as mesmas que o RabbitMQ no Docker-Compose.

Agora vamos preparar nossa imagem do HAProxy, poderiamos utilizar a default mas precisamos alterar o arquivo de configuração para atender as nossas necessidades.

Primeiro vamos criar um Dockerfile para fazer uma imagem do HAProxy que atenda os nossos requisitos: `vim Dockerfile`

Agora cole no Dockerfile os comandos abaixo:

```
FROM haproxy:1.6

 ENV HAPROXY_USER haproxy

 RUN groupadd --system ${HAPROXY_USER} && \
   useradd --system --gid ${HAPROXY_USER} ${HAPROXY_USER} && \
   mkdir --parents /var/lib/${HAPROXY_USER} && \
   chown -R ${HAPROXY_USER}:${HAPROXY_USER} /var/lib/${HAPROXY_USER}

 COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg

 CMD ["haproxy", "-db", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
 ```

Com esse Dockerfile será criada uma imagem com um usuário haproxy e o nosso arquivo de configuração já será copiado para dentro dela, deixando-a pronta para uso.

No mesmo diretório, vamos criar um arquivo chamado `haproxy.cfg` com o seguinte conteúdo:

```
global
 	  log 127.0.0.1	local0 info
    chroot /var/lib/haproxy
    user haproxy
    group haproxy

listen stats
    bind  *:20000
    mode  http
    stats enable
    stats refresh 3s
    stats uri /
    timeout client 10000
    timeout connect 10000
    timeout server 30000
    stats auth guest:guest

listen rabbitmq-tcp
    bind    *:5672
    mode    tcp
    log global
 	  retries 4
 	  option tcplog
    option persist
    balance roundrobin
    server  rabbitmq1 rabbitmq1:5672 check inter 5s rise 2 fall 5
    server  rabbitmq2 rabbitmq2:5672 check inter 5s rise 2 fall 5
    server  rabbitmq3 rabbitmq3:5672 check inter 5s rise 2 fall 5
    server  rabbitmq4 rabbitmq4:5672 check inter 5s rise 2 fall 5
```

Essas configurações vão permitir o balanceamento dos nossos nós, checando, em um intervalo de 5 segundos, se estão online ou não. Roundrobin é um algoritmo simples de escalonamento, é um mecanismo que equilibra a carga local e distribui entre os nós do cluster. Nossos nós estarão sendo ouvidos e checados pela porta 5672, que é a porta padrão de comunicação TCP do RabbitMQ.

Pronto :exclamation:, já criamos os arquivos de configuração  :clap:. Agora vamos fazer o build da nossa nova imagem do HAProxy. No mesmo diretório que criamos o Dockerfile e o haproxy.cfg, digite:

`docker build -t haproxy:1.6 .`

O processo deverá ser algo parecido com isso:

![build_imagem_haproxy](imagens/build_imagem_haproxy.png)

Agora vamos juntar tudo isso :scream:. Calma que tudo vai dar certo  :sweat_smile:, eu prometo. Vamos criar um shell script bem básico para subir tudo de uma vez :heart_eyes:. Nesse mesmo diretório que criamos os outros arquivos, vamos criar um arquivo (prometo que é o último :pray:) com o nome de start_all.sh: `vim start_all.sh` e cole o código abaixo:

```
#!/bin/bash
# "Bruno Luis Cardoso Novo <bruno.novo@concrete.com.br>"
# Script to manage the containers execution

## Variables for images build.
JOIN_RABBIT2_RABBIT1="rabbitmqctl stop_app; rabbitmqctl join_cluster rabbit@rabbitmq1; rabbitmqctl start_app"
JOIN_RABBIT3_RABBIT1="rabbitmqctl stop_app; rabbitmqctl join_cluster rabbit@rabbitmq1; rabbitmqctl start_app"
JOIN_RABBIT4_RABBIT1="rabbitmqctl stop_app; rabbitmqctl join_cluster rabbit@rabbitmq1; rabbitmqctl start_app"
OPTIONAL_COMMAND="rabbitmqctl set_policy ha-all '' '{\"ha-mode\":\"all\", \"ha-sync-mode\":\"automatic\"}'"


#Subindo os container's do rabbitmq
echo -n "Starting container..."
docker-compose down
docker-compose up -d
sleep 15
docker exec -ti rabbitmq2 bash -c "$JOIN_RABBIT2_RABBIT1"
docker exec -ti rabbitmq3 bash -c "$JOIN_RABBIT3_RABBIT1"
docker exec -ti rabbitmq4 bash -c "$JOIN_RABBIT4_RABBIT1"
docker exec -ti rabbitmq1 bash -c "$OPTIONAL_COMMAND"
```

Como estamos utilizando Docker-Compose fica muito mais fácil de "subir" um container. Ele verifica se essa stack já está em execução e termina a mesma caso seja necessário (`docker-compose down`), depois, em modo background(`-d`), ele sobe todos os nossos serviços, um por um. Foi colocado um sleep de 15 segundos para dar tempo de todos eles subirem e, depois da ligação entre eles, executar, dentro de cada um dos nossos containers do RabbitMQ, os comandos para os nós se juntarem ao cluster. Logo depois disso foi colocado um OPTIONAL_COMMAND para definirmos uma política para nosso cluster.

:warning: No caso da política, foi definido que o cluster funcionará em modo mirror, todos os nós terão as mesmas informações. Dependendo do tipo de cluster que sua aplicação necessite, o tipo de política pode ser alterada sem problemas.  :grin:

Dê permissão à ele: `chmod +x start_all.sh`

E execute: `./start_all.sh`

Com isso vamos poder ver algo parecido com isso:

![start_all.sh](imagens/start_all.png)

Para verificarmos se tudo subiu como planejado, vamos digitar: `docker ps`

![docker_ps](imagens/docker_ps.png)

Podemos abrir o browser e digitar: `localhost:8080`, por exemplo, e veremos a tela de login do [RabbitMQ](http://www.rabbitmq.com/). Digite o usuário e a senha que configuramos e Voilá:

![rabbit_admin_page](imagens/rabbit_admin_page.png)

Aleluia irmãos  :raised_hands:! Tudo funcionando a pleno vapor, mas e se tivermos algum problema em algum dos nós? E se algum deles cair? Vamos fazer um teste  :smiling_imp:. Vamos parar um dos nós e verificar como o RabbitMQ vai se comportar. Volte para o terminal e digite: `docker stop rabbitmq1`

Quando voltamos para o browser, não conseguimos mais ver a página do RabbitMQ porque paramos ele. Ao invés de utilizar a porta `8080`, vamos alterar para `8081` que é a porta do `rabbitmq2` que configuramos no docker-compose.

![rabbit_admin_page_rabbitmq1_stopped](imagens/rabbit_admin_page_rabbitmq1_stopped.png)

Podemos verificar que o `rabbitmq1` que paramos está off mas o próprio RabbitMQ já verificou isso e já promoveu um novo nó para ser nosso "master" e continuarem seu funcionamento.

Para retornar o `rabbitmq1` de volta das cinzas é só digitar: `docker start rabbitmq1` e pronto. Tudo de volta à normalidade.

Uma outra maneira de ficarmos de olho :eyes: nos Rabbit's é utilizar o painel do HAProxy: `localhost:20000`

![HAProxy_admin_page](imagens/HAProxy_admin_page.png)

Link's interessantes
-----------

Bom pessoal, é isso. Essa foi uma maneira mais simples de subirmos um RabbitMQ e começarmos a utilizá-lo nas nossas futuras aplicações. Se quiser se aprofundar ainda mais no assunto existem vários links bem interessantes e que darão uma visão bem mais técnica de sua utilização.

[RabbitMQ - Site Oficial](https://www.rabbitmq.com/)

[Introducao ao AMQP com Rabbitmq](http://www.devmedia.com.br/introducao-ao-amqp-com-rabbitmq/33036)

[Por que utilizar filas? Implementação com RabbitMq](http://www.pognao.com.br/2013/04/rabbitmq.html)

[O que é RabbitMQ?](https://www.portalgsti.com.br/rabbitmq/sobre/)

Obrigado pelo tempo que disponibilizou para ler este artigo e qualquer dúvida é só chamar. Gostaria de agradecer ao Wesley Silva, Pedro Azevedo e o Kaio Kardamone que me ajudaram muito, esse post é de vocês também.

Ficou alguma dúvida ou tem algum comentário a fazer? Aproveite os campos abaixo. 

--
Quer trabalhar com DevOps em uma empresa verdadeiramente ágil? [Acesse aqui](http://conteudo.concretesolutions.com.br/trabalhe-conosco-concrete-solutions). Até a próxima!
