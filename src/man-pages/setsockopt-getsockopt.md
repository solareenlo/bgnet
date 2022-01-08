# 9.20 `setsockopt()`, `getsockopt()`

ソケットの各種オプションを設定します。

### 9.20.1 書式

```c
#include <sys/types.h>
#include <sys/socket.h>

int getsockopt(int s, int level, int optname, void *optval,
               socklen_t *optlen);
int setsockopt(int s, int level, int optname, const void *optval,
               socklen_t optlen);
```

### 9.20.2 解説

ソケットはかなり設定しやすい獣です。実際、とても設定しやすいので、ここですべてをカバーするつもりはありません。どうせシステム依存でしょうから。しかし、基本的なことについてはお話します。

当然ながら、これらの関数はソケットの特定のオプションを取得・設定します。Linux では、ソケットの情報はすべて "man page for socket" のセクション 7 にあります。（これらの情報を得るには "`man 7 socket`" とタイプしてください。）

パラメータとしては、`s` はあなたが話しているソケットで、`level` は `SOL_SOCKET` に設定する必要があります。そして、`optname` に興味のある名前を設定します。繰り返しになりますが、すべてのオプションについては man ページを参照してください。しかし、ここでは最も楽しいものをいくつか紹介します。

| `optname`         | 解説                                         |
|-------------------|------------------------------------------------------|
| `SO_BINDTODEVICE` | このソケットは `bind()` を使って IP アドレスにバインドするのではなく、 `eth0` のようなシンボリックなデバイス名にバインドしてください。Unix で `ifconfig` コマンドを入力すると、デバイス名が表示されます。|
| `SO_REUSEADDR` | このポートにバインドしているアクティブなソケットがない限り、他のソケットが `bind()` を行えるようにします。これにより、クラッシュ後にサーバを再起動しようとしたときに表示される "Address already in use" エラーメッセージを回避することができます。|
| `SOCK_DGRAM` | UDP データグラム [SOCK_DGRAM](`SOCK_DGRAM`) ソケットがブロードキャストアドレスに送信されたパケットを送受信できるようにします。TCP ストリームソケットには何もしません---NOTHING!!---ははは！|

パラメータ `optval` については、通常、問題の値を示す `int` へのポインタとなります。ブール値の場合、`0` は偽で、`0` 以外は真です。そしてそれは、あなたのシステムで異なっていない限り、絶対的な事実です。渡すべきパラメータがない場合、`optval` は `NULL` にすることができます。

最後のパラメータ `optlen` には、`optval` の長さ、おそらく `sizeof(int)` を設定する必要がありますが、オプションによって異なります。`getsockopt()` の場合、これは `socklen_t` へのポインタであり、`optval` に格納される最大サイズのオブジェクトを指定することに注意してください（バッファオーバーフローを防止するためです）。そして `getsockopt()` は `optlen` の値を、実際に設定されたバイト数を反映するように変更します。

**警告**：いくつかのシステム（特に Sun と Windows）では、このオプションは `int` の代わりに `char` とすることができ、例えば `int` 値 `1` の代わりに `'1'` という文字値を設定することができます。繰り返しになりますが、より詳しい情報は "`man setsockopt`" と "`man 7 socket`" で各自の man ページをチェックしてください！

### 9.20.3 返り値

成功した場合は `0` を、エラーの場合は `-1` を返します（それに応じて `errno` が設定されます）。

### 9.20.4 例

```c
int optval;
int optlen;
char *optval2;

// set SO_REUSEADDR on a socket to true (1):
optval = 1;
setsockopt(s1, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof optval);

// bind a socket to a device name (might not work on all systems):
optval2 = "eth1"; // 4 bytes long, so 4, below:
setsockopt(s2, SOL_SOCKET, SO_BINDTODEVICE, optval2, 4);

// see if the SO_BROADCAST flag is set:
getsockopt(s3, SOL_SOCKET, SO_BROADCAST, &optval, &optlen);
if (optval != 0) {
    print("SO_BROADCAST enabled on s3!\n");
}
```

### 9.20.5 参照

[`fcntl()`](./fcntl.md)
