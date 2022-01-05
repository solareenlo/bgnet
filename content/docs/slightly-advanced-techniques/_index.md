---
weight: 1
bookFlatSection: true
title: "7 Slightly Advanced Techniques"
---

# 7 Slightly Advanced Techniques

これらは本当に高度なものではありませんが、私たちがすでにカバーしたより基本的なレベルから抜け出したものです。実際、ここまでくれば、Unix ネットワークプログラミングの基本をかなり習得したと考えてよいでしょう！おめでとうございます！

さて、ここからは、より難解な事柄の勇敢な新世界に突入します。ソケットについて学ぶことができます。どうぞお楽しみに！


## 7.1 Blocking {#blocking}

ブロッキング。聞いたことがあると思います---さて、一体何でしょう？一言で言えば、"ブロック"は技術用語で"スリープ"のことです。上で `listener` を実行したとき、パケットが到着するまでただそこに座っていることに気付いたと思います。何が起こったかというと、`recvfrom()` を呼び出したのですが、データがなかったので、`recvfrom()` はデータが到着するまで "block"（つまり、そこで眠る）と言われているのです。

多くの関数がブロックします。`accept()` がブロックします。すべての `recv()` 関数がブロックします。このようなことができるのは、ブロックすることが許されているからです。最初に `socket()` でソケットディスクリプタを作成するとき、カーネルはそれをブロッキングに設定します。もし、ソケットをブロッキングさせたくなければ、`fcntl()` を呼び出す必要があります。

```{.c .numberLines}
#include <unistd.h>
#include <fcntl.h>
.
.
.
sockfd = socket(PF_INET, SOCK_STREAM, 0);
fcntl(sockfd, F_SETFL, O_NONBLOCK);
.
.
.
```

ソケットをノンブロッキングに設定することで、効果的にソケットの情報を"ポール"することができます。ノンブロッキングソケットから読み込もうとしたときに、そこにデータがない場合、ブロックすることは許されません---その際には `-1` が返り、`errno` には `EAGAIN` または `EWOULDBLOCK` がセットされます。

（待てよ--`EAGAIN` や `EWOULDBLOCK` を返すこともあるのか？どちらをチェックする？仕様では実際にあなたのシステムがどちらを返すかは指定されていないので、移植性のために両方チェックしましょう。）

しかし、一般的に言って、この種のポーリングは悪い考えです。ソケットのデータを探すためにプログラムをビジーウェイト状態にすると、流行遅れのように CPU 時間を吸い取られてしまうからです。読み込み待ちのデータがあるかどうかを確認するための、よりエレガントなソリューションが、次の `poll()` の節で紹介されています。


## 7.2 `poll()`---Synchronous I/O Multiplexing {#poll}

本当にやりたいことは、一度にたくさんのソケットを監視して、データの準備ができたものを処理することです。そうすれば、すべてのソケットを継続的にポーリングして、どれが読み込み可能な状態にあるかを確認する必要がなくなります。

> 警告: `poll()` は膨大な数のコネクションを持つ場合、恐ろしく遅くなります。そのような状況では、システムで利用可能な最も高速なメソッドを使用しようとする [libevent](https://libevent.org/) のようなイベントライブラリの方が良いパフォーマンスを得ることができるでしょう。

では、どうすればポーリングを回避できるのでしょうか。少し皮肉なことに、`poll()` システムコールを使えばポーリングを避けることができます。簡単に言うと、オペレーティングシステムにすべての汚い仕事を代行してもらい、どのソケットでデータが読めるようになったかだけを知らせてもらうのです。その間、我々のプロセスはスリープして、システムリソースを節約することができます。

一般的なゲームプランは、どのソケットディスクリプタを監視したいか、どのような種類のイベントを監視したいかという情報を `struct pollfd` の配列として保持することです。OS は、これらのイベントのいずれかが発生するか（例えば "socket ready to read!"）またはユーザが指定したタイムアウトが発生するまで `poll()` 呼び出しでブロックします。

便利なことに、 `listen()`しているソケットは、新しい接続が `accept()` される準備ができたときに "ready to read" を返します。

雑談はこのくらいにして。これをどう使うかです？

``` c
#include <poll.h>

int poll(struct pollfd fds[], nfds_t nfds, int timeout);
```

`fds` は情報の配列（どのソケットの何を監視するか）、`nfds` は配列の要素数、そして `timeout` はミリ秒単位のタイムアウトです。`timeout` はミリ秒単位のタイムアウトで、`poll` はイベントが発生した配列の要素数を返します。

`struct pollfd` を見てみましょう。

``` c
struct pollfd {
    int fd;         // the socket descriptor
    short events;   // bitmap of events we're interested in
    short revents;  // when poll() returns, bitmap of events that occurred
};
```

そして、その配列を用意するんだ。各要素の `fd` フィールドに、監視したいソケットディスクリプタを指定します。そして、`events` フィールドには、監視するイベントの種類を指定します。

`events` フィールドは、以下のビット単位の OR です。

| Macro     | 説明                                                                           |
|-----------|--------------------------------------------------------------------------------|
| `POLLIN`  | このソケットで `recv()` のためのデータが準備できたときに警告を出す。           |
| `POLLOUT` | このソケットにブロックせずにデータを `send()` できるようになったら警告します。 |

一旦 `struct pollfd` の配列を整えたら、それを `poll()` に渡すことができます。配列のサイズと、ミリ秒単位のタイムアウト値も一緒に渡してください。（タイムアウトに負の値を指定すると、永遠に待つことができます。）

`poll()` が返った後、`revents` フィールドをチェックして、`POLLIN` または `POLLOUT` がセットされているかどうかを確認し、イベントが発生したことを示すことができます。

（実際には `poll()` の呼び出しでできることはもっとたくさんあります。詳細は以下の [`poll()` man ページ](#pollman)を参照してください。）

ここでは、標準入力からデータを読み込めるようになるまで、つまり `RETURN` を押したときに 2.5 秒間待つ[例](https://beej.us/guide/bgnet/examples/poll.c)を示します。

``` {.c .numberLines}
#include <stdio.h>
#include <poll.h>

int main(void)
{
    struct pollfd pfds[1]; // More if you want to monitor more

    pfds[0].fd = 0;          // Standard input
    pfds[0].events = POLLIN; // Tell me when ready to read

    // If you needed to monitor other things, as well:
    //pfds[1].fd = some_socket; // Some socket descriptor
    //pfds[1].events = POLLIN;  // Tell me when ready to read

    printf("Hit RETURN or wait 2.5 seconds for timeout\n");

    int num_events = poll(pfds, 1, 2500); // 2.5 second timeout

    if (num_events == 0) {
        printf("Poll timed out!\n");
    } else {
        int pollin_happened = pfds[0].revents & POLLIN;

        if (pollin_happened) {
            printf("File descriptor %d is ready to read\n", pfds[0].fd);
        } else {
            printf("Unexpected event occurred: %d\n", pfds[0].revents);
        }
    }

    return 0;
}
```

`poll()` が `pfds` 配列の中でイベントが発生した要素の数を返していることに再度注目してください。これは配列のどの要素かを教えてくれるわけではありませんが（そのためにはまだスキャンしなければなりません）、`revents` フィールドが `0` 以外のエントリがいくつあるかを教えてくれます（したがって、その数がわかったらスキャンをやめることができます。）

ここで、いくつかの疑問が出てくるかもしれません。`poll()` に渡したセットに新しいファイルディスクリプタを追加するにはどうしたらいいのでしょうか？これについては、単に配列に必要なだけのスペースがあることを確認するか、必要に応じて `realloc()` でスペースを追加してください。

セットから項目を削除する場合はどうすればよいのでしょうか。この場合は、配列の最後の要素をコピーして、削除する要素の上に置くことができます。そして、その数をひとつ減らして `poll()` に渡します。もうひとつの方法として、`fd` フィールドに負の数を設定すると、`poll()` はそれを無視します。

どうすれば、`telnet` できるチャットサーバにまとめることができるのでしょうか？

これから行うのは、リスナーソケットを起動し、それをファイルディスクリプタのセットに追加して `poll()` に送ることです。（これは、接続があったときに読み込み可能な状態を表示します。）

そして、新しい接続を `struct pollfd` 配列に追加していきます。そして、容量が足りなくなったら、動的にそれを増やしていきます。

接続が終了したら、その接続を配列から削除します。

そして、ある接続が読み取り可能になったら、そこからデータを読み取り、そのデータを他のすべての接続に送ることで、他のユーザが入力した内容を見ることができるようにします。

そこで、[このポール・サーバ](https://beej.us/guide/bgnet/examples/pollserver.c)を試してみてください。あるウィンドウで実行し、他の多くのターミナルウィンドウから `telnet localhost 9034` を実行してみてください。一つのウィンドウで入力したものが他のウィンドウでも（RETURNを押した後で）見られるようになるはずです。

それだけでなく、`CTRL-]` を押して `quit` とタイプして `telnet` を終了すると、サーバは切断を検出し、ファイルディスクリプタの配列からあなたを削除するはずです。

``` {.c .numberLines}
/*
** pollserver.c -- a cheezy multiperson chat server
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
#include <poll.h>

#define PORT "9034"   // Port we're listening on

// Get sockaddr, IPv4 or IPv6:
void *get_in_addr(struct sockaddr *sa)
{
    if (sa->sa_family == AF_INET) {
        return &(((struct sockaddr_in*)sa)->sin_addr);
    }

    return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

// Return a listening socket
int get_listener_socket(void)
{
    int listener;     // Listening socket descriptor
    int yes=1;        // For setsockopt() SO_REUSEADDR, below
    int rv;

    struct addrinfo hints, *ai, *p;

    // Get us a socket and bind it
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

        // Lose the pesky "address already in use" error message
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(int));

        if (bind(listener, p->ai_addr, p->ai_addrlen) < 0) {
            close(listener);
            continue;
        }

        break;
    }

    freeaddrinfo(ai); // All done with this

    // If we got here, it means we didn't get bound
    if (p == NULL) {
        return -1;
    }

    // Listen
    if (listen(listener, 10) == -1) {
        return -1;
    }

    return listener;
}

// Add a new file descriptor to the set
void add_to_pfds(struct pollfd *pfds[], int newfd, int *fd_count, int *fd_size)
{
    // If we don't have room, add more space in the pfds array
    if (*fd_count == *fd_size) {
        *fd_size *= 2; // Double it

        *pfds = realloc(*pfds, sizeof(**pfds) * (*fd_size));
    }

    (*pfds)[*fd_count].fd = newfd;
    (*pfds)[*fd_count].events = POLLIN; // Check ready-to-read

    (*fd_count)++;
}

// Remove an index from the set
void del_from_pfds(struct pollfd pfds[], int i, int *fd_count)
{
    // Copy the one from the end over this one
    pfds[i] = pfds[*fd_count-1];

    (*fd_count)--;
}

// Main
int main(void)
{
    int listener;     // Listening socket descriptor

    int newfd;        // Newly accept()ed socket descriptor
    struct sockaddr_storage remoteaddr; // Client address
    socklen_t addrlen;

    char buf[256];    // Buffer for client data

    char remoteIP[INET6_ADDRSTRLEN];

    // Start off with room for 5 connections
    // (We'll realloc as necessary)
    int fd_count = 0;
    int fd_size = 5;
    struct pollfd *pfds = malloc(sizeof *pfds * fd_size);

    // Set up and get a listening socket
    listener = get_listener_socket();

    if (listener == -1) {
        fprintf(stderr, "error getting listening socket\n");
        exit(1);
    }

    // Add the listener to set
    pfds[0].fd = listener;
    pfds[0].events = POLLIN; // Report ready to read on incoming connection

    fd_count = 1; // For the listener

    // Main loop
    for(;;) {
        int poll_count = poll(pfds, fd_count, -1);

        if (poll_count == -1) {
            perror("poll");
            exit(1);
        }

        // Run through the existing connections looking for data to read
        for(int i = 0; i < fd_count; i++) {

            // Check if someone's ready to read
            if (pfds[i].revents & POLLIN) { // We got one!!

                if (pfds[i].fd == listener) {
                    // If listener is ready to read, handle new connection

                    addrlen = sizeof remoteaddr;
                    newfd = accept(listener,
                        (struct sockaddr *)&remoteaddr,
                        &addrlen);

                    if (newfd == -1) {
                        perror("accept");
                    } else {
                        add_to_pfds(&pfds, newfd, &fd_count, &fd_size);

                        printf("pollserver: new connection from %s on "
                            "socket %d\n",
                            inet_ntop(remoteaddr.ss_family,
                                get_in_addr((struct sockaddr*)&remoteaddr),
                                remoteIP, INET6_ADDRSTRLEN),
                            newfd);
                    }
                } else {
                    // If not the listener, we're just a regular client
                    int nbytes = recv(pfds[i].fd, buf, sizeof buf, 0);

                    int sender_fd = pfds[i].fd;

                    if (nbytes <= 0) {
                        // Got error or connection closed by client
                        if (nbytes == 0) {
                            // Connection closed
                            printf("pollserver: socket %d hung up\n", sender_fd);
                        } else {
                            perror("recv");
                        }

                        close(pfds[i].fd); // Bye!

                        del_from_pfds(pfds, i, &fd_count);

                    } else {
                        // We got some good data from a client

                        for(int j = 0; j < fd_count; j++) {
                            // Send to everyone!
                            int dest_fd = pfds[j].fd;

                            // Except the listener and ourselves
                            if (dest_fd != listener && dest_fd != sender_fd) {
                                if (send(dest_fd, buf, nbytes, 0) == -1) {
                                    perror("send");
                                }
                            }
                        }
                    }
                } // END handle data from client
            } // END got ready-to-read from poll()
        } // END looping through file descriptors
    } // END for(;;)--and you thought it would never end!

    return 0;
}
```

次の節では、似たような古い関数である `select()` について見ていきます。`select()` と `poll()` はどちらも似たような機能とパフォーマンスを持っており、どのように使うかが違うだけです。`select()` の方が若干移植性が高いかもしれませんが、使い勝手は少し悪いかもしれません。あなたのシステムでサポートされている限り、一番好きなものを選んでください。


## 7.3 `select()`---Synchronous I/O Multiplexing, Old School {#select}

この関数、ちょっと不思議なんですが、とても便利なんです。次のような状況を考えてみましょう。あなたはサーバで、入ってくるコネクションをリッスンするだけでなく、すでに持っているコネクションを読み続けたいのです。

問題ありません。`accept()` と `recv()` を数回実行するだけです。そうはいかないよ、バスター！もし `accept()` の呼び出しがブロックされていたらどうでしょう？どうやって `recv()` を同時に行うんだ？"ノンブロッキングソケットを使いましょう！"まさか！CPU を占有するようなことはしない方がいい。じゃあ、何？

`select()` は同時に複数のソケットを監視する力を与えてくれます。どのソケットが読み込み可能で、どのソケットが書き込み可能か、そしてどのソケットが例外を発生させたか、本当に知りたければ教えてくれるでしょう。

> 警告: `select()` は非常にポータブルですが、巨大な数の接続が発生した場合には恐ろしく遅くなります。そのような状況では、[libevent](https://libevent.org/) のようなイベントライブラリの方が、あなたのシステムで利用可能な最も高速なメソッドを使用しようとするため、より良いパフォーマンスを得ることができることでしょう。

さっそくですが、`select()`の概要を説明します。

```c
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int numfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

この関数は、ファイルディスクリプタの"集合"、特に `readfds`、`writefds`、`exceptfds` を監視します。標準入力とソケットディスクリプタ `sockfd` から読み込めるかどうかを確認したい場合、ファイルディスクリプタ `0` と `sockfd` を `readfds` の集合に追加するだけでよいです。パラメータ `numfds` には、最も大きいファイルディスクリプタの値に `1` を足した値を設定する必要があります。この例では、標準入力 (`0`) よりも確実に大きいので、`sockfd+1` に設定する必要があります。

`select()` が戻ると、`readfds` は、選択したファイルディスクリプタのうち、どれが読み込める状態にあるかを反映するように変更されます。以下のマクロ `FD_ISSET()` を用いて、それらをテストすることができます。

この先に進む前に、これらのセットを操作する方法について説明します。各セットは `fd_set` 型です。以下のマクロはこの型を操作します。

| Function                         | 説明                                             |
|----------------------------------|--------------------------------------------------|
| `FD_SET(int fd, fd_set *set);`   | `set` に `fd` を追加します。                     |
| `FD_CLR(int fd, fd_set *set);`   | `set` から `fd` を削除します。                   |
| `FD_ISSET(int fd, fd_set *set);` | `fd` が `set` に含まれる場合は true を返します。 |
| `FD_ZERO(fd_set *set);`          | `set` からすべてのエントリをクリアします。       |

最後に、この奇妙な `struct timeval` とは何でしょうか？まあ、誰かがデータを送ってくるのをいつまでも待っていたくない場合もあるでしょう。例えば、96 秒ごとに "Still Going..." とターミナルに表示させたい、でも何も起きていない。この time 構造体では、タイムアウト時間を指定することができます。タイムアウト時間を超えても `select()` がまだ準備のできたファイルディスクリプタを見つけられなければ、処理を続行できるように返されます。

`struct timeval` は以下のフィールドを持ちます。

```c
struct timeval {
    int tv_sec;     // seconds
    int tv_usec;    // microseconds
};
```

`tv_sec` に待ち時間の秒数を、`tv_usec` に待ち時間のマイクロ秒数を設定するだけです。そう、これはミリ秒ではなくマイクロ秒なのです。ミリ秒の中には 1000 マイクロ秒があり、1 秒の中には 1000 ミリ秒があります。したがって、1 秒の中には 1,000,000 マイクロ秒があることになります。なぜ "usec"なのか？"u"は、私たちが"マイクロ"に使っているギリシャ文字の μ（ミュー）に似ていると思われるからです。また、関数が戻ってきたとき、`timeout` はまだ残っている時間を表示するように更新されるかもしれません。これは、あなたが使っている Unix のフレーバーに依存します。

やったー！マイクロ秒の分解能のタイマーを手に入れたぞ！まあ、当てにしない方がいいです。どんなに小さな `struct timeval` を設定しても、おそらく標準的な Unix のタイムスライスの一部を待つ必要があります。

その他、気になること。もし `struct timeval` のフィールドを `0` に設定すると、`select()` は直ちにタイムアウトし、セット内のすべてのファイルディスクリプタを効率よくポーリングします。パラメータ `timeout` を `NULL` に設定すると、決してタイムアウトせず、最初のファイルディスクリプタが準備できるまで待ちます。最後に、特定のセットを待つことを気にしないのであれば、`select()` のコールでそれを `NULL` に設定することができます。

[次のコード](https://beej.us/guide/bgnet/examples/select.c)では、標準入力に何か表示されるまで 2.5 秒待ちます。

```{.c .numberLines}
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

リードセット内のソケットがコネクションをクローズした場合はどうなるのでしょうか？その場合、`select()` はそのソケットディスクリプタを "ready to read" に設定して返す。実際にそこから `recv()` を実行すると、`recv()` は `0` を返します。これが、クライアントが接続を閉じたことを知るための方法です。

もうひとつ `select()` について書いておくと、`listen()` しているソケットがある場合、そのソケットのファイルディスクリプタを `readfds` セットに入れておけば、新しい接続があるかどうかチェックすることができます。

以上、全能の関数 `select()` の概要を簡単に説明しました。

しかし、ご要望の多かった、より詳細な例をご紹介します。残念ながら、上記のごく簡単な例と、こちらの例では、大きな違いがあります。しかし、ご覧になってから、その後に続く説明をお読みください。

[このプログラム](https://beej.us/guide/bgnet/examples/selectserver.c)は、簡単なマルチユーザチャットサーバのように動作します。一つのウィンドウで起動し、他の複数のウィンドウから `telnet` ("`telnet hostname 9034`") で接続してください。ある `telnet` セッションで何かを入力すると、他のすべてのウィンドウに表示されるはずです。

```{.c .numberLines}
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

このコードでは、2つのファイル記述子セットを持っていることに注意してください。`master` と `read_fds` です。最初の `master` は、現在接続されているすべてのソケットディスクリプタと、新しい接続を待ち受けているソケットディスクリプタを保持します。

`master` のセットを持っている理由は、`select()` が実際に渡すセットを変更して、どのソケットが読み込み可能な状態にあるかを反映させるためです。ある `select()` から次の `select()` への呼び出しまでの接続を追跡する必要があるので、これらをどこかに安全に保存しておかなければなりません。最後の最後で、`master` を `read_fds` にコピーしてから `select()` を呼び出します。

しかし、これでは新しい接続を得るたびに、それを `master` セットに追加しなければならないのではありませんか？そうです。そして接続が終了するたびに、それを `master` セットから削除しなければならないのですか？はい、その通りです。

注目すべきは、`listener` ソケットが読み込み可能な状態になったかどうかをチェックしていることです。このとき、新しい接続が保留されていることを意味するので、それを `accept()` して `master` セットに追加します。同様に、クライアントの接続が読み込み可能な状態になったときに、`recv()` が `0` を返したら、クライアントが接続を閉じたことがわかるので、`master` セットからそれを削除しなければなりません。

しかし、クライアントの `recv()` がゼロ以外を返した場合、何らかのデータを受信したことが分かります。そこで私はそれを取得し、`master` リストを経由して、接続されている残りのすべてのクライアントにそのデータを送信します。

以上が、全能の関数 `select()` の簡単でない概要です。

Linux ファンの皆さんへ：まれに、Linux の `select()` が "ready-to-read" を返した後、実際には読み込む準備ができていないことがあります！これは、Linux の `select()` が "ready-to-read" を返した後、実際には読み込む準備ができていないことを意味します。これはつまり、`select()` が読まないと言っているのに、`read()` でブロックしてしまうということです！なぜだ、この野郎---！とにかく、回避策は受信側のソケットで `O_NONBLOCK` フラグをセットして、 `EWOULDBLOCK` でエラーにすることです（これは発生しても無視しても大丈夫です）。ソケットをノンブロッキングに設定する方法については、[`fcntl()` リファレンスページ](#fcntlman)を参照してください。

さらに、ここでボーナス的な余談ですが、`poll()` という別の関数があります。これは `select()` とほぼ同じ動作をしますが、ファイルディスクリプタ集合を管理するシステムが異なります。[チェックしてみてください！](#pollman)


## 7.4 Handling Partial `send()`s {#sendall}

上の [`send()` の節](docs/system-calls-or-bust/#sendrecv)で、`send()` はあなたが頼んだバイトをすべて送らないかもしれない、と言ったのを覚えていますか？つまり、512バイト送って欲しいのに、412バイトが返ってきたとします。残りの100バイトはどうなったのでしょうか?

さて、それらはまだあなたの小さなバッファの中で送信されるのを待っています。あなたがコントロールできない状況のために、カーネルはすべてのデータを1つのチャンクで送信しないことを決定しました、そして今、私の友人は、データを外に出すのはあなた次第です。

このような関数を書いてやってもいいんじゃないでしょうか。

```{.c .numberLines}
#include <sys/types.h>
#include <sys/socket.h>

int sendall(int s, char *buf, int *len)
{
    int total = 0;        // how many bytes we've sent
    int bytesleft = *len; // how many we have left to send
    int n;

    while(total < *len) {
        n = send(s, buf+total, bytesleft, 0);
        if (n == -1) { break; }
        total += n;
        bytesleft -= n;
    }

    *len = total; // return number actually sent here

    return n==-1?-1:0; // return -1 on failure, 0 on success
}
```

この例では、`s` がデータを送信するソケット、`buf` がデータを格納するバッファ、`len` がバッファのバイト数を格納する `int` へのポインタです。

この関数は、エラーが発生すると `-1` を返します（また、`send()` の呼び出しによって `errno` が設定されたままです）。また、実際に送信されたバイト数は `len` として返されます。これは、エラーがない限り、あなたが送信するように頼んだバイト数と同じになります。`sendall()` はデータを送信するために最善を尽くし、ハァハァ言っていますが、もしエラーがあればすぐに返されます。

念のため、この関数の呼び出しのサンプルを示します。

```{.c .numberLines}
char buf[10] = "Beej!";
int len;

len = strlen(buf);
if (sendall(s, buf, &len) == -1) {
    perror("sendall");
    printf("We only sent %d bytes because of the error!\n", len);
}
```

パケットの一部が到着した場合、受信側ではどのようなことが起こるのでしょうか？パケットの長さがまちまちな場合、あるパケットが終わり、別のパケットが始まるのを受信側はどうやって知るのでしょうか？そう、現実のシナリオはロバの王道なのです。あなたはおそらくカプセル化しなければならないでしょう（冒頭の[データのカプセル化のセクション](docs/what-is-a-socket/#lowlevel)を覚えていますか？）詳細はこちらをご覧ください。


## 7.5 Serialization---How to Pack Data {#serialization}

ネットワーク上でテキストデータを送るのは簡単ですが、`int` や `float` のような"バイナリ"データを送りたい場合はどうしたらよいでしょうか？その結果、いくつかの選択肢があることがわかりました。

1. 数字を `sprintf()` などの関数でテキストに変換し、送信します。受信側は `strtol()` などの関数を使ってテキストを解析し、数値に戻します。

2. `send()` にデータへのポインタを渡して、データをそのまま送信します。

3. 数値を携帯可能な2進数にエンコードします。受信側はそれをデコードします。

スニークプレビュー！今夜だけ！

[_カーテン上昇_]

Beejは、"私は、上の方法3を好みます！"と言っています。

[_終_]

（このセクションを本格的に始める前に、これを行うためのライブラリは世の中に存在し、自分でローリングして移植性とエラーのない状態を維持することはかなり困難であることをお伝えしておきます。ですから、自分で実装することを決める前に、いろいろと調べて下調べをしてください。私は、このようなことがどのように機能するのかに興味がある人のために、ここに情報を記載します。）

実は、上記の方法はどれも欠点と利点があるのですが、一般的には、先ほど言ったように、私は3番目の方法を好みます。しかし、まず、他の2つの方法の欠点と利点について説明しましょう。

最初の方法は、数字をテキストとしてエンコードしてから送信するもので、電線を伝わってくるデータを簡単に印刷して読むことができるという利点があります。[インターネットリレーチャット（IRC）](https://en.wikipedia.org/wiki/Internet_Relay_Chat)のように、帯域幅を必要としない状況で使用するには、人間が読めるプロトコルが優れている場合もあります。しかし、変換に時間がかかるという欠点があり、その結果はほとんど常に元の数値よりも多くのスペースを取ってしまいます。

方法2：生データを渡します。これは非常に簡単です（しかし危険です！）。送信するデータへのポインタを取り、それを使って send を呼び出すだけです。

```c
double d = 3490.15926535;

send(s, &d, sizeof d, 0);  /* DANGER--non-portable! */
```

受け手はこのように受け取ります。

```c
double d;

recv(s, &d, sizeof d, 0);  /* DANGER--non-portable! */
```

速くて、シンプルで、いいことずくめじゃないですか。しかし、すべてのアーキテクチャが `double`（あるいは `int`）を同じビット表現で、あるいは同じバイト順序で表現しているわけではないことがわかりました！このコードは明らかに非移植的です。（おいおい---もしかしたら移植性は必要ないかもしれない。その場合は、これはいいし、速い。）

整数型をパッキングするとき、`htons()` クラスの関数が、数値をネットワークバイトオーダーに変換することによって、いかに移植性を保つのに役立つか、そして、それがいかに正しい行為であるかをすでに見てきました。残念ながら、`float` 型に対する同様の関数はありません。希望は失われてしまったのでしょうか？

恐るべし！（一瞬、怖くなったか？いいえ？少しも？）私たちにできることがあります。データを既知のバイナリ形式にパックし（または"マーシャル"、"シリアライズ"、あるいは他の1億の名前のうちの1つ）、受信者がリモート側で解凍できるようにすることができるのです。

"既知のバイナリ形式"とはどういう意味でしょうか？さて、`htons()` の例はもう見ましたね？これは、ホスト側のフォーマットが何であれ、数値をネットワークバイトオーダーに変更（あるいは"エンコード"）します。数字を逆変換（アンエンコード）するために、受信側は `ntohs()` を呼び出します。

でも、他の非整数型にはそんな関数はないって、さっき言い終わったばかりじゃないですか。そうです。そうなんだ。そして、C 言語にはこれを行う標準的な方法がないので、ちょっと困ったことになります（Python ファンにとってはありがたいダジャレですね）。

そのためには、データを既知の形式にパックし、それを電送してデコードする必要があります。例えば、`float` をパックするために、以下は[迅速で汚い方法ですが、改善の余地はたくさんあります](https://beej.us/guide/bgnet/examples/pack.c)。

```{.c .numberLines}
#include <stdint.h>

uint32_t htonf(float f)
{
    uint32_t p;
    uint32_t sign;

    if (f < 0) { sign = 1; f = -f; }
    else { sign = 0; }

    p = ((((uint32_t)f)&0x7fff)<<16) | (sign<<31); // whole part and sign
    p |= (uint32_t)(((f - (int)f) * 65536.0f))&0xffff; // fraction

    return p;
}

float ntohf(uint32_t p)
{
    float f = ((p>>16)&0x7fff); // whole part
    f += (p&0xffff) / 65536.0f; // fraction

    if (((p>>31)&0x1) == 0x1) { f = -f; } // sign bit set

    return f;
}
```

上記のコードは、32 ビットの数値に `float` を格納する素朴な実装のようなものです。上位ビット（31）は数値の符号（"1"は負を意味します）を格納するために使用され、次の7ビット（30-16）は `float` の整数部を格納するために使用されます。最後に残りのビット（15-0）は、数値の小数部分を格納するために使用されます。

使い方はいたって簡単です。

```{.c .numberLines}
#include <stdio.h>

int main(void)
{
    float f = 3.1415926, f2;
    uint32_t netf;

    netf = htonf(f);  // convert to "network" form
    f2 = ntohf(netf); // convert back to test

    printf("Original: %f\n", f);        // 3.141593
    printf(" Network: 0x%08X\n", netf); // 0x0003243F
    printf("Unpacked: %f\n", f2);       // 3.141586

    return 0;
}
```

プラス面は、小さくてシンプル、そして速いことです。32767 より大きい数を格納しようとすると、とても満足できるものではありません！マイナス面は、スペースを有効活用できないことと、範囲が大きく制限されることです。上の例では、小数点以下の桁数が正しく保存されていないこともおわかりいただけると思います。

代わりに何ができるのか？浮動小数点数を保存するための標準規格は [IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) として知られています。ほとんどのコンピュータは内部でこのフォーマットを使って浮動小数点演算を行っているので、厳密に言えば変換する必要はないのです。しかし、ソースコードの移植性を重視するのであれば、必ずしもそのような前提は成り立ちません。（一方、高速に動作させたいのであれば、変換を行う必要のないプラットフォームでは最適化すべきです！それが `htons()` やその類いの処理です。）

以下は、浮動小数点と倍数を IEEE-754 フォーマットにエンコードするコードです。（ほとんど--- NaN や Infinity はエンコードしませんが、そのように修正することができます。）

```{.c .numberLines}
#define pack754_32(f) (pack754((f), 32, 8))
#define pack754_64(f) (pack754((f), 64, 11))
#define unpack754_32(i) (unpack754((i), 32, 8))
#define unpack754_64(i) (unpack754((i), 64, 11))

uint64_t pack754(long double f, unsigned bits, unsigned expbits)
{
    long double fnorm;
    int shift;
    long long sign, exp, significand;
    unsigned significandbits = bits - expbits - 1; // -1 for sign bit

    if (f == 0.0) return 0; // get this special case out of the way

    // check sign and begin normalization
    if (f < 0) { sign = 1; fnorm = -f; }
    else { sign = 0; fnorm = f; }

    // get the normalized form of f and track the exponent
    shift = 0;
    while(fnorm >= 2.0) { fnorm /= 2.0; shift++; }
    while(fnorm < 1.0) { fnorm *= 2.0; shift--; }
    fnorm = fnorm - 1.0;

    // calculate the binary form (non-float) of the significand data
    significand = fnorm * ((1LL<<significandbits) + 0.5f);

    // get the biased exponent
    exp = shift + ((1<<(expbits-1)) - 1); // shift + bias

    // return the final answer
    return (sign<<(bits-1)) | (exp<<(bits-expbits-1)) | significand;
}

long double unpack754(uint64_t i, unsigned bits, unsigned expbits)
{
    long double result;
    long long shift;
    unsigned bias;
    unsigned significandbits = bits - expbits - 1; // -1 for sign bit

    if (i == 0) return 0.0;

    // pull the significand
    result = (i&((1LL<<significandbits)-1)); // mask
    result /= (1LL<<significandbits); // convert back to float
    result += 1.0f; // add the one back on

    // deal with the exponent
    bias = (1<<(expbits-1)) - 1;
    shift = ((i>>significandbits)&((1LL<<expbits)-1)) - bias;
    while(shift > 0) { result *= 2.0; shift--; }
    while(shift < 0) { result /= 2.0; shift++; }

    // sign it
    result *= (i>>(bits-1))&1? -1.0: 1.0;

    return result;
}
```

32 ビット（おそらく `float`）と 64 ビット（おそらく `double`）の数値のパッキングとアンパッキングのための便利なマクロをトップに置きましたが、`pack754()` 関数を直接呼んで `bits` 分のデータ（`expbits` は正規化した数値の指数用に予約されています）をエンコードするように指示することができます。

以下は使用例です。

```{.c .numberLines}

#include <stdio.h>
#include <stdint.h> // defines uintN_t types
#include <inttypes.h> // defines PRIx macros

int main(void)
{
    float f = 3.1415926, f2;
    double d = 3.14159265358979323, d2;
    uint32_t fi;
    uint64_t di;

    fi = pack754_32(f);
    f2 = unpack754_32(fi);

    di = pack754_64(d);
    d2 = unpack754_64(di);

    printf("float before : %.7f\n", f);
    printf("float encoded: 0x%08" PRIx32 "\n", fi);
    printf("float after  : %.7f\n\n", f2);

    printf("double before : %.20lf\n", d);
    printf("double encoded: 0x%016" PRIx64 "\n", di);
    printf("double after  : %.20lf\n", d2);

    return 0;
}
```


上記のコードでは、このように出力されます。

```
float before : 3.1415925
float encoded: 0x40490FDA
float after  : 3.1415925

double before : 3.14159265358979311600
double encoded: 0x400921FB54442D18
double after  : 3.14159265358979311600
```

もう一つの疑問は、`struct` をどのようにパックするかということです。残念ながら、コンパイラは `struct` の中に自由にパディングを入れることができるので、全体を1つのチャンクでポータブルに送信することはできません。（"これができない"、"あれができない"というのはもう聞き飽きた？すみません。友人の言葉を借りれば、"何か問題が起きると、いつもマイクロソフトのせいにする "ということです。これは確かにマイクロソフトのせいではないかもしれませんが、友人の発言は完全に事実です。）

話を戻すと、`struct` を電線で送るには、各フィールドを独立してパックし、反対側に到着したらそれらを `struct` にアンパックするのが一番良い方法です。

それは大変なことだ、とお考えでしょう。そうなんです。ひとつは、データをパックするのを手伝ってくれるヘルパー関数を書くことです。これは楽しいぞ。本当に！？

Kernighan と Pike の [The Practice of Programming](https://www.amazon.com/gp/product/020161586X/ref=as_li_tl?ie=UTF8&tag=beejus0c-20&camp=1789&creative=9325&linkCode=as2&creativeASIN=020161586X&linkId=b3ac9370d2df122adcce0316aed99924) という本の中で、彼らは `printf()` に似た関数である `pack()` と `unpack()` を実装し、まさにこのようなことをやっています。リンクしたいのですが、どうやらこれらの関数はこの本の他のソースと一緒にオンラインにないようです。

（The Practice of Programming は素晴らしい読み物です。ゼウスは私が勧めるたびに子猫を救ってくれます。）

この時点で、私は使ったことはありませんが、完全に立派に見える [Protocol Buffers implementation in C](https://github.com/protobuf-c/protobuf-c) へのポインタを落とすつもりです。Python や Perl のプログラマは、同じことを実現するために、それぞれの言語の `pack()` と `unpack()` 関数をチェックアウトしたいと思うでしょう。また、Java には大きな Serializable インターフェースがあり、同じような方法で使用することができます。


しかし、C言語で独自のパッキングユーティリティを書きたい場合、K&P のトリックは、変数の引数リストを使って `printf()` 風の関数を作り、パケットを構築することです。 以下は、それを元に[[私が自作したバージョン](https://beej.us/guide/bgnet/examples/pack2.c)ですが、うまくいけば、このようなものがどのように動作するかのアイデアを与えるのに十分なものです。

（このコードは、上記の `pack754()` 関数を参照しています。`packi*()` 関数はおなじみの `htons()` ファミリーと同じように動作しますが、別の整数の代わりに `char` 配列にパックする点が異なります。）

```{.c .numberLines}
#include <stdio.h>
#include <ctype.h>
#include <stdarg.h>
#include <string.h>

/*
** packi16() -- store a 16-bit int into a char buffer (like htons())
*/
void packi16(unsigned char *buf, unsigned int i)
{
    *buf++ = i>>8; *buf++ = i;
}

/*
** packi32() -- store a 32-bit int into a char buffer (like htonl())
*/
void packi32(unsigned char *buf, unsigned long int i)
{
    *buf++ = i>>24; *buf++ = i>>16;
    *buf++ = i>>8;  *buf++ = i;
}

/*
** packi64() -- store a 64-bit int into a char buffer (like htonl())
*/
void packi64(unsigned char *buf, unsigned long long int i)
{
    *buf++ = i>>56; *buf++ = i>>48;
    *buf++ = i>>40; *buf++ = i>>32;
    *buf++ = i>>24; *buf++ = i>>16;
    *buf++ = i>>8;  *buf++ = i;
}

/*
** unpacki16() -- unpack a 16-bit int from a char buffer (like ntohs())
*/
int unpacki16(unsigned char *buf)
{
    unsigned int i2 = ((unsigned int)buf[0]<<8) | buf[1];
    int i;

    // change unsigned numbers to signed
    if (i2 <= 0x7fffu) { i = i2; }
    else { i = -1 - (unsigned int)(0xffffu - i2); }

    return i;
}

/*
** unpacku16() -- unpack a 16-bit unsigned from a char buffer (like ntohs())
*/
unsigned int unpacku16(unsigned char *buf)
{
    return ((unsigned int)buf[0]<<8) | buf[1];
}

/*
** unpacki32() -- unpack a 32-bit int from a char buffer (like ntohl())
*/
long int unpacki32(unsigned char *buf)
{
    unsigned long int i2 = ((unsigned long int)buf[0]<<24) |
                           ((unsigned long int)buf[1]<<16) |
                           ((unsigned long int)buf[2]<<8)  |
                           buf[3];
    long int i;

    // change unsigned numbers to signed
    if (i2 <= 0x7fffffffu) { i = i2; }
    else { i = -1 - (long int)(0xffffffffu - i2); }

    return i;
}

/*
** unpacku32() -- unpack a 32-bit unsigned from a char buffer (like ntohl())
*/
unsigned long int unpacku32(unsigned char *buf)
{
    return ((unsigned long int)buf[0]<<24) |
           ((unsigned long int)buf[1]<<16) |
           ((unsigned long int)buf[2]<<8)  |
           buf[3];
}

/*
** unpacki64() -- unpack a 64-bit int from a char buffer (like ntohl())
*/
long long int unpacki64(unsigned char *buf)
{
    unsigned long long int i2 = ((unsigned long long int)buf[0]<<56) |
                                ((unsigned long long int)buf[1]<<48) |
                                ((unsigned long long int)buf[2]<<40) |
                                ((unsigned long long int)buf[3]<<32) |
                                ((unsigned long long int)buf[4]<<24) |
                                ((unsigned long long int)buf[5]<<16) |
                                ((unsigned long long int)buf[6]<<8)  |
                                buf[7];
    long long int i;

    // change unsigned numbers to signed
    if (i2 <= 0x7fffffffffffffffu) { i = i2; }
    else { i = -1 -(long long int)(0xffffffffffffffffu - i2); }

    return i;
}

/*
** unpacku64() -- unpack a 64-bit unsigned from a char buffer (like ntohl())
*/
unsigned long long int unpacku64(unsigned char *buf)
{
    return ((unsigned long long int)buf[0]<<56) |
           ((unsigned long long int)buf[1]<<48) |
           ((unsigned long long int)buf[2]<<40) |
           ((unsigned long long int)buf[3]<<32) |
           ((unsigned long long int)buf[4]<<24) |
           ((unsigned long long int)buf[5]<<16) |
           ((unsigned long long int)buf[6]<<8)  |
           buf[7];
}

/*
** pack() -- store data dictated by the format string in the buffer
**
**   bits |signed   unsigned   float   string
**   -----+----------------------------------
**      8 |   c        C
**     16 |   h        H         f
**     32 |   l        L         d
**     64 |   q        Q         g
**      - |                               s
**
**  (16-bit unsigned length is automatically prepended to strings)
*/

unsigned int pack(unsigned char *buf, char *format, ...)
{
    va_list ap;

    signed char c;              // 8-bit
    unsigned char C;

    int h;                      // 16-bit
    unsigned int H;

    long int l;                 // 32-bit
    unsigned long int L;

    long long int q;            // 64-bit
    unsigned long long int Q;

    float f;                    // floats
    double d;
    long double g;
    unsigned long long int fhold;

    char *s;                    // strings
    unsigned int len;

    unsigned int size = 0;

    va_start(ap, format);

    for(; *format != '\0'; format++) {
        switch(*format) {
        case 'c': // 8-bit
            size += 1;
            c = (signed char)va_arg(ap, int); // promoted
            *buf++ = c;
            break;

        case 'C': // 8-bit unsigned
            size += 1;
            C = (unsigned char)va_arg(ap, unsigned int); // promoted
            *buf++ = C;
            break;

        case 'h': // 16-bit
            size += 2;
            h = va_arg(ap, int);
            packi16(buf, h);
            buf += 2;
            break;

        case 'H': // 16-bit unsigned
            size += 2;
            H = va_arg(ap, unsigned int);
            packi16(buf, H);
            buf += 2;
            break;

        case 'l': // 32-bit
            size += 4;
            l = va_arg(ap, long int);
            packi32(buf, l);
            buf += 4;
            break;

        case 'L': // 32-bit unsigned
            size += 4;
            L = va_arg(ap, unsigned long int);
            packi32(buf, L);
            buf += 4;
            break;

        case 'q': // 64-bit
            size += 8;
            q = va_arg(ap, long long int);
            packi64(buf, q);
            buf += 8;
            break;

        case 'Q': // 64-bit unsigned
            size += 8;
            Q = va_arg(ap, unsigned long long int);
            packi64(buf, Q);
            buf += 8;
            break;

        case 'f': // float-16
            size += 2;
            f = (float)va_arg(ap, double); // promoted
            fhold = pack754_16(f); // convert to IEEE 754
            packi16(buf, fhold);
            buf += 2;
            break;

        case 'd': // float-32
            size += 4;
            d = va_arg(ap, double);
            fhold = pack754_32(d); // convert to IEEE 754
            packi32(buf, fhold);
            buf += 4;
            break;

        case 'g': // float-64
            size += 8;
            g = va_arg(ap, long double);
            fhold = pack754_64(g); // convert to IEEE 754
            packi64(buf, fhold);
            buf += 8;
            break;

        case 's': // string
            s = va_arg(ap, char*);
            len = strlen(s);
            size += len + 2;
            packi16(buf, len);
            buf += 2;
            memcpy(buf, s, len);
            buf += len;
            break;
        }
    }

    va_end(ap);

    return size;
}

/*
** unpack() -- unpack data dictated by the format string into the buffer
**
**   bits |signed   unsigned   float   string
**   -----+----------------------------------
**      8 |   c        C
**     16 |   h        H         f
**     32 |   l        L         d
**     64 |   q        Q         g
**      - |                               s
**
**  (string is extracted based on its stored length, but 's' can be
**  prepended with a max length)
*/
void unpack(unsigned char *buf, char *format, ...)
{
    va_list ap;

    signed char *c;              // 8-bit
    unsigned char *C;

    int *h;                      // 16-bit
    unsigned int *H;

    long int *l;                 // 32-bit
    unsigned long int *L;

    long long int *q;            // 64-bit
    unsigned long long int *Q;

    float *f;                    // floats
    double *d;
    long double *g;
    unsigned long long int fhold;

    char *s;
    unsigned int len, maxstrlen=0, count;

    va_start(ap, format);

    for(; *format != '\0'; format++) {
        switch(*format) {
        case 'c': // 8-bit
            c = va_arg(ap, signed char*);
            if (*buf <= 0x7f) { *c = *buf;} // re-sign
            else { *c = -1 - (unsigned char)(0xffu - *buf); }
            buf++;
            break;

        case 'C': // 8-bit unsigned
            C = va_arg(ap, unsigned char*);
            *C = *buf++;
            break;

        case 'h': // 16-bit
            h = va_arg(ap, int*);
            *h = unpacki16(buf);
            buf += 2;
            break;

        case 'H': // 16-bit unsigned
            H = va_arg(ap, unsigned int*);
            *H = unpacku16(buf);
            buf += 2;
            break;

        case 'l': // 32-bit
            l = va_arg(ap, long int*);
            *l = unpacki32(buf);
            buf += 4;
            break;

        case 'L': // 32-bit unsigned
            L = va_arg(ap, unsigned long int*);
            *L = unpacku32(buf);
            buf += 4;
            break;

        case 'q': // 64-bit
            q = va_arg(ap, long long int*);
            *q = unpacki64(buf);
            buf += 8;
            break;

        case 'Q': // 64-bit unsigned
            Q = va_arg(ap, unsigned long long int*);
            *Q = unpacku64(buf);
            buf += 8;
            break;

        case 'f': // float
            f = va_arg(ap, float*);
            fhold = unpacku16(buf);
            *f = unpack754_16(fhold);
            buf += 2;
            break;

        case 'd': // float-32
            d = va_arg(ap, double*);
            fhold = unpacku32(buf);
            *d = unpack754_32(fhold);
            buf += 4;
            break;

        case 'g': // float-64
            g = va_arg(ap, long double*);
            fhold = unpacku64(buf);
            *g = unpack754_64(fhold);
            buf += 8;
            break;

        case 's': // string
            s = va_arg(ap, char*);
            len = unpacku16(buf);
            buf += 2;
            if (maxstrlen > 0 && len >= maxstrlen) count = maxstrlen - 1;
            else count = len;
            memcpy(s, buf, count);
            s[count] = '\0';
            buf += len;
            break;

        default:
            if (isdigit(*format)) { // track max str len
                maxstrlen = maxstrlen * 10 + (*format-'0');
            }
        }

        if (!isdigit(*format)) maxstrlen = 0;
    }

    va_end(ap);
}
```

そして、[上記のコード](https://beej.us/guide/bgnet/examples/pack2.c)で、あるデータを `buf` にパックし、それを変数に展開するデモプログラムを以下に示します。なお、文字列の引数（フォーマット指定子 "`s`"）を指定して `unpack()` を呼び出す場合は、バッファオーバーランを防ぐために最大長を "`96s`" のように前に置くことが賢明です。ネットワーク経由で受け取ったデータを解凍するときには注意が必要です。悪意のあるユーザが、あなたのシステムを攻撃するために、うまく構成されたパケットを送るかもしれません！

```{.c .numberLines}
#include <stdio.h>

// various bits for floating point types--
// varies for different architectures
typedef float float32_t;
typedef double float64_t;

int main(void)
{
    unsigned char buf[1024];
    int8_t magic;
    int16_t monkeycount;
    int32_t altitude;
    float32_t absurdityfactor;
    char *s = "Great unmitigated Zot! You've found the Runestaff!";
    char s2[96];
    int16_t packetsize, ps2;

    packetsize = pack(buf, "chhlsf", (int8_t)'B', (int16_t)0, (int16_t)37, 
            (int32_t)-5, s, (float32_t)-3490.6677);
    packi16(buf+1, packetsize); // store packet size in packet for kicks

    printf("packet is %" PRId32 " bytes\n", packetsize);

    unpack(buf, "chhl96sf", &magic, &ps2, &monkeycount, &altitude, s2,
        &absurdityfactor);

    printf("'%c' %" PRId32" %" PRId16 " %" PRId32
            " \"%s\" %f\n", magic, ps2, monkeycount,
            altitude, s2, absurdityfactor);

    return 0;
}
```

自分でコードをロールアップするにしても、他人のコードを使うにしても、毎回手作業で各ビットをパッキングするのではなく、バグを抑えるために一般的なデータパッキングルーチンのセットを用意するのは良いアイデアだと思います。

データをパッキングする場合、どのような形式が良いのでしょうか？素晴らしい質問です。幸いなことに、[RFC4506](https://datatracker.ietf.org/doc/html/rfc4506)（外部データ表現規格）では、浮動小数点型、整数型、配列、生データなど、さまざまな型のバイナリ形式をすでに定義しているんです。もし自分でデータをロールバックするのであれば、それに準拠することをお勧めします。でも、そうする義務はありません。パケット警察は、あなたのドアのすぐ外にいるわけではありません。少なくとも、私は彼らがそうだとは思いません。

いずれにせよ、データを送信する前に何らかの方法でエンコードするのが正しい方法なのです！


## 7.6 Son of Data Encapsulation {#sonofdataencap}

ところで、データをカプセル化するとはどういうことでしょうか。最も単純なケースでは、識別情報かパケット長のどちらか、あるいは両方を含むヘッダーをそこに貼り付けるということです。

ヘッダーはどのようなものにすべきでしょうか？まあ、プロジェクトを完成させるために必要だと思うものを表すバイナリデータに過ぎません。

うわー。漠然としてますね。

なるほど。例えば、`SOCK_STREAM` を使用したマルチユーザチャットプログラムがあるとします。ユーザが何かをタイプする ("発言する") と、2つの情報がサーバに送信される必要があります: 何を発言したか、誰が発言したかです。

ここまでは良いですか？"では何が問題なのか？"とあなたは聞いているのでしょう。

問題は、メッセージの長さがまちまちであることです。ある人は "tom" と名乗り、ある人は "Benjamin" と名乗り、"Hey guys what is up?" と言うかもしれません。

そこで、これらのものが入ってきたときに `send()` でクライアントに送るわけです。送信するデータストリームは次のようになります。

```
t o m H i B e n j a m i n H e y g u y s w h a t i s u p ?
```

といった具合に。あるメッセージが始まり、別のメッセージが止まったとき、クライアントはどうやって知るのでしょうか？もし望むなら、すべてのメッセージを同じ長さにして、[上で実装](docs/slightly-advanced-techniques/#sendall)した `sendall()` を呼び出すだけでよいでしょう。しかし、それはバンド幅を浪費します！"tom" が "Hi" と言うために 1024 バイトも `send()` したくはないでしょう。

そこで、データを小さなヘッダーとパケット構造でカプセル化します。クライアントもサーバも、このデータをどのようにパックしたりアンパックしたりするか（"マーシャル""アンマーシャル"と呼ばれることもあります）知っています。今は見ないでください、私たちは、クライアントとサーバがどのように通信するかを記述するプロトコルを定義し始めているのです

この場合、ユーザ名は固定長で8文字、パディングは `'\0'` とします。そして、データは最大 128 文字の可変長とします。このような場合に使用するパケット構造の例を見てみましょう。

1. `len` (1 byte, unsigned)---8 バイトのユーザ名とチャットデータをカウントしたパケットの総長です。

2. `name` (8 bytes)---ユーザの名前。必要なら NUL パッドされます。

3. `chatdata` (n-bytes) --- データ自体で、128 バイト以下です。パケットの長さは、このデータの長さに 8（上記の名前フィールドの長さ）を加えたものとします。

なぜ、8 バイトと 128 バイトというフィールドの制限を選んだのか？十分な長さがあると思い、思いつきで選んだのです。しかし、8 バイトでは制約が多すぎる、30 バイトの名前フィールドがあってもいい、などということもあるかもしれません。 選択はあなた次第です。

上記のパケット定義を用いると、最初のパケットは以下の情報（16 進数、ASCII）で構成されることになります。

```
   0A     74 6F 6D 00 00 00 00 00      48 69
(length)  T  o  m    (padding)         H  i
```

そして、2つ目も同様です。

```
   18     42 65 6E 6A 61 6D 69 6E      48 65 79 20 67 75 79 73 20 77 ...
(length)  B  e  n  j  a  m  i  n       H  e  y     g  u  y  s     w  ...
```

（もちろん、長さはネットワークバイトオーダーで格納されます。この場合は 1 バイトだけなので問題ないですが、一般的にはパケット内の 2 進整数はすべてネットワークバイトオーダーで格納されるようにしたいです。）

このデータを送信するときには、安全策をとって上記の [`sendall()`](docs/slightly-advanced-techniques/#sendall) のようなコマンドを使うべきです。そうすれば、たとえすべてのデータを送信するために `send()` を複数回呼び出す必要があったとしても、すべてのデータが送信されたことを確認できます。

同様に、このデータを受信するときにも、少し余分な作業をする必要があります。念のため、部分的なパケットを受け取るかもしれないことを想定しておく必要があります（例えば、上の Benjamin から "`18 42 65 6E 6A`" を受け取るかもしれませんが、この `recv()` のコールではそれがすべてです。）。パケットが完全に受信されるまで、何度も何度も `recv()` を呼び出す必要があります。

でも、どうやって？さて、私たちはパケットが完成するために必要な合計バイト数を知っています。その数はパケットの前面に貼られているからです。 また、パケットの最大サイズは 1+8+128、つまり 137 バイトであることも知っています（パケットの定義がそうなっているからです）。

実は、ここでできることがいくつかあるんです。すべてのパケットが長さで始まることを知っているので、パケットの長さを取得するためだけに `recv()` を呼び出すことができます。そして、一度それを手に入れたら、パケットの残りの長さを正確に指定して（おそらくすべてのデータを得るために繰り返し）、完全なパケットを手に入れるまで再びそれを呼び出すことができます。この方法の利点は、1つのパケットに対して十分な大きさのバッファが必要なことで、欠点は、すべてのデータを取得するために少なくとも2回 `recv()` を呼び出す必要があることです。

もう一つの方法は、単に `recv()` を呼び出して、受信してもよい量をパケットの最大バイト数として言うことです。そして、受け取ったものはすべてバッファの後ろに貼り付け、最後にパケットが完了したかどうかを確認します。もちろん、次のパケットの一部を受け取るかもしれないので、そのためのスペースが必要です。

そこで、2つのパケットに十分な大きさの配列を宣言します。これは作業用の配列で、到着したパケットを再構築します。

データを `recv()` するたびに、ワークバッファにデータを追加し、パケットが完成したかどうかチェックします。すなわち、バッファ内のバイト数がヘッダで指定された長さ以上であること（ヘッダの長さには長さ自体のバイト数は含まれないので、+1）です。バッファ内のバイト数が1より小さい場合、明らかにパケットは完全ではありません。この場合、特別なケースを作る必要があります。しかし、最初の 1 バイトはゴミなので、正しいパケット長を知るためにそれを当てにすることはできないからです。

パケットを完成させたら、あとは好きなように使ってください。使って、ワークバッファから削除してください。

ふぅ〜。もう頭の中でグルグルしてますか？さて、ここでワンツーパンチの2つ目です。1回の `recv()` 呼び出しで、あるパケットの終わりを越えて次のパケットを読んでしまったかもしれません。つまり、一つの完全なパケットと、次のパケットの不完全な部分を持つワークバッファを持っているのです！なんてこったい。（しかし、このような事態を想定して、ワークバッファを2つのパケットを保持できるような大きさにしたのです！）

最初のパケットの長さはヘッダからわかっているし、ワークバッファのバイト数も記録しているので、ワークバッファのバイト数のうち何バイトが2番目の（不完全な）パケットに属しているかを差し引いて計算することができるのです。最初のパケットを処理したら、それをワークバッファから取り除き、部分的な2番目のパケットをバッファの先頭に移動させ、次の `recv() ` のためにすべての準備をすることができます。

（読者の中には、実際に部分的な2番目のパケットをワークバッファの先頭に移動させるのに時間がかかることを指摘する人もいるでしょう。プログラムは、循環バッファを使用することによって、これを必要としないようにコード化することができます。残念ながら、円形バッファに関する議論は、この記事の範囲外です。それでも気になるなら、データ構造の本を読んでみてください。）

簡単だとは言っていません。なるほど、簡単だとは言いました。練習すれば、すぐに自然にできるようになりますよ。エクスカリバーに誓って！


## 7.7 Broadcast Packets---Hello, World!

これまで、このガイドでは、1つのホストから他の1つのホストにデータを送信することについて話してきました。しかし、適切な権限があれば、同時に複数のホストにデータを送信することが可能です、と私は主張します！

UDP（TCP ではなく UDP のみ）と標準的な IPv4 では、これはブロードキャストと呼ばれるメカニズムによって行われます。IPv6 ではブロードキャストはサポートされていないため、マルチキャストという優れた技術を利用する必要があります。しかし、未来に目を向けるのはもう十分です。私たちは 32 ビットの現在に留まっています。

しかし、ちょっと待ってください！いきなりブロードキャストを始めることはできません。ブロードキャストパケットをネットワークに送信する前に、ソケットオプション `SO_BROADCAST` をセットしなければなりません。まるでミサイルの発射スイッチの上に被せる小さなプラスチックのカバーのようなものです。そのくらいのパワーを秘めているのです。

しかし、真面目な話、ブロードキャストパケットを使うには危険性があるのです。ブロードキャストパケットを受信したすべてのシステムは、そのデータの行き先がどのポートか分かるまで、何層にもわたるデータのカプセル化を解かなければならないのです。そして、そのデータを渡すか、廃棄するかしなければなりません。どちらの場合でも、ブロードキャストパケットを受信した各マシンは、ローカルネットワーク上のすべてのマシンで、多くのマシンが不必要な作業をすることになりかねません。ゲーム Doom が発売された当初、これはそのネットワークコードに対する不満でした。

さて、猫の皮を剥ぐ方法は一つではない......ちょっと待った。本当に猫の皮を剥ぐ方法は1つではないのですか？どんな表現なんだ？それと同じように、ブロードキャストパケットを送る方法も1つではありません。では、本題のブロードキャストメッセージの宛先アドレスはどのように指定するのでしょうか？一般的には2つの方法があります。

1. 特定のサブネットのブロードキャストアドレスにデータを送信します。これは、サブネットのネットワーク番号に、アドレスのホスト部分のすべての 1 ビットを設定したものです。例えば、私の家のネットワークは `192.168.1.0`、ネットマスクは `255.255.255.0` で、アドレスの最後のバイトは私のホスト番号です（ネットマスクによると、最初の 3 バイトはネットワーク番号だからです）。つまり、私のブロードキャストアドレスは `192.168.1.255` です。Unix では、`ifconfig` コマンドが実際にこのすべてのデータを与えてくれます。（気になる方は、ブロードキャストアドレスを得るためのビット論理は `network_number` OR (NOT `netmask`) です。）このタイプのブロードキャストパケットは、自分のローカルネットワークだけでなく、遠隔地のネットワークにも送ることができますが、宛先のルータによってパケットがドロップされる危険性があります。（もしドロップされなかったら、どこかのランダムなスマーフが彼らの LAN にブロードキャストトラフィックを流し始めるかもしれません。）

2. "グローバル"ブロードキャストアドレスにデータを送信します。これは `255.255.255.255` で、別名 `INADDR_BROADCAST` と呼ばれるものです。多くのマシンはこれをネットワーク番号と自動的にビット演算して、ネットワークブロードキャストアドレスに変換しますが、そうでないものもあります。ルータは、皮肉なことに、この種のブロードキャストパケットをローカルネットワークから転送しません。

では、ソケットオプション `SO_BROADCAST` を設定せずに、ブロードキャストアドレスでデータを送信しようとするとどうなるでしょうか。それでは、古き良き時代の [`talker` and `listener`](docs/client-server-background/#datagram) を起動し、何が起こるか見てみましょう。

```
$ talker 192.168.1.2 foo
sent 3 bytes to 192.168.1.2
$ talker 192.168.1.255 foo
sendto: Permission denied
$ talker 255.255.255.255 foo
sendto: Permission denied
```

そう、全然嬉しくないんです...ソケットオプションの `SO_BROADCAST` を設定していなかったからです。そうすれば、どこでも `sendto()` ができるようになります！

実際、これがブロードキャストできる UDP アプリケーションとできない UDP アプリケーションの唯一の違いなのです。そこで、古い `talker` アプリケーションに `SO_BROADCAST` ソケットオプションを設定するセクションを一つ追加してみましょう。このプログラムを [`broadcaster.c`](https://beej.us/guide/bgnet/examples/broadcaster.c) と呼ぶことにします。

```{.c .numberLines}
/*
** broadcaster.c -- a datagram "client" like talker.c, except
**                  this one can broadcast
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define SERVERPORT 4950 // the port users will be connecting to

int main(int argc, char *argv[])
{
    int sockfd;
    struct sockaddr_in their_addr; // connector's address information
    struct hostent *he;
    int numbytes;
    int broadcast = 1;
    //char broadcast = '1'; // if that doesn't work, try this

    if (argc != 3) {
        fprintf(stderr,"usage: broadcaster hostname message\n");
        exit(1);
    }

    if ((he=gethostbyname(argv[1])) == NULL) {  // get the host info
        perror("gethostbyname");
        exit(1);
    }

    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) == -1) {
        perror("socket");
        exit(1);
    }

    // this call is what allows broadcast packets to be sent:
    if (setsockopt(sockfd, SOL_SOCKET, SO_BROADCAST, &broadcast,
        sizeof broadcast) == -1) {
        perror("setsockopt (SO_BROADCAST)");
        exit(1);
    }

    their_addr.sin_family = AF_INET;     // host byte order
    their_addr.sin_port = htons(SERVERPORT); // short, network byte order
    their_addr.sin_addr = *((struct in_addr *)he->h_addr);
    memset(their_addr.sin_zero, '\0', sizeof their_addr.sin_zero);

    if ((numbytes=sendto(sockfd, argv[2], strlen(argv[2]), 0,
             (struct sockaddr *)&their_addr, sizeof their_addr)) == -1) {
        perror("sendto");
        exit(1);
    }

    printf("sent %d bytes to %s\n", numbytes,
        inet_ntoa(their_addr.sin_addr));

    close(sockfd);

    return 0;
}
```

"通常の" UDP クライアント/サーバーの状況と何が違うのでしょうか？何もありません。（この場合、クライアントがブロードキャストパケットの送信を許可されていることを除けば。）このように、古い UDP [`listener`](docs/client-server-background/#datagram) プログラムを1つのウィンドウで実行し、`broadcaster` を別のウィンドウで実行してみてください。これで、上で失敗した送信をすべて行うことができるはずです。

```
$ broadcaster 192.168.1.2 foo
sent 3 bytes to 192.168.1.2
$ broadcaster 192.168.1.255 foo
sent 3 bytes to 192.168.1.255
$ broadcaster 255.255.255.255 foo
sent 3 bytes to 255.255.255.255
```

そして、`listener` がパケットを受け取ったと応答するのが見えるはずです。（もし `listener` が応答しないなら、それは IPv6 アドレスにバインドされているからかもしれません。`listener.c` の `AF_UNSPEC` を `AF_INET` に変更して、強制的に IPv4 にすることを試してみてください。）

なるほど、それはちょっと楽しみですね。しかし、今度は同じネットワーク上にある隣のマシンで `listener` を起動して、それぞれのマシンに2つのコピーを作成し、ブロードキャストアドレスを指定して `broadcaster` を再度実行してみてください... おい！両方の `listener` がパケットを受け取るぞ！`sendto()` を一度しか呼んでいないのにな。かっこいい！

もし `listener` が、あなたが直接送ったデータを受け取って、ブロードキャストアドレスのデータを受け取らないなら、あなたのローカルマシンのファイアウォールがパケットをブロックしている可能性があります。（そうそう、Pat とBapper、これが私のサンプルコードが動作しない理由だと、私より先に気づいてくれてありがとうございます。ガイドの中で紹介すると言っておいたのに、こうして紹介してくれて。というわけで、にゃー。）

繰り返しになりますが、ブロードキャストパケットには注意が必要です。LAN 上のすべてのマシンが `recvfrom()` したかどうかに関わらず、そのパケットを処理しなければならないので、計算機ネットワーク全体にかなりの負荷を与える可能性があります。ブロードキャストパケットは控えめに、そして適切に使用されるべきものであることは間違いありません。
