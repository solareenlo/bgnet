# 9.5 `getaddrinfo()`, `freeaddrinfo()`, `gai_strerror()`

ホスト名やサービスに関する情報を取得し、その結果を `struct sockaddr` にロードします。

### 9.5.1 書式

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

int getaddrinfo(const char *nodename, const char *servname,
                const struct addrinfo *hints, struct addrinfo **res);

void freeaddrinfo(struct addrinfo *ai);

const char *gai_strerror(int ecode);

struct addrinfo {
  int     ai_flags;          // AI_PASSIVE, AI_CANONNAME, ...
  int     ai_family;         // AF_xxx
  int     ai_socktype;       // SOCK_xxx
  int     ai_protocol;       // 0 (auto) or IPPROTO_TCP, IPPROTO_UDP

  socklen_t  ai_addrlen;     // length of ai_addr
  char   *ai_canonname;      // canonical name for nodename
  struct sockaddr  *ai_addr; // binary address
  struct addrinfo  *ai_next; // next structure in linked list
};
```

### 9.5.2 解説

`getaddrinfo()` は特定のホスト名に関する情報（IPアドレスなど）を返し、`struct sockaddr` を読み込んで、細かい部分（IPv4 か IPv6 か）を処理してくれる優れた関数です。これは、古い関数である `gethostbyname()` と `getservbyname()` を置き換えるものです。以下の説明には、少し難しく感じるかもしれない多くの情報が含まれていますが、実際の使い方はとてもシンプルです。まずはサンプルを見てみるのもいいかもしれません。

興味のあるホスト名は `nodename` パラメータに入れます。アドレスは "www.example.com" のようなホスト名か、IPv4 または IPv6 のアドレス（文字列として渡される）のいずれかになります。このパラメータは、`AI_PASSIVE` フラグを使用している場合は `NULL` にすることもできます（下記参照）。

`servname` パラメータは、基本的にポート番号です。ポート番号（"80" のような文字列として渡される）、または "http"、"tftp"、"smtp"、"pop" などのサービス名であることができます。よく知られたサービス名は、[IANA ポートリスト](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)や、`/etc/services` ファイルで見つけることができます。

最後に、入力パラメータとして `hints` があります。これは `getaddrinfo()` 関数が何をしようとしているかを定義する場所です。`memset()` で使用する前に、構造体全体をゼロにします。それでは、使用する前に設定する必要があるフィールドを見てみましょう。

`ai_flags` には様々なものを設定することができますが、ここでは重要なものをいくつか紹介します。（複数のフラグを指定するには、`|` 演算子でビット単位の OR を指定します。）フラグの完全なリストについては、man ページをチェックしてください。

`AI_CANONNAME` は、結果の `ai_canonname` をホストの標準的な（実際の）名前で埋め尽くすようにします。`AI_PASSIVE` は、結果の IP アドレスを `INADDR_ANY`（IPv4）または `in6addr_any`（IPv6）で埋めます。これにより、その後の `bind()` への呼び出しで、`struct sockaddr` の IP アドレスが現在のホストのアドレスで自動的に埋められるようになります。これは、アドレスをハードコードしたくない場合に、サーバをセットアップするのに優れています。

もし、`AI_PASSIVE` フラグを使用する場合は、`nodename` に `NULL` を渡すことができます（`bind()` が後でそれを埋めてくれるからです）。

続けて、入力パラメータについてですが、おそらく `ai_family` を `AF_UNSPEC` に設定すると、`getaddrinfo()` に IPv4 と IPv6 の両方のアドレスを検索させることができるようになります。また、`AF_INET` や `AF_INET6` でどちらか一方に限定することもできます。

次に、`socktype` フィールドを `SOCK_STREAM` または `SOCK_DGRAM` のどちらかに設定する必要があります。

最後に、`ai_protocol` を `0` にしておくと、自動的にプロトコルの種類が選択されます。

さて、このようなものを全部入れたら、いよいよ `getaddrinfo()` を呼び出すことができます！

もちろん、ここからが楽しいところです。`res` は `struct addrinfo` のリンクリストを指すようになり、このリストを通して、hints で渡したものと一致するすべてのアドレスを取得することができるようになります。

さて、何らかの理由で動作しないアドレスを取得することは可能です。Linux の man ページでは、`socket()` と `connect()`（または `AI_PASSIVE` フラグでサーバをセットアップしている場合は `bind()`）を成功するまで呼び出すリストをループしています。

最後に、リンクリストを使い終わったら、`freeaddrinfo()` を呼んでメモリを解放する必要があります （さもないと、メモリがリークしてしまい、Some People が怒ることになります）。

### 9.5.3 返り値

成功した場合は `0` を、エラーが発生した場合は `0` 以外を返します。非ゼロを返した場合、関数 `gai_strerror()` を用いて、エラーコードのプリント可能なバージョンを戻り値に含めることができる。

### 9.5.4 例

```c
// code for a client connecting to a server
// namely a stream socket to www.example.com on port 80 (http)
// either IPv4 or IPv6

int sockfd;
struct addrinfo hints, *servinfo, *p;
int rv;

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC; // use AF_INET6 to force IPv6
hints.ai_socktype = SOCK_STREAM;

if ((rv = getaddrinfo("www.example.com", "http", &hints, &servinfo)) != 0) {
    fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv));
    exit(1);
}

// loop through all the results and connect to the first we can
for(p = servinfo; p != NULL; p = p->ai_next) {
    if ((sockfd = socket(p->ai_family, p->ai_socktype,
            p->ai_protocol)) == -1) {
        perror("socket");
        continue;
    }

    if (connect(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
        perror("connect");
        close(sockfd);
        continue;
    }

    break; // if we get here, we must have connected successfully
}

if (p == NULL) {
    // looped off the end of the list with no connection
    fprintf(stderr, "failed to connect\n");
    exit(2);
}

freeaddrinfo(servinfo); // all done with this structure
```

```c
// code for a server waiting for connections
// namely a stream socket on port 3490, on this host's IP
// either IPv4 or IPv6.

int sockfd;
struct addrinfo hints, *servinfo, *p;
int rv;

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC; // use AF_INET6 to force IPv6
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE; // use my IP address

if ((rv = getaddrinfo(NULL, "3490", &hints, &servinfo)) != 0) {
    fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv));
    exit(1);
}

// loop through all the results and bind to the first we can
for(p = servinfo; p != NULL; p = p->ai_next) {
    if ((sockfd = socket(p->ai_family, p->ai_socktype,
            p->ai_protocol)) == -1) {
        perror("socket");
        continue;
    }

    if (bind(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
        close(sockfd);
        perror("bind");
        continue;
    }

    break; // if we get here, we must have connected successfully
}

if (p == NULL) {
    // looped off the end of the list with no successful bind
    fprintf(stderr, "failed to bind socket\n");
    exit(2);
}

freeaddrinfo(servinfo); // all done with this structure
```

### 9.5.5 参照

[`gethostbyname()`](./gethostbyname-gethostbyaddr.md),
[`getnameinfo()`](./getnameinfo.md)
