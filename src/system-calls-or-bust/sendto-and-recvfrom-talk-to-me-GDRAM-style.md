# 5.8 `sendto()` and `recvfrom()`---DGRAM スタイルで話して。

"これはすべて素晴らしく、ダンディーです"、"しかし、データグラムソケットを接続しないままにしておくのはどうなんだ？"、という声が聞こえてきそうです。大丈夫だ、アミーゴ。ちょうどいいものがありますよ。

データグラムソケットはリモートホストに接続されていないので、パケットを送信する前にどのような情報を与える必要があるか分かりますか？そうです！宛先アドレスです！これがそのスコープです。

```c
int sendto(int sockfd, const void *msg, int len, unsigned int flags,
           const struct sockaddr *to, socklen_t tolen);
```

見ての通り、この呼び出しは基本的に `send()` の呼び出しと同じで、他に2つの情報が追加されています。`to` は `struct sockaddr` へのポインタで（おそらく直前にキャストした別の `struct sockaddr_in` や `struct sockaddr_in6`、`struct sockaddr_storage` になるでしょう）、送信先の IP アドレスとポートが含まれています。`tolen` は `int` 型ですが、単純に `sizeof *to` または `sizeof(struct sockaddr_storage)` に設定することができます。

宛先アドレスの構造体を手に入れるには、`getaddrinfo()` や以下の `recvfrom()` から取得するか、手で記入することになると思います。

`send()` と同様、`sendto()` は実際に送信したバイト数 (これも、送信するように指示したバイト数よりも少ないかもしれません！) を返し、エラーの場合は `-1` を返します。

同様に、`recv()` と `recvfrom()` も類似しています。`recvfrom()` の概要は以下の通りです。

```c
int recvfrom(int sockfd, void *buf, int len, unsigned int flags,
             struct sockaddr *from, int *fromlen);
```

これも `recv()` と同様であるが、いくつかのフィールドが追加されています。`from` はローカルの `struct sockaddr_storage` へのポインタで、送信元のマシンの IP アドレスとポートが格納されます。`fromlen` はローカルの `int` へのポインタであり、`sizeof *from` または `sizeof(struct sockaddr_storage)` に初期化する必要があります。この関数が戻ったとき、`fromlen` は実際に `from` に格納されたアドレスの長さを含みます。

`recvfrom()` は受信したバイト数を返し、エラーの場合は `-1` を返します（`errno` はそれに応じて設定されます）。

そこで質問ですが、なぜソケットの型として `struct sockaddr_storage` を使うのでしょうか？なぜ、`struct sockaddr_in` ではないのでしょうか？なぜなら、私たちは IPv4 や IPv6 に縛られたくないからです。そこで、汎用的な構造体である `sockaddr_storage` を使用するのですが、これはどちらにも十分な大きさであることが分かっています。

（そこで...ここでまた疑問なのですが、なぜ `struct sockaddr` 自体はどんなアドレスに対しても十分な大きさがないのでしょうか？汎用 `struct sockaddr_storage` を汎用 `struct sockaddr` にキャストしているくらいなのに！？余計なことをしたような気がしますね。答えは、十分な大きさがなく、この時点で変更するのは問題がある、ということでしょう。だから新しいのを作ったんだ。）

データグラムソケットを `connect()` すれば、すべてのトランザクションに `send()` と `recv()` を使用できることを覚えておいてください。ソケット自体はデータグラムソケットであり、パケットは UDP を使用しますが、ソケットインターフェイスが自動的に宛先と送信元の情報を追加してくれるのです。
