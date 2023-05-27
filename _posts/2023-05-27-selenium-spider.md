---
title: 使用Selenium实现浏览器爬取网页
layout: post
---

> 尽管早已经进入移动App的世代，但是大部分服务商还是会提供便于PC使用的网页版，而在Web时代最长做的一个事情就是抓取网络上的数据，并将抓取的数据进行整理分析，其实如今的不少类似的需求应该还是延续这种方式，毕竟通用性强，不需要针对不同类型的App逐个反向研究App内部的通信协议。
>
> 而在之前通常使用的方法是使用基于各中编程语言提供的HTTP客户端API进行直接的数据抓取，这种方法直接获取有用的数据，有着效率高同时节约网络带宽的特点。但如今即使是Web端也同样利用的大量的Javascrip技术，网页中大量数据都是在网页HTML框架加载完成后才进行，要想抓取其中有用的数据还需要搞清楚整个网页的运行机理，同时各种Javascript代码还进行了各种精简化很难读懂，为此通脚本直接过操控浏览器的方式就成为了一种实现更加简单，并且由于是完全模拟浏览器的行为，在服务端进行识别和封禁也相对会比较难，有着更好的适应性，缺点也同样明显——效率低占用资源也相对较高。

#### 1. Selenium简介

> Selenium 通过使用WebDriver支持市场上所有主流浏览器的自动化。 Webdriver是一个API和协议，它定义了一个语言中立的接口，用于控制Web浏览器的行为。每个浏览器都有一个特定的WebDriver实现，称为驱动程序。驱动程序是负责委派给浏览器的组件，并处理与 Selenium 和浏览器之间的通信。其本质就是一种浏览器的操作接口，可以实现对各种浏览器的自动化操作。
>
> 这里将以Python语言和Firefox浏览器为例介绍Selenium的使用方法，使用前需要下载Firefox浏览器对应的驱动程序Geckodriver，在Windows下可将下载好的geckodriver.exe程序放入Firefox的安装目录下。

#### 2. 初始化

> 在使用前需要通过对应的驱动程序初始化对应的浏览器实例，其中比较关键的是设置浏览器的工作目录，如果不进行手动设置Firefox会每次自己都建立一个新的临时目录，会导致每次运行是一些持久化数据都消失掉
```python
from selenium import webdriver;
from selenium.webdriver.firefox.service import Service;
> 
#增加环境变量，指向驱动程序所在目录
os.putenv(NAME, os.getenv(NAME) + ';C:\Program Files\Mozilla Firefox');
#设置浏览器配置
options = webdriver.FirefoxOptions();
#不显示浏览器的图形界面
options.add_argument('--headless');
#设置浏览器的工作目录
options.add_argument('-profile');
options.add_argument('E:\TMP\FF');
#初始化浏览器实例
browser = webdriver.Firefox(options=options, service=Service(log_path=os.devnull));
```

#### 3. 访问页面和查找元素
>
> 通过get方法可以访问对应URL的页面
```python
browser.get('https://www.baidu.com');
```
>
> 通过find\_element或是find\_elements方法可以找到对应的元素，其中这两个方法的不同在于find\_element只会找其中第一个匹配的元素，同时在找不到时抛出异常，而find\_elements则会返回全部的结果列表，同时在找不到时返回空列表，可根据实际的需要使用对应的方法。查找的方式可以通过css选择其或是通过连接的内容等进行，与jQuery等浏览器中的操作方式基本相同。
```python
from selenium.webdriver.common.by import By;
e = browser.find_element(By.CSS_SELECTOR, 'div .sku-name');
e = browser.find_element(By.CSS_SELECTOR, '#store-prompt > strong');
e = browser.find_element(By.LINK_TEXT, '登录');
```

#### 4. 元素的操作
>
> 对于获得的元素可以进行类似浏览器前端中的各种操作——触发点击、输入等动作或是获取元素的属性及内部文本信息等。
```python
e.send_keys('username');
e.click();
print(e.text);
```

#### 5. 完整的例子
>
> 以下是一个完整的例子，可以用于获取列表中京东的商品价格，出于省事用于每天看看自己想买的东西有没有降价^_^
```python
from selenium import webdriver;
from selenium.webdriver.firefox.service import Service;
from selenium.webdriver.common.by import By;
import os;
import time;
>
if __name__ == '__main__':
    NAME = 'PATH';
    os.putenv(NAME, os.getenv(NAME) + ';C:\Program Files\Mozilla Firefox');
    options = webdriver.FirefoxOptions();
    options.add_argument('--headless');
    options.add_argument('-profile');
    options.add_argument('E:\TMP\FF');
    browser = webdriver.Firefox(options=options, service=Service(log_path=os.devnull));
    #商品连接列表
    URLS = [''];
    for url in URLS:
        browser.get(url);
        time.sleep(0.5);
        #获取商品名称
        e = browser.find_element(By.CSS_SELECTOR, 'div .sku-name');
        name = e.text;
        #获取商品状态，即是否有货
        e = browser.find_element(By.CSS_SELECTOR, '#store-prompt > strong');
        status = e.text;
        #获取价格
        elements = browser.find_elements(By.CSS_SELECTOR, 'span .price');
        e1 = elements[0];
        e2 = elements[1];
        #如果有第二个价格说明是预购状态，进一步确定尾款金额
        if e2.text == '':
            price = e1.text;
        else:
            price = e2.text;
            elements = browser.find_elements(By.CSS_SELECTOR, '.yy-category');
            if elements != '':
                e3 = elements[0];
                #如果定金可额外抵扣一部分尾款，则计算实际的商品价格
                if e3.text != '':
                    #获取定价额度
                    e4 = browser.find_element(By.CSS_SELECTOR, 'span .J-earnest');
                    #产品实际价格为：尾款金额-抵扣金额+定金
                    price = float(e2.text) - float(e3.text.replace('可抵¥', '')) + float(e4.text);
        #输出得到的价格
        print(price, "\t", status, "\t", name);
        time.sleep(1.0);
    time.sleep(2.0);
    #退出
    browser.quit();
```
