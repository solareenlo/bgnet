# 9.23 `socket()`

ソケットディスクリプタをアロケートします。

### 9.23.1 書式

```c
#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

### 9.23.2 解説

ソケット的なことをするのに使える新しいソケットディスクリプタを返します。これは一般に、ソケットプログラムを書くための膨大なプロセスの最初の呼び出しとなり、その後の `listen()`, `bind()`, `accept()` や他の様々な関数の呼び出しでその結果を利用することができるようになります。

通常の使い方では、以下の例のように `getaddrinfo()` を呼び出して、これらのパラメータの値を取得します。しかし、本当に必要であれば、手動でこれらの値を埋めることができます。

| Macro      | 解説                                                |
|------------|-------------------------------------------------------------|
| `domain` | `domain` には、どのような種類のソケットに興味があるのかを記述します。これは様々なものがありますが、これはソケットガイドなので、IPv4では `PF_INET` 、IPv6では `PF_INET6` となります。|
| `type` | また、`type` パラメータには様々なものを指定できますが、信頼性の高い TCP ソケット（`send()`, `recv()`）には `SOCK_STREAM` を、信頼性の低い高速な UDP ソケット（`sendto()`, `recvfrom()`）には `SOCK_DGRAM` を設定することになると思われます。（もうひとつの興味深いソケットタイプは `SOCK_RAW` で、これはパケットを手動で構築するために使用することができます。これはかなりクールです。）|
| `protocol` | 最後に、`protocol` パラメータは、特定のソケットタイプでどのプロトコルを使用するかを指示します。既に述べたように、例えば `SOCK_STREAM` は TCP を使用します。幸いなことに、`SOCK_STREAM` や `SOCK_DGRAM` を使用する場合は、プロトコルを `0` に設定すれば、自動的に適切なプロトコルを使用することができます。そうでない場合は、`getprotobyname()` を使用して適切なプロトコル番号を調べることができます。|

### 9.23.3 返り値

それ以降の呼び出しで使用される新しいソケットディスクリプタ、またはエラーの場合は `-1`（それに応じて `errno` が設定されます）。

### 9.23.4 例

```c
struct addrinfo hints, *res;
int sockfd;

// first, load up address structs with getaddrinfo():

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;     // AF_INET, AF_INET6, or AF_UNSPEC
hints.ai_socktype = SOCK_STREAM; // SOCK_STREAM or SOCK_DGRAM

getaddrinfo("www.example.com", "3490", &hints, &res);

// make a socket using the information gleaned from getaddrinfo():
sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
```

### 9.23.5 参照

[`accept()`](#acceptman), [`bind()`](#bindman),
[`getaddrinfo()`](#getaddrinfoman), [`listen()`](#listenman)
