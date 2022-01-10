# 6.3 データグラムソケット

UDP データグラムソケットの基本は、上記の [5.8 sendto() and recvfrom()](../system-calls-or-bust/sendto-and-recvfrom-talk-to-me-GDRAM-style.md) ですでに説明しましたので、ここでは `talker.c` と `listener.c` という2つのサンプルプログラムのみを紹介します。

`listener` は、ポート 4950 で入ってくるパケットを待つマシンに座っています。`talker` は、指定されたマシンのそのポートに、ユーザがコマンドラインに入力したものを含むパケットを送信します。

データグラムソケットはコネクションレス型であり、パケットを無慈悲に発射するだけなので、クライアントとサーバには IPv6 を使用するように指示することにしています。こうすることで、サーバが IPv6 でリッスンしていて、クライアントが IPv4 で送信するような状況を避けることができます。（接続された TCP ストリームソケットの世界では、まだ不一致があるかもしれませんが、一方のアドレスファミリーの `connect()` でエラーが発生すると、他方のアドレスファミリーの再試行が行われます。）

[listener.c ソースコード](https://beej.us/guide/bgnet/examples/listener.c)

```c
/*
** listener.c -- a datagram sockets "server" demo
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

#define MYPORT "4950"	// the port users will be connecting to

#define MAXBUFLEN 100

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
	int sockfd;
	struct addrinfo hints, *servinfo, *p;
	int rv;
	int numbytes;
	struct sockaddr_storage their_addr;
	char buf[MAXBUFLEN];
	socklen_t addr_len;
	char s[INET6_ADDRSTRLEN];

	memset(&hints, 0, sizeof hints);
	hints.ai_family = AF_INET6; // set to AF_INET to use IPv4
	hints.ai_socktype = SOCK_DGRAM;
	hints.ai_flags = AI_PASSIVE; // use my IP

	if ((rv = getaddrinfo(NULL, MYPORT, &hints, &servinfo)) != 0) {
		fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv));
		return 1;
	}

	// loop through all the results and bind to the first we can
	for(p = servinfo; p != NULL; p = p->ai_next) {
		if ((sockfd = socket(p->ai_family, p->ai_socktype,
				p->ai_protocol)) == -1) {
			perror("listener: socket");
			continue;
		}

		if (bind(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
			close(sockfd);
			perror("listener: bind");
			continue;
		}

		break;
	}

	if (p == NULL) {
		fprintf(stderr, "listener: failed to bind socket\n");
		return 2;
	}

	freeaddrinfo(servinfo);

	printf("listener: waiting to recvfrom...\n");

	addr_len = sizeof their_addr;
	if ((numbytes = recvfrom(sockfd, buf, MAXBUFLEN-1 , 0,
		(struct sockaddr *)&their_addr, &addr_len)) == -1) {
		perror("recvfrom");
		exit(1);
	}

	printf("listener: got packet from %s\n",
		inet_ntop(their_addr.ss_family,
			get_in_addr((struct sockaddr *)&their_addr),
			s, sizeof s));
	printf("listener: packet is %d bytes long\n", numbytes);
	buf[numbytes] = '\0';
	printf("listener: packet contains \"%s\"\n", buf);

	close(sockfd);

	return 0;
}
```

`getaddrinfo()` の呼び出しで、最終的に `SOCK_DGRAM` を使用していることに注意してください。また、`listen()` や `accept()` は必要ないことに注意してください。これは非接続型データグラムソケットを使用する利点の1つです！

[talker.c ソースコード](https://beej.us/guide/bgnet/examples/talker.c)

```c
/*
** talker.c -- a datagram "client" demo
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

#define SERVERPORT "4950"	// the port users will be connecting to

int main(int argc, char *argv[])
{
	int sockfd;
	struct addrinfo hints, *servinfo, *p;
	int rv;
	int numbytes;

	if (argc != 3) {
		fprintf(stderr,"usage: talker hostname message\n");
		exit(1);
	}

	memset(&hints, 0, sizeof hints);
	hints.ai_family = AF_INET6; // set to AF_INET to use IPv4
	hints.ai_socktype = SOCK_DGRAM;

	if ((rv = getaddrinfo(argv[1], SERVERPORT, &hints, &servinfo)) != 0) {
		fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rv));
		return 1;
	}

	// loop through all the results and make a socket
	for(p = servinfo; p != NULL; p = p->ai_next) {
		if ((sockfd = socket(p->ai_family, p->ai_socktype,
				p->ai_protocol)) == -1) {
			perror("talker: socket");
			continue;
		}

		break;
	}

	if (p == NULL) {
		fprintf(stderr, "talker: failed to create socket\n");
		return 2;
	}

	if ((numbytes = sendto(sockfd, argv[2], strlen(argv[2]), 0,
			 p->ai_addr, p->ai_addrlen)) == -1) {
		perror("talker: sendto");
		exit(1);
	}

	freeaddrinfo(servinfo);

	printf("talker: sent %d bytes to %s\n", numbytes, argv[1]);
	close(sockfd);

	return 0;
}
```

と、これだけです！`listener` をあるマシンで実行し、次に `takler` を別のマシンで実行します。それらのコミュニケーションをご覧ください！核家族で楽しめる G 級興奮体験です！

今回はサーバを動かす必要もありません。`talker` はただ楽しくパケットをエーテルに発射し、相手側に `recvfrom()` の準備が出来ていなければ消えてしまうのです。UDP データグラムソケットを使用して送信されたデータは、到着が保証されていないことを思い出してください！

過去に何度も述べた、もうひとつの小さなディテールを除いては、コネクテッド・データグラム・ソケットです。このドキュメントのデータグラムの章にいるので、ここでこれについて話す必要があります。例えば、`talker` が `connect()` を呼び出して `listener` のアドレスを指定したとします。それ以降、`talker` は `connect()` で指定されたアドレスにのみ送信と受信ができます。このため、`sendto()` と `recvfrom()` を使う必要はなく、単に `send()` と `recv()` を使えばいいのです。
