# 7.7 ブロードキャストパケット---Hello, World!

これまで、このガイドでは、1つのホストから他の1つのホストにデータを送信することについて話してきました。しかし、適切な権限があれば、同時に複数のホストにデータを送信することが可能です、と私は主張します！

UDP（TCP ではなく UDP のみ）と標準的な IPv4 では、これはブロードキャストと呼ばれるメカニズムによって行われます。IPv6 ではブロードキャストはサポートされていないため、マルチキャストという優れた技術を利用する必要があります。しかし、未来に目を向けるのはもう十分です。私たちは 32 ビットの現在に留まっています。

しかし、ちょっと待ってください！いきなりブロードキャストを始めることはできません。ブロードキャストパケットをネットワークに送信する前に、ソケットオプション `SO_BROADCAST` をセットしなければなりません。まるでミサイルの発射スイッチの上に被せる小さなプラスチックのカバーのようなものです。そのくらいのパワーを秘めているのです。

しかし、真面目な話、ブロードキャストパケットを使うには危険性があるのです。ブロードキャストパケットを受信したすべてのシステムは、そのデータの行き先がどのポートか分かるまで、何層にもわたるデータのカプセル化を解かなければならないのです。そして、そのデータを渡すか、廃棄するかしなければなりません。どちらの場合でも、ブロードキャストパケットを受信した各マシンは、ローカルネットワーク上のすべてのマシンで、多くのマシンが不必要な作業をすることになりかねません。ゲーム Doom が発売された当初、これはそのネットワークコードに対する不満でした。

さて、猫の皮を剥ぐ方法は一つではない......ちょっと待った。本当に猫の皮を剥ぐ方法は1つではないのですか？どんな表現なんだ？それと同じように、ブロードキャストパケットを送る方法も1つではありません。では、本題のブロードキャストメッセージの宛先アドレスはどのように指定するのでしょうか？一般的には2つの方法があります。

1. 特定のサブネットのブロードキャストアドレスにデータを送信します。これは、サブネットのネットワーク番号に、アドレスのホスト部分のすべての 1 ビットを設定したものです。例えば、私の家のネットワークは `192.168.1.0`、ネットマスクは `255.255.255.0` で、アドレスの最後のバイトは私のホスト番号です（ネットマスクによると、最初の 3 バイトはネットワーク番号だからです）。つまり、私のブロードキャストアドレスは `192.168.1.255` です。Unix では、`ifconfig` コマンドが実際にこのすべてのデータを与えてくれます。（気になる方は、ブロードキャストアドレスを得るためのビット論理は `network_number` OR (NOT `netmask`) です。）このタイプのブロードキャストパケットは、自分のローカルネットワークだけでなく、遠隔地のネットワークにも送ることができますが、宛先のルータによってパケットがドロップされる危険性があります。（もしドロップされなかったら、どこかのランダムなスマーフが彼らの LAN にブロードキャストトラフィックを流し始めるかもしれません。）

2. "グローバル"ブロードキャストアドレスにデータを送信します。これは `255.255.255.255` で、別名 `INADDR_BROADCAST` と呼ばれるものです。多くのマシンはこれをネットワーク番号と自動的にビット演算して、ネットワークブロードキャストアドレスに変換しますが、そうでないものもあります。ルータは、皮肉なことに、この種のブロードキャストパケットをローカルネットワークから転送しません。

では、ソケットオプション `SO_BROADCAST` を設定せずに、ブロードキャストアドレスでデータを送信しようとするとどうなるでしょうか。それでは、古き良き時代の [`talker` and `listener`](docs/client-server-background/#datagram) を起動し、何が起こるか見てみましょう。

```
$ talker 192.168.1.2 foo
sent 3 bytes to 192.168.1.2
$ talker 192.168.1.255 foo
sendto: Permission denied
$ talker 255.255.255.255 foo
sendto: Permission denied
```

そう、全然嬉しくないんです...ソケットオプションの `SO_BROADCAST` を設定していなかったからです。そうすれば、どこでも `sendto()` ができるようになります！

実際、これがブロードキャストできる UDP アプリケーションとできない UDP アプリケーションの唯一の違いなのです。そこで、古い `talker` アプリケーションに `SO_BROADCAST` ソケットオプションを設定する章を一つ追加してみましょう。このプログラムを [`broadcaster.c`](https://beej.us/guide/bgnet/examples/broadcaster.c) と呼ぶことにします。

```c
/*
** broadcaster.c -- a datagram "client" like talker.c, except
**                  this one can broadcast
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define SERVERPORT 4950 // the port users will be connecting to

int main(int argc, char *argv[])
{
    int sockfd;
    struct sockaddr_in their_addr; // connector's address information
    struct hostent *he;
    int numbytes;
    int broadcast = 1;
    //char broadcast = '1'; // if that doesn't work, try this

    if (argc != 3) {
        fprintf(stderr,"usage: broadcaster hostname message\n");
        exit(1);
    }

    if ((he=gethostbyname(argv[1])) == NULL) {  // get the host info
        perror("gethostbyname");
        exit(1);
    }

    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) == -1) {
        perror("socket");
        exit(1);
    }

    // this call is what allows broadcast packets to be sent:
    if (setsockopt(sockfd, SOL_SOCKET, SO_BROADCAST, &broadcast,
        sizeof broadcast) == -1) {
        perror("setsockopt (SO_BROADCAST)");
        exit(1);
    }

    their_addr.sin_family = AF_INET;     // host byte order
    their_addr.sin_port = htons(SERVERPORT); // short, network byte order
    their_addr.sin_addr = *((struct in_addr *)he->h_addr);
    memset(their_addr.sin_zero, '\0', sizeof their_addr.sin_zero);

    if ((numbytes=sendto(sockfd, argv[2], strlen(argv[2]), 0,
             (struct sockaddr *)&their_addr, sizeof their_addr)) == -1) {
        perror("sendto");
        exit(1);
    }

    printf("sent %d bytes to %s\n", numbytes,
        inet_ntoa(their_addr.sin_addr));

    close(sockfd);

    return 0;
}
```

"通常の" UDP クライアント/サーバーの状況と何が違うのでしょうか？何もありません。（この場合、クライアントがブロードキャストパケットの送信を許可されていることを除けば。）このように、古い UDP [`listener`](docs/client-server-background/#datagram) プログラムを1つのウィンドウで実行し、`broadcaster` を別のウィンドウで実行してみてください。これで、上で失敗した送信をすべて行うことができるはずです。

```
$ broadcaster 192.168.1.2 foo
sent 3 bytes to 192.168.1.2
$ broadcaster 192.168.1.255 foo
sent 3 bytes to 192.168.1.255
$ broadcaster 255.255.255.255 foo
sent 3 bytes to 255.255.255.255
```

そして、`listener` がパケットを受け取ったと応答するのが見えるはずです。（もし `listener` が応答しないなら、それは IPv6 アドレスにバインドされているからかもしれません。`listener.c` の `AF_UNSPEC` を `AF_INET` に変更して、強制的に IPv4 にすることを試してみてください。）

なるほど、それはちょっと楽しみですね。しかし、今度は同じネットワーク上にある隣のマシンで `listener` を起動して、それぞれのマシンに2つのコピーを作成し、ブロードキャストアドレスを指定して `broadcaster` を再度実行してみてください... おい！両方の `listener` がパケットを受け取るぞ！`sendto()` を一度しか呼んでいないのにな。かっこいい！

もし `listener` が、あなたが直接送ったデータを受け取って、ブロードキャストアドレスのデータを受け取らないなら、あなたのローカルマシンのファイアウォールがパケットをブロックしている可能性があります。（そうそう、Pat とBapper、これが私のサンプルコードが動作しない理由だと、私より先に気づいてくれてありがとうございます。ガイドの中で紹介すると言っておいたのに、こうして紹介してくれて。というわけで、にゃー。）

繰り返しになりますが、ブロードキャストパケットには注意が必要です。LAN 上のすべてのマシンが `recvfrom()` したかどうかに関わらず、そのパケットを処理しなければならないので、計算機ネットワーク全体にかなりの負荷を与える可能性があります。ブロードキャストパケットは控えめに、そして適切に使用されるべきものであることは間違いありません。
