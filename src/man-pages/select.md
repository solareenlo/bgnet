# 9.19 `select()`

ソケット記述子が読み込み/書き込み可能かどうか確認します。

### 9.19.1 書式

```c
#include <sys/select.h>

int select(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
           struct timeval *timeout);

FD_SET(int fd, fd_set *set);
FD_CLR(int fd, fd_set *set);
FD_ISSET(int fd, fd_set *set);
FD_ZERO(fd_set *set);
```

### 9.19.2 解説

`select()` 関数は、複数のソケットを同時にチェックして、受信待ちのデータがあるか、ブロッキングせずに `send()` できるか、何らかの例外が発生したかを確認する方法を提供します。

上記の `FD_SET()` のようなマクロを使用して、ソケット記述子のセットを投入します。セットができたら、それを以下のいずれかのパラメータとして関数に渡します。`readfds`（セット内のいずれかのソケットが `recv()` データを受信できる状態になったとき）、`writefds`（セット内のいずれかのソケットが `send()` データを送信できる状態になったとき）、または `exceptfds`（いずれかのソケットで例外が発生したときに知る必要がある場合）。もし、これらのタイプのイベントに興味がなければ、これらのパラメータのいずれか、あるいはすべてを `NULL` にすることができます。`select()` が返された後、セット内の値が変更され、どれが読み書き可能で、どれが例外を発生させたかが表示されます。

最初のパラメータ `n` は、最も大きい番号のソケット記述子（これらは単なる `int`s, 覚えていますか？）に `1` を加えたものです。

最後に、最後に `struct timeval`, `timeout` を記述します。これは `select()` に、これらのセットをどれくらいの時間チェックするかを指示します。タイムアウト後、あるいはイベントが発生したときのどちらか早いほうに戻ります。`timeval` 構造体は2つのフィールドを持っています。`tv_sec` は秒数で、これに `tv_usec` というマイクロ秒（1,000,000 マイクロ秒が 1 秒）を加えたものです。

ヘルパーマクロは次のような働きをします。

| Macro                            | 解説                                         |
|----------------------------------|----------------------------------------------|
| `FD_SET(int fd, fd_set *set);`   | `set` に `fd` を追加します。                 |
| `FD_CLR(int fd, fd_set *set);`   | `set` から `fd` を削除します。               |
| `FD_ISSET(int fd, fd_set *set);` | `fd` が `set` に含まれる場合は true を返す。 |
| `FD_ZERO(fd_set *set);`          | `set` からすべてのエントリをクリアします。   |

Linux ユーザーへの注意事項：Linux の `select()` は "ready-to-read" を返すことがありますが、実際には読み込む準備ができていないため、その後の `read()` 呼び出しがブロックされることがあります。このバグを回避するには、受信側のソケットに `O_NONBLOCK` フラグを設定し、`EWOULDBLOCK` でエラーになるようにし、このエラーが発生しても無視します。ソケットをノンブロッキングに設定する方法については、[`fcntl()` の man ページ](./fcntl.md) を参照してください。

### 9.19.3 返り値

成功した場合はセットに含まれる記述子の数を、タイムアウトに達した場合は `0` を、エラーの場合は `-1` を返します（それに応じて `errno` が設定されます）。また、どのソケットが準備できているかを示すために、セットも変更されます。

### 9.19.4 例

```c
int s1, s2, n;
fd_set readfds;
struct timeval tv;
char buf1[256], buf2[256];

// pretend we've connected both to a server at this point
//s1 = socket(...);
//s2 = socket(...);
//connect(s1, ...)...
//connect(s2, ...)...

// clear the set ahead of time
FD_ZERO(&readfds);

// add our descriptors to the set
FD_SET(s1, &readfds);
FD_SET(s2, &readfds);

// since we got s2 second, it's the "greater", so we use that for
// the n param in select()
n = s2 + 1;

// wait until either socket has data ready to be recv()d (timeout 10.5 secs)
tv.tv_sec = 10;
tv.tv_usec = 500000;
rv = select(n, &readfds, NULL, NULL, &tv);

if (rv == -1) {
    perror("select"); // error occurred in select()
} else if (rv == 0) {
    printf("Timeout occurred! No data after 10.5 seconds.\n");
} else {
    // one or both of the descriptors have data
    if (FD_ISSET(s1, &readfds)) {
        recv(s1, buf1, sizeof buf1, 0);
    }
    if (FD_ISSET(s2, &readfds)) {
        recv(s2, buf2, sizeof buf2, 0);
    }
}
```

### 9.19.5 参照

[`poll()`](./poll.md)
