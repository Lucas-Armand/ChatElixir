# ChatScalable
Objetivo dese projeto é criar um serviço de chat cloud native, capaz de escalar horizontalmente e que possa rodar de maneira única em mais de um servidor. Um requisito é que o servidor seja feitos em Elixir. Como esse projeto será o meu primeiro projeto em Elixir, tentarei montar esse documente em um formato passo-a-passo e construir uma lista de links relevaantes para usar como referencia em projetos futuros. Esse trabalho também será uma resposta a um desafio então tentarei realiza-lo numa design sprint de quatro dias.

# Criando seu primeiro Chat no Elixir:

Usaremos o Raxx.Kit para desenvolver o nosso chat. Para iniciar um novo app basta fazer:

```
mix archive.install hex raxx_kit
mix raxx.kit my_chat
```

Se tudo deu certo (caso não, veja a [documentação](https://hexdocs.pm/raxx/readme.html)), você já tem um app web. Para acessa-lo basta seguir as orientações inciais:

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

Então, "www.ex" é o **router** do aplicativo. Nele é definido a porta de acesso do servidor, qual função chamar para cada acesso de url (no caso de um acesso em '/' á função 'MyChat.WWW.HomePage' será chamada e, para qualquer outro acesso, a função 'MyChat.WWW.NotFoundPage'), alem de importar alguns recursos externos.

```elixir
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

Nesse caso "home_page.ex" faz o papel de **view** associado ao acesso á pagina da aplicação. O código gerencia o que acontece com ocorre um GET e um POST, o que fazer em caso de erro e quaiS são as informações que o template vai receber:

```elixir
#lib/my_chat/www/home_page.ex

defmodule MyChat.WWW.HomePage do
  use Raxx.Server
  use MyChat.WWW.Layout, arguments: [:title]

  @impl Raxx.Server
  def handle_request(_request = %{method: :GET}, _state) do
    title = "Raxx.Kit"

    response(:ok)
    |> render(title)
  end

  def handle_request(request = %{method: :POST}, _state) do
    case URI.decode_query(request.body) do
      %{"name" => name} ->
        greeting = "Hello, #{name}!"

        response(:ok)
        |> render(greeting)
      _ ->
        response(:bad_request)
        |> render("Bad Request")
    end
  end
end

```

Por fim, temos o arquvio "home_page.html.eex" que é o **template**. Aqui podemos ver aonde "title" (que é introduzido no código da view) é referenciado. Esse arquivo html, junto aos arquivos em "/public" compõem o frontend da aplicação.


```html
#lib/my_chat/www/home_page.html.eex

<main class="centered">
  <section class="accent">
    <h1><%%= title %></h1>
    <form action="/" method="post">
      <label>
        <span>tell us your name</span>
        <input type="text" name="name" value="" autofocus>
      </label>
      <button type="submit">Submit</button>
    </form>
    <h2>Find out more</h2>
    <nav>
      <a href="https://github.com/CrowdHailer/raxx">SOURCE CODE</a>
      <a href="https://elixir-lang.slack.com/messages/C56H3TBH8/">CHAT</a>
    </nav>
    <%%= javascript_variables title: title %>
    <script type="text/javascript">
      document.addEventListener('DOMContentLoaded', function () {
        app.show(title)
      })
    </script>
  </section>
</main>

```
## Altarando o aplicativo Elixir para construir um Chat:

Dai, podemo começar fazendo uma alteração no router. No caso vamos manter o acesso em "/" á função "MyChat.WWW.HomePage" (que será alterada a frente), para um POST em "publish" será chamada "MyChat.WWW.Publish", para um acesso em "listen" será chamada "MyChat.WWW.Listen" e qualquer outro acesso a função "MyChat.WWW.NotFoundPage". Além de importar alguns recursos externos.

```elixir
defmodule MyChat.WWW do
  use Ace.HTTP.Service, [port: 8080, cleartext: true]
  
  use Raxx.Router, [
    {%{method: :GET, path: []}, MyChat.WWW.HomePage},
    {%{method: :POST, path: ["publish"]}, MyChat.WWW.Publish},
    {%{method: :GET, path: ["listen"]}, MyChat.WWW.Listen},
    {_, MyChat.WWW.NotFoundPage}
  ]

  @external_resource "lib/my_chat/public/main.css"
  @external_resource "lib/my_chat/public/main.js"
  use Raxx.Static, "./public"
  use Raxx.Logger, level: :info
end
```

Agora devemos criar as views para realizar adicionar novas funcionalidades a aplicação. Incialmente faremos uma função para lidar com um request inicial. A informação denomida "node", que é o ID do nó que está executando a tarefa, será importante quando, mais a frente, tivermos multiplos node executando o código em paralelo, passando essa informação para o template poderemo saber em que node a aplicação está rodando.

```elixir
#lib/my_chat/www/home_page.ex

defmodule MyChat.WWW.HomePage do
  use Raxx.Server
  use MyChat.WWW.Layout, arguments: [:node]


  @impl Raxx.Server
  def handle_request(_request, _state) do
    node = Node.self()

    response(:ok)
    |> render(node)
  end
end
```

Agora devemos construir um metodo publisher e um listener:

```elixir
#lib/my_chat/www/publish.ex

defmodule MyChat.WWW.Publish do
  use Raxx.Server

  @impl Raxx.Server
  def handle_request(request, _state) do
    "message=" <> message = request.body 
     message = URI.decode_www_form(message) 

    {:ok, _} = MyChat.publish(message) # 2.

    redirect("/") # 3.
  end
end
```

Esse método recebe as informações "postadadas" pela pagina e limpa os dados do request para enviar somente a "messagem" para o backend. Mais a frente vamos criar esse MyChat.publish.


```elixir
#lib/my_chat/www/listen.ex

defmodule MyChat.WWW.Listen do
  use Raxx.Server
  alias ServerSentEvent, as: SSE

  @impl Raxx.Server
  def handle_request(_request, state) do
    {:ok, _} = MyChat.listen()

    response = response(:ok)
    |> set_header("content-type", SSE.mime_type())
    |> set_body(true) 

    {[response], state} 
  end

  @impl Raxx.Server
  def handle_info({:mychat, message}, state) do
    event = SSE.serialize(message)
    {[Raxx.data(event)], state} 
  end
end
```
Em listen.ex você tem duas ações. A primeira, handle_request, evoca MyChat.listen() (que iremos discutir a seguir) e retorna uma respota de "ok esta tudo funcionando". O segundo método "handle_info" é evocado toda vez que alguma mensagem é enviada. Ele serializa a mensagem e usa o Raxx.data para tornar essa informação disponibel para js (que atualizará as mensagens na tela). Esse método usa um biblioteca de serialização chamada "ServerSentEvent".

Agora, nos aprofundando no backend, vamos fazer o **controler** da nossa aplicação.

```elixir
#lib/my_chat.ex
defmodule MyChat do
  @group :mychat

  def publish(message) do
    :ok = :pg2.create(@group)
    for client <- :pg2.get_members(@group) do
      send(client, {@group, message})
    end

    {:ok, message}
  end

  def listen() do
    :ok = :pg2.create(@group)
    :ok = :pg2.join(@group, self())
    {:ok, @group}
  end
end
```
A idéia de separar a lógica da aplicação view (local aonde eu chamo outras funções do código e onde eu faço um processamento básico das informações de saida e entrada do template) do controler (processamento dos dados, lógica da aplicação, queries e filtros do banco de dados, etc...) é interessante, porque muitas vezes as funções criadas no controller podem ser aproveitadas em outras aplicaçãos. Nesse código tempo "MyChat.publish" e "MyChat.listen" (como prometido) e podemos perveber que a funcionalidade desses métodos é baseada em um biblioteca "pg2". A pg2 é um biblioteca do erlang (que pra ser sincero ainda não entendo muito bem), mas, pelo que eu entendo, ela permite que você crie "nodes" que fiquem inscritos a um grupo, assim você pode mandar para eles mensagens que são recebidos como eventos. 

Em seguida, podemos nos avançar,  e atualizar o frontend para se adequar aquilo a aplicação de um chat:

```html
#lib/my_chat/www/home_page.html.eex

<main>
  <h1>My First Chat</h1>
  <h2>Node: <%= node %></h2>
  <iframe name="myframe" width="0" height="0"></iframe>
  <form action="/publish" method="post" target="myframe">
    <input type="text" name="message">
    <button type="submit">Publish</button>
  </form>
  <hr>
  <h1>Messages</h1>
  <ul id="messages"></ul>
</main>
```
Por fim algum css:

```css
#lib/my_chat/www/public/main.css

main {
  max-width: 720px;
  margin-left: auto;
  margin-right: auto;
}

h1 {
  text-align: center;
}

iframe {
  border: none;
}
```
E agora, se rodarmos o serviço (``` iex -S mix ```) já temos a nossa nova aplicação. Podemos abri-la em multiplas abas o brownser para testar o recebimento de mensagem, mas a seguir discutiremos como rodar em multiplos nós:

![nonode print](https://github.com/Lucas-Armand/ChatScalable/blob/master/img/nonode.png)

Abrindo em multiplos terminais

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
