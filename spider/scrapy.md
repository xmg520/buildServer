# Scrapy：
Scrapy是一个为了爬取网站数据，提取结构性数据而编写的应用框架。 可以应用在包括数据挖掘，信息处理或存储历史数据等一系列的程序中。

其最初是为了 页面抓取 (更确切来说, 网络抓取 )所设计的， 也可以应用在获取API所返回的数据(例如 Amazon Associates Web Services ) 或者通用的网络爬虫。


## 创建一个新的`Scrapy`项目

```
scrapy startproject boss
```

## 配置文件
```
scrapy.cfg: 项目的配置文件
boss/: 该项目的python模块。之后您将在此加入代码。
boss/items.py: 项目中的item文件，Item 是保存爬取到的数据的容器.
boss/pipelines.py: 项目中的pipelines文件 指定存储地址.
boss/settings.py: 项目的设置文件.
boss/spiders/: 放置spider代码的目录.
```

## 例子
[前程无忧Scrapy爬虫](https://gitee.com/9035/scrapyQjwy)  
[boss直聘Scrapy爬虫](https://gitee.com/9035/ScrapyBoos)
