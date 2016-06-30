---
title: 基于TrieTree的关键词检索算法
layout: post
---

> 在实际应用中经常会遇到需要对给定的文本进行关键词搜索的操作，而在关键词较少时，通常的做法是将文本对每一个关键词都进行一次匹配，而随着关键词数量的增加此种算法的效率下降会非常明显。而这里将给出一种给予TrieTree（字典树）的检索算法，能够达到对于关键词个数O(1)的检索效率。

#### 1. TrieTree数据结构

> TrieTree可以像字典一样进行数据的存储，例如对于PHP Perl Pear三个单词，用TrieTree进行存储的结果大致如下
>
```php
<?php
$trie_tree = array(
    'P' => array(
        'H' => array(
            'P' => array(
            ),
        ),
        'e' => array(
            'r' => array(
                'l' => array(
                ),
            ),
            'a' => array(
                'r' => array(
                ),
            ),
        ),
    ),
);
```
> 多于多字节组成的编码，如GBK或UTF-8，这里也只需要按照其单子节的编码进行处理即可，因此算法并不依赖于文本的编码，只要保证关键词列表与待检索文本的编码一致即可。另外对于每一个节点还需要标注其是否是某个关键词的结尾，例如PHP和PHP5都是关键词，如果不进行结尾标记，在生成TrieTree时PHP这个关键词就相当于不存在了，当文本中只还有PHP关键词时将不能够匹配成功。
>

#### 2. TrieTree生成和匹配算法

> TrieTree节点包含两部分——子节点散列表和自身是否是某个词的结尾，生成算法是对每一个词，将其按字节插入到TrieTree的每一层中，并在结尾处做标记。匹配时是将文本从第一个字节开始按子节匹配，如果能够匹配到则返回(如果需要匹配到全部，则这里对匹配结果进行记录而不返回)，未能匹配到时，如果第一个字节就没有匹配到则将下一个字节做为开头进行递归匹配，不是第一个字节没有匹配到，则将此位置做为开头进行递归匹配。
>
```php
<?php
class TrieTreeNode {
    public $h = array();
    public $b = false;    
}
>
class TrieTree {
    private $root;
>
    public function __construct() {
        $this->root = new TrieTreeNode();
    }
>
    public function init($file) {
        $fp = fopen($file, 'r');
        while (($buffer = fgets($fp)) != null) {
            //插入一个词
            $node = $this->root;
            $w = trim($buffer);
            $len = strlen($w);
            for ($i = 0; $i < $len; $i++) {
                //插入每一个字节
                $c = $w[$i];
                //如果不存在对应的节点则进行插入
                if (!array_key_exists($c, $node->h)) {
                    $next = new TrieTreeNode();
                    $node->h[$c] = $next;
                    $node = $next;
                } else {
                    $node = $node->h[$c];
                }
            }
            $node->b = true;
        }
    }
>
    protected function _match($text, $pos) {
        $len = strlen($text);
        //如果匹配位置已超过文本长度，则匹配失败
        if ($pos >= $len) {
            return false;
        }
        $node = $this->root;
        $match = '';
        for ($i = $pos; $i < $len; $i++) {
            $c = $text[$i];
            if (array_key_exists($c, $node->h)) {
                $match .= $c;
                $next = $node->h[$c];
                $node = $next;
                //匹配到结束为止，返回结果
                if ($next->b) {
                    //匹配的词和匹配的位置
                    return array('w' => $match, 'p' => $pos);
                }
            } else {
                //已到达TrieTree末端还未完成匹配，则退出后重新开始匹配
                break;
            }
        }
        //递归匹配
        return $this->_match($text, $i > $pos ? $i : $pos + 1);
    }
>
    public function match($text) {
        return $this->_match($text, 0);
    }
}
```
> 其时间复杂度只与关键词长（TrieTree高度）和待匹配的文本长度有关，与关键词的个数无关，在关键词非常多时仍能保持较好的性能。但TrieTree的生成过程与关键词的个数有关，如果每次匹配都重新生成TrieTree会丧失TrieTree算法的优势，因此通常可以将生成的TrieTree保持住，可以使用服务化或是将TrieTree结构进行序列化来避免每次重新生成。

#### 3. 服务化

> 对于此类功能单一的业务逻辑，可以将其作为一种服务单独进行维护，其好处是服务可以在不同的业务之间进行共享。最简单的服务化是将其作为一个HTTP服务，这里使用PHP的swoole扩展进行实现，测试证明其能够提供较好的性能
>
```php
<?php
$tree = new TrieTree();
$tree->init($argv[1]);
>
$http = new swoole_http_server("0.0.0.0", 9501);
$http->on('request', function ($request, $response) use($tree) {
    $args = $request->get;
    $match = $tree->match($args['c']);
    $response->end(json_encode($match));
});
$http->start();
```
>
> 使用时将待匹配的文本通过URL参数c进行传递，如果文本较长则可以使用POST来实现。
>
```
curl "http://127.0.0.1:9501/?c=%E4%BD%9C%E5%BC%8A%E5%99%A8" -v
>
HTTP/1.1 200 OK
Server: swoole-http-server
Content-Type: text/html
Connection: keep-alive
Date: Sat, 21 May 2016 05:45:57 GMT
Content-Length: 32
>
{"w":"\u4f5c\u5f0a\u5668","p":0}
```
> 对1000左右数量的关键词进行压力测试，发现性能还是不错的
>
```
Server Software:        swoole-http-server
Server Hostname:        127.0.0.1
Server Port:            9501
>
Document Path:          /?c=%E4%BD%9C%E5%BC%8A%E5%99%A8
Document Length:        32 bytes
>
Concurrency Level:      20
Time taken for tests:   11.601 seconds
Complete requests:      100000
Failed requests:        0
Total transferred:      18000000 bytes
HTML transferred:       3200000 bytes
Requests per second:    8619.72 [#/sec] (mean)
Time per request:       2.320 [ms] (mean)
Time per request:       0.116 [ms] (mean, across all concurrent requests)
Transfer rate:          1515.18 [Kbytes/sec] received
```
