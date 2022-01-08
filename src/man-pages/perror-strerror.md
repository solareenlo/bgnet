# 9.16 `perror()`, `strerror()`

エラーを人間が読める文字列で表示します。

### 9.16.1 書式

```c
#include <stdio.h>
#include <string.h>   // for strerror()

void perror(const char *s);
char *strerror(int errnum);
```

### 9.16.2 解説

多くの関数がエラー時に `-1` を返し、変数 `errno` の値を何らかの数値に設定するので、それを自分にとって意味のある形で簡単に表示できたら確かに良いでしょう。

幸いなことに、`perror()` がそれをやってくれます。もし、エラーの前にもっと説明を表示させたい場合は、パラメータ `s` を指定します（あるいは `s` を `NULL` にしておけば、何も追加で表示されません）。

簡単に言うと、この関数は `ECONNRESET` のような `errno` 値を受け取り、それらを "Connection reset by peer" のようにうまく表示します。

関数 `strerror()` は `perror()` に非常に似ていますが、与えられた値（通常は変数 `errno` を渡します）に対するエラーメッセージの文字列へのポインタを返す点が異なります。

### 9.16.3 返り値

`strerror()` はエラーメッセージの文字列へのポインタを返します。

### 9.16.4 例

```c
int s;

s = socket(PF_INET, SOCK_STREAM, 0);

if (s == -1) { // some error has occurred
    // prints "socket error: " + the error message:
    perror("socket error");
}

// similarly:
if (listen(s, 10) == -1) {
    // this prints "an error: " + the error message from errno:
    printf("an error: %s\n", strerror(errno));
}
```

### 9.16.5 参照

[`errno`](#errnoman)
