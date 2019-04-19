### MySQL源码解读——password

#### 源码所在位置

```
https://github.com/mysql/mysql-server/blob/5.7/sql/auth/password.c
```

#### 1 密码检查思路

主要的思想是：1 客户端与服务器端的网络连接上不能传递密码 2 在MySQL端保存的密码不能以任何形式反解码出来。第1点说明网络传输的密码要加密；第2点说明密码在MySQL上要通过单向加密算法加密后才能存储，从而无法反解析出来。

连接建立的时候，生成一个随机字符串并将其发送给客户端。客户端使用随机生成器，使用密码的hash值与传递过来的字符串生成一个新的字符串。该新字符串发送到服务器端，与存储的密码的hash值和随机生成的字符串进行比较。

密码在服务器端通过函数PASSWORD()保存到user.password字段中。

该文件之所以是.c文件，是因为其被libmysqlclient使用，而libmysqlclient是一个c程序。（我们需要使其能够移植到各种操作系统上）

例如：

```
update user set password=PASSWORD("hello") where user="test"
```

上述SQL语句会将密码以hash后的字符串形式保存到user.password字段中。

新的身份认证需要按照一下方式执行：

```
SERVER: public_seed=generate_user_salt()
        send(public_seed)
CLIENT: recv(public_seed)
        hash_stage1=sha1("password")
        hash_stage2=sha1(hash_stage1)
        reply=xor(hash_stage1, sha1(public_seed, hash_stage2))
        //this three steps are done in scramble()
        
SERVER: recv(reply)
        hash_stage1=xor(reply, sha1(public_seed, hash_stage2))
        candidate_hash2=sha1(hash_stage1)
        check(candidate_hash2==hash_stage2)
        //this three steps are done in check_scramble()
```

#### 2 函数解析

- void randominit()

该函数是初始化rand_struct结构体的，初始化的时候，保证不超过seed的最大值（rand_st->max_value_dbl）


- void hash_password()

将原始密码生成二机制的hash值，在Pre-4.1之前使用。hash函数会对字符串类型的密码每一个字节做运算，最终得到一个ulong类型的hash值。

- static inline uint8 char_val()

这个函数主要是将字符（a-z/A-z/0-9）转换成数字，该函数输入输出参数可以换成：

```
uint char_val(char x) {
    return (uint)(x>='0'&&x<='9' ? x - '0':
                  x>='A'&&x<='Z'?x - 'A' + 10:x - 'a' + 10);
}
```

这个函数需要注意，传入的必须都是字符（a-z/A-z/0-9）组合，不然后会出现意想不到问题。

- char *octet2hex()

该函数将八进制的序列转化成十六进制的ascii字符串。

- static void hex2octet()

该函数将十六进制（0-9/a-f）的ascii字符串转换成8进制序列。

- static void my_crypt()

异或算法加密解密函数。注意以下公式成立：

```
XOR(s1, XOR(s1, s2)) == s2
XOR(s1, s2) == XOR(s2, s1)
```

- void compute_two_stage_sha1_hash()

计算两阶段SHA1后的密码：

```
hash_stage1=sha1("password")
hash_stage2=sha1(hash_stage1)
```

- void my_make_scrambled_password_sha1()

MySQL4.1.1使用的密码hash方法。SHA转换两次应用于密码，然后生成8位序列转换为十六进制字符串。

该函数的返回结果用作函数PASSWORD()的返回值和存储在数据库中。

- void make_scrambled_password()

该函数是对`my_make_scrambled_password_sha1 `函数的封装，用来维护客户端ABI兼容性。

在server端代码中最好使用`my_make_scrambled_password_sha1`来避免`strlen()`。

- void scramble()

通过密码和随机字符串生成一个八位字符串，该字符串将发往服务器。该序列对应于密码，但密码不容易从它恢复。序列会发往服务器进行权限认证。其不需要尾部的零字符表示结束。

客户端使用该函数在服务器握手后创建权限认证的响应。

- my_bool check_scramble_sha1()

检查客户端传过来的认证信息是否符合密码；这个函数用于检查接收到的回复是否真实。该函数不检查给定的字符串的长度：消息必须以NULL结尾的回复，并且hash_stage2必须至少为SHA1_HASH_SIZE这么长。

- my_bool check_scramble()

该函数是对`check_scramble_sha1`的封装。


- void get_salt_from_password()

将打乱的密码从ascii十六进制字符串转化为二进制形式。

- void make_password_from_salt()

将二进制形式的密码转换成ascii十六进制字符串。
