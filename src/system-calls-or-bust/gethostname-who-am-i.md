# 5.11 `gethostname()`---私は誰なのか？

`getpeername()` よりもさらに簡単なのは、`gethostname()` という関数です。これは、あなたのプログラムが動作しているコンピュータの名前を返します。この名前は、後述の `gethostbyname()` でローカルマシンの IP アドレスを決定するために使用されます。

これ以上楽しいことはないでしょう？いくつか思いつきましたが、ソケットプログラミングには関係ないですね。とにかく、書式はこんな感じです。

```c
#include <unistd.h>

int gethostname(char *hostname, size_t size);
```

引数は単純で、`hostname` はこの関数が戻ったときにホスト名を格納する文字列の配列へのポインタ、`size` はホスト名配列のバイト長です。

この関数は、正常に終了した場合は `0` を、エラーの場合は `-1` を返し、通常通り `errno` を設定します。
