---
title: 微信机器人
layout: post
---
> 有些使用我们可能希望使用普通的微信帐号来实现对消息的自动处理和回复功能，之前使用过XMPP协议做过一个GTalk的机器人，能够自动使用歌词不断更换签名档。而微信使用的是自行设计的协议，所以要实现微信的机器人需要先分析其通信协议，不过目前已有不少开源的根据Web版微信分析出的接口库可以使用，主要是使用Python和Nodejs实现的。

#### 1. wxBot的安装
> 这里使用的是一个名为wxBot的开源项目提供的微信接口，其官方Github地址为[https://github.com/liuwons/wxBot](https://github.com/liuwons/wxBothttps://github.com/liuwons/wxBot)
>
> 其安装需要依赖两个不太常见的Python的类库——[pyqrcode](https://github.com/mnooner256/pyqrcode)和[pypng](https://pypi.python.org/pypi/pypng)，用于生成登录用的二维码。使用以下的代码可以完成wxBot库及其依赖的下载
>
```sh
git clone https://github.com/liuwons/wxBot.git
cd wxBot
git clone https://github.com/mnooner256/pyqrcode.git pyqrcode-git
git clone https://pypi.python.org/pypi/pypng.git pypng-git
cp -r pyqrcode-git/pyqrcode ./
cp pypng-git/code/png.py ./
```
>
> 由于在使用的过程中需要使用到Python处理中文字符，需要保证系统支持了en\_US.UTF-8字符集，并且LANG环境变量设置为en\_US.UTF-8
>

#### 2. wxBot的使用
> 可以运行代码目录中的test.py脚本进行测试，由于wxBot的代码不能兼容Python3，因此在默认使用Python3的系统中需要使用python2来运行该脚本
>
```sh
python2 test.py
[INFO] Please use WeChat to scan the QR code .
```
>
> 出现以上提示时会在当前目录下生成一个temp目录，其中会包含文件wxqr.png，其就是登录需要使用的二维码，打开手机微信App，扫描此二维码，并点击确认登录后会出现以下输出
>
```
[INFO] Please confirm to login .
[INFO] Web WeChat login succeed .
[INFO] Web WeChat init succeed .
[INFO] Get 184 contacts
[INFO] Start to process messages .
```
>
> 说明已经正确获取到了联系人列表，程序开始等待接收消息，如果收到消息，则会输出以下提示
>
```
[MSG] XX:
    [Text] [呲牙]
```
>
> 其中第一行为消息的发送者，第二行为消息的类型和消息的内容，如果内容中包含文件，则文件会被下载到当前目录的temp目录下，消息内容会是文件名
>
```
[MSG] XX:
    [Image] img_6556032082643325149.jpg
```
>
> 根据自己的需求处理消息则可以参考test.py中的内容，其主要思想就是继承WXBot类，并覆盖相应的消息处理方法既可完成对消息的处理操作，例如test.py中的代码，收到对方的文本消息后会自动回复一个hi
>
```python
class MyWXBot(WXBot):
    def handle_msg_all(self, msg):
        if msg['msg_type_id'] == 4 and msg['content']['type'] == 0:
            #给对方发送消息
            self.send_msg_by_uid(u'hi', msg['user']['id'])
```
