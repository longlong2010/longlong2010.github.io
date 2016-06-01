---
title: Java PHP共享Memcache数据
layout: post
---

> 如果在应用中同时存在Java和PHP两套系统，并且需要共享数据，这种情况下MySQL是比较容易处理的，而Memcache则会面临两个问题，首先采用了基于一致性散列算法的分布式架构下需要Java和PHP使用同一种算法，已保证可以根据键找到对应的Memcache实例，另外就是数据存储格式上需要具有一致性，一般都用JSON格式序列化即可，但还需要保证存储时压缩算法和标志位使用的一致性，才能正确的处理压缩过的数据。

#### 1. Java和PHP访问Memcache

> 对于PHP目前使用较为广泛的是[php-memcached扩展](https://github.com/php-memcached-dev/php-memcached)，其使用的一致性散列算法为Ketama，是一种简单并且广泛支持的算法，其它大多语言中都有对此算法的支持（Python Perl Go等）。为此需要Java中使用支持这一算法的连接驱动，这里尝试使用了[xmemcached](https://github.com/killme2008/xmemcached)。
>

> PHP通过php-memcached扩展访问Memcache
>
```php
<?php
$mem = new Memcached();
$mem->setOption(Memcached::DISTRIBUTION_CONSISTENT, true);
$mem->setOption(Memcached::OPT_LIBKETAMA_COMPATIBLE, true);
$mem->addServers(array(
array('127.0.0.1', 11211),
array('127.0.0.1', 11212),
array('127.0.0.1', 11213),
array('127.0.0.1', 11214),
));
```

> Java通过xmemcached访问Memcache
>
```java
try {
    XMemcachedClientBuilder builder = new XMemcachedClientBuilder();
    //注意这里需要开启cwNginxUpstreamConsistent方式，兼容libmemcached中的11211 hack
    builder.setSessionLocator(new KetamaMemcachedSessionLocator(true));
    client = builder.build();
    client.addServer("127.0.0.1", 11211);
    client.addServer("127.0.0.1", 11212);
    client.addServer("127.0.0.1", 11213);
    client.addServer("127.0.0.1", 11214);
    client.shutdown();
} catch (Exception ex) {
    ex.printStackTrace();
}
```

#### 2. libmemcached 11211 hack

> 这里简单说明一下libmemcached中的11211 hack，其为了让不加端口号的情况与使用默认11211端口号时保持一致性，其在端口号为11211时会自动去掉:11211，为此在使用很多其它的ketama库时会发现散列结果不一致，其大多是因为这个逻辑导致的，为了保持兼容在使用时也做相同的处理即可，具体逻辑可以参考以下libmemcached的源代码。
>
```c++
if (list[host_index].port() == MEMCACHED_DEFAULT_PORT)
{   
  sort_host_length= snprintf(sort_host, sizeof(sort_host),
                             "%s-%u",
                             list[host_index]._hostname,
                             pointer_index - 1); 
}   
else
{   
  sort_host_length= snprintf(sort_host, sizeof(sort_host),
                             "%s:%u-%u",
                             list[host_index]._hostname,
                             (uint32_t)list[host_index].port(),
                             pointer_index - 1); 
} 
```
#### 3. 带权重的一致性散列算法

> 在xmemcached中带权重的一致性散列算法的处理方法是将当前节点的虚节点个数（默认为160）直接乘以其权重，而在libmemcached中除了乘以了权重之外，还乘以了总节点个数/总权重值（好处是只要各节点权重比相同，散列的结果就是相同的）。为了保证散列结果的一致性，需要修改xmemcached的源代码（或者增加一个散列算法类型 AbstractMemcachedSessionLocator），其patch如下
>
```diff
diff --git a/src/main/java/net/rubyeye/xmemcached/impl/KetamaMemcachedSessionLocator.java b/src/main/java/net/rubyeye/xmemcached/impl/KetamaMemcachedSessionLocator.java
index 2e9548d..2c5a191 100644
--- a/src/main/java/net/rubyeye/xmemcached/impl/KetamaMemcachedSessionLocator.java
+++ b/src/main/java/net/rubyeye/xmemcached/impl/KetamaMemcachedSessionLocator.java
@@ -91,7 +91,12 @@ AbstractMemcachedSessionLocator {
> 
        private final void buildMap(Collection<Session> list, HashAlgorithm alg) {
                TreeMap<Long, List<Session>> sessionMap = new TreeMap<Long, List<Session>>();
-
+               int totalWeight = 0;
+               for (Session session : list) {
+                       if (session instanceof MemcachedTCPSession) {
+                               totalWeight += ((MemcachedSession) session).getWeight();
+                       }
+               }
                for (Session session : list) {
                        String sockStr = null;
                        if (this.cwNginxUpstreamConsistent) {
@@ -117,7 +122,9 @@ AbstractMemcachedSessionLocator {
                         */
                        int numReps = NUM_REPS;
                        if (session instanceof MemcachedTCPSession) {
-                               numReps *= ((MemcachedSession) session).getWeight();
+                               int weight = ((MemcachedSession) session).getWeight();
+                               float pct = (float) weight / (float) totalWeight;
+                               numReps = (int) ((Math.floor((float)(pct * NUM_REPS / 4 * list.size() + 0.0000000001))) * 4);
                        }
                        if (alg == HashAlgorithm.KETAMA_HASH) {
                                for (int i = 0; i < numReps / 4; i++) {
```
>

#### 4. 压缩算法和标志位的处理 

> 在较早版本的php-memcached中使用的是zlib进行的数据压缩，新版本中默认使用fastlz，这里为了保证兼容性依然使用zlib进行数据压缩，同时Java中默认就可以支持zlib压缩。同时在php-memcached的不同版本中有两套标志位的用法，这里需要分别予以支持。

> xmemcached中默认情况下进行数据的读写时会采用其自身的序列化和标志位的规则进行处理，但可以通过传递Transcoder自行定制对原始数据在存储和获取时的处理行为。具体的处理方法如下
>
```java
//定义针对PHP的数据处理器
abstract class PHPTranscoder extends PrimitiveTypeTranscoder<String> {
}
>
//针对旧版本的php-memcached，由于在PHP5中使用了这个版本，所以叫做PHP5
class PHP5Transcoder extends PHPTranscoder {
>    
    //使用压缩的最小长度
    final int COMPRESSION_THRESHOLD = 100;
    //压缩的标志位
    final int COMPRESSION_MASK = 1 << 1;
>
    //取出数据的处理
    public String decode(CachedData d) {
        byte[] data = d.getData();
        int flag = d.getFlag();
        //如果带有压缩标志位，则进行解压缩处理
        if ((flag & COMPRESSION_MASK) == COMPRESSION_MASK) {
            setCompressionMode(CompressionMode.ZIP);
            data = decompress(data);
        }
        return new String(data);
    }
>
    //存储数据的处理
    public CachedData encode(String o) {
        byte[] data = o.getBytes();
        int flag = 0;
        //如果数据超过了压缩的最小长度，则进行压缩，并更改标志位
        if (data.length > COMPRESSION_THRESHOLD) {
            setCompressionMode(CompressionMode.ZIP);
            data = compress(data);
            flag = COMPRESSION_MASK;
        }
        return new CachedData(flag, data);
    }
}
>
//针对新版本的php-memcached，由于在PHP7中使用了这个版本，所以叫做PHP7
class PHP7Transcoder extends PHPTranscoder {
>
    final int COMPRESSION_THRESHOLD = 2000;
    final int COMPRESSION_MASK = 3 << 4;
>
    public String decode(CachedData d) {
        byte[] data = d.getData();
        int flag = d.getFlag();
        if ((flag & COMPRESSION_MASK) == COMPRESSION_MASK) {
            setCompressionMode(CompressionMode.ZIP);
            //解压缩时需要处理size_t hack，存储时开头使用了一个size_t保存了压缩前数据的长度，为了支持fastlz
            byte[] realdata = new byte[data.length - 4];
            System.arraycopy(data, 4, realdata, 0, data.length - 4);
            data = decompress(realdata);
        }
        return new String(data);
    }
>
    public CachedData encode(String o) {
        byte[] realdata = o.getBytes();
        byte[] data = realdata;
        int flag = 0;
        if (realdata.length > COMPRESSION_THRESHOLD) {
            //上面提到的size_t hack，将长度转换为四个字节
            byte[] lenbytes = new byte[4];
            for (int i = 0; i < lenbytes.length; i++) {
                lenbytes[i] = (byte) ((realdata.length >> (8 * i)) & 0xff);
            }
            //压缩数据
            setCompressionMode(CompressionMode.ZIP);
            realdata = compress(realdata);           
            //合并数据
            data = new byte[realdata.length + 4];
            System.arraycopy(lenbytes, 0, data, 0, 4);
            System.arraycopy(realdata, 0, data, 4, realdata.length);
            flag = COMPRESSION_MASK;
        }
        return new CachedData(flag, data);
    }
}
```
>
> 这里没有对PHP中的数据类型做相关的处理，只是处理了压缩，即PHP中的任何数据类型在Java中都会变为String类型，并且Java中目前也只是提供了写入String的接口

#### 5. Java接口类

> 仿照php-memcached中的Memcached类实现了Java的访问接口，只实现了对String类型的读写操作
>
```java
class Memcached {
>
    private MemcachedClient client;
    protected PHPTranscoder transcoder;
>
    public Memcached() throws IOException {
        transcoder = new PHP7Transcoder();
        XMemcachedClientBuilder builder = new XMemcachedClientBuilder();
        builder.setSessionLocator(new KetamaMemcachedSessionLocator(true));
        client = builder.build();
    }
>
    public void setTranscoder(PHPTranscoder transcoder) {
        this.transcoder = transcoder;
    }
>
    public void addServer(String server, int port, int weight) {
        try {
            client.addServer(server, port, weight);
        } catch (IOException ex) {
            ex.printStackTrace();
        }
    }
>
    public void addServer(String server, int port) {
        this.addServer(server, port, 1);
    }
>
    public String get(String key) throws TimeoutException, InterruptedException, MemcachedException {
        return client.get(key, transcoder);
    }
>
    public boolean set(String key, String value, int expire) {
        boolean b = false;
        try {
            b = client.set(key, expire, value, transcoder);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        return b;
    }
>
    public boolean set(String key, String value) {
        return set(key, value, 0);
    }
>
    public boolean add(String key, String value, int expire) {
        boolean b = false;
        try {
            b = client.add(key, expire, value, transcoder);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        return b;
    }
>
    public boolean add(String key, String value) {
        return add(key, value, 0);
    }
>
    public boolean replace(String key, String value, int expire) {
        boolean b = false;
        try {
            b = client.add(key, expire, value, transcoder);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        return b;
    }
>
    public boolean replace(String key, String value) {
        return replace(key, value, 0);
    }
>
    public void quit() throws IOException {
        client.shutdown();
    }
}
```
