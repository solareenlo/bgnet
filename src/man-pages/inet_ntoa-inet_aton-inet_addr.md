# 9.13 `inet_ntoa()`, `inet_aton()`, `inet_addr`

IP アドレスをドットアンドナンバーの文字列から `struct in_addr` に変換し、元に戻します。

### 9.13.1 書式

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

// ALL THESE ARE DEPRECATED! Use inet_pton()  or inet_ntop() instead!!

char *inet_ntoa(struct in_addr in);
int inet_aton(const char *cp, struct in_addr *inp);
in_addr_t inet_addr(const char *cp);
```

### 9.13.2 解説

これらの関数は IPv6 を扱えないため、非推奨です！代わりに `inet_ntop()` または `inet_pton()` を使ってください！これらの関数がここに含まれているのは、まだ野放しになっていることがあるからです。

これらの関数はすべて `struct in_addr`（おそらく `struct sockaddr_in` の一部）からドットアンドナンバー形式の文字列（例："192.168.5.10"）に変換し、その逆もまた可能です。コマンドラインなどで IP アドレスが渡された場合、`connect()` の相手となる `struct in_addr` を得るには、これが一番簡単な方法だと思います。もっと強力なものが必要なら、`gethostbyname()` のような DNS 関数を試してみたり、自分の住んでいる国でクーデターを起こしてみたりしてください。

関数 `inet_ntoa()` は `struct in_addr` に格納されたネットワークアドレスをドットアンドナンバー形式の文字列に変換します。"ntoa" の "n" はネットワークを表し、"a" は歴史的な理由から ASCII を表しています（つまり "Network To ASCII" です。"toa" というサフィックスは C ライブラリに `atoi()` という ASCII 文字列に変換する類似の友人があります）。

関数 `inet_aton()` はその逆で、ドットや数字を含む文字列から `in_addr_t`（これは `struct in_addr` のフィールド `s_addr` の型）に変換します。

最後に、関数 `inet_addr()` は古い関数で、基本的に `inet_aton()` と同じことをします。理論的には非推奨ですが、よく目にしますし、使っても警察には捕まりません。

### 9.13.3 返り値

`inet_aton()` はアドレスが有効な場合は `0` 以外を返し、アドレスが無効な場合は `0` を返します。

`inet_ntoa()` はドットアンドナンバーズの文字列を静的バッファに格納し、この関数を呼び出すたびに上書きされるように返します。

`net_addr()` はアドレスを `in_addr_t` として返し、エラーがある場合は `-1` を返します。（これは、有効な IP アドレスである "`255.255.255.255`" という文字列を変換しようとした場合と同じ結果です。これが `inet_aton()` の方が良い理由です。）

### 9.13.4 例

```c
struct sockaddr_in antelope;
char *some_addr;

inet_aton("10.0.0.1", &antelope.sin_addr); // store IP in antelope

some_addr = inet_ntoa(antelope.sin_addr); // return the IP
printf("%s\n", some_addr); // prints "10.0.0.1"

// and this call is the same as the inet_aton() call, above:
antelope.sin_addr.s_addr = inet_addr("10.0.0.1");
```

### 9.13.5 参照

[`inet_ntop()`](#inet_ntopman), [`inet_pton()`](#inet_ntopman),
[`gethostbyname()`](#gethostbynameman), [`gethostbyaddr()`](#gethostbynameman)
