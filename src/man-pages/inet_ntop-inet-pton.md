# 9.14 `inet_ntop()`, `inet_pton()`

IPアドレスを人間が読める形に変換して戻します。

### 9.14.1 書式

```c
#include <arpa/inet.h>

const char *inet_ntop(int af, const void *src,
                      char *dst, socklen_t size);

int inet_pton(int af, const char *src, void *dst);
```

### 9.14.2 解説

これらの関数は、人間が読める IP アドレスを扱い、様々な関数やシステムコールで使用するためにバイナリ表現に変換するためのものです。"n" は "network"、"p" は "presentation" を表します。あるいは "text presentation" 。しかし、"printable" と考えてもよいです。"ntop" は "network to printable " です。ほらね？

IP アドレスを見るときに、2 進数の山を見たくないことがあります。`192.0.2.180` や `2001:db8:8714:3a90::12` のような、プリント可能な形式でありたいと思うことでしょう。そのような場合には、`inet_ntop()` を使用します。

`inet_ntop()` は `af` パラメータにアドレスファミリ（`AF_INET` または `AF_INET6`）を受け取ります。`src` パラメータには、文字列に変換したいアドレスを含む `struct in_addr` または `struct in6_addr` へのポインタを指定する必要があります。最後に `dst` と `size` は、変換先の文字列へのポインタと、その文字列の最大長を表します。

`dst` の文字列の最大長はどうすればよいのでしょうか？IPv4 アドレスと IPv6 アドレスの最大長はどのくらいですか？幸いなことに、あなたを助けてくれるマクロがいくつかあります。最大長は以下の通りです。`INET_ADDRSTRLEN` と `INET6_ADDRSTRLEN` です。

また、IP アドレスを可読形式で含む文字列を持っていて、それを `struct sockaddr_in` や `struct sockaddr_in6` に格納したい場合もあります。このような場合は、反対の関数 `inet_pton()` を使用します。

また、`inet_pton()` は `af` パラメータにアドレスファミリ（`AF_INET` または `AF_INET6`）を受け取ります。`src` パラメータには、IP アドレスをプリント可能な形式で格納した文字列へのポインタを指定します。最後に `dst` パラメータは、結果を格納する場所を指定します。これはおそらく `struct in_addr` または `struct in6_addr` となります。

これらの関数は DNS のルックアップを行いません---そのためには `getaddrinfo()` が必要です。

### 9.14.3 返り値

`inet_ntop()` は成功すると `dst` パラメータを返し、失敗すると `NULL` を返します（そして `errno` が設定されます）。

`inet_pton()` は成功すると `1` を返します。エラーがあった場合（`errno` が設定されます）は `-1` を返し、入力が有効な IP アドレスでなかった場合は `0` を返します。

### 9.14.4 例

```c
// IPv4 demo of inet_ntop() and inet_pton()

struct sockaddr_in sa;
char str[INET_ADDRSTRLEN];

// store this IP address in sa:
inet_pton(AF_INET, "192.0.2.33", &(sa.sin_addr));

// now get it back and print it
inet_ntop(AF_INET, &(sa.sin_addr), str, INET_ADDRSTRLEN);

printf("%s\n", str); // prints "192.0.2.33"
```

```c
// IPv6 demo of inet_ntop() and inet_pton()
// (basically the same except with a bunch of 6s thrown around)

struct sockaddr_in6 sa;
char str[INET6_ADDRSTRLEN];

// store this IP address in sa:
inet_pton(AF_INET6, "2001:db8:8714:3a90::12", &(sa.sin6_addr));

// now get it back and print it
inet_ntop(AF_INET6, &(sa.sin6_addr), str, INET6_ADDRSTRLEN);

printf("%s\n", str); // prints "2001:db8:8714:3a90::12"
```

```c
// Helper function you can use:

//Convert a struct sockaddr address to a string, IPv4 and IPv6:

char *get_ip_str(const struct sockaddr *sa, char *s, size_t maxlen)
{
    switch(sa->sa_family) {
        case AF_INET:
            inet_ntop(AF_INET, &(((struct sockaddr_in *)sa)->sin_addr),
                    s, maxlen);
            break;

        case AF_INET6:
            inet_ntop(AF_INET6, &(((struct sockaddr_in6 *)sa)->sin6_addr),
                    s, maxlen);
            break;

        default:
            strncpy(s, "Unknown AF", maxlen);
            return NULL;
    }

    return s;
}
```

### 9.14.5 参照

[`getaddrinfo()`](./getaddrinfo-freeaddrinfo-gai_strerror.md)
