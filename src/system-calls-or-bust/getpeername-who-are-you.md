# 5.10 `getpeername()`---あなたは誰ですか？

この関数はとても簡単です。

あまりに簡単なので、ほとんど独自のセクションを設けなかったほどです。でも、とりあえずここに書いておきます。

`getpeername()` 関数は、接続されたストリームソケットのもう一方の端にいるのが誰であるかを教えてくれます。その書式は

```c
#include <sys/socket.h>

int getpeername(int sockfd, struct sockaddr *addr, int *addrlen);
```

`sockfd` は接続したストリームソケットのディスクリプタ、`addr` は接続の相手側の情報を保持する `struct sockaddr`（または `struct sockaddr_in`）へのポインタ、`addrlen` は `int` へのポインタであり、 `sizeof *addr` または `sizeof(struct sockaddr)` で初期化される必要があります。

この関数は，エラーが発生すると `-1` を返し，それに応じて `errno` を設定します。

アドレスがわかれば、`inet_ntop()`、`getnameinfo()`、`gethostbyaddr()` を使って、より詳しい情報を表示したり取得したりすることができます。いいえ、ログイン名を取得することはできません。（OK、OK。相手のコンピュータで ident デーモンが動いていれば、可能です。しかし、これはこのドキュメントの範囲外です。詳しくは [RFC 1413](https://datatracker.ietf.org/doc/html/rfc1413) をチェックしてください。）
