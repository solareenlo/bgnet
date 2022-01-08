# 9.6 `gethostname()`

システムの名称を返します。

### 9.6.1 書式

```c
#include <sys/unistd.h>

int gethostname(char *name, size_t len);
```

### 9.6.2 解説

あなたのシステムには名前がついています。みんなそうです。これは、今まで話してきた他のネットワーク的なものよりも、若干 Unix 的なものですが、それでも使い道はあります。

例えば、ホスト名を取得してから `gethostbyname()` を呼び出すと、IP アドレスを知ることができます。

パラメータ `name` はホスト名を格納するバッファを指し、`len` はそのバッファのサイズ（バイト）を表します。`gethostname()` はバッファの終端を上書きせず（エラーを返すかもしれませんし、単に書き込みを止めるかもしれません）、バッファに文字列のスペースがある場合は `NUL`-ターミネイトを行います。

### 9.6.3 返り値

成功した場合は `0` を、エラーの場合は `-1` を返す（それに応じて `errno` が設定されます）。

### 9.6.4 例

```c
char hostname[128];

gethostname(hostname, sizeof hostname);
printf("My hostname: %s\n", hostname);
```

### 9.6.5 参照

[`gethostbyname()`](#gethostbynameman)
