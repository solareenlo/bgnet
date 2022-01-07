# 5.3 `bind()`---私はどのポートにいるのでしょうか？

ソケットを取得したら、そのソケットをローカルマシンのポートに関連付ける必要があるかもしれません。（これは、特定のポートへの接続を `listen()` する場合によく行われます。多人数参加型ネットワークゲームで "192.168.5.10 ポート 3490 に接続" と指示されたときに行います。）ポート番号はカーネルが受信パケットを特定のプロセスのソケットディスクリプタにマッチさせるために使用されます。もしあなたが `connect()` を行うだけなら（あなたはクライアントであり、サーバではないので）、これはおそらく不要でしょう。とにかく読んでみてください。

`bind()` システムコールの概要は以下のとおりです。

```c
#include <sys/types.h>
#include <sys/socket.h>

int bind(int sockfd, struct sockaddr *my_addr, int addrlen);
```

`sockfd` は `socket()` が返すソケットファイル記述子です。`my_addr` は自分のアドレスに関する情報、すなわちポートおよび IP アドレスを含む `sockaddr` 構造体へのポインタです。

ふぅー。一度に吸収するのはちょっと無理があるな。ソケットをプログラムが実行されているホスト、ポート 3490 にバインドする例を見てみましょう。

```c
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

```c
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

`bind()` を呼ぶときにもうひとつ気をつけなければならないのは、ポート番号で下手を打たないことです。1024 以下のポートはすべて予約済みです（あなたがスーパーユーザでない限り）！それ以上のポート番号は、（他のプログラムによってすでに使われていなければ） 65535 までの任意のポート番号を使用することができます。

時々、サーバを再実行しようとすると、`bind()` が "Address already in use" と言って失敗することに気がつくかもしれません。これはどういうことでしょう？それは、接続されたソケットの一部がまだカーネル内に残っていて、ポートを占有しているのです。それが消えるのを待つか（1分くらい）、次のようにポートが再利用できるようなコードをプログラムに追加します。

```c
int yes=1;
//char yes='1'; // Solaris people use this

// lose the pesky "Address already in use" error message
if (setsockopt(listener,SOL_SOCKET,SO_REUSEADDR,&yes,sizeof yes) == -1) {
    perror("setsockopt");
    exit(1);
}
```

`bind()` について、最後にちょっとした注意点があります。`bind()` を絶対に呼び出す必要がない場合があります。リモートマシンに `connect()` する際に、ローカルポートを気にしない場合（telnet のようにリモートポートを気にする場合）は、単に `connect()` をコールすれば、ソケットが未束縛かどうかをチェックし、必要なら未使用のローカルポートに `bind()` してくれます。
