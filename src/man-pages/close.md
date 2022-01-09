# 9.4 `close()`

ソケットディスクリプタを閉じます。

### 9.4.1 書式

```c
#include <unistd.h>

int close(int s);
```

### 9.4.2 解説

ソケットを使い終えて、`send()` や `recv()` 、あるいはそれ以外のことを一切したくなくなったら、 `close()` をすれば、ソケットは解放され、二度と使われることはありません。

リモート側は、このような事態が発生したかどうかを、2つの方法のいずれかで判断することができます。1つ目は、リモート側が `recv()` を呼び出した場合、`0` を返します。2つ目は、リモート側が `send()` を呼び出した場合、シグナル `SIGPIPE` を受け取り、`send()` は `-1` を返し、`errno` は `EPIPE` にセットされます。

**Windowsユーザ**：あなたが使うべき関数は `closesocket()` であって、`close()` ではありません。もし、ソケットディスクリプタ上で `close()` を使おうとすると、Windows が怒る可能性があります...。そして、怒られると嫌なものです。

### 9.4.3 返り値

成功した場合は `0` を、エラーの場合は `-1` を返します（それに応じて `errno` が設定されます）。

### 9.4.4 例

```c
s = socket(PF_INET, SOCK_DGRAM, 0);
.
.
.
// a whole lotta stuff...*BRRRONNNN!*
.
.
.
close(s);  // not much to it, really.
```

### 9.4.5 参照

[`socket()`](./socket.md),
[`shutdown()`](./shutdown.md)