# 9.18 `recv()`, `recvfrom()`

ソケットでデータを受信します。

### 9.18.1 書式

```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recv(int s, void *buf, size_t len, int flags);
ssize_t recvfrom(int s, void *buf, size_t len, int flags,
                 struct sockaddr *from, socklen_t *fromlen);
```

### 9.18.2 解説

ソケットを立ち上げて接続したら、`recv()`（TCP `SOCK_STREAM` ソケットの場合）と `recvfrom()`（UDP `SOCK_DGRAM` ソケットの場合）を使ってリモート側から受信したデータを読み込むことができるようになります。

どちらの関数も、ソケット記述子 `s`、バッファへのポインタ `buf`、バッファのサイズ（バイト数）`len`、そして関数の動作を制御するための `flags` を受け取ります。

さらに、`recvfrom()` は `struct sockaddr*` を受け取り、`from` はデータがどこから来たかを示し、 `fromlen` には `struct sockaddr` のサイズを記入することになります。（また、`fromlen` を `from` または `struct sockaddr` のサイズに初期化する必要があります。）

では、この関数にどんな不思議なフラグを渡すことができるのでしょうか？以下にそのいくつかを紹介しますが、より詳細な情報やあなたのシステムで実際にサポートされているものについては、ローカルのマニュアルページをチェックする必要があります。また、通常のバニラの `recv() ` にしたい場合は、`flags` を `0` に設定します。

| Macro         | 解説                                             |
|---------------|----------------------------------------------------------|
| `MSG_OOB`     | Out of Band データを受信します。これは、`send()` で `MSG_OOB` フラグを指定して送信されたデータを取得する方法です。受信側としては、緊急のデータがあることを伝えるシグナル `SIGURG` が発生しているはずです。このシグナルに対するハンドラで、この `MSG_OOB` フラグを指定して `recv()` を呼び出すことができます。|
| `MSG_PEEK`    | もし `recv()` を "見せかけだけ" に呼び出したい場合は、このフラグを付けて呼び出すことができます。これは、`recv()` を "本当の意味で"（つまり `MSG_PEEK` フラグなしで）呼び出したときに、バッファの中で何が待っているかを教えてくれるものです。これは、次の `recv()` 呼び出しに対するスニークプレビューのようなものです。|
| `MSG_WAITALL` | `recv()` に、`len` パラメータで指定したデータがすべて揃うまで戻らないように指示します。しかし、シグナルによって呼び出しが中断された場合、何らかのエラーが発生した場合、リモート側が接続を閉じた場合など、極端な状況下ではあなたの希望を無視することができます。怒らないでください。|

`recv()` を呼び出すと、読み込むべきデータがあるまでブロックされます。ブロックしたくない場合は、ソケットをノンブロッキングに設定するか、`select()` や `poll()` で受信データがあるかどうかを確認してから `recv()` や `recvfrom()` を呼び出してください。

### 9.18.3 返り値

実際に受け取ったバイト数（`len` パラメータで指定した値よりも少ないかもしれません）を返します。エラーの場合は `-1` を返します（それに応じて `errno` がセットされます）。

リモート側が接続を閉じていた場合、`recv()` は `0` を返します。これは、リモート側が接続を閉じたかどうかを判断するための通常の方法です。正常なのは良いことだ、反乱軍！

### 9.18.4 例

```c
// stream sockets and recv()

struct addrinfo hints, *res;
int sockfd;
char buf[512];
int byte_count;

// get host info, make socket, and connect it
memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;  // use IPv4 or IPv6, whichever
hints.ai_socktype = SOCK_STREAM;
getaddrinfo("www.example.com", "3490", &hints, &res);
sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
connect(sockfd, res->ai_addr, res->ai_addrlen);

// all right! now that we're connected, we can receive some data!
byte_count = recv(sockfd, buf, sizeof buf, 0);
printf("recv()'d %d bytes of data in buf\n", byte_count);
```

```c
// datagram sockets and recvfrom()

struct addrinfo hints, *res;
int sockfd;
int byte_count;
socklen_t fromlen;
struct sockaddr_storage addr;
char buf[512];
char ipstr[INET6_ADDRSTRLEN];

// get host info, make socket, bind it to port 4950
memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;  // use IPv4 or IPv6, whichever
hints.ai_socktype = SOCK_DGRAM;
hints.ai_flags = AI_PASSIVE;
getaddrinfo(NULL, "4950", &hints, &res);
sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
bind(sockfd, res->ai_addr, res->ai_addrlen);

// no need to accept(), just recvfrom():

fromlen = sizeof addr;
byte_count = recvfrom(sockfd, buf, sizeof buf, 0, &addr, &fromlen);

printf("recv()'d %d bytes of data in buf\n", byte_count);
printf("from IP address %s\n",
    inet_ntop(addr.ss_family,
        addr.ss_family == AF_INET?
            ((struct sockadd_in *)&addr)->sin_addr:
            ((struct sockadd_in6 *)&addr)->sin6_addr,
        ipstr, sizeof ipstr);
```

### 9.18.5 参照

[`send()`](./send-sendto.md),
[`sendto()`](./send-sendto.md),
[`select()`](./select.md),
[`poll()`](./poll.md),
[Blocking](../slightly-advanced-techniques/blocking.md)
