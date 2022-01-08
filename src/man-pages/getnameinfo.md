# 9.8 `getnameinfo()`

与えられた `struct sockaddr` のホスト名とサービス名の情報を検索します。

### 9.8.1 書式

```c
#include <sys/socket.h>
#include <netdb.h>

int getnameinfo(const struct sockaddr *sa, socklen_t salen,
                char *host, size_t hostlen,
                char *serv, size_t servlen, int flags);
```

### 9.8.2 解説

この関数は `getaddrinfo()` の逆で、すでにロードされている `struct sockaddr` を受け取り、その名前とサービス名のルックアップを行う関数です。これは、古い `gethostbyaddr()` と `getservbyport()` 関数を置き換えるものです。

`sa` パラメータに `struct sockaddr`（実際には `struct sockaddr_in` または `struct sockaddr_in6` をキャストしたもの）へのポインタを渡し、`salen` にその `struct` の長さを渡す必要があります。

結果として得られるホスト名とサービス名は、`host` と `serv` パラメータで指定された領域に書き込まれます。もちろん、これらのバッファの最大長を `hostlen` と `servlen` で指定する必要があります。

最後に、渡すことのできるフラグがいくつかありますが、ここでは良いものを2つ紹介します。`NI_NOFQDN` は `host` にドメイン名全体ではなく、ホスト名のみを含めるようにする。`NI_NAMEREQD` は、DNS ルックアップで名前が見つからない場合に、関数を失敗させます（このフラグを指定せず、名前が見つからない場合は、`getnameinfo()` が代わりに IP アドレスの文字列を `host` に格納します。）

いつも通り、ローカルの man ページで全容を確認してください。

### 9.8.3 返り値

成功した場合は `0` を、エラーが発生した場合は `0` 以外を返します。戻り値が `0` でない場合、それを `gai_strerror()` に渡すと、人間が読める文字列を得ることができます。詳しくは `getaddrinfo` を参照してください。

### 9.8.4 例

```c
struct sockaddr_in6 sa; // could be IPv4 if you want
char host[1024];
char service[20];

// pretend sa is full of good information about the host and port...

getnameinfo(&sa, sizeof sa, host, sizeof host, service, sizeof service, 0);

printf("   host: %s\n", host);    // e.g. "www.example.com"
printf("service: %s\n", service); // e.g. "http"
```

### 9.8.5 参照

[`getaddrinfo()`](./getaddrinfo-freeaddrinfo-gai_strerror.md),
[`gethostbyaddr()`](./gethostbyname-gethostbyaddr.md)
