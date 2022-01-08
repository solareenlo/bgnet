# 9.10 `errno`

最後のシステムコールのエラーコードを保持します。

### 9.10.1 書式

```c
#include <errno.h>

int errno;
```

### 9.10.2 解説

これは、多くのシステムコールのエラー情報を保持する変数です。思い起こせば、`socket()` や `listen()` などはエラーが発生すると `-1` を返し、どのエラーが発生したかを具体的に知らせるために `errno` という正確な値を設定します。

ヘッダーファイル `errno.h` には、`EADDRINUSE`, `EPIPE`, `ECONNREFUSED` などのような、エラーに対する定数シンボル名の束がリストアップされています。ローカルの man ページには、どのようなコードがエラーとして返されるのかが書かれています。そして、実行時にこれらを使用して、異なるエラーを異なる方法で処理することができます。

あるいは、より一般的には `perror()` や `strerror()` を呼び出して、人間が読める形式のエラーを取得することができます。

マルチスレッドの愛好家のために一つ注意しておきたいのは、ほとんどのシステムで `errno` はスレッドセーフな方法で定義されているということです。（つまり、実際にはグローバル変数ではないのですが、シングルスレッド環境ではグローバル変数と同じように振る舞います。）

### 9.10.3 返り値

この変数の値は、最後に発生したエラーであり、最後のアクションが成功した場合は "success "のコードになるかもしれません。

### 9.10.4 例

```c
s = socket(PF_INET, SOCK_STREAM, 0);
if (s == -1) {
    perror("socket"); // or use strerror()
}

tryagain:
if (select(n, &readfds, NULL, NULL) == -1) {
    // an error has occurred!!

    // if we were only interrupted, just restart the select() call:
    if (errno == EINTR) goto tryagain;  // AAAA! goto!!!

    // otherwise it's a more serious error:
    perror("select");
    exit(1);
}
```

### 9.10.5 参照

[`perror()`](./perror-strerror.md),
[`strerror()`](./perror-strerror.md)
