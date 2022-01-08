# 5.1 `getaddrinfo()`---起動の準備をしよう！

この関数は多くのオプションを持つ真の主力関数ですが、使い方はいたってシンプルです。後で必要な構造体をセットアップするのに役立ちます。

昔は、`gethostbyname()` という関数を使って DNS のルックアップを行っていました。そして、その情報を `sockaddr_in` 構造体に手作業でロードし、それを呼び出しに使用するのです。

これは、ありがたいことに、もう必要ありません。（IPv4 と IPv6 の両方で動作するコードを書きたいのであれば、望ましいことではありません！）現代では、DNS やサービス名のルックアップなど、あらゆる種類の良いことをやってくれる `getaddrinfo()` という関数があり、さらに必要な `struct` も埋めてくれます！

それでは、ご覧ください！

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

int getaddrinfo(const char *node,     // e.g. "www.example.com" or IP
                const char *service,  // e.g. "http" or port number
                const struct addrinfo *hints,
                struct addrinfo **res);
```

この関数に3つの入力パラメータを与えると、結果のリンクリストである `res` へのポインタが得られます。

`node` パラメータには、接続先のホスト名、または IP アドレスを指定します。

次にパラメータ `service` ですが、これは "80" のようなポート番号か、"http", "ftp", "telnet", "smtp" などの特定のサービスの名前（[IANAポートリスト](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)や Unix マシンの `/etc/services` ファイルで見つけることができます）であることができます。

最後に、`hints` パラメータは、関連情報をすでに記入した `addrinfo` 構造体を指します。

以下は、自分のホストの IP アドレス、ポート 3490 をリッスンしたいサーバの場合の呼び出し例です。これは実際にはリスニングやネットワークの設定を行っていないことに注意してください。

```c
int status;
struct addrinfo hints;
struct addrinfo *servinfo;  // will point to the results

memset(&hints, 0, sizeof hints); // make sure the struct is empty
hints.ai_family = AF_UNSPEC;     // don't care IPv4 or IPv6
hints.ai_socktype = SOCK_STREAM; // TCP stream sockets
hints.ai_flags = AI_PASSIVE;     // fill in my IP for me

if ((status = getaddrinfo(NULL, "3490", &hints, &servinfo)) != 0) {
    fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(status));
    exit(1);
}

// servinfo now points to a linked list of 1 or more struct addrinfos

// ... do everything until you don't need servinfo anymore ....

freeaddrinfo(servinfo); // free the linked-list
```

`ai_family` を `AF_UNSPEC` に設定することで、IPv4 や IPv6 を使うかどうかを気にしないことを表明していることに注意してください。もし、どちらか一方だけを使いたい場合は、`AF_INET` または `AF_INET6` に設定することができます。

また、`AI_PASSIVE` フラグがあるのがわかると思いますが、これは `getaddrinfo()` にローカルホストのアドレスをソケット構造体に割り当てるように指示しています。これは、ハードコードする必要がないのがいいところです。(あるいは、`getaddrinfo()` の最初のパラメータとして特定のアドレスを入れることもできます。私は現在 `NULL` を持っています。)

そして、呼び出しを行います。エラー（`getaddrinfo()` が `0` 以外を返します）があれば、ご覧のように関数 `gai_strerror()` を使ってそれを表示することができます。しかし、すべてがうまくいけば、`servinfo` は `struct addrinfos` のリンクリストを指し、それぞれのリストには後で使用できる何らかの `sockaddr` 構造体が含まれています！素晴らしい！

最後に、`getaddrinfo()` が快く割り当ててくれたリンクリストをすべて使い終わったら、`freeaddrinfo()` を呼び出してすべてを解放することができます（そうすべき）です。

ここでは、クライアントが特定のサーバ、例えば "www.example.net" ポート 3490 に接続したい場合のサンプルコールを紹介します。繰り返しますが、これは実際には接続しませんが、後で使用する構造をセットアップしています。

```c
int status;
struct addrinfo hints;
struct addrinfo *servinfo;  // will point to the results

memset(&hints, 0, sizeof hints); // make sure the struct is empty
hints.ai_family = AF_UNSPEC;     // don't care IPv4 or IPv6
hints.ai_socktype = SOCK_STREAM; // TCP stream sockets

// get ready to connect
status = getaddrinfo("www.example.net", "3490", &hints, &servinfo);

// servinfo now points to a linked list of 1 or more struct addrinfos

// etc.
```

`servinfo` は、あらゆるアドレス情報を持つリンクリストだと言い続けています。この情報を披露するために、簡単なデモプログラムを書いてみよう。[この短いプログラム](https://beej.us/guide/bgnet/examples/showip.c)は、コマンドラインで指定された任意のホストの IP アドレスを表示します。

```c
/*
** showip.c -- show IP addresses for a host given on the command line
*/

#include <stdio.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <arpa/inet.h>
#include <netinet/in.h>

int main(int argc, char *argv[])
{
    struct addrinfo hints, *res, *p;
    int status;
    char ipstr[INET6_ADDRSTRLEN];

    if (argc != 2) {
        fprintf(stderr,"usage: showip hostname\n");
        return 1;
    }

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC; // AF_INET or AF_INET6 to force version
    hints.ai_socktype = SOCK_STREAM;

    if ((status = getaddrinfo(argv[1], NULL, &hints, &res)) != 0) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(status));
        return 2;
    }

    printf("IP addresses for %s:\n\n", argv[1]);

    for(p = res;p != NULL; p = p->ai_next) {
        void *addr;
        char *ipver;

        // get the pointer to the address itself,
        // different fields in IPv4 and IPv6:
        if (p->ai_family == AF_INET) { // IPv4
            struct sockaddr_in *ipv4 = (struct sockaddr_in *)p->ai_addr;
            addr = &(ipv4->sin_addr);
            ipver = "IPv4";
        } else { // IPv6
            struct sockaddr_in6 *ipv6 = (struct sockaddr_in6 *)p->ai_addr;
            addr = &(ipv6->sin6_addr);
            ipver = "IPv6";
        }

        // convert the IP to a string and print it:
        inet_ntop(p->ai_family, addr, ipstr, sizeof ipstr);
        printf("  %s: %s\n", ipver, ipstr);
    }

    freeaddrinfo(res); // free the linked list

    return 0;
}
```

ご覧のように、このコードはコマンドラインで渡されたものに対して `getaddrinfo()` を呼び出し、`res` が指すリンクリストを埋めて、そのリストを繰り返し表示して何かを出力したりすることができます。

（そこには、IP バージョンによって異なるタイプの `struct sockaddrs` を掘り下げなければならない、ちょっとした醜さがあります。申し訳ありません。他にいい方法はないかなぁ...）

サンプル走行！みんな大好きスクリーンショット。

```
$ showip www.example.net
IP addresses for www.example.net:

  IPv4: 192.0.2.88

$ showip ipv6.example.com
IP addresses for ipv6.example.com:

  IPv4: 192.0.2.101
  IPv6: 2001:db8:8c00:22::171
```

これで、`getaddrinfo()` の結果を他のソケット関数に渡して、ついにネットワーク接続を確立することができます。引き続きお読みください。
