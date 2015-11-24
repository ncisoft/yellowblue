---
layout: post
styles: [syntax]
title: ss 最佳使用实践
categories: gfw
tags: ['shadowsocks']
comments: true

---

## 背景知识

[shadowsocks](https://github.com/shadowsocks/shadowsocks) 是最新流行的翻墙方法，通过提供 socks5代理方式达到翻墙的目的。在各种vpn被封锁的情况下，尤为简洁高效。

## 基础用法

[cow](https://github.com/cyfdecyf/cow) 是一个支持 shadowsocks 的代理工具，除了基本的将socks5 转化为 http 代理功能之外，还能自动检测被墙网站，仅对这些网站使用二级代理，从而实现国内国外自动分流的目的，具体请详见其[项目介绍](https://github.com/cyfdecyf/cow)

## 常见问题

* *dns被污染怎么办？*

必须要说，这是最常见的误区。要知道，socks5接口本身是提供了远端域名解析功能的，所以本地dns再怎么被污染，都没有关系。shadowsocks也是如此，基于socks5和http代理（over socks5）的使用方式，在浏览器的使用场景下，都不需要存在dns污染的顾虑。

至于命令行操作，如果能指定代理，也是一样的，比如curl，有三种指定方式：
在测试之前，手动修改 本地hosts文件，将www.google.com指向127.0.0.1，模拟dns被污染的场景。

{% highlight bash linenos %}
    curl --proxy http://127.0.0.1:3128 -o /dev/nul https://www.google.com/images/google_favicon_128.png
    curl --socks5-hostname 127.0.0.1:1080 -o /dev/null  https://www.google.com/images/google_favicon_128.png
    curl --socks5 127.0.0.1:1080 -o /dev/null https://www.google.com/images/google_favicon_128.png
    $docker pull ubuntu:14.04
{% endhighlight %}


{% highlight java linenos %}
// test
public void testThread() {
	TestThread thread = new TestThread(this);
	thread.start();
	System.out.println("begin wait");
	try
	{
		this.result.wait();
		System.out.println("after wait, the result = 0 " + result);
	} catch (InterruptedException e)
	{
		// TODO Auto-generated catch block
		e.printStackTrace();
	}
	System.out.println("end wait");
}
{% endhighlight %}

{% highlight javascript linenos %}
//给id元素绑定两个单击和一个鼠标离开事件
$('#id').bind('click', function() { alert('once'); });
$('#id').bind('click', function() { alert('second'); });
$('#id').bind('mouseout', function(){ alert('mouseout'); });

//events对象的值
{
  'click': [
    {// 此对象还有:data,guid,namespace属性
      handler: function() { alert('once'); },
      type: 'click'
    },
    {
      handler: function() { alert('second'); },
      type: 'click'
    }
  ],
  'mouseout': [
    {
      handler: function(){ alert('mouseover'); },
      type: 'mouseout'
    }
  ]
}
{% endhighlight %}




测试结果表明就方式3不能正常访问，因为它用的是本地dns解析，而前两者都工作正常。

* *为什么用了ss和cow还是有些网站不能正常访问？*

一般来说，你应该是使用了商业的ss服务，由于该ss服务商的服务器被多人使用，所以较大几率不是没有翻墙成功，而是ss服务商的ip被别的网站所拒绝，出现的错误常见reset, refused。

这是个很关键的分析，请大家记住结论。问题如何解决呢，下面我们进入进阶用法

## 那些踩过的坑

* *ss-redir好用吗？*

我试过全局代理，能用，但是针对部分ip、port的代理则一直没有成功，redsocks则好很多，一配就过。而且ss-redir的日志输出简陋得很，基本没用，远没有ss-local、redsocks丰富。所以如果你想用ss-redir，建议改用redsocks、

* *通过ss-tunnel udprelay代理解析域名好用吗？*

我在v2ex的节点上看到很多人问通过ss-tunnel udp代理解析域名的问题。这里有三个个误区，第一，不是所有的ss服务商都开启了udp relay功能的，比如我用的也只是部分节点才有提供，服务商也建议慎用；第二，参见上一节的分析，用了shadowsocks其实已经不用考虑域名解析纯净的问题，第三，如果有洁癖，仍然希望得到纯净的dns解析，正确的姿势是用ss-tunnel 的 tcprelay，shadowsocks支持tcprelay毫无鸭梨呀。且得到的ip是到你的ss服务器的最佳ip（dns智能解析）。你在本地用比如pdnsd桥接一下即可。示例代码如下：
<pre> <code class="language-markup"> ss-tunnel -c /tmp/ss-local.conf -b 0.0.0.0 -l 53 -L 8.8.8.8:53 -v </code></pre>

## 进阶用法

接上条，既然被reset。refused的原因是ss服务商被多人使用造成被拒绝，要想稳定科学上网，必须要有一个稳定的外网socks5服务才行，至于怎么才能得到稳定的外网socks5服务，这留在下一环节再做分析。

那么假设你现在有了一个商业的ss服务（以下简称服务A），和一个稳定但可能速度略差的socks5服务（以下简称服务B），那么问题就变成了服务A快速但不稳定，服务B稳定但不快速。当然如果你的ss服务即快速又稳定，看youtube 720p没有压力，那么基础用法就足够适合你，以下就不用看了。

答案是[privoxy](http://www.privoxy.org/)。privoxy是个很常用的socks5-to-http代理，一般人都这么用，又或者当做垃圾广告拦截器来使用。但这严重浪费了privoxy的能力。privoxy的功能还包括基于hostname、user-agent（比如针对智能手机上的qq、微信、微博）分别做代理的能力（透过action进行配置），我的做法是国内网站直接出去，不经过A/B代理服务，而[gfw白名单](https://github.com/breakwa11/gfw_whitelist)上的网站走代理A，关键重点是服务A走不通的走代理B。这样就完美地利用了国内网速、代理A的速度、代理B的稳定性。要注意这些action的执行顺序。比如我用的是这样的

<pre> <code class="language-markup"> 
	actionsfile direct.action      #国内网站直通，网站名做为判断依据
    actionsfile ua-direct.action #国内网站直通，user-agent 做为判断依据
    actionsfile gfw.action         #gfw白名单
    actionsfile me.action         #自己维护的
</code></pre>


[gfw白名单](https://github.com/breakwa11/gfw_whitelist)有人给做好了，怎么判断哪些网站是国内的呢？我的做法是把cow当做机器训练，也即privoxy配置成缺省走cow代理，通过定期分析cow的proxy.pac用工具转化成direct.action，这个工具我刚开始实现，等做出来了再给大家分享。

* *为什么不用cow当前端，privoxy当后端？*

理论上这是更好的方案，direct.action、ua-direct.action都可以省略掉了，可是我配置成privoxy当后端时卡顿异常严重，只有配置成socks5才快。如果你能配置成这种方式且使用顺利，那么恭喜了，我给cow提了[issue](https://github.com/cyfdecyf/cow/issues/303)，希望作者能尽快出新版解决这个问题。

* *privoxy是否很配置起来复杂？*

privoxy的规则不算复杂（不包括广告拦截这块，这部分我还没研究），比如[网站名匹配的规则(http://www.privoxy.org/user-manual/actions-file.html#AF-PATTERNS)]就很简单，如果有人能像[gfw白名单](https://github.com/breakwa11/gfw_whitelist)那样开箱即用，就更没难度了。

没码完，待续。。。后面是最精华的部分，敬请期待，预期明天4月6日写完。
<pp>没码完，待续。。。后面是最精华的部分，敬请期待，预期明天4月6日写完。</p>
