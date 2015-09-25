##Scrapy 一瞥##

scrapy 是一个用于抓取网站、提取结构化数据的应用框架。广泛应用于数据挖掘、信息处理和历史归档等应用中。

虽然原本本设计用来做[数据抓取](http://en.wikipedia.org/wiki/Web_scraping)，它也可以用来通过 API 提取数据(例如 [Amazon Associates Web Services](http://aws.amazon.com/associates/)) 或者一般目的的爬虫。

## 来看一个简单爬虫 ##

为了让你看看 Scrapy 能干什么，我们通过一个能运行的简单蜘蛛来了解一下。

下面是一个蜘蛛的代码，它的作用是跟踪StackOverflow上最热门的问题的链接，并从每个页面中提取一些数据：

```python
import scrapy


class StackOverflowSpider(scrapy.Spider):
    name = 'stackoverflow'
    start_urls = ['http://stackoverflow.com/questions?sort=votes']

    def parse(self, response):
        for href in response.css('.question-summary h3 a::attr(href)'):
            full_url = response.urljoin(href.extract())
            yield scrapy.Request(full_url, callback=self.parse_question)

    def parse_question(self, response):
        yield {
            'title': response.css('h1 a::text').extract()[0],
            'votes': response.css('.question .vote-count-post::text').extract()[0],
            'body': response.css('.question .post-text').extract()[0],
            'tags': response.css('.question .post-tag::text').extract(),
            'link': response.url,
        }
```

把这段代码保存到`stackoverflow_spider.py`中，用 `runspider`命令运行一下：

```shell
scrapy runspider stackoverflow_spider.py -o top-stackoverflow-questions.json
```

运行结束时会生成`top-stackoverflow-questions.json`文件。以 json 格式保存了StackOverflow上最热门的问题，一个 tags 列表和 HTML 格式的问题内容，就是这个（为了方便阅读整理了格式）：

```
[{
    "body": "... LONG HTML HERE ...",
    "link": "http://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-an-unsorted-array",
    "tags": ["java", "c++", "performance", "optimization"],
    "title": "Why is processing a sorted array faster than an unsorted array?",
    "votes": "9924"
},
{
    "body": "... LONG HTML HERE ...",
    "link": "http://stackoverflow.com/questions/1260748/how-do-i-remove-a-git-submodule",
    "tags": ["git", "git-submodules"],
    "title": "How do I remove a Git submodule?",
    "votes": "1764"
},
...]
```

##发生了什么？##

当你运行命令`scrapy runspider somefile.py`的时候，Scrapy在文件中查找蜘蛛定义(Spider definition)并且通过爬虫引擎运行了它。


抓取从定义在`start_urls`属性中的 URL 生成请求开始，由默认的回调方法`parse`调用（传入响应对象）。在`parse`方法中，我们用 CSS 选择器从问题页中提取数据，接着我们又抛出几个待发送的请求，把`parse_question`作为这几个新请求的回调方法，直到每个请求都完成。

这时你注意到一个 scrapy 的主要优势：**请求是异步地被调度和处理的**。这意味着 Scrapy 不需要等一个请求返回并处理完成，它就可以同时发送另一个请求，或者做些其他事情。这也意味着即使一些请求失败了，或者处理过程中出错了，其他请求可以继续处理。

在给你快速抓取能力（以fault-tolerant方式可以并发发送多个请求）的同时，你可以通过一些设置控制它。你可以设置每个请求之间的下载延时，限制每个域或者 IP 的并发数，甚至通过auto-throttling扩展尝试自动限制并发数。

While this enables you to do very fast crawls (sending multiple concurrent requests at the same time, in a fault-tolerant way) Scrapy also gives you control over the politeness of the crawl through a few settings. You can do things like setting a download delay between each request, limiting amount of concurrent requests per domain or per IP, and even using an auto-throttling extension that tries to figure out these automatically.

Finally, the parse_question callback scrapes the question data for each page yielding a dict, which Scrapy then collects and writes to a JSON file as requested in the command line.


