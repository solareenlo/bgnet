# 5.6 `accept()`---"3490番ポートにコールいただきありがとうございます。"

`accept()` の呼び出しはちょっと変です。これから起こることはこうです。遠く離れた誰かが、あなたが `listen()` しているポートであなたのマシンに `connect()` しようとするでしょう。その接続は、`accept()` されるのを待つためにキューに入れられることになります。あなたは `accept()` をコールし、保留中の接続を取得するように指示します。すると、この接続に使用する新しいソケットファイル記述子が返されます！そうです、1つの値段で2つのソケットファイル記述子を手に入れたことになります。元のソケットファイル記述子はまだ新しい接続を待ち続けており、新しく作成されたソケットファイル記述子はようやく `send()` と `recv()` を行う準備が整いました。着いたぞ！

書式は以下の通りです。

```c
#include <sys/types.h>
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

`sockfd` は `listen()` するソケット記述子です。簡単ですね。`addr` は通常、ローカルの `struct sockaddr_storage` へのポインタになります。この構造体には、着信接続に関する情報が格納されます（これにより、どのホストがどのポートからコールをかけてきたかを判断することができます）。`addrlen` はローカルの整数型変数で、そのアドレスが `accept()` に渡される前に `sizeof(struct sockaddr_storage)` に設定されなければなりません。`accept()` は、`addr` に `addrlen` 以上のバイト数を入れることはありません。もし、それ以下のバイト数であれば、それを反映するように `addrlen` の値を変更します。

何だと思いますか？`accept()` はエラーが発生した場合は `-1` を返し、`errno` をセットします。そうだったんですか。

前回と同様、一度に吸収するのは大変なので、サンプルコードの一部をご覧ください。

```c
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

#define MYPORT "3490"  // the port users will be connecting to
#define BACKLOG 10     // how many pending connections queue will hold

int main(void)
{
    struct sockaddr_storage their_addr;
    socklen_t addr_size;
    struct addrinfo hints, *res;
    int sockfd, new_fd;

    // !! don't forget your error checking for these calls !!

    // first, load up address structs with getaddrinfo():

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;  // use IPv4 or IPv6, whichever
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE;     // fill in my IP for me

    getaddrinfo(NULL, MYPORT, &hints, &res);

    // make a socket, bind it, and listen on it:

    sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
    bind(sockfd, res->ai_addr, res->ai_addrlen);
    listen(sockfd, BACKLOG);

    // now accept an incoming connection:

    addr_size = sizeof their_addr;
    new_fd = accept(sockfd, (struct sockaddr *)&their_addr, &addr_size);

    // ready to communicate on socket descriptor new_fd!
    .
    .
    .
```

ここでも、すべての `send()` と `recv()` の呼び出しに、ソケット記述子 `new_fd` を使用することに注意してください。もし、一度しか接続がないのであれば、同じポートからの接続を防ぐために、`listen` している `sockfd` を `close()` することができます。
