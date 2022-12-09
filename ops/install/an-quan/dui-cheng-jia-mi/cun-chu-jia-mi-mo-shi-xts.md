# 存储加密xts

磁盘加密通常使用特殊目的、专门设计的模式。可以调节的小数据块加密模式（LRW，XEX和XTS）和大数据块的模式（CMC和EME）是设计用于加密磁盘区块的。



在对磁盘的加密中，通常一个扇区大小为512 Byte。加密时将要写入扇区的明文数据进行加密，然后存储到扇区上。解密时，希望能直接读取到一个扇区上的信息进行解密



我的理解是，存储加密的基本要求是：\(1\) 磁盘加密希望密文不能有扩张，即不能有IV信息等额外的存储信息。\(2\) 便于随机访问。



详情参见http://en.wikipedia.org/wiki/Disk\_encryption\_theory



XTS属于可调密码。可调密码和通常的密码相比，输入多了一个可以公开的tweak value。在具体操作中，tweak value与明文tweak后（比如做异或）将所得结果送入加密模块，加密后得到的密文再次与tweak value做一次 tweak后才得到输出密文（也可能没有加密后的tweak）。为什么需要此tweak value呢？主要原因在于：（1） tweak可以增加密文的多变性，不改变密钥只改变此值就可以改变密文；（2） 改变密钥的代价比改变tweak value大的多；（3） tweak value是公开的，不担心泄露。



XTS能满足磁盘加密存储的要求。在XTS中，tweak value通常是data unit所在位置，所以就不需要额外的空间存储tweak value。data unit内部类似CTR模式，data unit之间则是相互独立的，因此便于随机访问。



在XTS工作模式中对数据做了以下划分：各术语由大到小解释如下：



KeyScope：密钥的有效范围，在一个KeyScope里面包含了N多个DataUnit。同一个KeyScope内，所有DataUnit使用相同的密钥，且每个DataUnit的长度都是一样的（不同的KeyScope之间的DataUnit的长度可以不一样），每个DataUnit拥有各自的tweak value\(比如使用其所在地址作为tweak value\)。比如有如下KeyScope的例：&lt;KeyScope&gt;: KeyScopeStart =0, DataUnitSize = 4096, KeyScopeLength = 1083。这表示KeyScope从位置0开始，其中共有1083个DataUnit，每个DataUnit的大小都是4096字节，即这个KeyScope的长度为4096\*1083 =4435968字节

DataUnit：由N个128-bit Block组成。

128-bit Block：是AES的分组大小。每个128-bit Block都有自己的这个DataUnit内部的编号（从0开始）。每个128-bit Block在进行AES加密前后都会与TweakValue的衍生值做一次XOR。

2. 有限域GF\(2^128\)

标准中使用的有限域为GF\(2^128\)，本原元alpha，生成多項式 f\(x\) = x^128 + x^7 + x^2 + x^1 + 1, 数 135就是 x^7 + x^2 + x^1 + 1.。有限域上的乘法参见有限域介绍。此标准中任意元与alpha的乘法就是一个线性反馈移位寄存器。



B \* alpha = \( B &lt;&lt; 1 \)  XOR   \( B\_127 &gt; 0 \) ? 135：0



3. 128-bit Block加密

C ← XTS-AES-blockEnc\(Key, P, i, j\)



Key 是256或512 bit密钥，长度是普通AES密钥长度的2倍，因为key会被等分为两个AES密钥 key = key1\|\|key2

P 128bit明文

i 128-bit tweak value

j 当前128-bit block在DataUnit内的序号

C 128bit密文

步骤如下：



1. T ← AES-enc\(Key2 , i\) × alphaj（注：× alphaj可以看作是一个线性反馈移位寄存器）



2. PP ← P  XOR  T



3. CC ← AES-enc\(Key1 , PP\)



4. C ← CC  XOR  T



XTS-AES blockEnc



4. DataUnit加密

C ← XTS-AES-Enc \(Key, P, i\)



Key 是256或512 bit密钥，长度是普通AES密钥长度的2倍，因为key会被等分为两个AES密钥 key = key1\|\|key2

P 明文

i 128-bit tweak value

C 密文

步骤如下：



将明文按128bit划分 P = P0 \|… \|Pm−1\|Pm，最后一个块为0——127bit



1.   for \( q = 0; q &lt;= m-2; q++ \)//前面的块直接做块加密



{



1.1        Cq ← XTS-AES-blockEnc\(Key, Pj, i, q\);



}



2     b ← bit-size of Pm;//最后两个块的加密要特殊点



3     if \( b == 0 \)



{



3.1        Cm-1 ← XTS-AES-blockEnc\(Key, Pm-1, i, m-1\)



3.2        Cm ← empty



}



4     else



{



4.1        CC ← XTS-AES-blockEnc\(Key, Pm-1, i, m-1\);



4.2        Cm ← first b bits of CC;



4.3        CP ← last \(128-b\) bits of CC;



4.4        PP ← Pm \| CP;



4.5        Cm-1 ← XTS-AES-blockEnc\(Key, PP, i, m\);



}



2.    C ← C0\|… \|Cm-1\|Cm;



最后一个分组不完整时的处理方案



5. KeyScope的加密

使用同一个密钥，对各个DataUnit分别进行加密即可。各DataUnit间无相关性。



6. 解密

解密与加密类似。在XTS中需要AES解密模块，用在128-bit Block的解密中。



略

————————————————

版权声明：本文为CSDN博主「艾米的爸爸」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/samsho2/article/details/84561212

