# 9.17 `poll()`

複数のソケットで同時にイベントをテストします。

### 9.17.1 書式

```c
#include <sys/poll.h>

int poll(struct pollfd *ufds, unsigned int nfds, int timeout);
```

### 9.17.2 解説

この関数は `select()` と非常によく似ており、どちらもファイルディスクリプタの集合を監視して、`recv()` の準備ができた受信データ、`send()` へのソケットの準備、`recv()` への帯域外データの準備、エラーなどのイベントを検出します。

基本的な考え方は、 `ufds` に `nfds` `struct pollfd` の配列と、ミリ秒単位のタイムアウト（1秒は1000ミリ秒）を渡すというものです。永遠に待ちたい場合は、タイムアウトを負数にすることでできます。タイムアウトまでにどのソケットディスクリプタでもイベントが発生しなかった場合、`poll()` が返ります。

配列 `struct pollfd` の各要素は、1つのソケットディスクリプタを表し、以下のフィールドを含みます。

```c
struct pollfd {
    int fd;         // the socket descriptor
    short events;   // bitmap of events we're interested in
    short revents;  // when poll() returns, bitmap of events that occurred
};
```

`poll()` を呼び出す前に、ソケットディスクリプタ `fd` をロードし（`fd` に負の数を設定すると、この `struct pollfd` は無視され、その `revents` フィールドはゼロになります）、次に以下のマクロのビット単位の OR により `event` フィールドを構築します。

| Macro     | 解説                                                                      |
|-----------|---------------------------------------------------------------------------|
| `POLLIN`  | このソケットで `recv()` のためのデータが準備できたときに警告を出します。  |
| `POLLOUT` | このソケットにブロックせずにデータを `send()` できるようになったら警告を出します。|
| `POLLPRI` | このソケットで帯域外データが `recv()` できるようになったら警告を出します。|

`poll()` コールが戻ってくると、`revents` フィールドは上記のフィールドのビット単位の OR となり、どのディスクリプタで実際にそのイベントが発生したかがわかります。さらに、これらの他のフィールドが存在する可能性があります。

| Macro      | 解説                                    |
|------------|-----------------------------------------|
| `POLLERR`  | このソケットでエラーが発生しました。    |
| `POLLHUP`  | リモート側の接続がハングアップしました。|
| `POLLNVAL` | ソケットディスクリプタ `fd` に何か問題があったようだ---たぶん初期化されていないのだろう？|

### 9.17.3 返り値

配列 `ufds` のうち、イベントが発生した要素の数を返します。タイムアウトが発生した場合は `0` になります。また、エラーが発生した場合は `-1` を返します（それに応じて `errno` が設定されます。）

### 9.17.4 例

```c
int s1, s2;
int rv;
char buf1[256], buf2[256];
struct pollfd ufds[2];

s1 = socket(PF_INET, SOCK_STREAM, 0);
s2 = socket(PF_INET, SOCK_STREAM, 0);

// pretend we've connected both to a server at this point
//connect(s1, ...)...
//connect(s2, ...)...

// set up the array of file descriptors.
//
// in this example, we want to know when there's normal or out-of-band
// data ready to be recv()'d...

ufds[0].fd = s1;
ufds[0].events = POLLIN | POLLPRI; // check for normal or out-of-band

ufds[1].fd = s2;
ufds[1].events = POLLIN; // check for just normal data

// wait for events on the sockets, 3.5 second timeout
rv = poll(ufds, 2, 3500);

if (rv == -1) {
    perror("poll"); // error occurred in poll()
} else if (rv == 0) {
    printf("Timeout occurred! No data after 3.5 seconds.\n");
} else {
    // check for events on s1:
    if (ufds[0].revents & POLLIN) {
        recv(s1, buf1, sizeof buf1, 0); // receive normal data
    }
    if (ufds[0].revents & POLLPRI) {
        recv(s1, buf1, sizeof buf1, MSG_OOB); // out-of-band data
    }

    // check for events on s2:
    if (ufds[1].revents & POLLIN) {
        recv(s1, buf2, sizeof buf2, 0);
    }
}
```

### 9.17.5 参照

[`select()`](./select.md)
