# 3.3 構造体

さて、ついにここまで来ました。そろそろプログラミングの話をしましょう。この章では、ソケットインターフェイスで使用される様々なデータ型について説明します。

まず、簡単なものからです。ソケットディスクリプタです。ソケットディスクリプタは以下のような型です。

```c
int
```

普通の `int` です。

ここからは変な話なので、我慢して読んでください。

My First Struct™---`struct addrinfo`。この構造体は最近開発されたもので、ソケットアドレス構造体を後で使用するために準備するために使用されます。また、ホスト名のルックアップやサービス名のルックアップにも使用されます。これは、後で実際の使い方を説明するときに、より意味をなすと思いますが、今は、接続を行うときに最初に呼び出されるものの1つであることを知っておいてください。

```c
struct addrinfo {
    int              ai_flags;     // AI_PASSIVE, AI_CANONNAME, etc.
    int              ai_family;    // AF_INET, AF_INET6, AF_UNSPEC
    int              ai_socktype;  // SOCK_STREAM, SOCK_DGRAM
    int              ai_protocol;  // use 0 for "any"
    size_t           ai_addrlen;   // size of ai_addr in bytes
    struct sockaddr *ai_addr;      // struct sockaddr_in or _in6
    char            *ai_canonname; // full canonical hostname

    struct addrinfo *ai_next;      // linked list, next node
};
```

この構造体を少し読み込んでから、`getaddrinfo()` を呼び出します。この構造体のリンクリストへのポインタが返され、必要なものがすべて満たされます。

`ai_family` フィールドで IPv4 か IPv6 を使うように強制することもできますし、`AF_UNSPEC` のままにして何でも使えるようにすることも可能です。これは、あなたのコードが IP バージョンに依存しないので、クールです。

これはリンクされたリストであることに注意してください：`ai_next` は次の要素を指しています---そこから選択するためにいくつかの結果があるかもしれません。私は最初にうまくいった結果を使いますが、あなたは異なるビジネスニーズを持っているかもしれません。何でもかんでも知ってるわけじゃないんです！

`struct addrinfo` の `ai_addr` フィールドは `struct sockaddr` へのポインタであることがわかります。ここからが、IP アドレス構造体の中身についての細かい話になります。

通常、これらの構造体に書き込む必要はありません。多くの場合、`addrinfo` 構造体を埋めるために `getaddrinfo()` を呼び出すだけでよいでしょう。しかし、これらの構造体の内部を覗いて値を取得する必要があるため、ここでそれらを紹介します。

(また、構造体 `addrinfo` が発明される前に書かれたコードはすべて、これらのものをすべて手作業で梱包していたので、まさにそのような IPv4 コードを多く見かけることができます。このガイドの古いバージョンなどでもそうです)。

ある構造体は IPv4 で、ある構造体は IPv6 で、ある構造体はその両方です。どれが何なのか、メモしておきます。

とにかく、構造体 `sockaddr` は、多くの種類のソケットのためのソケットアドレス情報を保持します。

```c
struct sockaddr {
    unsigned short    sa_family;    // address family, AF_xxx
    char              sa_data[14];  // 14 bytes of protocol address
};
```

`sa_family` には様々なものを指定できますが、この文書ではすべて `AF_INET` (IPv4) または `AF_INET6` (IPv6) とします。`sa_data` にはソケットの宛先アドレスとポート番号を指定します。`sa_data` にアドレスを手で詰め込むのは面倒なので、これはかなり扱いにくいです。

構造体 `sockaddr` を扱うために、プログラマは IPv4 で使用する構造体 `sockaddr_in`（"in" は "Internet" の意）を並列に作成しました。

`sockaddr_in` 構造体へのポインタは `sockaddr` 構造体へのポインタにキャストすることができ、その逆も可能です。つまり、`connect()` が `struct sockaddr*` を要求しても、`struct sockaddr_in` を使用して、最後の最後でキャストすることができるのです！

```c
// (IPv4 only--see struct sockaddr_in6 for IPv6)

struct sockaddr_in {
    short int          sin_family;  // Address family, AF_INET
    unsigned short int sin_port;    // Port number
    struct in_addr     sin_addr;    // Internet address
    unsigned char      sin_zero[8]; // Same size as struct sockaddr
};
```

この構造体により、ソケットアドレスの要素を簡単に参照することができます。`sin_zero` (構造体を `struct sockaddr` の長さに合わせるために含まれます) は、関数 `memset()` ですべて 0 に設定する必要があることに注意すること。また、`sin_family` は `struct sockaddr` の `sa_family` に相当し、"`AF_INET`" に設定されることに注意します。最後に、`sin_port` はネットワークバイトオーダーでなければなりません（`htons()` を使用することで！）。

もっと掘り下げましょう！`sin_addr` フィールドは `in_addr` 構造体であることがわかりますね。あれは何なんだ？まあ、大げさではなく、史上最も恐ろしい組合せの1つです。

```c
// (IPv4 only--see struct in6_addr for IPv6)

// Internet address (a structure for historical reasons)
struct in_addr {
    uint32_t s_addr; // that's a 32-bit int (4 bytes)
};
```

うおぉ まあ、昔はユニオンだったんだけど、今はもうそういう時代じゃないみたいだね。おつかれさまでした。つまり、`ina` を `struct sockaddr_in` 型と宣言した場合、`ina.sin_addr.s_addr` は4バイトの IP アドレス（ネットワークバイトオーダー）を参照することになります。あなたのシステムがまだ `struct in_addr` のための神々しいユニオンを使用している場合でも、あなたはまだ私が上記のように全く同じ方法で4バイトの IP アドレスを参照することができます（これは `#defines` によるものです）ことに注意してください。

IPv6 ではどうでしょうか。これについても同様の構造体が存在します。

```c
// (IPv6 only--see struct sockaddr_in and struct in_addr for IPv4)

struct sockaddr_in6 {
    u_int16_t       sin6_family;   // address family, AF_INET6
    u_int16_t       sin6_port;     // port number, Network Byte Order
    u_int32_t       sin6_flowinfo; // IPv6 flow information
    struct in6_addr sin6_addr;     // IPv6 address
    u_int32_t       sin6_scope_id; // Scope ID
};

struct in6_addr {
    unsigned char   s6_addr[16];   // IPv6 address
};
```

IPv4 が IPv4 アドレスとポート番号を持つように、IPv6 も IPv6 アドレスとポート番号を持つことに注意してください。

また、IPv6 フロー情報やスコープ ID のフィールドについては、今のところ触れないことに注意してください。`:-)`

最後になりますが、こちらもシンプルな構造体である `struct sockaddr_storage` は、IPv4 と IPv6 の両方の構造体を保持できるように十分な大きさに設計されています。コールによっては、`struct sockaddr` に IPv4 と IPv6 のどちらのアドレスが記入されるのか事前にわからないことがありますよね。そこで、この並列構造体を渡しますが、サイズが大きい以外は `struct sockaddr` とよく似ており、必要な型にキャストします。

```c
struct sockaddr_storage {
    sa_family_t  ss_family;     // address family

    // all this is padding, implementation specific, ignore it:
    char      __ss_pad1[_SS_PAD1SIZE];
    int64_t   __ss_align;
    char      __ss_pad2[_SS_PAD2SIZE];
};
```

重要なのは、`ss_family` フィールドでアドレスファミリーを確認できることで、これが `AF_INET` か `AF_INET6`（IPv4 か IPv6 か）かを確認することです。それから、必要なら `struct sockaddr_in` や `struct sockaddr_in6` にキャストすることができます。
