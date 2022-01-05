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

[このプログラム](https://beej.us/guide/bgnet/examples/selectserver.c)は、簡単なマルチユーザチャットサーバーのように動作します。一つのウィンドウで起動し、他の複数のウィンドウから `telnet` ("`telnet hostname 9034`") で接続してください。ある `telnet` セッションで何かを入力すると、他のすべてのウィンドウに表示されるはずです。

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


