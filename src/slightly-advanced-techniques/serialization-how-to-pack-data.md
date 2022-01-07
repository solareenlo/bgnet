# 7.5 シリアライゼーション---データの詰め方

ネットワーク上でテキストデータを送るのは簡単ですが、`int` や `float` のような"バイナリ"データを送りたい場合はどうしたらよいでしょうか？その結果、いくつかの選択肢があることがわかりました。

1. 数字を `sprintf()` などの関数でテキストに変換し、送信します。受信側は `strtol()` などの関数を使ってテキストを解析し、数値に戻します。

2. `send()` にデータへのポインタを渡して、データをそのまま送信します。

3. 数値を携帯可能な2進数にエンコードします。受信側はそれをデコードします。

スニークプレビュー！今夜だけ！

[_カーテン上昇_]

Beejは、"私は、上の方法3を好みます！"と言っています。

[_終_]

（この章を本格的に始める前に、これを行うためのライブラリは世の中に存在し、自分でローリングして移植性とエラーのない状態を維持することはかなり困難であることをお伝えしておきます。ですから、自分で実装することを決める前に、いろいろと調べて下調べをしてください。私は、このようなことがどのように機能するのかに興味がある人のために、ここに情報を記載します。）

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

```c
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

```c
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

```c
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

```c

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

```c
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

```c
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
