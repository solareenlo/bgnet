# 9.12 `htons()`, `htonl()`, `ntohs()`, `ntohl()`

マルチバイト整数型をホストバイトオーダーからネットワークバイトオーダーに変換します。

### 9.12.1 書式

```c
#include <netinet/in.h>

uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);
```

### 9.12.2 解説

不幸なことに、コンピュータによってマルチバイト整数（つまり `char` より大きな整数）の内部でのバイト順序が異なっています。この結果、2 バイトの `short int` を Intel のコンピュータから Mac（Intel のコンピュータになる前）に `send()` すると、一方のコンピュータが `1` という数字だと思っていても、他方は `256` という数字だと思い、またその逆もしかりということになるのです。

この問題を回避する方法は、皆がそれぞれの違いを脇に置き、モトローラと IBM は正しく、インテルは奇妙なやり方をしたのだから、送信する前にバイトオーダーを "ビッグエンディアン" に変換することに同意することです。Intel は "リトルエンディアン" マシンなので、私たちが好むバイト順序を "ネットワークバ イトオーダー" と呼ぶ方が、はるかに政治的に正しいのです。そこで、これらの関数は、ネイティブのバイトオーダーからネットワークバイトオーダーに変換し、また元に戻します。

（つまり、Intel ではこれらの関数はすべてのバイトを入れ替えますが、PowerPC ではバイトがすでにネットワークバイトオーダーになっているため、何もしません。しかし、Intel マシンでビルドしても正常に動作するようにしたい人がいるかもしれないので、いずれにせよコードでは常にこれらを使用する必要があります。）

32 ビット（4バイト、おそらく `int`）と 16 ビット（2バイト、おそらく `short`）の数値が含まれることに注意してください。64 ビットマシンには 64 ビットの `int` 用の `htonll()` があるかもしれませんが、私は見たことがありません。自分で書くしかないでしょう。

とにかく、これらの関数の動作は、まず、ホスト（あなたのマシン）のバイトオーダーから変換するのか、ネットワークのバイトオーダーから変換するのかを決定することです。もし "ホスト" なら、呼び出す関数の最初の文字は "h" です。そうでなければ、"network" の場合は "n" です。関数名の真ん中は、ある "もの" から別の "もの" に変換するため、常に "to" であり、最後の1文字は何に変換するかを示しています。最後の文字はデータの大きさを表し、`sort` の場合は "s"、`long` の場合は "l" です。このように

| 関数      | 解説                          |
|-----------|-------------------------------|
| `htons()` | `h`ost `to` `n`etwork `s`hort |
| `htonl()` | `h`ost `to` `n`etwork `l`ong  |
| `ntohs()` | `n`etwork `to` `h`ost `s`hort |
| `ntohl()` | `n`etwork `to` `h`ost `l`ong  |

### 9.12.3 返り値

各関数は変換後の値を返します。

### 9.12.4 例

```c
uint32_t some_long = 10;
uint16_t some_short = 20;

uint32_t network_byte_order;

// convert and send
network_byte_order = htonl(some_long);
send(s, &network_byte_order, sizeof(uint32_t), 0);

some_short == ntohs(htons(some_short)); // this expression is true
```
