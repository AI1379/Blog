---
title: MD5算法解析
date: 2020-02-20 21:14:00
tags: 加密
mathjax: true
categories:
 - Cryptography
 - Hash
---

MD5即Message-Digest Algorithm 5, 信息学中使用广泛的哈希算法

<!--more-->

# MD5简介

**MD5**即Message-Digest Algorithm 5, 信息学中使用广泛的哈希算法

这个算法具有很多性质:

1. 压缩性: 对于任意长度的输入, 输出长度总是相同的
2. 容易计算: **线性时间复杂度**
3. 抗修改性: 对原数据的一点点修改都会导致最终结果的巨大变化
4. 抗碰撞性: 已知原数据和MD5值很难生成与原数据不同但MD5值相同的数据

MD5可以生成任意一个文件的“数字指纹”，对文件的微小改动都会直接导致数字指纹的巨大变化。

> 注：MD5加密中文需要使用UTF-8编码，但Windows下默认是GBK编码，两种编码得到的结果是不一样的

# 加密步骤

## 填充

MD5中，首先要对信息进行填充，先填充一个1，后面都填充0，使得信息的长度 $len \equiv 448 \pmod{512}$ ，即 $len\bmod 512 = 448$。为什么要求模出来是448呢？因为448=512-64，而填充完后后面还要再填上64位的原数据长度，如果超出64位则填充原数据长度的后64位，这样可以使得最终的数据长度为512的整数倍，才可以满足后面继续加密的需要。

## 初始化变量

初始化4个128位链接变量如下:

``` text
A = 0x67452301
B = 0xEFCDAB89
C = 0x98BADCFE
D = 0x10325476
```

## 处理数据

### 分组

将原始数据每512bits为一个分组 (这就是前面要求填充到512整数倍的原因)，对每组分别进行处理

### 处理每个分组的数据

每个分组有四个变量$a,b,c,d$，第一分组的$a,b,c,d$分别为上面所写了的4个16进制数，此后每一个分组的这四个变量都是上一个分组计算的结果。

现在定义四个非线性函数 (逻辑运算) :
$$
\begin{align}
&F(x,y,z)=(x\land y)\lor((\neg x)\land z)\\
&G(x,y,z)=(x\land y)\lor(y\land(\neg z))\\
&H(x,y,z)=x\oplus y\oplus z\\
&I(x,y,z)=y\oplus (x\lor (\neg z))
\end{align}
$$
其中$\land$表示按位与，$\lor$表示按位或，$\neg$表示按位非，$\oplus$表示按位异或

接下来，我们设$M_j$为这一组信息的第$j$个子分组 ($0\le j\le 15$) ，$t_j$为$2^{32}\cdot |\sin{i}|\ \ (1\le i\le 64)$的整数部分，$i$是整数，单位是弧度($2^{32}=4294967296$)

现在定义四个操作:
$$
\begin{align}
&FF(a,b,c,d,M_j,s,t_i):\quad a=b+((a+F(b,c,d)+M_j+t_i)<<s) \\
&GG(a,b,c,d,M_j,s,t_i):\quad a=b+((a+G(b,c,d)+M_j+t_i)<<s) \\
&HH(a,b,c,d,M_j,s,t_i):\quad a=b+((a+H(b,c,d)+M_j+t_i)<<s) \\
&II(a,b,c,d,M_j,s,t_i):\quad a=b+((a+I(b,c,d)+M_j+t_i)<<s) \\
\end{align}
$$
**注意：<<在这里是左环移而不是一般的左移！！！**

> 左环移和左移的区别：
>
> 现有一个二进制数11010010，左环移三位后是10010110，而左移三位后是10010000

接下来就可以操作了，共4轮循环，每轮16次，伪代码如下：
$$
\begin{align}
&for\ i\ \leftarrow\ 0\ to\ 4:\\
&\qquad for\ j\ \leftarrow\ 0\ to\ 16:\\
&\qquad\qquad func_i(a_{(j+3)\bmod 4},a_{(j+2)\bmod 4},a_{(j+1)\bmod 4},a_{j\bmod 4},M_j,s_{i,j\bmod 4},t_{16i+j})
\end{align}
$$
其中:
$$
func_0:\quad FF\\func_1:\quad GG\\func_2:\quad HH\\func_3:\quad II\ \ \ \ 
$$
$s_{i,j\bmod 4}$为常数数组，具体数值见下面代码里的s数组，$M_j$和$t_{16i-j}$上文已经讲过了

最后把$a,b,c,d$分别加上$A,B,C,D$

然后进行下一组运算

## 输出

输出数据为$a,b,c,d$的级联

# 代码(C++)

编译指令：`g++ MD5.cpp -o md5.exe -Wall -Wextra`

MD5.hpp:

```cpp
#ifndef CSTRING
#include <cstring>
#endif // !CSTRING
#include <string>

#ifndef MD5_HPP
#define MD5_HPP

//----------------------------------------宏定义----------------------------------------
#define F_MD5(x, y, z) ((x & y) | (~x & z))
#define G_MD5(x, y, z) ((x & z) | (y & ~z))
#define H_MD5(x, y, z) (x ^ y ^ z)
#define I_MD5(x, y, z) (y ^ (x | ~z))
#define ROTATE_LEFT_MD5(x, n) ((x << n) | (x >> (32 - n)))

#define FF_MD5(a, b, c, d, m, s, t) (a = b + (ROTATE_LEFT_MD5((a + F_MD5(b, c, d) + m + t), s)))
#define GG_MD5(a, b, c, d, m, s, t) (a = b + (ROTATE_LEFT_MD5((a + G_MD5(b, c, d) + m + t), s)))
#define HH_MD5(a, b, c, d, m, s, t) (a = b + (ROTATE_LEFT_MD5((a + H_MD5(b, c, d) + m + t), s)))
#define II_MD5(a, b, c, d, m, s, t) (a = b + (ROTATE_LEFT_MD5((a + I_MD5(b, c, d) + m + t), s)))


//----------------------------------------MD5结构体定义----------------------------------------
struct MD5_CTX
{
    unsigned int count[2];
    unsigned int state[4]; //加密结果
    unsigned char buffer[64];
};

//----------------------------------------常量定义----------------------------------------
const unsigned int t[64] = {
    0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee, 0xf57c0faf, 0x4787c62a, 0xa8304613, 0xfd469501,
    0x698098d8, 0x8b44f7af, 0xffff5bb1, 0x895cd7be, 0x6b901122, 0xfd987193, 0xa679438e, 0x49b40821,
    0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa, 0xd62f105d, 0x2441453, 0xd8a1e681, 0xe7d3fbc8,
    0x21e1cde6, 0xc33707d6, 0xf4d50d87, 0x455a14ed, 0xa9e3e905, 0xfcefa3f8, 0x676f02d9, 0x8d2a4c8a,
    0xfffa3942, 0x8771f681, 0x6d9d6122, 0xfde5380c, 0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70,
    0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x4881d05, 0xd9d4d039, 0xe6db99e5, 0x1fa27cf8, 0xc4ac5665,
    0xf4292244, 0x432aff97, 0xab9423a7, 0xfc93a039, 0x655b59c3, 0x8f0ccc92, 0xffeff47d, 0x85845dd1,
    0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1, 0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391};

const unsigned int s[4][4]{
    {7, 12, 17, 22},
    {5, 9, 14, 20},
    {4, 11, 16, 23},
    {6, 10, 15, 21}};

unsigned char PADDING[] = {
    0x80, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};

//----------------------------------------函数预定义----------------------------------------
void MD5Init(MD5_CTX *context);
void MD5Update(MD5_CTX *context, unsigned char *input, unsigned int inputlen);
void MD5Final(MD5_CTX *context, unsigned char digest[16]);
void MD5Transform(unsigned int state[4], unsigned char block[64]);
void MD5Encode(unsigned char *output, unsigned int *input, unsigned int len);
void MD5Decode(unsigned int *output, unsigned char *input, unsigned int len);

//----------------------------------------主函数----------------------------------------
void MD5Init(MD5_CTX *context)
{
    context->count[0] = 0;
    context->count[1] = 0;
    context->state[0] = 0x67452301;
    context->state[1] = 0xEFCDAB89;
    context->state[2] = 0x98BADCFE;
    context->state[3] = 0x10325476;
}

void MD5Update(MD5_CTX *context, unsigned char *input, unsigned int inputlen)
{
    //context: 已初始化的结构体
    //input: 需加密的信息
    //inputlen: 需加密的信息的长度
    unsigned int i = 0, index = 0, partlen = 0;
    index = (context->count[0] >> 3) & 0x3F;
    partlen = 64 - index;
    context->count[0] += inputlen << 3;
    if (context->count[0] < (inputlen << 3))
        context->count[1]++;
    context->count[1] += inputlen >> 29;

    if (inputlen >= partlen)
    {
        memcpy(&context->buffer[index], input, partlen);
        MD5Transform(context->state, context->buffer);
        for (i = partlen; i + 64 <= inputlen; i += 64)
            MD5Transform(context->state, &input[i]);
        index = 0;
    }
    else
    {
        i = 0;
    }
    memcpy(&context->buffer[index], &input[i], inputlen - i);
}

void MD5Final(MD5_CTX *context, unsigned char digest[16])
{
    //context: 加密好的 MD5结构体
    //digest: 最终保存位置
    unsigned int index = 0, padlen = 0;
    unsigned char bits[8];
    index = (context->count[0] >> 3) & 0x3F;
    padlen = (index < 56) ? (56 - index) : (120 - index);
    MD5Encode(bits, context->count, 8);
    MD5Update(context, PADDING, padlen);
    MD5Update(context, bits, 8);
    MD5Encode(digest, context->state, 16);
}

void MD5Encode(unsigned char *output, unsigned int *input, unsigned int len)
{
    unsigned int i = 0, j = 0;
    while (j < len)
    {
        output[j] = input[i] & 0xFF;
        output[j + 1] = (input[i] >> 8) & 0xFF;
        output[j + 2] = (input[i] >> 16) & 0xFF;
        output[j + 3] = (input[i] >> 24) & 0xFF;
        i++;
        j += 4;
    }
}

void MD5Decode(unsigned int *output, unsigned char *input, unsigned int len)
{
    unsigned int i = 0, j = 0;
    while (j < len)
    {
        output[i] = (input[j]) |
                    (input[j + 1] << 8) |
                    (input[j + 2] << 16) |
                    (input[j + 3] << 24);
        i++;
        j += 4;
    }
}

void MD5Transform(unsigned int state[4], unsigned char block[64])
{
    //state: MD5结构体里的 state数组
    //block: 要加密的 512bits数据块
    unsigned int a[4];
    //注意一下，这里 a[0],a[1],a[2],a[3]分别代表 d,c,b,a, 顺序是反的, 方便后面操作
    a[0] = state[3];
    a[1] = state[2];
    a[2] = state[1];
    a[3] = state[0];
    unsigned int x[64];
    int i, j, k;
    int kinit[4] = {0, 1, 5, 0}; //k的初始值

    MD5Decode(x, block, 64);
    for (i = 0; i < 4; i++)
    {
        k = kinit[i];
        for (j = 0; j < 16; j++)
        {
            switch (i)
            {
            case 0:
                FF_MD5(a[(j + 3) % 4], a[(j + 2) % 4], a[(j + 1) % 4], a[j % 4],
                       x[k], s[i][j % 4], t[i * 16 + j]);
                k = (k + 1) % 16;
                break;
            case 1:
                GG_MD5(a[(j + 3) % 4], a[(j + 2) % 4], a[(j + 1) % 4], a[j % 4],
                       x[k], s[i][j % 4], t[i * 16 + j]);
                k = (k + 5) % 16;
                break;
            case 2:
                HH_MD5(a[(j + 3) % 4], a[(j + 2) % 4], a[(j + 1) % 4], a[j % 4],
                       x[k], s[i][j % 4], t[i * 16 + j]);
                k = (k + 3) % 16;
                break;
            case 3:
                II_MD5(a[(j + 3) % 4], a[(j + 2) % 4], a[(j + 1) % 4], a[j % 4],
                       x[k], s[i][j % 4], t[i * 16 + j]);
                k = (k + 7) % 16;
                break;
            }
        }
    }
    state[0] += a[3];
    state[1] += a[2];
    state[2] += a[1];
    state[3] += a[0];
}
void MD5(unsigned char *input, unsigned char digest[16]) //最终的加密函数
{
    MD5_CTX MD5;
    MD5Init(&MD5);
    MD5Update(&MD5, input, strlen((char *)input));
    MD5Final(&MD5, digest);
    return;
}
std::string MD5string(std::string input) //对输入的 std::string进行 MD5加密并返回 std::string
{
    unsigned char digest[16];
    char tmp[2];
    std::string res;
    int i;
    MD5((unsigned char *)(input.c_str()), digest);
    for (i = 0; i < 16; i++)
    {
        sprintf(tmp, "%02x", digest[i]);
        res.push_back(tmp[0]);
        res.push_back(tmp[1]);
    }
    return res;
}
#endif // !MD5_HPP
```

MD5.cpp:

```cpp
#include <bits/stdc++.h>
#include "MD5.hpp"
using namespace std;
int main()
{
    string s, res;
    unsigned char decryption[16];
    cin >> s;
    MD5((unsigned char *)(s.c_str()), decryption);
    res = MD5string(s);
    for (int i = 0; i < 16; i++)
    {
        printf("%02x", decryption[i]);
    }
    printf("\n");
    cout << res << endl;
    system("pause");
    return 0;
}
```

生成t数组的Python代码(C++的cmath库是真的弱)

```python
import math

def main():
    for i in range(1, 65):
        print("%#x" % int(math.floor((2**32)*math.fabs(math.sin(i)))), end='')
        print(',', end='')
        if i % 8 == 0:
            print('\n', end='')


if __name__ == "__main__":
    main()

```





> 参考资料：
>
> https://zhuanlan.zhihu.com/p/37257569
>
> https://www.cnblogs.com/foxclever/p/7668369.html