# 3.4 IP アドレス、パート2

幸いなことに、IP アドレスを操作するための関数がたくさんあります。手書きで把握して `<<` 演算子で `long` に詰め込む必要はありません。

まず、`struct sockaddr_in ina` があり、そこに格納したい IP アドレスが `10.12.110.57` または `2001:db8:63b3:1::3490` だとしましょう。`inet_pton()` という関数は、数字とドットで表記された IP アドレスを、`AF_INET` か `AF_INET6` の指定によって、`in_addr` 構造体か `in6_addr` 構造体に変換する関数です。("`pton`" は "presentation to network" の略で、覚えやすければ "printable to network" と呼んでも構いません)。変換は次のように行うことができます。

```c
struct sockaddr_in sa; // IPv4
struct sockaddr_in6 sa6; // IPv6

inet_pton(AF_INET, "10.12.110.57", &(sa.sin_addr)); // IPv4
inet_pton(AF_INET6, "2001:db8:63b3:1::3490", &(sa6.sin6_addr)); // IPv6
```

(クイックメモ: 古い方法では、`inet_addr()` という関数や `inet_aton()` という別の関数を使っていましたが、これらはもう時代遅れで IPv6 では動きません。)

さて、上記のコードスニペットは、エラーチェックがないため、あまり堅牢ではありません。`inet_pton()` はエラー時に `-1` を返し、アドレスがめちゃくちゃになった場合は 0 を返します。ですから、使用する前に結果が 0 よりも大きいことを確認してください！

さて、これで文字列の IP アドレスをバイナリ表現に変換することができるようになりました。では、その逆はどうでしょうか？`in_addr` 構造体を持っていて、それを数字とドットの表記で印刷したい場合はどうでしょうか。(この場合、関数 `inet_ntop()` ("`ntop`" は "network to presentation" という意味です。覚えやすければ "network to printable" と呼んでも構いません) を次のように使用します。

```c
// IPv4:

char ip4[INET_ADDRSTRLEN];  // space to hold the IPv4 string
struct sockaddr_in sa;      // pretend this is loaded with something

inet_ntop(AF_INET, &(sa.sin_addr), ip4, INET_ADDRSTRLEN);

printf("The IPv4 address is: %s\n", ip4);


// IPv6:

char ip6[INET6_ADDRSTRLEN]; // space to hold the IPv6 string
struct sockaddr_in6 sa6;    // pretend this is loaded with something

inet_ntop(AF_INET6, &(sa6.sin6_addr), ip6, INET6_ADDRSTRLEN);

printf("The address is: %s\n", ip6);
```

呼び出す際には、アドレスの種類（IPv4 または IPv6）、アドレス、結果を格納する文字列へのポインタ、その文字列の最大長を渡すことになります。(2つのマクロは、最大の IPv4 または IPv6 アドレスを保持するために必要な文字列のサイズを都合よく保持します。`INET_ADDRSTRLEN` と `INET6_ADDRSTRLEN` です)。

(古いやり方についてもう一度簡単に触れておくと、この変換を行う歴史的な関数は `inet_ntoa()` と呼ばれるものでした。これも時代遅れで、IPv6 では動きません。)

最後に、これらの関数は数値の IP アドレスに対してのみ動作します。"`www.example.com`" のようなホスト名に対してネームサーバの DNS ルックアップは行いません。後ほど説明するように、そのためには `getaddrinfo()` を使用します。
