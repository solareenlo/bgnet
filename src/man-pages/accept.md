# 9.1 `accept()`

リスニング中のソケットで着信接続を承認します。

### 書式

```c
#include <sys/types.h>
#include <sys/socket.h>

int accept(int s, struct sockaddr *addr, socklen_t *addrlen);
```

### 解説

一旦、 `SOCK_STREAM` ソケットを取得して、 `listen()` で着信接続のための設定を行うという手間をかけると、そのソケットを使用することができます。次に `accept()` を呼び出して、新しく接続したクライアントとの通信に使用する新しいソケットディスクリプタを実際に取得します。

リスニングに使用している古いソケットはまだ残っています。そして、さらに `accept()` を呼び出す際に使用されます。

| パラメータ | 説明                                                  |
|------------|-------------------------------------------------------|
| `s`        | `listen()` 中のソケットディスクリプタ。               |
| `addr`     | ここには、接続先のサイトのアドレスが記入されています。|
| `addrlen`  | これは `addr` パラメータで返された構造体の `sizeof()` で埋められています。これは `addr` に渡された型なので、 `struct sockaddr_in` が返ってきたと仮定すれば、安全に無視することができます。|

`accept()` は通常ブロックされますが、`select()` を使用して、リスニング中のソケットディスクリプタが "ready to read" になっているかどうかを前もって確認することができます。もしそうなら、新しい接続が `accept()` を待っていることになります！やったー！あるいは、`fcntl()` を使って、リスニング中のソケットに `O_NONBLOCK` フラグを設定することもできます。そうすると、決してブロックせず、代わりに `errno` を `EWOULDBLOCK` に設定して `-1` を返します。

`accept()` が返すソケットディスクリプタは、正真正銘のソケットディスクリプタであり、リモートホストに開いて接続されています。それを使い終わったら `close()` しなければなりません。

### 返り値

`accept()` は新しく接続されたソケットディスクリプタを返すが、エラーの場合は `-1` を返し、 `errno` は適切に設定されます。

### 例

```c
struct sockaddr_storage their_addr;
socklen_t addr_size;
struct addrinfo hints, *res;
int sockfd, new_fd;

// first, load up address structs with getaddrinfo():

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;  // use IPv4 or IPv6, whichever
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;     // fill in my IP for me

getaddrinfo(NULL, MYPORT, &hints, &res);

// make a socket, bind it, and listen on it:

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
bind(sockfd, res->ai_addr, res->ai_addrlen);
listen(sockfd, BACKLOG);

// now accept an incoming connection:

addr_size = sizeof their_addr;
new_fd = accept(sockfd, (struct sockaddr *)&their_addr, &addr_size);

// ready to communicate on socket descriptor new_fd!
```

### 参照

[`socket()`](#socketman), [`getaddrinfo()`](#getaddrinfoman),
[`listen()`](#listenman), [`struct sockaddr_in`](#sockaddr_inman)
