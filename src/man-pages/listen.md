# 9.15 `listen()`

ソケットに着信を待つように指示します。

### 9.15.1 書式

```c
#include <sys/socket.h>

int listen(int s, int backlog);
```

### 9.15.2 解説

ソケットディスクリプタ（`socket()`システムコールで作成）を受け取り、着信接続をリッスンするように指示することができます。これがサーバとクライアントを区別するポイントです、みんな。

`backlog` パラメータはシステムによって異なる意味を持ちますが、大まかに言うと、カーネルが新しい接続を拒否し始めるまでに保留できる接続の数です。つまり、新しい接続が来たら、バックログがいっぱいにならないように、素早く `accept()` する必要があります。10 程度に設定してみて、もしクライアントが高負荷で "Connection refused" を受け始めたら、もっと高く設定してみてください。

`listen()` を呼び出す前に、サーバは `bind()` を呼び出して特定のポート番号に自分自身をアタッチする必要があります。そのポート番号（サーバの IP アドレス上）は、クライアントが接続するポート番号になります。

### 9.15.3 返り値

成功した場合は `0` を、エラーの場合は `-1` を返します（それに応じて `errno` が設定されます）。

### 9.15.4 例

```c
struct addrinfo hints, *res;
int sockfd;

// first, load up address structs with getaddrinfo():

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;  // use IPv4 or IPv6, whichever
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;     // fill in my IP for me

getaddrinfo(NULL, "3490", &hints, &res);

// make a socket:

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

// bind it to the port we passed in to getaddrinfo():

bind(sockfd, res->ai_addr, res->ai_addrlen);

listen(sockfd, 10); // set s up to be a server (listening) socket

// then have an accept() loop down here somewhere
```

### 9.15.5 参照

[`accept()`](./accept.md),
[`bind()`](./bind.md),
[`socket()`](./socket.md)
