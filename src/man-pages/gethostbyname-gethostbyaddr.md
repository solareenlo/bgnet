# 9.7 `gethostbyname()`, `gethostbyaddr()`

ホスト名に対するIPアドレスの取得、またはその逆を行います。


### 9.7.1 書式

```c
#include <sys/socket.h>
#include <netdb.h>

struct hostent *gethostbyname(const char *name); // DEPRECATED!
struct hostent *gethostbyaddr(const char *addr, int len, int type);
```

### 9.7.2 解説

注意：この2つの関数は `getaddrinfo()` と `getnameinfo()` に取って代わられています！特に、`gethostbyname()` は IPv6 ではうまく動きません。

これらの関数は、ホスト名と IP アドレスの間を行き来します。例えば、"www.example.com" があれば、 `gethostbyname()` を使ってその IP アドレスを取得し、`struct in_addr` に格納することができます。

逆に、`struct in_addr` や `struct in6_addr` があれば、`gethostbyaddr()` を使用してホスト名を取得することができます。`gethostbyaddr()` は IPv6 互換ですが、より新しいピカピカな `getnameinfo()` を代わりに使用すべきです。

（ホスト名を調べたい IP アドレスをドットアンドナンバー形式で表した文字列がある場合、`AI_CANONNAME` フラグを指定して `getaddrinfo()` を使った方がよいでしょう。）

`gethostbyname()` は "www.yahoo.com" のような文字列を受け取り、IP アドレスを含む膨大な情報を含む `struct hostent` を返します。（他の情報は、正式なホスト名、エイリアスのリスト、アドレスの種類、アドレスの長さ、そしてアドレスのリストです---これは汎用の構造体で、一度方法を見れば我々の特定の目的のために使うのはかなり簡単です。）

`gethostbyaddr()` は `struct in_addr` または `struct in6_addr` を受け取って、対応するホスト名を表示します（もし存在すれば）。したがって、`gethostbyname()` の逆バージョンということになります。パラメータについては、 `addr` は `char*`、実際には `struct in_addr` へのポインタを渡したいのです。`len` は `sizeof(struct in_addr)` で、`type` は `AF_INET` であるべきです。

では、返される `struct hostent` とは何なのでしょうか？これは、問題のホストに関する情報を含むいくつかのフィールドを持っています。

| Field                | 解説                                              |
|----------------------|---------------------------------------------------|
| `char *h_name`       | 実際の正規のホスト名。                            |
| `char **h_aliases`   | 配列でアクセスできるエイリアスのリスト--- 最後の要素は `NULL` です。|
| `int h_addrtype`     | 結果のアドレスの型。この目的のためには、本当は `AF_INET` でなければなりません。|
| `int length`         | アドレスの長さをバイト数で表したもので、IP（バージョン4）アドレスの場合は4となります。|
| `char **h_addr_list` | このホストの IP アドレスのリスト。これは `char**` ですが、実際には `struct in_addr*` の配列で、偽装されています。配列の最後の要素は `NULL` です。|
| `h_addr`             | `h_addr_list[0]` の共通定義エイリアスです。もし、このホストの古い IP アドレスが欲しいだけなら（ええ、複数持つこともできます)、このフィールドを使えばいいのです。|

### 9.7.3 返り値

成功すれば `struct hostent` へのポインタを、エラーなら `NULL` を返します。

通常の `perror()` やエラー報告に使うようなものの代わりに、これらの関数は `h_errno` という変数に並列の結果を持ち、`herror()` や `hstrerror()` という関数を使って表示することができるようになりました。 これらの関数は、通常の `errno`、`perror()`、`strerror()` 関数と同じように動作します。

### 9.7.4 例

```c
// THIS IS A DEPRECATED METHOD OF GETTING HOST NAMES
// use getaddrinfo() instead!

#include <stdio.h>
#include <errno.h>
#include <netdb.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int main(int argc, char *argv[])
{
    int i;
    struct hostent *he;
    struct in_addr **addr_list;

    if (argc != 2) {
        fprintf(stderr,"usage: ghbn hostname\n");
        return 1;
    }

    if ((he = gethostbyname(argv[1])) == NULL) {  // get the host info
        herror("gethostbyname");
        return 2;
    }

    // print information about this host:
    printf("Official name is: %s\n", he->h_name);
    printf("    IP addresses: ");
    addr_list = (struct in_addr **)he->h_addr_list;
    for(i = 0; addr_list[i] != NULL; i++) {
        printf("%s ", inet_ntoa(*addr_list[i]));
    }
    printf("\n");

    return 0;
}
```

```c
// THIS HAS BEEN SUPERCEDED
// use getnameinfo() instead!

struct hostent *he;
struct in_addr ipv4addr;
struct in6_addr ipv6addr;

inet_pton(AF_INET, "192.0.2.34", &ipv4addr);
he = gethostbyaddr(&ipv4addr, sizeof ipv4addr, AF_INET);
printf("Host name: %s\n", he->h_name);

inet_pton(AF_INET6, "2001:db8:63b3:1::beef", &ipv6addr);
he = gethostbyaddr(&ipv6addr, sizeof ipv6addr, AF_INET6);
printf("Host name: %s\n", he->h_name);
```

### 9.7.5 参照

[`getaddrinfo()`](./getaddrinfo-freeaddrinfo-gai_strerror.md),
[`getnameinfo()`](./getnameinfo.md),
[`gethostname()`](./gethostname.md),
[`errno`](./errno.md),
[`perror()`](./perror-strerror.md),
[`strerror()`](./perror-strerror.md),
[`struct in_addr`](./struct-sockaddr-and-pals.md)
