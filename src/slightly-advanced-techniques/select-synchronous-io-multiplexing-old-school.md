# 7.3 `select()`---同期式 I/O 多重化、旧式

この関数、ちょっと不思議なんですが、とても便利なんです。次のような状況を考えてみましょう。あなたはサーバで、入ってくるコネクションをリッスンするだけでなく、すでに持っているコネクションを読み続けたいのです。

問題ありません。`accept()` と `recv()` を数回実行するだけです。そうはいかないよ、バスター！もし `accept()` の呼び出しがブロックされていたらどうでしょう？どうやって `recv()` を同時に行うんだ？"ノンブロッキングソケットを使いましょう！" まさか！CPU を占有するようなことはしない方がいい。じゃあ、何？

`select()` は同時に複数のソケットを監視する力を与えてくれます。どのソケットが読み込み可能で、どのソケットが書き込み可能か、そしてどのソケットが例外を発生させたか、本当に知りたければ教えてくれるでしょう。

> 警告：`select()` は非常にポータブルですが、巨大な数の接続が発生した場合には恐ろしく遅くなります。そのような状況では、[libevent](https://libevent.org/) のようなイベントライブラリの方が、あなたのシステムで利用可能な最も高速なメソッドを使用しようとするため、より良いパフォーマンスを得ることができることでしょう。

さっそくですが、`select()` の概要を説明します。

```c
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int numfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

この関数は、ファイル記述子の "集合"、特に `readfds`、`writefds`、`exceptfds` を監視します。標準入力とソケット記述子 `sockfd` から読み込めるかどうかを確認したい場合、ファイル記述子 `0` と `sockfd` を `readfds` の集合に追加するだけでよいです。パラメータ `numfds` には、最も大きいファイル記述子の値に `1` を足した値を設定する必要があります。この例では、標準入力（`0`）よりも確実に大きいので、`sockfd+1` に設定する必要があります。

`select()` が戻ると、`readfds` は、選択したファイル記述子のうち、どれが読み込める状態にあるかを反映するように変更されます。以下のマクロ `FD_ISSET()` を用いて、それらをテストすることができます。

この先に進む前に、これらのセットを操作する方法について説明します。各セットは `fd_set` 型です。以下のマクロはこの型を操作します。

| Function                         | 説明                                             |
|----------------------------------|--------------------------------------------------|
| `FD_SET(int fd, fd_set *set);`   | `set` に `fd` を追加します。                     |
| `FD_CLR(int fd, fd_set *set);`   | `set` から `fd` を削除します。                   |
| `FD_ISSET(int fd, fd_set *set);` | `fd` が `set` に含まれる場合は true を返します。 |
| `FD_ZERO(fd_set *set);`          | `set` からすべてのエントリをクリアします。       |

最後に、この奇妙な `struct timeval` とは何でしょうか？まあ、誰かがデータを送ってくるのをいつまでも待っていたくない場合もあるでしょう。例えば、96 秒ごとに "Still Going..." とターミナルに表示させたい、でも何も起きていない。この time 構造体では、タイムアウト時間を指定することができます。タイムアウト時間を超えても `select()` がまだ準備のできたファイル記述子を見つけられなければ、処理を続行できるように返されます。

`struct timeval` は以下のフィールドを持ちます。

```c
struct timeval {
    int tv_sec;     // seconds
    int tv_usec;    // microseconds
};
```

`tv_sec` に待ち時間の秒数を、`tv_usec` に待ち時間のマイクロ秒数を設定するだけです。そう、これはミリ秒ではなくマイクロ秒なのです。ミリ秒の中には 1000 マイクロ秒があり、1 秒の中には 1000 ミリ秒があります。したがって、1 秒の中には 1,000,000 マイクロ秒があることになります。なぜ "usec" なのか？"u" は、私たちが "マイクロ" に使っているギリシャ文字の μ（ミュー）に似ていると思われるからです。また、関数が戻ってきたとき、`timeout` はまだ残っている時間を表示するように更新されるかもしれません。これは、あなたが使っている Unix のフレーバーに依存します。

やったー！マイクロ秒の分解能のタイマーを手に入れたぞ！まあ、当てにしない方がいいです。どんなに小さな `struct timeval` を設定しても、おそらく標準的な Unix のタイムスライスの一部を待つ必要があります。

その他、気になること。もし `struct timeval` のフィールドを `0` に設定すると、`select()` は直ちにタイムアウトし、セット内のすべてのファイル記述子を効率よくポーリングします。パラメータ `timeout` を `NULL` に設定すると、決してタイムアウトせず、最初のファイル記述子が準備できるまで待ちます。最後に、特定のセットを待つことを気にしないのであれば、`select()` のコールでそれを `NULL` に設定することができます。

[次のコード](https://beej.us/guide/bgnet/examples/select.c)では、標準入力に何か表示されるまで 2.5 秒待ちます。

```c
/*
** select.c -- a select() demo
*/

#include <stdio.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

#define STDIN 0  // file descriptor for standard input

int main(void)
{
    struct timeval tv;
    fd_set readfds;

    tv.tv_sec = 2;
    tv.tv_usec = 500000;

    FD_ZERO(&readfds);
    FD_SET(STDIN, &readfds);

    // don't care about writefds and exceptfds:
    select(STDIN+1, &readfds, NULL, NULL, &tv);

    if (FD_ISSET(STDIN, &readfds))
        printf("A key was pressed!\n");
    else
        printf("Timed out.\n");

    return 0;
}
```

ラインバッファ端末の場合、押すキーは RETURN でないと、とにかくタイムアウトしてしまいます。

さて、この方法はデータグラムソケットでデータを待つのに最適な方法だと思う人もいるかもしれませんね。Unice の中にはこの方法で select を使えるものもあれば、使えないものもあります。試してみたいなら、ローカルの man ページに何が書いてあるか見てみるといいです。

Unices の中には、タイムアウトまでの残り時間を反映して、`struct timeval` の時間を更新するものがあります。しかし、そうでないものもあります。ポータブルにしたいのであれば、そのようなことが起こることを当てにしないでください。（経過時間を追跡する必要がある場合は、`gettimeofday()` を使ってください。残念なことですが、それが現実なのです。）

リードセット内のソケットがコネクションをクローズした場合はどうなるのでしょうか？その場合、`select()` はそのソケット記述子を "ready to read" に設定して返す。実際にそこから `recv()` を実行すると、`recv()` は `0` を返します。これが、クライアントが接続を閉じたことを知るための方法です。

もうひとつ `select()` について書いておくと、`listen()` しているソケットがある場合、そのソケットのファイル記述子を `readfds` セットに入れておけば、新しい接続があるかどうかチェックすることができます。

以上、全能の関数 `select()` の概要を簡単に説明しました。

しかし、ご要望の多かった、より詳細な例をご紹介します。残念ながら、上記のごく簡単な例と、こちらの例では、大きな違いがあります。しかし、ご覧になってから、その後に続く説明をお読みください。

[このプログラム](https://beej.us/guide/bgnet/examples/selectserver.c)は、簡単なマルチユーザチャットサーバのように動作します。一つのウィンドウで起動し、他の複数のウィンドウから `telnet` ("`telnet hostname 9034`") で接続してください。ある `telnet` セッションで何かを入力すると、他のすべてのウィンドウに表示されるはずです。

```c
/*
** selectserver.c -- a cheezy multiperson chat server
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define PORT "9034"   // port we're listening on

// get sockaddr, IPv4 or IPv6:
void *get_in_addr(struct sockaddr *sa)
{
    if (sa->sa_family == AF_INET) {
        return &(((struct sockaddr_in*)sa)->sin_addr);
    }

    return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

int main(void)
{
    fd_set master;    // master file descriptor list
    fd_set read_fds;  // temp file descriptor list for select()
    int fdmax;        // maximum file descriptor number

    int listener;     // listening socket descriptor
    int newfd;        // newly accept()ed socket descriptor
    struct sockaddr_storage remoteaddr; // client address
    socklen_t addrlen;

    char buf[256];    // buffer for client data
    int nbytes;

    char remoteIP[INET6_ADDRSTRLEN];

    int yes=1;        // for setsockopt() SO_REUSEADDR, below
    int i, j, rv;

    struct addrinfo hints, *ai, *p;

    FD_ZERO(&master);    // clear the master and temp sets
    FD_ZERO(&read_fds);

    // get us a socket and bind it
    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE;
    if ((rv = getaddrinfo(NULL, PORT, &hints, &ai)) != 0) {
        fprintf(stderr, "selectserver: %s\n", gai_strerror(rv));
        exit(1);
    }

    for(p = ai; p != NULL; p = p->ai_next) {
        listener = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (listener < 0) {
            continue;
        }

        // lose the pesky "address already in use" error message
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(int));

        if (bind(listener, p->ai_addr, p->ai_addrlen) < 0) {
            close(listener);
            continue;
        }

        break;
    }

    // if we got here, it means we didn't get bound
    if (p == NULL) {
        fprintf(stderr, "selectserver: failed to bind\n");
        exit(2);
    }

    freeaddrinfo(ai); // all done with this

    // listen
    if (listen(listener, 10) == -1) {
        perror("listen");
        exit(3);
    }

    // add the listener to the master set
    FD_SET(listener, &master);

    // keep track of the biggest file descriptor
    fdmax = listener; // so far, it's this one

    // main loop
    for(;;) {
        read_fds = master; // copy it
        if (select(fdmax+1, &read_fds, NULL, NULL, NULL) == -1) {
            perror("select");
            exit(4);
        }

        // run through the existing connections looking for data to read
        for(i = 0; i <= fdmax; i++) {
            if (FD_ISSET(i, &read_fds)) { // we got one!!
                if (i == listener) {
                    // handle new connections
                    addrlen = sizeof remoteaddr;
                    newfd = accept(listener,
                        (struct sockaddr *)&remoteaddr,
                        &addrlen);

                    if (newfd == -1) {
                        perror("accept");
                    } else {
                        FD_SET(newfd, &master); // add to master set
                        if (newfd > fdmax) {    // keep track of the max
                            fdmax = newfd;
                        }
                        printf("selectserver: new connection from %s on "
                            "socket %d\n",
                            inet_ntop(remoteaddr.ss_family,
                                get_in_addr((struct sockaddr*)&remoteaddr),
                                remoteIP, INET6_ADDRSTRLEN),
                            newfd);
                    }
                } else {
                    // handle data from a client
                    if ((nbytes = recv(i, buf, sizeof buf, 0)) <= 0) {
                        // got error or connection closed by client
                        if (nbytes == 0) {
                            // connection closed
                            printf("selectserver: socket %d hung up\n", i);
                        } else {
                            perror("recv");
                        }
                        close(i); // bye!
                        FD_CLR(i, &master); // remove from master set
                    } else {
                        // we got some data from a client
                        for(j = 0; j <= fdmax; j++) {
                            // send to everyone!
                            if (FD_ISSET(j, &master)) {
                                // except the listener and ourselves
                                if (j != listener && j != i) {
                                    if (send(j, buf, nbytes, 0) == -1) {
                                        perror("send");
                                    }
                                }
                            }
                        }
                    }
                } // END handle data from client
            } // END got new incoming connection
        } // END looping through file descriptors
    } // END for(;;)--and you thought it would never end!

    return 0;
}
```

このコードでは、2つのファイル記述子セットを持っていることに注意してください。`master` と `read_fds` です。最初の `master` は、現在接続されているすべてのソケット記述子と、新しい接続を待ち受けているソケット記述子を保持します。

`master` のセットを持っている理由は、`select()` が実際に渡すセットを変更して、どのソケットが読み込み可能な状態にあるかを反映させるためです。ある `select()` から次の `select()` への呼び出しまでの接続を追跡する必要があるので、これらをどこかに安全に保存しておかなければなりません。最後の最後で、`master` を `read_fds` にコピーしてから `select()` を呼び出します。

しかし、これでは新しい接続を得るたびに、それを `master` セットに追加しなければならないのではありませんか？そうです。そして接続が終了するたびに、それを `master` セットから削除しなければならないのですか？はい、その通りです。

注目すべきは、`listener` ソケットが読み込み可能な状態になったかどうかをチェックしていることです。このとき、新しい接続が保留されていることを意味するので、それを `accept()` して `master` セットに追加します。同様に、クライアントの接続が読み込み可能な状態になったときに、`recv()` が `0` を返したら、クライアントが接続を閉じたことがわかるので、`master` セットからそれを削除しなければなりません。

しかし、クライアントの `recv()` がゼロ以外を返した場合、何らかのデータを受信したことが分かります。そこで私はそれを取得し、`master` リストを経由して、接続されている残りのすべてのクライアントにそのデータを送信します。

以上が、全能の関数 `select()` の簡単でない概要です。

Linux ファンの皆さんへ：まれに、Linux の `select()` が "ready-to-read" を返した後、実際には読み込む準備ができていないことがあります！これは、Linux の `select()` が "ready-to-read" を返した後、実際には読み込む準備ができていないことを意味します。これはつまり、`select()` が読まないと言っているのに、`read()` でブロックしてしまうということです！なぜだ、この野郎---！とにかく、回避策は受信側のソケットで `O_NONBLOCK` フラグをセットして、 `EWOULDBLOCK` でエラーにすることです（これは発生しても無視しても大丈夫です）。ソケットをノンブロッキングに設定する方法については、[`fcntl()` のリファレンスページ](../man-pages/fcntl.md)を参照してください。

さらに、ここでボーナス的な余談ですが、`poll()` という別の関数があります。これは `select()` とほぼ同じ動作をしますが、ファイル記述子集合を管理するシステムが異なります。[チェックしてみてください！](#pollman)
