# 5.4 `connect()`---やあ、こんにちは！

ちょっとだけ、あなたが telnet アプリケーションであることを仮定してみましょう。ユーザが（映画 TRON のように）ソケットファイル記述子を取得するように命令します。あなたはそれに応じ、`socket()` を呼び出します。次に、ユーザはポート "`23`"（標準的な telnet ポート）で "`10.12.110.57`" に接続するように指示します。やったー！どうするんだ？

幸運なことに、あなたは今、`connect()` の章---リモートホストに接続する方法を読んでいるところです。だから、猛烈に読み進めよう！時間がない！

`connect()` の呼び出しは以下の通りです。

```c
#include <sys/types.h>
#include <sys/socket.h>

int connect(int sockfd, struct sockaddr *serv_addr, int addrlen);
```

`sockfd` は `socket()` コールで返される、我々の身近なソケットファイル記述子、`serv_addr` は宛先ポートと IP アドレスを含む `struct sockaddr`、`addrlen` はサーバアドレス構造体のバイト長です。

これらの情報はすべて、`getaddrinfo()` の呼び出しの結果から得ることができ、これはロックします。

だんだん分かってきたかな？ここからは聞こえないので、そうであることを祈るしかないですね。ポート `3490` の "`www.example.com`" にソケット接続する例を見てみましょう。

```c
struct addrinfo hints, *res;
int sockfd;

// first, load up address structs with getaddrinfo():

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;

getaddrinfo("www.example.com", "3490", &hints, &res);

// make a socket:

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

// connect!

connect(sockfd, res->ai_addr, res->ai_addrlen);
```

繰り返しになりますが、古いタイプのプログラムでは、独自の `struct sockaddr_ins` を作成して `connect()` に渡していました。必要であれば、そうすることができます。上の [`bind()`](docs/system-calls-or-bust/#bind) の節で同様のことを書いています。

`connect()` の戻り値を必ず確認してください。エラー時に `-1` が返され、`errno` という変数がセットされます。

また、`bind()` を呼んでいないことに注意してください。基本的に、私たちはローカルのポート番号には関心がありません。カーネルは私たちのためにローカルポートを選択し、接続先のサイトは自動的にこの情報を取得します。心配はいりません。
