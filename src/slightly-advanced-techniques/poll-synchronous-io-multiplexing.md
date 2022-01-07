# 7.2 `poll()`---同期式 I/O 多重化

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

``` c
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

``` c
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
