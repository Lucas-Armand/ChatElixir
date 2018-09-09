# ChatScalable
Objetivo dese projeto é criar um serviço de chat cloud native, capaz de escalar horizontalmente e que possa rodar de maneira única em mais de um servidor. Um requisito é que o servidor seja feitos em Elixir. Como esse projeto será o meu primeiro projeto em Elixir, tentarei montar esse documente em um formato passo-a-passo e construir uma lista de links relevaantes para usar como referencia em projetos futuros. Esse trabalho também será uma resposta a um desafio então tentarei realiza-lo numa design sprint de quatro dias.

# Criando seu primeiro Chat no Elixir:

Usaremos o Raxx.Kit para desenvolver o nosso chat. Para iniciar um novo app basta fazer:

```
mix archive.install hex raxx_kit
mix raxx.kit my_chat
```

Se tudo deu certo (caso não veja a [documentação](https://hexdocs.pm/raxx/readme.html)), você já tem um app web. Para acessa-lo basta seguir as orientações inciais:

```
cd my_chat
iex -S mix
```
O resultado é a aplicação padrão do Raxx.kit pode ser acessado em http://localhost:8080/:

![resultado 1](https://github.com/Lucas-Armand/ChatScalable/blob/master/img/resultado_raxxkit_app.png)

## Analisando o aplicativo Elixir:

Mesmo que não seja o foco desse trabalho discutir as basses de como funciona uma aplicação web de forma geral e como esses modelo aparecem dentro da linguagem em questão (até porque já existe muito material sobre isso), acredito que vale a pena explorar rápidamente essa aplicação padrão antes de altera-la.

Quando Raxx.kit inicia seu nomo aplicativo ele cria um diretório para o projeto. A seguir destaco as parte principal para nossa análise dessa estrutura:
```
├── _build
├── config
├── deps
├── lib
│   └── my_chat.ex
│   └── my_chat
│   │   └── application.ex  
│   │   └── public  
│   │   └── www.ex (*)
│   │   └── www
│   │   │   └── home_page.ex  (*)
│   │   │   └── home_page.html.eex  (*)
│   │   │   └── layout.ex  
│   │   │   └── layout.html.eex  
│   │   │   └── not_found_page.ex  
│   │   │   └── not_found_page.html.eex   
├── priv
├── test
```

Então, "www.ex" é o router do aplicativo. Nele é definido a porta de acesso do servidor, qual função chamar para cada acesso de url (no caso de um acesso em '/' á função 'MyChat.WWW.HomePage' será chamada e, para qualquer outro acesso, a função 'MyChat.WWW.NotFoundPage'), alem de importar alguns recursos externos.

```
#lib/my_chat/www.ex
  
defmodule MyChat.WWW do
  use Ace.HTTP.Service, [port: 8080, cleartext: true]
  
  use Raxx.Router, [
    {%{path: []}, MyChat.WWW.HomePage},
    {_, MyChat.WWW.NotFoundPage}
  ]
  
  @external_resource "lib/my_chat/public/main.css"
  @external_resource "lib/my_chat/public/main.js"
  use Raxx.Static, "./public"
  use Raxx.Logger, level: :info
end                         

```




.

.

.

.

.

# Testando o projeto: (ATÉ 07/11)

Para facilitar a visualização do projeto e os testes futuros eu coloquei todo o código para rodar em um repositório em uma máquina virtual. 

### "Hello World" App Cloud Elixir/Docker:

A seguir temos o um "HelloWorld" feito Elixir/Docker podem ser acessados a partir de: 

http://138.68.228.9:8080 (RESULTADO PARA UM SÓ NODE)

Para testar com dois servidores usando o "swarm" do Docker, só foi possível rodando localmente. Então, é necessário acessar ssh:

```
ssh root@138.197.222.53
password:targetso
```

E então faça o acesso a porta 8080:


```
curl http://localhost:8080
```

Se você acessar múltiplas vezes verá que ID do "nó" alterna entre dois valores, como na imagem abaixo: 

![duas maquinas](https://github.com/Lucas-Armand/ElixirDocker/blob/master/linux_result.png)

### Chat Elixir (2 Nodes):

E o projeto de servidor chat no Elixir em múltiplos "nodes" pode ser acessado por: 

http://138.68.228.9:8081

http://138.68.228.9:8082

Caso deseje acessar a máquina virtual para testar localmente:

```
ssh root@138.68.228.9
password:targetso
```

Dai, só seguir os passos descritos a seguir.

# Executando os projetos na própria máquina:

Primeiro é necessário baixar o repositória na sua máquina:

```
https://github.com/Lucas-Armand/ElixirDocker.git
```

# Criando um "Hello World" Cloud Nativo no Elixir/Docker:

## Criando uma imagem do app no docker:

```
cd elixir-on-docker/
docker-compose run --rm www mix deps.get
```
( Nesse ponto é possível fazer um teste :  ```docker-compose up --build ``` )

## Iniciando o swarm:

```
docker swarm init < coloque aqui seu IP (138.68.228.9)>
```
( O resultado desse comando gera um token que será necessário mais adiante!)

## Criando virtual machines no docker:

Para facilitar a implementação e reprodutibilidade do código usaremos as chamadas "docker-machine" que são máquinas virtuais geradas pelo próprio docker. Então:

```
docker-machine create --driver virtualbox myvm1
```

## Conectando demais nodes no swarm:

Agora, usando o token do nosso manager node, devemos iniciar os nós nas virtual machines:
```
docker-machine ssh myvm1 "docker swarm join --token SWMTKN-1-299w4q8qco8zfede9n622zskmttmgs7zast89vbg7mncsi4vfa-ao4za4snopb11g7a3z1zbt3od 138.68.228.9:2377"
```
Nesse ponto se executarmos o comando ``` docker node ls ``` teremos um resultado como :
```
ID                            HOSTNAME                     STATUS              AVAILABILITY        MANAGER STATUS
p2hnk9awi5oqp1zd3fimymiyy *   docker-s-2vcpu-4gb-sfo2-03   Ready               Active              Leader
awzsljg0ya0s9v0jfrzytcvbg     myvm1                        Ready               Active                         
```
Por fim, se rodarmos o comando:

```
docker stack deploy -c docker-compose.yml www
```
o código começa rodar e podemos fazer o acesso:


```
curl http://localhost:8080
```


# Criando um Chat Elixir:

Esse projeto eu me baseei nesse tutorial :

http://crowdhailer.me/2018-05-01/building-a-distributed-chatroom-with-raxx-kit/

Mesmo tendo aprendido muito! Eu não consegui alterar ele, a tempo, para que funcionasse com múltiplas máquinas.
para executá-lo basta:

```
cd 
cd ElixirDocker/watercooler/
``` 

E abra em dois terminais:

```
# terminal 1
$ PORT=8081 SECURE_PORT=8441 iex \
  --name node1@127.0.0.1 \
  --erl "-config sys.config" \
  -S mix

# terminal 2
$ PORT=8082 SECURE_PORT=8442 iex \
  --name node2@127.0.0.1 \
  --erl "-config sys.config" \
  -S mix

```

# Referências:
[Elixir first Project](https://elixir-lang.org/getting-started/mix-otp/introduction-to-mix.html#our-first-project)

[Elixir first web aplication](https://codewords.recurse.com/issues/five/building-a-web-framework-from-scratch-in-elixir)

[Elixir hello word](https://angelika.me/2016/08/14/hello-world-web-app-in-elixir-part-3-phoenix/)

[Elixir+vim](https://github.com/elixir-editors/vim-elixir)

[Elixir Chat Cloud Aplication](https://www.youtube.com/watch?v=JEXT-qaeeyg)

[what's pg2?](https://speakerdeck.com/antipax/pg2-and-you-getting-distributed-with-elixir?slide=12)

[Building a distributed chatroom with Raxx.Kit](http://crowdhailer.me/2018-05-01/building-a-distributed-chatroom-with-raxx-kit/)

[Elixir on Docker](https://github.com/CrowdHailer/elixir-on-docker)

[Elixir Meetup Londom](https://skillsmatter.com/skillscasts/11072-elixir-london-october) 

[Tutorial Docker - Part 3 (swarm)](https://docs.docker.com/get-started/part3/#take-down-the-app-and-the-swarm)


```

###  Instalando Elixir em Server Linux:

```
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb && sudo dpkg -i erlang-solutions_1.0_all.deb
sudo apt-get update
sudo apt-get install esl-erlang
sudo apt-get install elixir
