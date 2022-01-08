# 9.21 `send()`, `sendto()`

ソケットでデータを送信します。

### 9.21.1 書式

```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t send(int s, const void *buf, size_t len, int flags);
ssize_t sendto(int s, const void *buf, size_t len,
               int flags, const struct sockaddr *to,
               socklen_t tolen);
```

### 9.21.2 解説

これらの関数は、ソケットにデータを送信します。一般に、`send()` は TCP の `SOCK_STREAM` 接続型ソケットに、`sendto()` は UDP の `SOCK_DGRAM` 非接続型データグラムソケットに用います。非接続型ソケットでは、パケットを送信するたびにその宛先を指定する必要があり、そのため `sendto()` の最後のパラメータでパケットの行き先を定義しています。

`send()` と `sendto()` では、パラメータ `s` がソケット、`buf` が送信するデータへのポインタ、`len` が送信するバイト数、`flags` がデータの送信方法に関する詳細情報を指定することが可能です。通常の "データ" にしたい場合は、`flags` を `0` に設定します。以下は一般的に使用されるフラグですが、詳細はローカルの `send()` man ページを確認してください。

| Macro           | 解説                                           |
|-----------------|--------------------------------------------------------|
| `MSG_OOB` | "帯域外" データとして送信します。TCP はこれをサポートしており、このデータが通常のデータよりも優先順位が高いことを受信側のシステムに伝える方法です。受信側はシグナル `SIGURG` を受け取り、キューにある残りの通常のデータをすべて最初に受信することなく、このデータを受信することができます。|
| `MSG_DONTROUTE` | このデータはルータで送らず、ローカルに置いてください。|
| `MSG_DONTWAIT` | 送信トラフィックが詰まっているために `send()` がブロックされる場合、`EAGAIN` を返すようにします。これは "この送信のときだけノンブロックを有効にする" ようなものです。詳しくは [blocking](#blocking) の章を参照してください。|
| `MSG_NOSIGNAL` | もし `recv()` が終了したリモートホストに `send()` した場合、通常 `SIGPIPE` というシグナルを受け取ります。このフラグを追加することで、このシグナルが発生するのを防ぐことができます。|

### 9.21.3 返り値

実際に送信されたバイト数を返します。エラーの場合は `-1` を返します（それに応じて `errno` が設定されます）。実際に送信されたバイト数は、送信を要求したバイト数よりも少なくなる可能性があることに注意してください！これを回避するためのヘルパー関数については、[handling partial `send()`s](../slightly-advanced-techniques/handling-partial-sends.html) の章を参照してください。

また、どちらかの側でソケットが閉じられていた場合、`send()` を呼び出したプロセスはシグナル `SIGPIPE` を受け取ります。（ただし、`send()` が `MSG_NOSIGNAL` フラグ付きで呼び出された場合を除きます。）

### 9.21.4 例

```c
int spatula_count = 3490;
char *secret_message = "The Cheese is in The Toaster";

int stream_socket, dgram_socket;
struct sockaddr_in dest;
int temp;

// first with TCP stream sockets:

// assume sockets are made and connected
//stream_socket = socket(...
//connect(stream_socket, ...

// convert to network byte order
temp = htonl(spatula_count);
// send data normally:
send(stream_socket, &temp, sizeof temp, 0);

// send secret message out of band:
send(stream_socket, secret_message, strlen(secret_message)+1, MSG_OOB);

// now with UDP datagram sockets:
//getaddrinfo(...
//dest = ... // assume "dest" holds the address of the destination
//dgram_socket = socket(...

// send secret message normally:
sendto(dgram_socket, secret_message, strlen(secret_message)+1, 0,
       (struct sockaddr*)&dest, sizeof dest);
```

### 9.21.5 参照

[`recv()`](#recvman), [`recvfrom()`](#recvman)
