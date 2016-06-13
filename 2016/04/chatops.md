# ChatOps - passado, presente e futuro.

ChatOps surgiu nos últimos anos como um modelo colaborativo no qual pessoas e ferramentas são conectadas, o que permite uma melhor interação entre os times dentro do ambiente como um todo.

## Sobre passado 

No tempo em que __Cloud Computing__ era um termo muito utilizado, mas ninguém sabia para que seria utilizado mesmo, eu estava lendo um artigo sobre computação distribuída que dizia que os conceitos de cloud datavam da década de 70, e isso fazia muito sentido. Naquela época já iniciavam-se os trabalhos com máquinas virtuais e já era prevista a utilização de computadores __on-demand__ (pagando mensalmente a conta, assim como fazemos com água e luz). Desde então, passei a observar um pouco melhor os "**novos conceitos**" que surgem.

Nos exemplos citados basicamente toda a tecnologia já existia, mas ainda não havia um cenário ideal para que elas fossem aplicadas. Podemos listar na história diversos momentos em que isso aconteceu. Vocês lembram, por exemplo, dos "terminais burros" conectados aos mainframes da IBM? E dos "novos" conceitos de VDI (Virtual Desktop Infrastructure) da VMware?  

A utilização dos chats e de certa automação deles também já teve sua tentativa de aplicação. Desde o __BBSs__ (Bulletin Board System) na década de 70 até o __mIRC__ e __AOL__ na década de 90 passamos por inúmeras ferramentas que utilizavam conceitos bem primitivos de comunicação colaborativa e notificação. Inclusive podemos citar o sugimento do primeiro chatbot em 1993, o [Eggdrop](http://www.eggheads.org), utilizado para notificações dentro de canais IRC. Ele surgiu como algo revolucionário, mas fora de contexto. Apesar da grande popularidade, no inîcio dos anos 2000 os chats perderam bastante espaço devido aos serviços de mensagens instantâneas e redes sociais, voltando a ter certo protagonismo mais recentemente, em serviços de atendimento ao cliente incorporados a sites de empresas, mas ainda assim sem uma grande aplicação.

## E agora?

Mais um termo que surgiu recentemente e vem causando alvoroço é o tal **DevOps**, mas, diferente de outros posts, o meu não tem por intenção discutir o que é ou deixa de ser DevOps. Não quero deixar pessoas emocionalmente abaladas pelo que escrevo. E já que eu não estou aqui para definir se podemos ou não chamar DevOps de profissão, eu só queria compartilhar que temos alguns pilares comuns aos quem aderem ao tal DevOps. Esses pilares principais ficaram conhecidos como CAMS:

- Culture: o principal intuito é diminuir as barreiras entre os times e permitir maior integração por meio de processos e sistemas menos burocráticos.
- Automation: automatizar tudo que for possível para permitir ganho de performance e de produtividade.
- Measurement: buscar dados de tudo o que for possível, para maior inteligência em tomada de decisões e, o mais importante, transparência na apresentação destes dados.
- Sharing: a chave maior de tudo é compartilhar, compartilhar tudo que for possível. 

Então, a comunidade busca formas de se aproximar desses pilares para que os processos fiquem mais ajustados ou mesmo para permitir melhor gerência das informações. Mas em um ambiente de medição, automatização e compartilhamento total alguns problemas surgem:

- Como deixar todos os times (muitas vezes não só os times de produto, como desenvolvimento e operações) atualizados no processo? Como deixar todas as equipes na mesma "página"?
- Como garantir que todos possam fazer de forma ágil as suas atividades e isso fique visível a todos? Não queremos ter mais heróis que magicamente resolvem um problema sem ninguém mais saber como?
- Como garantir certa uniformidade e processo no acesso às informações?
- Como permitir a unificação das ferramentas em um ambiente só e não ter que fazer a "Dashboard das dashboards", aquela ferramenta que gerencia outras ferramentas?

A resposta é simples: utilizando uma ferramenta interativa com a qual podemos criar comandos e scripts para automatizar tarefas, desde as mais básicas até as mais complexas. O CHAT, então, volta à tona, e independentemente do serviço utilizado (Slack, Hipchat, Campfire ou outros), temos inúmeras opções de bots para serem conectados. 

- [Hubot](https://hubot.github.com) (JavaScript e CoffeeScript)
- [Lita](https://www.lita.io) (Ruby)
- [Err..](http://errbot.io/en/latest/) (Python)  

Verificando as documentações dessas ferramentas, é possível verificar o quão customizåveis e extensíveis elas são. Só precisamos adicionar um script da linguagem suportada para adicionar uma funcionalidade e converter algo digitado no chat para um comando para execução de uma tarefa específica. 

Recentemente, eu fiz um pequeno projeto usando Docker para subir um Hubot e conectar ao meu Slack pessoal e poder brincar tranquilamente. Encorajo a todos fazerem o mesmo, então se quiserem dar uma brincada inicial com ChatOps, verifiquem esse repositório:

- [ChatOps Begginers](https://github.com/pedrocesar-ti/hubot-slack-docker)

Grande parte dos plugins são configurados via variável de ambiente, então se você passar seu token de acesso ao Slack, o nome e o canal para o bot tudo já funcionará com o simples comando abaixo:

```shell
docker run --name mybot -it  \
 -e HUBOT_SLACK_TOKEN='#######' \
 -e Hubot_SLACK_TEAM='#######' \
 -e Hubot_SLACK_BOTNAME='#######' \
 -d pedrocesarti/hubot-slack
```
Após o comando acima, seu bot já estará conectado ao chat e esperando por comandos. Basta chamar o nome dele e dar um help. Meu bot se chama macgyver, então o comando:

```
macgyver help
```
Já mostraria todos os plugins ativados e como utilizá-los, conforme imagem abaixo.

<p align="center"><img src="https://dl.dropboxusercontent.com/s/d0mld4njpw70bmw/Screen%20Shot%202016-06-08%20at%207.13.36%20PM.png?dl=0"Hubot - Slack"></p>

A documentação do repositório mostra algumas outras opções e customizações que podem ser criadas, divirta-se.

## E daqui para a frente?
Depois de passarmos por muitas dores de cabeça com integrações no passado, estamos em uma época áurea em relação à integração e grande parte deste sucesso é a adoção cada vez mais pesada de APIs. Tudo se tornou mais fácil, endpoints configurados, e tudo se resolve por meio de GETs e POSTs que retornam um JSON perfeito para ser tratado ;).

Levando em consideração que já temos inúmeros módulos já desenvolvidos para todos os bots, que permitem as mais variadas funcionalidades - como gerar um gráfico de utilização da rede, como inicializar uma instância na AWS ou até mesmo como criar uma regra no firewall - e que eles permanecem expansíveis, aguardando só um novo script para criar uma nova funcionalidade, acredito então que a popularidade tende a continuar crescendo.  

ChatOps talvez não seja realmente para todos os ambientes, mas quanto mais destruímos barreiras entre os times e deixamos todo o processo mais transparente tudo se torna mais fácil. Então, se eu tenho uma mensagem para deixar no final deste post é:

**USE CHATOPS E SEJA MAIS ÁGIL!!**

Ficou alguma dúvida ou tem algum comentário a fazer? Aproveite os campos abaixo. Quer trabalhar com DevOps em uma empresa verdadeiramente ágil? Acesse aqui. Até a próxima!

