## 0. 我的环境与工作目录
Ubuntu 20.04 LTS aarch64

如果你的环境是`x86_64`的，请把以下所有的`aarch64`换成`x86_64`

工作目录在
```
git clone https://github.com/sylar-yin/sylar.git
cd sylar
```

## 1. 退出conda base环境
```
conda deactivate
```
理由：有的终端会自动进入base环境，环境变量改变，会出问题

## 2. 配置环境

### 2.1 安装不在乎版本的包 
```
sudo apt update
sudo apt install protobuf-compiler libprotobuf-dev libboost-all-dev libmysqlclient-dev ragel libjsoncpp-dev libyaml-cpp-dev libtinyxml2-dev libjemalloc-dev sqlite3 libsqlite3-dev
```
注意！！可能不止这些包！

### 2.2 安装`zookeeper-3.7.0`

从这个网站上下载
```
wget https://archive.apache.org/dist/zookeeper/zookeeper-3.7.0/apache-zookeeper-3.7.0.tar.gz
```
我是直接从网页下载的。
```
tar -xvzf apache-zookeeper-3.7.0.tar.gz
cd apache-zookeeper-3.7.0
```
安装编译C API的必要依赖
```
sudo apt update
sudo apt install build-essential autoconf automake libtool libcppunit-dev maven
```
生成`zookeeper-jute.h`文件
```
cd ./zookeeper-jute
mvn compile
cd ../zookeeper-client/zookeeper-client-c
./configure
make
sudo make install
```
参考：
https://github.com/sylar-yin/sylar/issues/35#issuecomment-2434522195
https://stackoverflow.com/questions/64916994/build-xml-does-not-exist-when-install-zookeeper-client-c
https://groups.google.com/g/linux.debian.bugs.dist/c/tN4oYiffamo
https://github.com/apache/zookeeper/blob/master/zookeeper-docs/src/main/resources/markdown/zookeeperProgrammers.md#Installation

一定不要参考README！！那个过时了！！
里面犯了太多错误，比如根本没有build.xml，并且cppunit.m4文件也被废弃了

### 2.3 安装`hiredis-vip`
```
git clone https://github.com/vipshop/hiredis-vip.git
cd hiredis-vip
make
sudo make install
```

## 3. `./CMakeLists.txt` 添加、删除、修改
### 3.1 添加
#### 3.1.1
```
include_directories(/usr/local/include)
```
理由：这个目录是自行安装的软件，我怕cmake找不到上面我们自行安装的zookeeper和hiredis-vip
#### 3.1.2
```
include_directories(/usr/include/jsoncpp) 
```
不推荐：也可以把源代码里面的./sylar/util.h 里面<json/json.h>改为<jsoncpp/json/json.h>，但不推荐，因为我不确定是不是只有./sylar/util.h 里面写了<json/json.h>

理由：比较显然的原因：json.h在/usr/include/jsoncpp/json/下面，而不是/usr/include/json/下面

### 3.2 修改
所有`mysqlclient_r`改为`mysqlclient`

也可以通过软链接（注意！路径不一定是下面这个，可能要把aarch64改成x86_64）
```
ln -s /usr/lib/aarch64-linux-gnu/libmysqlclient.so /usr/lib/aarch64-linux-gnu/libmysqlclient_r.so
```	
两种方法一样的推荐

理由：mysqlclient_r似乎被淘汰了，需要的东西在新版里面被整合到mysqlclient里面，所以要么别链接带有_r的.so文件，要么就整个软链接骗一下链接器

### 3.3 删除
所有的`-Werror`

理由：这个东西会让编译时所有的warning变成error，很烦。具体说来，warning的东西就是代码会忽略一些函数的返回值，总不可能改代码吧。能用就行。

## 4.源代码修改
### 4.1 `./sylar/db/mysql.cc`
把所有的`my_bool`改为`bool`即可

理由：新版本的mysql弃用了`my_bool`，直接改为`bool`在网上似乎是推荐做法

### 4.2 `./sylar/util/hash_util.cc`
把所有的`SHA_Init()` `SHA_Update()` `SHA_Final()` 改成`SHA1_Init()` `SHA1_Update()` `SHA1_Final()`

###	4.3 `./sylar/util/crypto_util.cc`
65到82行改为：
```
EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
    do {
        //static CryptoInit s_crypto_init;
        EVP_CIPHER_CTX_init(ctx);
        EVP_CipherInit_ex(ctx, cipher, nullptr, (const uint8_t*)key
                ,(const uint8_t*)iv, enc);
        if(EVP_CipherUpdate(ctx, (uint8_t*)out, &tmp_len, (const uint8_t*)in, in_len) != 1) {
            has_error = true;
            break;
        }
        *out_len = tmp_len;
        if(EVP_CipherFinal_ex(ctx, (uint8_t*)out + tmp_len, &tmp_len) != 1) {
            has_error = true;
            break;
        }
        *out_len += tmp_len;
    } while(0);
    EVP_CIPHER_CTX_free(ctx);
```
理由：新的OpenSSL把这个上下文类的构造方式和消除方式变了一下。原来直接构造`EVP_CIPHER_CTX ctx;`，现在使用了指针`EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();`，同样的，最后释放内存的方式也不太一样，要`EVP_CIPHER_CTX_free(ctx);`。

### 4.4 `./tests/test_crypto.cc`
98-112行改为：
```
    {
        EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
        EVP_CIPHER_CTX_init(ctx);
        EVP_EncryptInit(ctx, EVP_aes_256_cbc(), (const uint8_t*)key.c_str(), (const uint8_t*)iv.c_str());

        std::string encode;
        encode.resize(str.size() + 30);
        int olen = 0;
        EVP_EncryptUpdate(ctx, (uint8_t*)&encode[0], &olen, (const uint8_t*)str.c_str(), str.length());
        int llen = 0;
        EVP_EncryptFinal(ctx, (uint8_t*)(&encode[0] + olen), &llen);
        encode.resize(olen + llen);
        std::cout << "encode: " << sylar::base64encode(encode) << " - len=" << encode.size() << std::endl;
        std::cout << "encode: " << to_hex(encode) << " - len=" << encode.size() << std::endl;
    }
```
理由：同上
