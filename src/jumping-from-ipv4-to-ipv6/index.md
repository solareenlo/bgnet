# 4 IPv4 から IPv6 へのジャンプ

しかし、IPv6 で動作させるためには、私のコードのどこを変えればいいのか知りたいのです！今すぐ教えてください！

Ok! Ok!

ここに書かれていることはほとんどすべて、私が上で説明したことですが、せっかちな人のためのショートバージョンです。（もちろん、これ以外にもありますが、このガイドに該当するのはこれです。）

1. まず、構造体を手で詰めるのではなく、[`getaddrinfo()`](docs/ip-addresses-structs-and-data-munging/#structs) を使ってすべての `sockaddr` 構造体の情報を取得するようにしてください。こうすることで、IP のバージョンに左右されず、また、その後の多くのステップを省くことができます。

1. IP バージョンに関連する何かをハードコーディングしていることが分かったら、ヘルパー関数でラップするようにします。

1. `AF_INET` を `AF_INET6` に変更します。

1. `PF_INET` を `PF_INET6` に変更します。

1. `INADDR_ANY` の割り当てを `in6addr_any` の割り当てに変更し、若干の差異が生じます。

   ```c
   struct sockaddr_in sa;
   struct sockaddr_in6 sa6;

   sa.sin_addr.s_addr = INADDR_ANY;  // use my IPv4 address
   sa6.sin6_addr = in6addr_any; // use my IPv6 address
   ```

   Also, the value `IN6ADDR_ANY_INIT` can be used as an initializer when
   the `struct in6_addr` is declared, like so:

   ```c
   struct in6_addr ia6 = IN6ADDR_ANY_INIT;
   ```

1. `struct sockaddr_in` の代わりに `struct sockaddr_in6` を使用し、必要に応じてフィールドに "6" を追加してください（上記の [`struct`s](docs/ip-addresses-structs-and-data-munging/#structs) を参照）。`sin6_zero` フィールドはありません。

1. `struct in_addr` の代わりに `struct in6_addr` を使用し、必要に応じてフィールドに "6" を追加してください（上記の [`struct`s](docs/ip-addresses-structs-and-data-munging/#structs) を参照）。

1. `inet_aton()` や `inet_addr()` の代わりに、`inet_apton()` を使用してください。

1. `inet_ntoa()` の代わりに `inet_ntop()` を使用してください。

1. `gethostbyname()` の代わりに、優れた `getaddrinfo()` を使用してください。

1. `gethostbyaddr()` の代わりに、優れた `getnameinfo()` を使用してください（`gethostbyaddr()`は IPv6 でも動作可能です）。

1. `INADDR_BROADCAST` は動作しなくなりました。代わりに IPv6 マルチキャストを使用してください。

出来上がり！
