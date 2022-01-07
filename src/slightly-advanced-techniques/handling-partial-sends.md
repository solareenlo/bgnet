# 7.4 部分的な `send()`s の処理

上の [`send()` の節](docs/system-calls-or-bust/#sendrecv)で、`send()` はあなたが頼んだバイトをすべて送らないかもしれない、と言ったのを覚えていますか？つまり、512バイト送って欲しいのに、412バイトが返ってきたとします。残りの100バイトはどうなったのでしょうか？

さて、それらはまだあなたの小さなバッファの中で送信されるのを待っています。あなたがコントロールできない状況のために、カーネルはすべてのデータを1つのチャンクで送信しないことを決定しました、そして今、私の友人は、データを外に出すのはあなた次第です。

このような関数を書いてやってもいいんじゃないでしょうか。

```c
#include <sys/types.h>
#include <sys/socket.h>

int sendall(int s, char *buf, int *len)
{
    int total = 0;        // how many bytes we've sent
    int bytesleft = *len; // how many we have left to send
    int n;

    while(total < *len) {
        n = send(s, buf+total, bytesleft, 0);
        if (n == -1) { break; }
        total += n;
        bytesleft -= n;
    }

    *len = total; // return number actually sent here

    return n==-1?-1:0; // return -1 on failure, 0 on success
}
```

この例では、`s` がデータを送信するソケット、`buf` がデータを格納するバッファ、`len` がバッファのバイト数を格納する `int` へのポインタです。

この関数は、エラーが発生すると `-1` を返します（また、`send()` の呼び出しによって `errno` が設定されたままです）。また、実際に送信されたバイト数は `len` として返されます。これは、エラーがない限り、あなたが送信するように頼んだバイト数と同じになります。`sendall()` はデータを送信するために最善を尽くし、ハァハァ言っていますが、もしエラーがあればすぐに返されます。

念のため、この関数の呼び出しのサンプルを示します。

```c
char buf[10] = "Beej!";
int len;

len = strlen(buf);
if (sendall(s, buf, &len) == -1) {
    perror("sendall");
    printf("We only sent %d bytes because of the error!\n", len);
}
```

パケットの一部が到着した場合、受信側ではどのようなことが起こるのでしょうか？パケットの長さがまちまちな場合、あるパケットが終わり、別のパケットが始まるのを受信側はどうやって知るのでしょうか？そう、現実のシナリオはロバの王道なのです。あなたはおそらくカプセル化しなければならないでしょう（冒頭の[データのカプセル化の章](docs/what-is-a-socket/#lowlevel)を覚えていますか？）詳細はこちらをご覧ください。
