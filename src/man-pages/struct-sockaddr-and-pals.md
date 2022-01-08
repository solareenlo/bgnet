# 9.24 `struct sockaddr` and pals

インターネットアドレスを扱うための構造体です。

### 9.24.1 書式

```c
#include <netinet/in.h>

// All pointers to socket address structures are often cast to pointers
// to this type before use in various functions and system calls:

struct sockaddr {
    unsigned short    sa_family;    // address family, AF_xxx
    char              sa_data[14];  // 14 bytes of protocol address
};


// IPv4 AF_INET sockets:

struct sockaddr_in {
    short            sin_family;   // e.g. AF_INET, AF_INET6
    unsigned short   sin_port;     // e.g. htons(3490)
    struct in_addr   sin_addr;     // see struct in_addr, below
    char             sin_zero[8];  // zero this if you want to
};

struct in_addr {
    unsigned long s_addr;          // load with inet_pton()
};


// IPv6 AF_INET6 sockets:

struct sockaddr_in6 {
    u_int16_t       sin6_family;   // address family, AF_INET6
    u_int16_t       sin6_port;     // port number, Network Byte Order
    u_int32_t       sin6_flowinfo; // IPv6 flow information
    struct in6_addr sin6_addr;     // IPv6 address
    u_int32_t       sin6_scope_id; // Scope ID
};

struct in6_addr {
    unsigned char   s6_addr[16];   // load with inet_pton()
};


// General socket address holding structure, big enough to hold either
// struct sockaddr_in or struct sockaddr_in6 data:

struct sockaddr_storage {
    sa_family_t  ss_family;     // address family

    // all this is padding, implementation specific, ignore it:
    char      __ss_pad1[_SS_PAD1SIZE];
    int64_t   __ss_align;
    char      __ss_pad2[_SS_PAD2SIZE];
};
```

### 9.24.2 解説

これらは、インターネット・アドレスを扱うすべてのシステムコールと関数の基本構造です。多くの場合、`getaddrinfo()` を使ってこれらの構造体を埋めておき、必要なときにそれを読み出すことになります。

メモリ上では、`struct sockaddr_in` と `struct sockaddr_in6` は `struct sockaddr` と同じ開始構造体を共有しており、一方の型のポインタを他方に自由にキャストしても、宇宙の終焉の可能性を除けば何の害もないのです。

もし、`struct sockaddr_in*` を `struct sockaddr*` にキャストしたときに宇宙が終わるとしたら、それは全くの偶然であり、心配する必要はないでしょう。

このことを念頭に置いて、関数が `struct sockaddr*` を受け取るときはいつでも、`struct sockaddr_in*`, `struct sockaddr_in6*`, または `struct sockadd_storage*` を簡単に安全にその型にキャストできることを覚えておいてください。

`sockaddr_in` 構造体は、IPv4 アドレス（例："192.0.2.10"）で使用される構造体です。アドレスファミリ（`AF_INET`）、ポート（`sin_port`）と IPv4 アドレス（`sin_addr`）を保持します。

また、`struct sockaddr_in` には `sin_zero` というフィールドがあり、ある人はこれを `0` に設定しなければならないと主張しています。他の人はそれについて何も主張していませんし（Linux のドキュメントでは全く言及されていません）、 `0` に設定することは実際には必要ないように思われます。ですから、もしあなたがそうしたいと思ったら、`memset()` を使って `0` にセットしてください。

さて、この `struct in_addr` は、システムによって奇妙な獣のようなものです。時には、あらゆる種類の `#define` やその他のナンセンスなものを含むクレイジーな `union` になることもあります。しかし、多くのシステムでは `s_addr` フィールドしか実装されていないので、この構造体の `s_addr` フィールドのみを使用する必要があります。

`struct sockadd_in6` と `struct in6_addr` は IPv6 で使用されることを除けば、非常によく似ています。

`struct sockaddr_storage` は、IP バージョンに依存しないコードを書こうとしているときに、新しいアドレスが IPv4 か IPv6 か分からない場合に `accept()` や `recvfrom()` に渡すことができる構造体です。`struct sockaddr_storage` 構造体は、元の小さな `struct sockaddr` とは異なり、両方のタイプを保持するのに十分な大きさを持っています。

### 9.24.3 例

```c
// IPv4:

struct sockaddr_in ip4addr;
int s;

ip4addr.sin_family = AF_INET;
ip4addr.sin_port = htons(3490);
inet_pton(AF_INET, "10.0.0.1", &ip4addr.sin_addr);

s = socket(PF_INET, SOCK_STREAM, 0);
bind(s, (struct sockaddr*)&ip4addr, sizeof ip4addr);
```

```c
// IPv6:

struct sockaddr_in6 ip6addr;
int s;

ip6addr.sin6_family = AF_INET6;
ip6addr.sin6_port = htons(4950);
inet_pton(AF_INET6, "2001:db8:8714:3a90::12", &ip6addr.sin6_addr);

s = socket(PF_INET6, SOCK_STREAM, 0);
bind(s, (struct sockaddr*)&ip6addr, sizeof ip6addr);
```

### 9.24.4 参照

[`accept()`](#acceptman), [`bind()`](#bindman), [`connect()`](#connectman),
[`inet_aton()`](#inet_ntoaman), [`inet_ntoa()`](#inet_ntoaman)
