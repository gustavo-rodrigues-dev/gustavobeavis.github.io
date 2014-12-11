---
layout: post
title: "web-socket-php"
date: 2014-09-11 22:14:11 -0200
comments: true
categories: ["websocket", "PHP", "IO", "NODE"]
author: Gustavo Rodrigues
---

![imagem](/../images/title_post_socket.png)

A web como conhecemos é baseada em solicitação/resposta de HTTP, ou seja, em cada site que você acessa, em cada solicitação AJAX que é feita ao servidor, temos uma alocação de um novo processo HTTP que entrará em uma fila pra ser respondido e o tempo de retorno pode variar de acordo com a capacidade do servidor em lidar com esses processos, sejam eles paralelos ou concorrentes, requisições simples ou requisições que dependam de uma grande capacidade computacional.

Agora quando pensamos em uma aplicação de respostas em tempo real, logo nos deparamos com um problema, chamado alta latência, pois, precisaremos consultar nosso servidor muito mais vezes, isso sem contar no número de clientes ativos fazendo requisições ao mesmo recurso ou áreas diferentes do mesmo sistema, causando lentidão no servidor e em casos mais graves, teremos problemas como timeout de execução ou queda do servidor.


##Event loop
![imagem](/../images/event_loop.png)

Toda aplicação realtime, necessariamente possui um loop de consulta de estado ou evento, isso pode estar tanto do lado cliente como do lado servidor. Esse loop ficará escutando a aplicação aguardando alguma mudança de estado, e caso haja atualização, o sistema deve enviar esses dados, a diferença entre o loop do cliente e o do servidor é que no caso do servidor temos um único serviço que irá ficar escutando todas as requisições e mandando as mudanças em tempo real para todos ou algum cliente específico via socket, já no caso do cliente, teremos um script long pooling em todos os clientes, que irá consultar periodicamente o servidor e receber as possíveis alterações do servidor.



##Primeiro socket PHP
Para criar um socket com php, você deve seguir 6 passos, que vão da abertura de um socket até remover a mascara da mensagem resolver seu encode.

####Passos:

 1. Abrir um socket.
 2. Vincular a um endereço.
 3. Escutar conexões de entrada
 4. Aceitar conexões
 5. WebSocket Handshake.

###Passo 1 – Abrir um socket:
Primeiro vamos criar um socket com [socket_create](http://php.net/manual/pt_BR/function.socket-create.php)(Domain, Type, Protocol) do PHP:

```php
$socket = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
```

###Passo 2 – Vincular a um endereço:
[socket_bind()](http://php.net/manual/pt_BR/function.socket-bind.php) recebe como parâmetro o resource do socket que já foi criado com o socket_create(), o endereço e  opcionalmente a porta. 
Isto tem que ser feito antes que uma conexão seja estabelecida, usando socket_connect() ou socket_listen().

```php
socket_bind($socket, 'localhost');
```

###Passo 3 – Escutar conexões de entrada
Após o socket ter sido criado usando socket_create e associado para um nome com socket_bind() , ele deve dizer para aguardar por escuta em conexões que irão entrar socket com o [socket_listen()](http://php.net/manual/pt_BR/function.socket-listen.php).

```php
socket_listen($socket);
```

###Passo 4 – Aceitar conexões
Após o socket ter sido criado usando socket_create(), passar um nome com socket_bind(), e dizer para listar conexões com socket_listen(), a função [socket_accept()](http://php.net/manual/pt_BR/function.socket-accept.php) irá aceitar conexões vindas neste socket. Uma vez que uma conexão é feita com sucesso , um novo “resource” do socket é retornado, que deve ser usado para comunicação. Se há múltiplas conexões na fila do socket, a primeira irá ser usada. Se não há conexões pendentes, socket_accept() irá bloquear até que uma conexão esteja presente.

```php
socket_accept($socket);
```

###Passo 5 – WebSocket Handshake
O cliente tem de apresentar-se, enviando uma solicitação WebSocket handshake para estabelecer uma conexão bem sucedida com o servidor, o pedido contém um Sec-WebSocket-Key que é uma chave em base64 de 16 bytes

```php
$secKey = $headers['Sec-WebSocket-Key'];
$secAccept = base64_encode(pack('H*', sha1($secKey . '258EAFA5-E914-47DA-95CA-C5AB0DC85B11')));
$upgrade  = "HTTP/1.1 101 Web Socket Protocol Handshake\r\n" .
"Upgrade: websocket\r\n" .
"Connection: Upgrade\r\n" .
"WebSocket-Origin: $host\r\n" .
"WebSocket-Location: ws://$host:$port/deamon.php\r\n".
"Sec-WebSocket-Accept:$secAccept\r\n\r\n";
socket_write($client_conn,$upgrade,strlen($upgrade));
```

###Passo 5 – WebSocket Handshake
Depois de fazer handshaking, o cliente pode enviar e receber mensagens, mas as mensagens enviadas são todas criptografadas, por isso, se queremos exibi-las, precisamos remover a mascara de cada frame de dado, conforme o [rfc6455](http://tools.ietf.org/html/rfc6455#section-5.2).

```php
//Unmask incoming framed message

function unmask($text) {
    $length = ord($text[1]) & 127;
    if($length == 126) {
        $masks = substr($text, 4, 4);
        $data = substr($text, 8);
    }
    elseif($length == 127) {
        $masks = substr($text, 10, 4);
        $data = substr($text, 14);
    }
    else {
        $masks = substr($text, 2, 4);
        $data = substr($text, 6);
    }
    $text = "";
    for ($i = 0; $i < strlen($data); ++$i) {
        $text .= $data[$i] ^ $masks[$i%4];
    }
    return $text;
}

//Encode message for transfer to client.
function mask($text)
{
    $b1 = 0x80 | (0x1 & 0x0f);
    $length = strlen($text);

    if($length <= 125)
        $header = pack('CC', $b1, $length);
    elseif($length > 125 && $length < 65536)
        $header = pack('CCn', $b1, 126, $length);
    elseif($length >= 65536)
        $header = pack('CCNN', $b1, 127, $length);
    return $header.$text;
}
```

###Executar
Para executar o Socket basta você acessar o seu terminal e executar o script que serve como servidor, pois se inicia-lo via browser, o seu socket não funcionará.

```
php -q path-to-server/server.php
```

Veja o Exemplo completo no [github](https://github.com/gustavobeavis/ws_pure_php).


##O React PHP
O React PHP é uma biblioteca que foi criada para suprir uma necessidade crescente de computação reativa, ela fornece entre outras coisas a possibilidade de criar aplicações com IO não blocante, como o NodeJS.

###Um chat com React e Ratchet
O Rachet possui um conjunto de soluções que abstraem e facilitam a criação, manutenção, envios e recebimentos de mensagens de um socket. Você terá que gerenciar os eventos do seu socket, com as funções.

####Funções
 1. onOpen 
 2. onMessage 
 3. onClose 
 4. onError

###onOpen
Essa função é chamada assim que ocorre uma nova conexão, nela você usa um método para guardar a nova conexão para enviar mensagens.

```php
public function onOpen(ConnectionInterface $conn) {
  // Store the new connection to send messages to later
  $this->clients->attach($conn);

  echo "New connection! ({$conn->resourceId})\n";
}
```

###onMessage
Essa função é chamada quando o evento de existe uma nova mensagem, e com ela, você pode encaminhar essas mensagens para todos ou para alguns clientes conectados ao socket

```php
public function onMessage(ConnectionInterface $from, $msg) {
    $numRecv = count($this->clients) - 1;
    echo sprintf('Connection %d sending message "%s" to %d other connection%s' . "\n"
        , $from->resourceId, $msg, $numRecv, $numRecv == 1 ? '' : 's');
    $dados = json_decode($msg, true);
    $dados['id'] = $from->resourceId;

    foreach ($this->clients as $client) {

            // The sender is not the receiver, send to each client connected
            $client->send(json_encode($dados));

    }
}
```

###onClose
Essa função é chamada assim que um um cliente se desconecta do websocket, e a partir dele, você pode atribuir callbacks para esse evento.

```php
public function onClose(ConnectionInterface $conn) {
    // The connection is closed, remove it, as we can no longer send it messages
    $this->clients->detach($conn);

    echo "Connection {$conn->resourceId} has disconnected\n";
}
```

###onError
Esse evento é disparado quando ocorre um erro interno no servidor.

```php
public function onError(ConnectionInterface $conn, \Exception $e) {
    echo "An error has occurred: {$e->getMessage()}\n";

    $conn->close();
}
```

###Executar
Para executar o Socket basta você acessar o seu terminal e executar o script que serve como servidor, pois se inicia-lo via browser, o seu socket não funcionará.

```
php -q path-to-server/server.php
```

Veja o Exemplo completo no [github](https://github.com/gustavobeavis/ws_react_basic).

##Javascript
Uma vez criado o socket, você pode acessa-lo do seu browser, lembrando que para acessa-lo de forma nativa, só browsers modernos suportem a API websocket, do contrário, existem diversas saídas, como flashsocket e javasocket, no qual você pode acessa-lo do seu script.

Assim como no caso do servidor, teremos funções com callbacks para todos os eventos de um socket, que são:

```javascript
var wsUri = "ws://localhost:1234";
var connection = new WebSocket(wsUri);
// When the connection is open, send some data to the server
connection.onopen = function () {
  connection.send('Ping'); // Send the message 'Ping' to the server
};

// Log errors
connection.onerror = function (error) {
  console.log('WebSocket Error ' + error);
};

// Log messages from the server
connection.onmessage = function (e) {
  console.log('Server: ' + e.data);
};
```

Eu fiz um exemplo completo usando Angular js, que encontra-se no [github](https://github.com/gustavobeavis/ws_angular).

##Vantagens e desvantagens
A vantagem de usar um websoket é que ao invés de termos muitos clientes fazendo requisições periódicas http ao mesmo endpoint para verificar possíveis mudanças de estado da aplicação, o socket que irá gerenciar as alterações e mandar para o(os) clientes essas atualizações logo que elas ocorrerem, evitando um tráfego desnecessário de requisições http, como consultas de estado sem mudança ou ainda muitos clientes consultando a mesma coisa. Em contra partida, teremos um script executando um loop infinito no servidor, aguardando algum evento para transmitir para os usuários, ou seja, embora você resolva a latência, você terá um processo que irá consumir recursos do servidor durante todo tempo de que você mante-lo rodando, independente se houver ou não novos eventos, solicitações ou usuários ativos.

Além disso, podemos ter situações onde o navegador do cliente não tenha suporte websoket, flash ou java. Nesses casos, não teremos como evitar o uso de long pooling.

O Websoket é o futuro, porém, o PHP não é a melhor saída para esse tipo de recurso, embora funcione bem na maioria dos casos. Eu aconselho o uso de Erlang ou Node JS, por serem linguagens de natureza não blocante e por lidarem melhor com concorrência.

##Aplicações
Os websockets podem ter muitas aplicações, sobre tudo, em:

- Chats
- Games multi-player online
- Sistemas de sincronização mobile/Web

##Referências
[HTML5 Rocks – Apresentando WebSockets – PT](http://www.html5rocks.com/pt/tutorials/websockets/basics/)

[Sergio Lopes – SPYD e HTTP 2.0 – PT](Sergio%20Lopes%20%E2%80%93%20SPYD%20e%20HTTP%202.0%20%E2%80%93%20PT)

[JustCode – Otimizando Performance com Compactação Gzip/Deflate](http://blog.glaucocustodio.com/2012/09/22/otimizando-performance-com-compactacao-gzip-deflate/)

[Sanwebe – mple Chat Using WebSocket and PHP Socket – EN](http://www.sanwebe.com/2013/05/chat-using-websocket-php-socket)

[Ratchet – Lib](http://socketo.me/)

[socket.io](http://socket.io/)

