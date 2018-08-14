## 解析Ajax Web页面方法

### 1. 分析Ajax页面

使用Chrome打开页面，右键点击**检查**，选中**Network**查看浏览器与服务器之间的真实请求与响应，过滤**XHR**页签可以看到类型为`xhr`的请求，即为`ajax`请求。

![](http://ww1.sinaimg.cn/large/67c0b572gy1fu7rjsxvs6j20yp0i8jsz.jpg)

往后拖动页面的时候，会发现页面也在源源不断地加载，其中请求的`URL`中的参数基本不变，只有`page`代表页面在刷新，在请求头`headers`中，可以看到应用层传输的数据是`json`或者纯文本格式。

`X-Requested-With: XMLHttpRequest`指定了该请求为`Ajax`请求。

![](http://ww1.sinaimg.cn/large/67c0b572gy1fu7rnvuzjkj20rs0icaao.jpg)

请求发出后，服务器会发回响应数据，在**Response**可以查看原始的服务器发回的数据，**Preview**是浏览器进行渲染处理后显示的便于观察的数据，可以看出返回的是`json`对象。

![](http://ww1.sinaimg.cn/large/67c0b572gy1fu7rru813uj20s50if74t.jpg)

### 2.获取页面数据

首先通过`urllib`中的`urlencode`来构建请求网址，指定参数为一个字典对象。

```python
params = {
        'type':'uid',
        'value':1640571365,
        'containerid':1076031640571365,
        'page':page,
    }
 url = base_url + urllib.parse.urlencode(params)
 print(url)
```

通过`Requests`库发起请求获取服务器的响应数据，通过`json()`函数获取该字典结构的响应数据，获取字典的键值可以通过`get()`函数或者中括号`['key']`来获取，在函数中加入异常处理，完整函数代码如下。

```python
def get_page(page):
    params = {
        'type':'uid',
        'value':1640571365,
        'containerid':1076031640571365,
        'page':page,
    }
    url = base_url + urllib.parse.urlencode(params)
    print(url)

    try:
        response = requests.get(url,headers=headers).json().get('data')
        # print(type(response))
        return response
    except urllib.error.HTTPError as err:
        print(err.reason, err.code, err.headers, sep='\n')
        return None
    except urllib.error.URLError as err:
        print(err.reason)
        return None
```

正常解析的话，函数会返回一个json中的`data`,如下截图，这也是一个`json`对象，在python中即为一个字典结果，后续对它的操作都是与字典操作一致。

![](http://ww1.sinaimg.cn/large/67c0b572gy1fu7sc381exj20s50hhdge.jpg)

### 3.解析页面数据

获取页面数据后，可以开始解析页面数据了，从上面截图中获取到`cards`字典中的数据，得到的会是一个列表`list`，需要通过迭代的方式来进行解析，如下图示我们解析每个列表中的评论数`comments_count`，创建时间`created_at`，`id`，转发数量`reposts_count`和微博内容`text`。

![](http://ww1.sinaimg.cn/large/67c0b572gy1fu7smhctvuj20s10i5js7.jpg)

```python
def parse_page(content):
    # 如果数据非空，则获取页面数据成功，开始解析页面数据
    if content:
        cards = content.get('cards')
        # print(type(cards))
        # print(type(content.get('cardlistInfo').get('page')))
        for card in cards:
            # print(type(card))
            item = card.get('mblog')
            # print(type(item))
            if item:
                data = {
                    '创建时间': item.get('created_at'),
                    '点赞次数': item.get('attitudes_count'),
                    '评论次数': item.get('comments_count'),
                    'id': item.get('id'),
                    '微博内容': pyquery.PyQuery(item.get('text')).text(),
                }
                print(data)
```

本来`微博内容`字段通过`item.get('text')`即可获取到，但是最终发现这个数据中包含了一些html格式的标签，所以通过`pyquery`的`text()`函数去掉标签。同时，经过观察发现如果是当天发出的微博或者昨天发出的，`创建时间`字段格式会变成`今天5分钟前`之类，所以可以通过正则表达式的方式来处理该字段。

```python
def get_date(time):
    reg_today = r'今天|分钟前|小时前'
    reg_yesterday = r'昨天'
    today = datetime.date.today()
    yesterday = today - datetime.timedelta(days=1)
    year = today.strftime('%Y')

    if re.search(reg_today,time):
        # 今天发出的微博，返回字符串格式的今天日期
        return str(today)
    elif re.search(reg_yesterday,time):
        # 昨天发出的微博，返回字符串格式的昨天日期
        return str(yesterday)
    elif len(time) < 6:
        # 今年发出的微博不会携带年份字段，所以手动加上年份
        return year + '-' + time
    else:
        # 非今年发出的微博，正常返回即可
        return time
```

将解析函数重新替换字段`创建时间`，键值改为上述函数即可。

### 4.保存数据到MongoDB

定义全局变量指定`连接`，`数据库db`和`集合collection`，然后定义保存函数，即：在集合中插入一条json数据(**字典结构**)。

```python
client = pymongo.MongoClient('localhost', 27017)
db = client.weibo
collection = db.luoyonghao

def save_to_mongodb(data):
    collection.insert_one(data) 
```

在解析函数`parse_page()`中调用保存函数即可。

### 5. 完善爬虫

给出完整代码如下。

```python
import urllib
import requests
import pyquery
import pymongo
import re
import datetime

base_url = 'https://m.weibo.cn/api/container/getIndex?'

headers = {
    'User-Agent':'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36',
    'X-Requested-With':'XMLHttpRequest',
    'Referer': 'https://m.weibo.cn/u/1640571365',
}

client = pymongo.MongoClient('localhost', 27017)
db = client.weibo
collection = db.luoyonghao

def get_page(page):
    params = {
        'type':'uid',
        'value':1640571365,
        'containerid':1076031640571365,
        'page':page,
    }
    url = base_url + urllib.parse.urlencode(params)
    print(url)

    try:
        response = requests.get(url,headers=headers).json().get('data')
        # print(type(response))
        return response
    except urllib.error.HTTPError as err:
        print(err.reason, err.code, err.headers, sep='\n')
        return None
    except urllib.error.URLError as err:
        print(err.reason)
        return None

def get_date(time):
    reg_today = r'今天|分钟前|小时前'
    reg_yesterday = r'昨天'
    today = datetime.date.today()
    yesterday = today - datetime.timedelta(days=1)
    year = today.strftime('%Y')

    if re.search(reg_today,time):
        return str(today)
    elif re.search(reg_yesterday,time):
        return str(yesterday)
    elif len(time) < 6:
        return year + '-' + time
    else:
        return time

def parse_page(content):
    # 如果数据非空，则获取页面数据成功，开始解析页面数据
    if content:
        cards = content.get('cards')
        # print(type(cards))
        # print(type(content.get('cardlistInfo').get('page')))
        for card in cards:
            # print(type(card))
            item = card.get('mblog')
            # print(type(item))
            if item:
                data = {
                    # '创建时间': item.get('created_at'),
                    '创建时间': get_date(item.get('created_at')),
                    '点赞次数': item.get('attitudes_count'),
                    '评论次数': item.get('comments_count'),
                    'id': item.get('id'),
                    '微博内容': pyquery.PyQuery(item.get('text')).text(),
                }
                print(data)
                # yield data
                if collection.find_one(data):
                    print('数据库已经有这页数据了')
                else:
                    save_to_mongodb(data=data)

def save_to_mongodb(data):
    collection.insert_one(data)         

if __name__ == '__main__':
    total_page = int(get_page(1).get('cardlistInfo').get('total')/11)
    for page in range(1,1+total_page):
        content = get_page(page)
        parse_page(content)
```

运行程序，最终保存数据到MongoDB中，查看前10000条数据。

![](http://ww1.sinaimg.cn/large/67c0b572gy1fu7t5ojonrj20xy0irt93.jpg)