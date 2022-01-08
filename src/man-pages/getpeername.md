# 9.9 `getpeername()`

リモート側の接続に関するアドレス情報を返す。

### 9.9.1 書式

```c
#include <sys/socket.h>

int getpeername(int s, struct sockaddr *addr, socklen_t *len);
```

### 9.9.2 解説

リモート接続を `accept()` したり、サーバに `connect()` したら、今度はピアとして知られるものを手に入れます。ピアとは、単に接続先のコンピュータのことで、IP アドレスとポートで識別されます。つまり...

`getpeername()` は単に、接続中のマシンに関する情報が詰まった `struct sockaddr_in` を返します。

なぜ"名前"なのか？このガイドで使っているようなインターネットソケットだけでなく、さまざまな種類のソケットがあるので、"名前"はすべてのケースをカバーする良い総称だったのです。この場合、相手の"名前"は相手の IP アドレスとポートです。

この関数は結果のアドレスのサイズを `len` で返しますが、`len` には `addr` のサイズをあらかじめ代入しておく必要があります。

### 9.9.3 返り値

成功した場合は `0` を、エラーの場合は `-1` を返します（それに応じて `errno` が設定されます。）

### 9.9.4 例

```c
// assume s is a connected socket

socklen_t len;
struct sockaddr_storage addr;
char ipstr[INET6_ADDRSTRLEN];
int port;

len = sizeof addr;
getpeername(s, (struct sockaddr*)&addr, &len);

// deal with both IPv4 and IPv6:
if (addr.ss_family == AF_INET) {
    struct sockaddr_in *s = (struct sockaddr_in *)&addr;
    port = ntohs(s->sin_port);
    inet_ntop(AF_INET, &s->sin_addr, ipstr, sizeof ipstr);
} else { // AF_INET6
    struct sockaddr_in6 *s = (struct sockaddr_in6 *)&addr;
    port = ntohs(s->sin6_port);
    inet_ntop(AF_INET6, &s->sin6_addr, ipstr, sizeof ipstr);
}

printf("Peer IP address: %s\n", ipstr);
printf("Peer port      : %d\n", port);
```

### 9.9.5 参照

[`gethostname()`](#gethostnameman), [`gethostbyname()`](#gethostbynameman),
[`gethostbyaddr()`](#gethostbynameman)
