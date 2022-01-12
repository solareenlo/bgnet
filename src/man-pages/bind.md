# 9.2 `bind()`

ソケットをIPアドレスとポート番号に関連付けます。

### 9.2.1 書式

```c
#include <sys/types.h>
#include <sys/socket.h>

int bind(int sockfd, struct sockaddr *my_addr, socklen_t addrlen);
```

### 9.2.2 解説

リモートマシンがあなたのサーバプログラムに接続しようとするとき、IP アドレスとポート番号という2つの情報が必要です。`bind()` の呼び出しによって、まさにそれが可能になります。

まず、`getaddrinfo()` を呼び出して、宛先アドレスとポート情報を持つ `struct sockaddr` をロードします。それから `socket()` を呼んでソケット記述子を取得し、ソケットとアドレスを `bind()` に渡すと、IP アドレスとポートが魔法のように（実際の魔法を使って）ソケットにバインドされるのです！

IP アドレスを知らない場合、あるいはマシンに IP アドレスが 1 つしかないことが分かっている場合、あるいはマシンの IP アドレスがどれでも構わない場合は、`getaddrinfo()` の `hints` パラメータに `AI_PASSIVE` フラグを渡すだけでよいでしょう。これは `struct sockaddr` の IP アドレス部分を特別な値で埋めるもので、`bind()` に対して、このホストの IP アドレスを自動的に埋めるように指示するものです。

何、何？現在のホストのアドレスを自動入力するために、`struct sockaddr` の IP アドレスにどんな特別な値が読み込まれているのでしょうか？もしそうでなければ、上記のように `getaddrinfo()` からの結果を使用してください。IPv4 では、`struct sockaddr_in` 構造体の `sin_addr.s_addr` フィールドは `INADDR_ANY` に設定されます。IPv6 では、`struct sockaddr_in6` 構造体の `sin6_addr` フィールドは、グローバル変数 `in6addr_any` から代入されます。また、新しい `struct in6_addr` を宣言する場合は、`IN6ADDR_ANY_INIT` で初期化することができます。

最後に、`addrlen` パラメータに `sizeof my_addr` を設定します。

### 9.2.3 返り値

成功した場合は `0` を、エラーの場合は `-1` を返します（それに応じて `errno` が設定されます）。

### 9.2.4 例

```c
// modern way of doing things with getaddrinfo()

struct addrinfo hints, *res;
int sockfd;

// first, load up address structs with getaddrinfo():

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;  // use IPv4 or IPv6, whichever
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;     // fill in my IP for me

getaddrinfo(NULL, "3490", &hints, &res);

// make a socket:
// (you should actually walk the "res" linked list and error-check!)

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

// bind it to the port we passed in to getaddrinfo():

bind(sockfd, res->ai_addr, res->ai_addrlen);
```

```c
// example of packing a struct by hand, IPv4

struct sockaddr_in myaddr;
int s;

myaddr.sin_family = AF_INET;
myaddr.sin_port = htons(3490);

// you can specify an IP address:
inet_pton(AF_INET, "63.161.169.137", &(myaddr.sin_addr));

// or you can let it automatically select one:
myaddr.sin_addr.s_addr = INADDR_ANY;

s = socket(PF_INET, SOCK_STREAM, 0);
bind(s, (struct sockaddr*)&myaddr, sizeof myaddr);
```

### 9.2.5 参照

[`getaddrinfo()`](./getaddrinfo-freeaddrinfo-gai_strerror.md),
[`socket()`](./socket.md),
[`struct sockaddr_in`](./struct-sockaddr-and-pals.md),
[`struct in_addr`](./struct-sockaddr-and-pals.md)
