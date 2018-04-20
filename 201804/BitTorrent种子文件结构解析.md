# BitTorrent种子文件结构解析

最近突然对种子的结构感兴趣，那就顺便了解一下。并用python解析看看。

### bencode介绍
我们先介绍 bencode 这种编码格式，因为 bt 种子文件，包括以后的 DHT 网络中，都是用这种编码的。网上有很多介绍，这里简单再重复一遍。bencode 有 4 种数据类型: string, integer, list 和 dictionary。

  1. **string**
  
  字符是以这种方式编码的: <字符串长度>:<字符串>。
  如 hell: 4:hell
  
  2. **integer**
  
  整数是一这种方式编码的: i<整数>e。
  如 1999: i1999e
  
  3. **list**
  
  列表是一这种方式编码的: l[数据1][数据2][数据3][…]e。
  如列表 [hello, world, 101]：l5:hello5:worldi101ee
  
  4. **dictionary**
  
  字典是一这种方式编码的: d[key1][value1][key2][value2][…]e，其中 key 必须是 string 而且按照字母顺序排序。
  如字典 {aa:100, bb:bb, cc:200}： d2:aai100e2:bb2:bb2:cci200ee

bt 种子文件是使用 bencode 编码的，整个文件就 dictionary，包含以下键。

  1. info, dictinary, 必选, 表示该bt种子文件的文件信息。

    * 文件信息包括文件的公共部分

      **piece length, integer**, 必选, 每一数据块的长度  
      **pieces, string**, 必选, 包含所有数据块的 SHA1 校验值，每个块的20个字节的SHAT Hash的值直接连起来(二进制格式)，所以这个字段的长度一定是20的整数倍  
      **publisher, string**, 可选, 发布者  
      **publisher.utf-8, string**, 可选, 发布者的 UTF-8 编码  
      **publisher-url, string**, 可选, 发布者的 URL  
      **publisher-url.utf-8, string**, 可选, 发布者的 URL 的 UTF-8 编码  
      
    * 如果 bt 种子包含的是单个文件，包含以下内容
      
      **name, string**, 必选, 推荐的文件名称  
      **name.utf-8, string**, 可选, 推荐的文件名称的 UTF-8 编码  
      **length, int**, 必选， 文件的长度单位是字节  
      
    * 如果是多文件，则包含以下部分:
      
      **name, string**, 必选, 推荐的文件夹名称  
      **name.utf-8, string**, 可选, 推荐的文件名称的 UTF-8 编码  
      **files, list**, 必选, 文件列表，每个文件列表下面是包括每一个文件的信息，文件信息是个字典。  

    * 文件字典
      
      **length, int**, 必选， 文件的长度单位是字节  
      **path, string**, 必选， 文件名称，包含文件夹在内  
      **path.utf-8, string**, 必选， 文件名称 UTF-8 表示，包含文件夹在内  
      **filehash，string**, 可选， 文件 hash。  
      **ed2k, string**, 可选, ed2k 信息。  

  2. announce, string, 必选, tracker 服务器的地址
  3. announce-list, list, 可选, 可选的 tracker 服务器地址
  4. creation date， interger， 必选, 文件创建时间
  5. comment， string, 可选, bt 文件注释
  6. created by， string， 可选， 文件创建者。

上面列举的可能不是很完整的，但是大体上主要的字段没有重大的错误。

综上，多文件Torrent的结构的树形图为：

    Multi-file Torrent
    ├─announce
    ├─announce-list
    ├─comment
    ├─comment.utf-8
    ├─creation date
    ├─encoding
    ├─info
    │ ├─files
    │ │ ├─length
    │ │ ├─path
    │ │ └─path.utf-8
    │ ├─name
    │ ├─name.utf-8
    │ ├─piece length
    │ ├─pieces
    │ ├─publisher
    │ ├─publisher-url
    │ ├─publisher-url.utf-8
    │ └─publisher.utf-8
    └─nodes

单文件Torrent的结构的树形图为：

    Single-File Torrent
    ├─announce
    ├─announce-list
    ├─comment
    ├─comment.utf-8
    ├─creation date
    ├─encoding
    ├─info
    │ ├─length
    │ ├─name
    │ ├─name.utf-8
    │ ├─piece length
    │ ├─pieces
    │ ├─publisher
    │ ├─publisher-url
    │ ├─publisher-url.utf-8
    │ └─publisher.utf-8
    └─nodes

**将上述文件中的info信息内容，bencode格式，求SHA1值，即为Torrent的HASH值。也就是磁力链接里`magnet:?xt=urn:btih:`后面的信息**

哈哈，了解了这个结构，可以自己编码写一个种子解析的三方包了。

### Python解析

Python 的 bencode 可在 pypi 里面找到: [bencode](https://pypi.org/project/bencode/1.0/)。

解析代码，以`CentOS-7-x86_64-Everything-1708.torrent`这个种子为例。
```python
import bencode, hashlib, base64, urllib

torrent = open('~/CentOS-7-x86_64-Everything-1708.torrent', 'rb').read()
metadata = bencode.bdecode(torrent)
hashcontents = bencode.bencode(metadata['info'])
print(hashcontents)
digest = hashlib.sha1(hashcontents).digest()
b16hash = base64.b16encode(digest)
print(b16hash)
params = {'xt': 'urn:btih:%s' % b16hash}
paramstr = urllib.urlencode(params)
magneturi = 'magnet:?%s' % paramstr
print(magneturi)
print(urllib.unquote(magneturi))
# print('e07cd65fe51f07980580bcf66eeec3419e136e01')
```
这个种子文件里面包含的内容有

    CentOS-7-x86_64-Everything-1708.torrent
    ├─announce {str}'http://torrent.centos.org:6969/announce'
    ├─announce-list
    │  ├─ {list}<type 'list'>: ['http://torrent.centos.org:6969/announce']
    │  └─ {list}<type 'list'>: ['http://ipv6.torrent.centos.org:6969/announce']
    ├─comment {str}'CentOS x86_64 Everything ISO'
    ├─create by {str}'mktorrent 1.0'
    ├─creation date {int}1505313544
    └─info
      ├─files
      │  ├─0
      │  │ ├─length {int}8694792192
      │  │ └─path {list}<type 'list'>: ['CentOS-7-x86_64-Everything-1708.iso']
      │  ├─1
      │  │ ├─length {int}454
      │  │ └─path {list}<type 'list'>: ['sha1sum.txt']
      │  ├─2
      │  │ ├─length {int}1314
      │  │ └─path {list}<type 'list'>: ['sha1sum.txt.asc']
      │  ├─3
      │  │ ├─length {int}598
      │  │ └─path {list}<type 'list'>: ['sha256sum.txt']
      │  └─4
      │    ├─length {int}1458
      │    └─path {list}<type 'list'>: ['sha256sum.txt.asc']
      ├─name {str}'CentOS-7-x86_64-Everything-1708'
      ├─piece length {int}524288
      └─pieces {str}一串二进制格式的代码，直接看不到内容，可以使用`base64.b16encode(metadata['info']['pieces'][0:20])`查看，每20位是一个分片的SHA1编码



### 引用
> http://luoguochun.cn/2014/09/17/bt-file-structure/  
> https://blog.csdn.net/mergerly/article/details/8013694  
> https://segmentfault.com/a/1190000000681331  
> https://blog.csdn.net/u012888602/article/details/47167841  

