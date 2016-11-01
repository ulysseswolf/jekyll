---
layout: post
title: "python用requests+tor搭建轮换ip爬虫"
description: ""
categories:    
- python
tags: [python]
---
{{ page.title }}
================

一般防爬虫有很多办法，最基本的是判断header是不是浏览器，比如python有个robot-detection库可以判断，不过这个太好模拟了，没有用。然后可以通过分析日志限定用户ip一定时间段内访问次数，或者通过cookie、session限定特定用户的访问次数等。理论上说，凡可以看到的内容都是可以通过技术手段爬到的，只是难易的差别。
下面记录一下在ubuntu server上用tor搭建一个轮换IP的代理，如果爬虫一定时间段内没有得到内容，就可以换一个IP了。你可以先参考最后的几个链接大致了解下。系统用的vultr日本机房ubuntu server14.04。这是一种不用花钱的解决方案，也可以直接购买一些提供IP代理的服务，不需要自己折腾了。(注意：国内的服务器提供商使用的ISP可能限制了tor，如果你按照下边方法发现总是ttl超时，看下tor的log。推荐用国外的服务器跑爬虫）


安装
安装tor，先sudo su一下。

apt-get update
apt-get install tor
/etc/init.d/tor restart
启动后socks监听9050端口。

tor --hash-password mypassword生成你的密码，之后编辑/etc/tor/torrc

加上
ControlPort 9051
HashedControlPassword 16:872860B76453A77D60CA2BB8C1A7042072093276A3D701AD684053EC4C
让ControlPort监听9051端口，后边那个16:开头的hash就是上一步得到的。

/etc/init.d/tor restart重启下tor。

Python使用
pip install requesocks

{% highlight python%}
#!/usr/bin/env python
# -*- coding:utf-8 -*-

import os
import requests
import requesocks

url = 'https://api.ipify.org?format=json'


def getip_requests(url):
    print "(+) Sending request with plain requests..."
    r = requests.get(url)
    print "(+) IP is: " + r.text.replace("\n", "")


def getip_requesocks(url):
    print "(+) Sending request with requesocks..."
    session = requesocks.session()
    session.proxies = {'http': 'socks5://127.0.0.1:9050',
                       'https': 'socks5://127.0.0.1:9050'}
    r = session.get(url)
    print "(+) IP is: " + r.text.replace("\n", "")


def main():
    print "Running tests..."
    getip_requests(url)
    getip_requesocks(url)
    os.system("""(echo authenticate '"yourpassword"'; echo signal newnym; echo quit) | nc localhost 9051""")
    getip_requesocks(url)


if __name__ == "__main__":
    main()
{% endhighlight %}
注意中间os.system这句，是用来重新生成一个新的ip的。一般在代码里根据内容是否有效判断是不是需要更新ip了，可以抛出一个异常之后执行这一句重新换个ip。

update：新的requests库版本2.10.0已经支持socks代理了，这下就非常方便了，代理写法和http一样，具体参考其文档，如此一来搭建一个轮换IP的爬虫就非常容易了。

参考
http://robbinfan.com/blog/11/anti-crawler-strategy

http://sacharya.com/crawling-anonymously-with-tor-in-python/

http://stackoverflow.com/questions/1969958/how-to-change-tor-exit-node-programmatically

https://github.com/dvska/requesocks

https://github.com/PegasusWang/PySiteCrawler

http://platinhom.github.io/2016/01/21/proxy-py/

http://tor.stackexchange.com/questions/4538/bootstrapped-at-10-for-hours ttl超时请参考这个方法使用obfsproxy