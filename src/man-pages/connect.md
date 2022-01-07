# 9.3 `connect()`

サーバーにソケットを接続します。

### 書式

```c
#include <sys/types.h>
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *serv_addr,
            socklen_t addrlen);
```

### 解説

`socket()` コールでソケットディスクリプタを構築したら、`connect()` システムコールを使って、そのソケットをリモートサーバに接続することができます。必要なことは、ソケットディスクリプタと、もっとよく知りたいと思うサーバのアドレスを渡すだけです。(それと、このような関数によく渡されるアドレスの長さも。)

通常、この情報は `getaddrinfo()` を呼び出した結果として得られますが、必要であれば自分自身で `struct sockaddr` を埋めることもできます。

ソケットディスクリプタの `bind()` をまだ呼んでいない場合、ソケットは自動的にあなたの IP アドレスとランダムなローカルポートに束縛されます。サーバーでない場合は、ローカルポートを気にする必要はなく、リモートポートを気にして `serv_addr` パラメータに設定するだけなので、通常はこれで十分です。クライアントソケットを特定の IP アドレスとポートにしたい場合は、`bind()` を呼び出すことができますが、これはかなりまれなケースです。

ソケットを `connect()` したら、あとは自由に `send()` や `recv()` をして、好きなだけデータを取り込めます。

特記事項：`SOCK_DGRAM` UDP ソケットをリモートホストに `connect()` した場合、`send()` と `recv()` だけでなく、`sendto()` と `recvfrom()` も使用できるようになります。もし必要なら。

### 返り値

成功した場合は `0` を、エラーの場合は `-1` を返す（それに応じて `errno` が設定されます）。

### 例

```c
// connect to www.example.com port 80 (http)

struct addrinfo hints, *res;
int sockfd;

// first, load up address structs with getaddrinfo():

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;  // use IPv4 or IPv6, whichever
hints.ai_socktype = SOCK_STREAM;

// we could put "80" instead on "http" on the next line:
getaddrinfo("www.example.com", "http", &hints, &res);

// make a socket:

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

// connect it to the address and port we passed in to getaddrinfo():

connect(sockfd, res->ai_addr, res->ai_addrlen);
```

### 参照

[`socket()`](#socketman), [`bind()`](#bindman)
