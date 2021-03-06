# 9.11 `fcntl()`

ソケット記述子を制御します。

### 9.11.1 書式

```c
#include <sys/unistd.h>
#include <sys/fcntl.h>

int fcntl(int s, int cmd, long arg);
```

### 9.11.2 解説

この関数は通常、ファイルロックやその他のファイル指向の処理を行うために使用されますが、時々見たり使ったりするようなソケット関連の関数もいくつか持っています。

パラメータ `s` には操作したいソケット記述子、`cmd` には `F_SETFL` を設定し、`arg` には以下のコマンドのいずれかを指定します。（ここで説明した以外にも `fcntl()` には様々な機能がありますが、ここではソケットに特化して説明します。）

| `cmd`        | 解説                                               |
|--------------|------------------------------------------------------------|
| `O_NONBLOCK` | ソケットをノンブロッキングに設定します。詳しくは [7.1 ブロッキング](../slightly-advanced-techniques/blocking.md)の章を参照してください。|
| `O_ASYNC`    | 非同期 I/O を行うようにソケットを設定します。ソケットでデータを `recv()` する準備ができたら、シグナル `SIGIO` を発生させます。これはめったに見られないことであり、ガイドの範囲を超えています。また、特定のシステムでのみ利用可能だと思います。|

### 9.11.3 返り値

成功した場合は `0` を、エラーの場合は `-1` を返します（それに応じて `errno` が設定されます）。

`fcntl()` システムコールの使い方によって、実際には異なる戻り値がありますが、ここではソケットに関係しないので取り上げていません。詳しくはローカルの `fcntl()` man ページを参照してください。

### 9.11.4 例

```c
int s = socket(PF_INET, SOCK_STREAM, 0);

fcntl(s, F_SETFL, O_NONBLOCK);  // set to non-blocking
fcntl(s, F_SETFL, O_ASYNC);     // set to asynchronous I/O
```

### 9.11.5 参照

[Blocking](../slightly-advanced-techniques/blocking.md),
[`send()`](./send-sendto.md)
