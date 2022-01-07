---
weight: 1
bookFlatSection: true
title: "5 System Calls or Bust"
---

# 5 System Calls or Bust

このセクションでは、Unix マシンのネットワーク機能にアクセスするためのシステムコールやその他のライブラリコールに触れることができますし、ソケット API をサポートしているあらゆるマシン (BSD, Windows, Linux, Mac, など) も同様です。これらの関数を呼び出すと、カーネルが引き継ぎ、すべての作業を自動で行ってくれます。

このあたりで多くの人がつまづくのは、これらのものをどのような順序で呼び出すかということです。これについては、皆さんもお分かりのように、`man` ページが役に立ちません。そこで、この恐ろしい状況を改善するために、以下の章のシステムコールを、あなたがプログラムの中で呼び出す必要があるのと全く（おおよそ）同じ順序で並べることにしました。

これに、あちこちにあるサンプルコード、ミルクとクッキー（自分で用意しなければならないのが怖い）、そして生粋のガッツと勇気があれば、ジョン・ポステルの息子のようにインターネット上でデータを発信することができるのです！

(なお、以下の多くのコードでは、簡潔にするため、必要なエラーチェックは行っていません。また、`getaddrinfo()` の呼び出しが成功し、リンクリストの有効なエントリを返すと仮定することが非常に一般的です。これらの状況はいずれもスタンドアロン・プログラムで適切に対処されているので、それらをモデルとして使用してください。)


## 5.1 `getaddrinfo()`---Prepare to launch!

この関数は多くのオプションを持つ真の主力関数ですが、使い方はいたってシンプルです。後で必要な構造体をセットアップするのに役立ちます。

昔は、`gethostbyname()` という関数を使って DNS のルックアップを行っていました。そして、その情報を `sockaddr_in` 構造体に手作業でロードし、それを呼び出しに使用するのです。

これは、ありがたいことに、もう必要ありません。(IPv4 と IPv6 の両方で動作するコードを書きたいのであれば、望ましいことではありません！) 現代では、DNS やサービス名のルックアップなど、あらゆる種類の良いことをやってくれる `getaddrinfo()` という関数があり、さらに必要な `struct` も埋めてくれます！

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

```{.c .numberLines}
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

そして、呼び出しを行います。エラー(`getaddrinfo()` が0以外を返す)があれば、ご覧のように関数 `gai_strerror()` を使ってそれを表示することができます。しかし、すべてがうまくいけば、`servinfo` は `struct addrinfos` のリンクリストを指し、それぞれのリストには後で使用できる何らかの `sockaddr` 構造体が含まれています！素晴らしい！

最後に、`getaddrinfo()` が快く割り当ててくれたリンクリストをすべて使い終わったら、`freeaddrinfo()` を呼び出してすべてを解放することができます(そうすべき)です。

ここでは、クライアントが特定のサーバ、例えば "www.example.net" ポート 3490 に接続したい場合のサンプルコールを紹介します。繰り返しますが、これは実際には接続しませんが、後で使用する構造をセットアップしています。

```{.c .numberLines}
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

```{.c .numberLines}
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

(そこには、IP バージョンによって異なるタイプの `struct sockaddrs` を掘り下げなければならない、ちょっとした醜さがあります。申し訳ありません。他にいい方法はないかなぁ...)

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


## 5.2 `socket()`---Get the File Descriptor! {#socket}

もう先延ばしにはできません。`socket()` システムコールの話をしなければならないのです。以下はその内訳です。

```c
#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

しかし、これらの引数は何なのでしょうか？これらは、どのようなソケットが欲しいか（IPv4 か IPv6 か、ストリームかデータグラムか、TCP か UDP か）を指定することができます。

以前は、これらの値をハードコードする人がいましたが、今でも絶対にそうすることができます。(ドメインは `PF_INET` または `PF_INET6`、タイプは `SOCK_STREAM` または `SOCK_DGRAM`、プロトコルは `0` に設定すると、与えられたタイプに適したプロトコルを選択することができます。あるいは `getprotobyname()` を呼んで、"tcp" や "udp" などの欲しいプロトコルを調べることもできます。)

(この `PF_INET` は、`struct sockaddr_in` の `sin_family` フィールドを初期化するときに使用できる `AF_INET` の近縁種です。実際、両者は非常に密接な関係にあり、実際に同じ値を持っているので、多くのプログラマは `socket()` を呼び出して `PF_INET` の代わりに `AF_INET` を第一引数に渡しています。さて、ミルクとクッキーを用意して、お話の時間です。昔々、あるアドレスファミリ(`AF_INET` の AF)が、プロトコルファミリ(`PF_INET` の PF)で参照される複数のプロトコルをサポートするかもしれないと考えられたことがあります。しかし、そうはなりませんでした。そして、みんな幸せに暮らした、ザ・エンド。というわけで、最も正しいのは `struct sockaddr_in` で `AF_INET` を使い、`socket()` の呼び出しで `PF_INET` を使うことです。)

とにかく、もう十分です。本当にやりたいことは、`getaddrinfo()` の呼び出しの結果の値を使い、以下のように直接 `socket()` に送り込むことです。

```{.c .numberLines}
int s;
struct addrinfo hints, *res;

// do the lookup
// [pretend we already filled out the "hints" struct]
getaddrinfo("www.example.com", "http", &hints, &res);

// again, you should do error-checking on getaddrinfo(), and walk
// the "res" linked list looking for valid entries instead of just
// assuming the first one is good (like many of these examples do).
// See the section on client/server for real examples.

s = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
```

`socket()` は単に、後のシステムコールで使用できるソケットディスクリプタを返すか、エラーの場合は `-1` を返します。グローバル変数 `errno` にはエラーの値が設定されます (詳細については [`errno`(#errnoman) のマニュアルページを参照してください。また、マルチスレッドプログラムで `errno` を使用する際の簡単な注意も参照してください)。

でも、このソケットは何の役に立つのでしょうか？答えは、これだけでは本当に意味がなく、もっと読み進めてシステムコールを作らないと意味がないのです。


## 5.3 `bind()`---What port am I on? {#bind}

ソケットを取得したら、そのソケットをローカルマシンのポートに関連付ける必要があるかもしれません。(これは、特定のポートへの接続を `listen()` する場合によく行われます。多人数参加型ネットワークゲームで "192.168.5.10 ポート 3490 に接続" と指示されたときに行います)。ポート番号はカーネルが受信パケットを特定のプロセスのソケットディスクリプタにマッチさせるために使用されます。もしあなたが `connect()` を行うだけなら(あなたはクライアントであり、サーバではないので)、これはおそらく不要でしょう。とにかく読んでみてください。

`bind()` システムコールの概要は以下のとおりです。

```c
#include <sys/types.h>
#include <sys/socket.h>

int bind(int sockfd, struct sockaddr *my_addr, int addrlen);
```

`sockfd` は `socket()` が返すソケットファイル記述子です。`my_addr` は自分のアドレスに関する情報、すなわちポートおよび IP アドレスを含む `sockaddr` 構造体へのポインタです。

ふぅー。一度に吸収するのはちょっと無理があるな。ソケットをプログラムが実行されているホスト、ポート 3490 にバインドする例を見てみましょう。

```{.c .numberLines}
struct addrinfo hints, *res;
int sockfd;

// first, load up address structs with getaddrinfo():

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;  // use IPv4 or IPv6, whichever
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;     // fill in my IP for me

getaddrinfo(NULL, "3490", &hints, &res);

// make a socket:

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

// bind it to the port we passed in to getaddrinfo():

bind(sockfd, res->ai_addr, res->ai_addrlen);
```

`AI_PASSIVE` フラグを使うことで、プログラムが動作しているホストの IP にバインドするように指示しているのです。もし、特定のローカル IP アドレスにバインドしたい場合は、`AI_PASSIVE` を削除して、`getaddrinfo()` の最初の引数に IP アドレスを入れてください。

`bind()` もエラー時には `-1` を返し、`errno` にエラーの値を設定します。

多くの古いコードでは、`bind()` を呼び出す前に、`struct sockaddr_in` を手動でパックしています。これは明らかに IPv4 特有のものですが、IPv6 で同じことをするのを止めるものは何もありません。ただし、一般的には `getaddrinfo()` を使う方が簡単になりそうです。とにかく、古いコードは次のようなものです。

```{.c .numberLines}
// !!! THIS IS THE OLD WAY !!!

int sockfd;
struct sockaddr_in my_addr;

sockfd = socket(PF_INET, SOCK_STREAM, 0);

my_addr.sin_family = AF_INET;
my_addr.sin_port = htons(MYPORT);     // short, network byte order
my_addr.sin_addr.s_addr = inet_addr("10.12.110.57");
memset(my_addr.sin_zero, '\0', sizeof my_addr.sin_zero);

bind(sockfd, (struct sockaddr *)&my_addr, sizeof my_addr);
```

上記のコードでは、ローカルの IP アドレスにバインドしたい場合、`s_addr` フィールドに `INADDR_ANY` を代入することもできます（上記の `AI_PASSIVE` フラグのようなものです）。`INADDR_ANY` の IPv6 バージョンはグローバル変数 `in6addr_any` で、`struct sockaddr_in6` の `sin6_addr` フィールドに代入されます。(変数の初期化で使用できるマクロ `IN6ADDR_ANY_INIT` も存在します。)また、`IN6ADDR_ANY_INIT` を使用することで、IPv6 の IP アドレスにバインドできます。

`bind()` を呼ぶときにもうひとつ気をつけなければならないのは、ポート番号で下手を打たないことです。1024 以下のポートはすべて予約済みです(あなたがスーパーユーザでない限り)！それ以上のポート番号は、(他のプログラムによってすでに使われていなければ) 65535 までの任意のポート番号を使用することができます。

時々、サーバを再実行しようとすると、`bind()` が "Address already in use" と言って失敗することに気がつくかもしれません。これはどういうことでしょう? それは、接続されたソケットの一部がまだカーネル内に残っていて、ポートを占有しているのです。それが消えるのを待つか(1分くらい)、次のようにポートが再利用できるようなコードをプログラムに追加します。

```{.c .numberLines}
int yes=1;
//char yes='1'; // Solaris people use this

// lose the pesky "Address already in use" error message
if (setsockopt(listener,SOL_SOCKET,SO_REUSEADDR,&yes,sizeof yes) == -1) {
    perror("setsockopt");
    exit(1);
}
```

`bind()` について、最後にちょっとした注意点があります。`bind()` を絶対に呼び出す必要がない場合があります。リモートマシンに `connect()` する際に、ローカルポートを気にしない場合 (telnet のようにリモートポートを気にする場合) は、単に `connect()` をコールすれば、ソケットが未束縛かどうかをチェックし、必要なら未使用のローカルポートに `bind()` してくれます。

## 5.4 `connect()`---Hey, you! {#connect}

ちょっとだけ、あなたが telnet アプリケーションであることを仮定してみましょう。ユーザが（映画 TRON のように）ソケットファイル記述子を取得するように命令します。あなたはそれに応じ、`socket()` を呼び出します。次に、ユーザはポート "`23`" (標準的な telnet ポート) で "`10.12.110.57`" に接続するように指示します。やったー! どうするんだ？

幸運なことに、あなたは今、`connect()`の章---リモートホストに接続する方法を読んでいるところです。だから、猛烈に読み進めよう！時間がない！

`connect()`の呼び出しは以下の通りです。

```c
#include <sys/types.h>
#include <sys/socket.h>

int connect(int sockfd, struct sockaddr *serv_addr, int addrlen);
```

`sockfd` は `socket()` コールで返される、我々の身近なソケットファイル記述子、`serv_addr` は宛先ポートと IP アドレスを含む `struct sockaddr`、`addrlen` はサーバアドレス構造体のバイト長です。

これらの情報はすべて、`getaddrinfo()` の呼び出しの結果から得ることができ、これはロックします。

だんだん分かってきたかな？ここからは聞こえないので、そうであることを祈るしかないですね。ポート `3490` の "`www.example.com`" にソケット接続する例を見てみましょう。

```{.c .numberLines}
struct addrinfo hints, *res;
int sockfd;

// first, load up address structs with getaddrinfo():

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;

getaddrinfo("www.example.com", "3490", &hints, &res);

// make a socket:

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

// connect!

connect(sockfd, res->ai_addr, res->ai_addrlen);
```

繰り返しになりますが、古いタイプのプログラムでは、独自の `struct sockaddr_ins` を作成して `connect()` に渡していました。必要であれば、そうすることができます。上の [`bind()`](docs/system-calls-or-bust/#bind) の節で同様のことを書いています。

`connect()` の戻り値を必ず確認してください。エラー時に `-1` が返され、`errno` という変数がセットされます。

また、`bind()` を呼んでいないことに注意してください。基本的に、私たちはローカルのポート番号には関心がありません。カーネルは私たちのためにローカルポートを選択し、接続先のサイトは自動的にこの情報を取得します。心配はいりません。


## 5.5 `listen()`---Will somebody please call me? {#listen}

よし、気分転換の時間です。リモートホストに接続したくない場合はどうすればいいのでしょう。例えば、接続が来るのを待ち、何らかの方法でそれを処理したいとします。この処理は2段階です。まず `listen()` を行い、次に `accept()` を行います (後述)。

`listen()` の呼び出しはかなり単純ですが、少し説明が必要です。

```c
int listen(int sockfd, int backlog);
```

`sockfd` は `socket()` システムコールから得られる通常のソケットファイル記述子です。これはどういう意味でしょうか？着信した接続は、`accept()` (後述) するまでこのキューで待機することになりますが、このキューに入れることができる数の上限を表しているのです。ほとんどのシステムでは、この数を黙って約 20 に制限しています。おそらく、`5` や `10` に設定しても大丈夫でしょう。

ここでも、いつものように `listen()` はエラー時に `-1` を返し、`errno` をセットします。

さて、想像がつくと思いますが、サーバが特定のポートで動作するように `listen()` を呼び出す前に `bind()` を呼び出す必要があります。(どのポートに接続するかを仲間に伝えることができなければなりません！) ですから、もし接続を待ち受けるのであれば、一連のシステムコールは次のようになります。

```{.c .numberLines}
getaddrinfo();
socket();
bind();
listen();
/* accept() goes here */
```

かなり自明なので、サンプルコードの代わりに置いておきます。(以下の `accept()` 章のコードはより完全なものです。) この全体の中で本当に厄介なのは、`accept()` の呼び出しです。


## 5.6 `accept()`---"Thank you for calling port 3490."

`accept()` の呼び出しはちょっと変です。これから起こることはこうです。遠く離れた誰かが、あなたが `listen()` しているポートであなたのマシンに `connect()` しようとするでしょう。その接続は、`accept()` されるのを待つためにキューに入れられることになります。あなたは `accept()` をコールし、保留中の接続を取得するように指示します。すると、この接続に使用する新しいソケットファイル記述子が返されます！そうです、1つの値段で2つのソケットファイル記述子を手に入れたことになります。元のソケットファイル記述子はまだ新しい接続を待ち続けており、新しく作成されたソケットファイル記述子はようやく `send()` と `recv()` を行う準備が整いました。着いたぞ！

コールは以下の通りです。

```c
#include <sys/types.h>
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

`sockfd` は `listen()` するソケットディスクリプタです。`addr` は通常、ローカルの`struct sockaddr_storage` へのポインタになります。この構造体には、着信接続に関する情報が格納されます(これにより、どのホストがどのポートから電話をかけてきたかを判断することができます)。`addrlen` はローカルの整数型変数で、そのアドレスが `accept()` に渡される前に `sizeof(struct sockaddr_storage)` に設定されなければなりません。`accept()` は、`addr` にそれ以上のバイト数を入れることはありません。もし、それ以下のバイト数であれば、`addrlen` の値を変更します。

何だと思いますか？`accept()` はエラーが発生した場合は `-1` を返し、`errno` をセットします。そうだったんですか。

前回と同様、一度に吸収するのは大変なので、サンプルコードの一部をご覧ください。

```{.c .numberLines}
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

ここでも、すべての `send()` と `recv()` の呼び出しに、ソケットディスクリプタ `new_fd` を使用することに注意してください。もし、一度しか接続がないのであれば、同じポートからの接続を防ぐために、`listen` している `sockfd` を `close()` することができます。


## 5.7 `send()` and `recv()`---Talk to me, baby! {#sendrecv}

この2つの関数は、ストリームソケットまたは接続されたデータグラムソケットで通信を行うためのものです。通常の非接続型データグラムソケットを使いたい場合は、以下の [`sendto()` and `recvfrom()`](docs/system-calls-or-bust/#sendtorecv) の節を参照する必要があります。

`send()` 呼び出し。

```c
int send(int sockfd, const void *msg, int len, int flags);
```

`sockfd` はデータを送信したいソケットディスクリプタ（`socket()` で返されたものでも `accept()` で取得したものでも可）、`msg` は送信したいデータへのポインタ、`len` はそのデータの長さ(バイト数)です。`flags` を `0` に設定するだけです(フラグに関する詳しい情報は `send()` の man ページを参照してください)。

サンプルコードとしては、以下のようなものがあります。

```{.c .numberLines}
char *msg = "Beej was here!";
int len, bytes_sent;
.
.
.
len = strlen(msg);
bytes_sent = send(sockfd, msg, len, 0);
.
.
.
```

`send()` は実際に送信されたバイト数を返しますが、これは送信するように指示した数よりも少ないかもしれません！つまり、大量のデータを送信するように指示しても、それが処理しきれないことがあるのです。その場合、できる限りのデータを送信し、残りは後で送信するように指示します。`send()` が返す値が `len` の値と一致しない場合、残りの文字列を送信するかどうかはあなた次第だということを覚えておいてください。良いニュースはこれです。パケットが小さければ（1K以下とか）、 おそらく全部を一度に送信することができるでしょう。ここでも、エラー時には `-1` が返され、 `errno` にはエラー番号がセットされます。

`recv()` 呼び出しは、多くの点で類似しています。

```c
int recv(int sockfd, void *buf, int len, int flags);
```

`sockfd` は読み込むソケットディスクリプタ、`buf` は情報を読み込むバッファ、`len` はバッファの最大長、`flags` は再び `0` に設定できます(フラグについては `recv()` の man ページを参照してください)。

`recv()` は、実際にバッファに読み込まれたバイト数を返し、エラーの場合は `-1` を返します（それに応じて `errno` が設定されます）。

待ってください！`recv()` は `0` を返すことがあります。これは、リモート側が接続を切断したことを意味します！`0` という返り値は、`recv()` がこのような事態が発生したことをあなたに知らせるためのものです。

ほら、簡単だったでしょう？これでストリームソケットでデータのやり取りができるようになったぞ。やったー！あなたは Unix ネットワークプログラマーです！


## 5.8 `sendto()` and `recvfrom()`---Talk to me, DGRAM-style {#sendtorecv}

"これはすべて素晴らしく、ダンディーだ"、"しかし、データグラムソケットを接続しないままにしておくのはどうなんだ？"、という声が聞こえてきそうです。大丈夫だ、アミーゴ。ちょうどいいものがありますよ。

データグラムソケットはリモートホストに接続されていないので、パケットを送信する前にどのような情報を与える必要があるか分かりますか？そうです！宛先アドレスです！これがそのスコープです。

```c
int sendto(int sockfd, const void *msg, int len, unsigned int flags,
           const struct sockaddr *to, socklen_t tolen);
```

見ての通り、この呼び出しは基本的に `send()` の呼び出しと同じで、他に2つの情報が追加されています。`to` は `struct sockaddr` へのポインタで（おそらく直前にキャストした別の `struct sockaddr_in` や `struct sockaddr_in6`、`struct sockaddr_storage` になるでしょう）、送信先の IP アドレスとポートが含まれています。`tolen` は `int` 型ですが、単純に `sizeof *to` または `sizeof(struct sockaddr_storage)` に設定することができます。

宛先アドレスの構造体を手に入れるには、`getaddrinfo()` や以下の `recvfrom()` から取得するか、手で記入することになると思います。

`send()` と同様、`sendto()` は実際に送信したバイト数 (これも、送信するように指示したバイト数よりも少ないかもしれません！) を返し、エラーの場合は `-1` を返します。

同様に、`recv()` と `recvfrom()` も類似しています。`recvfrom()` の概要は以下の通りです。

```c
int recvfrom(int sockfd, void *buf, int len, unsigned int flags,
             struct sockaddr *from, int *fromlen);
```

これも `recv()` と同様であるが、いくつかのフィールドが追加されています。`from` はローカルの `struct sockaddr_storage` へのポインタで、送信元のマシンの IP アドレスとポートが格納されます。`fromlen` はローカルの `int` へのポインタであり、`sizeof *from` または `sizeof(struct sockaddr_storage)` に初期化する必要があります。この関数が戻ったとき、`fromlen` は実際に `from` に格納されたアドレスの長さを含みます。

`recvfrom()` は受信したバイト数を返し、エラーの場合は `-1` を返します（`errno` はそれに応じて設定されます）。

そこで質問ですが、なぜソケットの型として `struct sockaddr_storage` を使うのでしょうか？なぜ、`struct sockaddr_in` ではないのでしょうか？なぜなら、私たちは IPv4 や IPv6 に縛られたくないからです。そこで、汎用的な構造体である `sockaddr_storage` を使用するのですが、これはどちらにも十分な大きさであることが分かっています。

（そこで...ここでまた疑問なのですが、なぜ `struct sockaddr` 自体はどんなアドレスに対しても十分な大きさがないのでしょうか？汎用 `struct sockaddr_storage` を汎用 `struct sockaddr` にキャストしているくらいなのに！？余計なことをしたような気がしますね。答えは、十分な大きさがなく、この時点で変更するのは問題がある、ということでしょう。だから新しいのを作ったんだ。）

データグラムソケットを `connect()` すれば、すべてのトランザクションに `send()` と `recv()` を使用できることを覚えておいてください。ソケット自体はデータグラムソケットであり、パケットは UDP を使用しますが、ソケットインターフェイスが自動的に宛先と送信元の情報を追加してくれるのです。


## 5.9 `close()` and `shutdown()`---Get outta my face!

ふぅー 一日中データの送受信（`send()`ing と `recv()`ing）をしていて、もう限界だ。ソケットディスクリプタの接続を閉じる準備ができました。これは簡単です。通常の Unix ファイルディスクリプタの `close()` 関数を使えばいいのです。

```c
close(sockfd);
```

これにより、それ以上のソケットへの読み書きができなくなります。リモート側でソケットの読み書きをしようとすると、エラーが発生します。

ソケットの閉じ方をもう少し制御したい場合は、`shutdown()` 関数を使用します。この関数では、特定の方向、あるいは両方の通信を遮断することができます (ちょうど `close()` がそうであるように)。概要:

```c
int shutdown(int sockfd, int how);
```

`sockfd` はシャットダウンしたいソケットファイル記述子、`how` は以下のいずれかです。

| `how` | Effect                                    |
|:-----:|-------------------------------------------|
|  `0`  | それ以上の受信は不可                      |
|  `1`  | それ以上の送信は禁止                      |
|  `2`  | それ以上の送受信は禁止(`close()`のように) |

`shutdown()` は成功すると `0` を、エラーが発生すると `-1` を返します（`errno` は適宜設定されます）。

データグラムソケットが接続されていない状態で `shutdown()` を使用すると、それ以降の `send()` および `recv()` 呼び出しに使用できなくなります（データグラムソケットを `connect()` した場合、これらを使用できることを忘れないでください）。

`shutdown()` は実際にはファイルディスクリプタを閉じないことに注意することが重要です。ソケットディスクリプタを解放するには、`close()` を使用する必要があります。

何もないんだけどね。

（ただし、Windows と Winsock を使用している場合は、`close()` ではなく `closesocket()` を呼び出すべきであることを忘れないでください。）


## 5.10 `getpeername()`---Who are you?

この関数はとても簡単です。

あまりに簡単なので、ほとんど独自のセクションを設けなかったほどです。でも、とりあえずここに書いておきます。

`getpeername()` 関数は、接続されたストリームソケットのもう一方の端にいるのが誰であるかを教えてくれます。その概要は

```c
#include <sys/socket.h>

int getpeername(int sockfd, struct sockaddr *addr, int *addrlen);
```

`sockfd` は接続したストリームソケットのディスクリプタ、`addr` は接続の相手側の情報を保持する `struct sockaddr`（または `struct sockaddr_in`）へのポインタ、`addrlen` は `int` へのポインタであり、 `sizeof *addr` または `sizeof(struct sockaddr)` で初期化される必要があります。

この関数は，エラーが発生すると `-1` を返し，それに応じて `errno` を設定します。

アドレスがわかれば、`inet_ntop()`、`getnameinfo()`、`gethostbyaddr()` を使って、より詳しい情報を表示したり取得したりすることができます。いいえ、ログイン名を取得することはできません。（OK、OK。相手のコンピュータで ident デーモンが動いていれば、可能です。しかし、これはこのドキュメントの範囲外です。詳しくは [RFC 1413](https://datatracker.ietf.org/doc/html/rfc1413) をチェックしてください。）


## 5.11 `gethostname()`---Who am I?

`getpeername()` よりもさらに簡単なのは、`gethostname()` という関数です。これは、あなたのプログラムが動作しているコンピュータの名前を返します。この名前は、後述の `gethostbyname()` でローカルマシンの IP アドレスを決定するために使用されます。

これ以上楽しいことはないでしょう？いくつか思いつきましたが、ソケットプログラミングには関係ないですね。とにかく、内訳はこんな感じです。

```c
#include <unistd.h>

int gethostname(char *hostname, size_t size);
```

引数は単純で、`hostname` はこの関数が戻ったときにホスト名を格納する文字列の配列へのポインタ、`size` はホスト名配列のバイト長です。

この関数は，正常に終了した場合は `0` を，エラーの場合は `-1` を返し，通常通り `errno` を設定します。
