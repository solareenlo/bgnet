# 5.2 `socket()`---ファイルディスクリプターを取得しよう！

もう先延ばしにはできません。`socket()` システムコールの話をしなければならないのです。以下はその書式です。

```c
#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

しかし、これらの引数は何なのでしょうか？これらは、どのようなソケットが欲しいか（IPv4 か IPv6 か、ストリームかデータグラムか、TCP か UDP か）を指定することができます。

以前は、これらの値をハードコードする人がいましたが、今でも絶対にそうすることができます。(ドメインは `PF_INET` または `PF_INET6`、タイプは `SOCK_STREAM` または `SOCK_DGRAM`、プロトコルは `0` に設定すると、与えられたタイプに適したプロトコルを選択することができます。あるいは `getprotobyname()` を呼んで、"tcp" や "udp" などの欲しいプロトコルを調べることもできます。)

（この `PF_INET` は、`struct sockaddr_in` の `sin_family` フィールドを初期化するときに使用できる `AF_INET` の近縁種です。実際、両者は非常に密接な関係にあり、実際に同じ値を持っているので、多くのプログラマは `socket()` を呼び出して `PF_INET` の代わりに `AF_INET` を第一引数に渡しています。さて、ミルクとクッキーを用意して、お話の時間です。昔々、あるアドレスファミリ（`AF_INET` の AF）が、プロトコルファミリ（`PF_INET` の PF）で参照される複数のプロトコルをサポートするかもしれないと考えられたことがあります。しかし、そうはなりませんでした。そして、みんな幸せに暮らした、ザ・エンド。というわけで、最も正しいのは `struct sockaddr_in` で `AF_INET` を使い、`socket()` の呼び出しで `PF_INET` を使うことです。）

とにかく、もう十分です。本当にやりたいことは、`getaddrinfo()` の呼び出しの結果の値を使い、以下のように直接 `socket()` に送り込むことです。

```c
int s;
struct addrinfo hints, *res;

// do the lookup
// [pretend we already filled out the "hints" struct]
getaddrinfo("www.example.com", "http", &hints, &res);

// again, you should do error-checking on getaddrinfo(), and walk
// the "res" linked list looking for valid entries instead of just
// assuming the first one is good (like many of these examples do).
// See the section on client/server for real examples.

s = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
```

`socket()` は単に、後のシステムコールで使用できるソケットディスクリプタを返すか、エラーの場合は `-1` を返します。グローバル変数 `errno` にはエラーの値が設定されます（詳細については [`errno`](#errnoman) のマニュアルページを参照してください。また、マルチスレッドプログラムで `errno` を使用する際の簡単な注意も参照してください。）

でも、このソケットは何の役に立つのでしょうか？答えは、これだけでは本当に意味がなく、もっと読み進めてシステムコールを作らないと意味がないのです。
