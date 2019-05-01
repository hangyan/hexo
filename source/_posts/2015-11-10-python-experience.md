---
layout: post
title: "一点 python 的经验"
modified: 2015-11-10
tags: [language]

image:
    feature: abstract-12.jpg
summary: 可能已经过时了。
---

工作中总结的一些 python 的小经验，分享一下~

## 返回值
当 python 项目越来越庞大的时候,返回值的不一致就成为了一个越来越严重的问题。有的返
回 None,有的返回一个 Object，有的返回一个 dict,有的抛出异常....有些函数我们可以从名
字上猜出它可能的返回值，有的必须要有健全的文档才能厘清。不一致的返回类型导致上层
调用者必须维护各种各样的判断逻辑，还要考虑到以后可能的改动，确实是一件非常头疼的
事。

所以可以考虑对所有函数的返回值进行一个封装，对上层调用者提供一个统一的判断逻辑。
下面是一个示例:

```python
def fail(msg, err_code=error.ERR_DATA_NOT_EXIST):
    return {
        'suc': False,
        'errmsg': msg,
        'code': err_code
    }


def suc(data=None):
    return {
        'suc': True,
        'data': data
    }
```


函数调用者在拿到返回值后只需判断 `result['suc']`的真假即可知道函数执行的情况。对
于`fail`来说，有一个错误信息和一个错误码，即可用于日志信息显示，也可以判断错误类
型。对于`suc`来说,数据存在`data`字段中，而大多数情况下仅需根据函数名字即可判断出
`data`是否为`None`。

### 异常
使用异常还是错误码，一直都是见仁见智的问题。但是如果两者混用，就会带来很多额外的
麻烦。尤其是在使用第三方库的时候，有的使用异常，有的不抛出异常，如果直接使用，调
用代码会显得非常混乱。使用上面的`suc/fail`封装后，可以将所有的异常捕获并以错误码
的形式返回，上层代码会显得更加具有一致性,示例如下:

```python
def _do_req(self, module, action, params):
    try:
        service = Api(module, self.config)
        url = service.generateUrl(action, params)
        LOG.debug("API http request url. | url=%s", url)
        resp = service.call(action, params)
        if resp['code'] != 0:
            return fail(resp['message'], err_code=error.ERR_REMOTE_FAILED)
        return suc(resp)
    except Exception, e:
        return fail(e.message, err_code=error.ERR_NETWORK_EXCEPTION)
```

这段代码需要处理两种情况,一种是网络异常,通过`fail`将其转化为错误码和错误信息。另
一种是成功调用但是 response 里面也有错误码,也可以用`fail`自己进行处理。调用者只需
判断返回值中的`suc`字段即可知道该如何处理返回值。


### 日志
很多时候我们希望既能返回错误信息也能同时将其打印到日志中,最简单的方式如下:
```python
msg = "error message"
LOG.error(msg)
return fail(msg, error_code)
```

这样写有点累赘，可以将其写成一个单独的函数:
```python
def log_fail(logger, msg, err_code=error.ERR_DATA_NOT_EXIST):
    logger.error(msg)
    return fail(msg, err_code=err_code)
```

这样每次只用将`logger`实例和`msg`传给`log_fail`即可。

Update:
这种方式并不是很好，最终显示调用 logger 的位置是不对的，需要一点 hack 才能正确地显示打日志的位置:

```python
class CallerLog(logging.Logger):
    def _error(self, msg, fn, lno, func, *args, **kwargs):
        if self.isEnabledFor(logging.ERROR):
            self.__log(logging.ERROR, msg, fn, lno, func, args, **kwargs)

    def __log(self, level, msg, fn, lno, func, args, exc_info=None, extra=None):
        if exc_info:
            if not isinstance(exc_info, tuple):
                exc_info = sys.exc_info()
        record = self.makeRecord(self.name, level, fn, lno, msg, args, exc_info,
                                 func, extra)
        self.handle(record)

    def _trace(self, msg, *args, **kwargs):
        self.critical(msg, *args, **kwargs)

logging.setLoggerClass(CallerLog)

def log_fail(msg, code=ERR_DATA_NOT_EXIST):
    """Log as caller function and return fail.

    Common scenario is a function want to log and return the same error message.

    Args:
        msg(str): error message
        code(str): error code
    """
    caller = getframeinfo(stack()[1][0])
    module_name = caller.filename[len(settings.BASE_DIR) + 1:].replace('.py', '').replace('/', '.')
    log = logging.getLogger(module_name)
    log._error(msg, caller.filename, caller.lineno, caller.function)
    return fail(msg, code=code)

```



## 日志打印
日志基本上使我们平时调试线上问题最重要的工具了。良好的日志分级，格式能够帮助我们
很快地定位问题所在，但是不像 python 语言本身有很多规范去约束代码，日志打印经常会出
现各种各样的问题影响我们的查看,比如:

1. 数量不够。导致看不出代码的执行逻辑
2. 格式混乱。看起来费力
3. 日志级别错误。 有些人喜欢将自己觉得重要的信息打在`ERROR`里，这是非常不好的一
   个习惯。
4. 没有高亮。 日志通常没有高亮显示功能,看起来非常费力。

下面是一些改善的方式:

### 书写习惯
这个就跟 git 的`commit message`一样，怎么写都行，但一定要只维持一种习惯。比如首字
母要大写都大写，要小写都小写，混用的话眼睛看起来会比较累。其他如标点符号， 使用
词汇等都是非常细节的问题，但如果能保持一致，会让日志的可读性提高不少。

这是我之前写的一个简单地一些日志规范,可以参考:
[日志打印规范](https://github.com/alaudacloud/style-guides/blob/master/logs.md)

日志信息描述中比较重要的一点是描述信息和实际数据的格式。当然我们使用格式化字符串
将变量的值嵌入到描述语句中，我个人比较偏爱的一种方式是:

![ ][1]

这样统一将描述放在前面，将数据放在后面，可读性会更好一点。`golang`有一个日志库(logrus)就
支持此种方式的打印:

![ ][2]

`logrus`做的更好，能够将数据统一向右对齐。可惜 python 没有这样的库。


### 高亮支持
高亮支持一般是对于终端显示用的,写在文件里就不是很合适，因为有好多颜色控制字符。
有一种折衷的办法就是同时往两个文件打印日志，一个是普通的，一个是可以高亮的。这样
当使用`cat`或者`tail`这样的工具时可以查看带高亮的日志，其他的情况查看普通的日志。


python 有一个[colorlog](https://github.com/borntyping/python-colorlog)库支持带高
亮的日志，下面是一个示例的配置:

formatters:

```python
'formatters': {
    'standard': {
        'format': '%(asctime)s [%(levelname)s][%(threadName)s]' +
                  '[%(name)s.%(funcName)s():%(lineno)d] %(message)s'
    },
    'color': {
        '()': 'util.log.SplitColoredFormatter',
        'format': "%(asctime)s " +
                  "%(log_color)s%(bold)s[%(levelname)s]%(reset)s" +
                  "[%(threadName)s][%(name)s.%(funcName)s():%(lineno)d] " +
                  "%(blue)s%(message)s"
    }
},
```

handlers:
```python
    'color': {
        'level': 'DEBUG',
        'class': 'logging.handlers.RotatingFileHandler',
        'filename': LOG_PATH  + '.color.log',
        'maxBytes': 1024 * 1024 * 1024,
        'backupCount': 5,
        'formatter': 'color',
    },
```

其中`SplitColoredformatter`是我自己重写的一个`formatter`,用来支持我上面所说的描
述和数据分离的格式:

```python
class SplitColoredFormatter(ColoredFormatter):
    def format(self, record):
        """Format a message from a record object."""
        record = ColoredRecord(record)
        record.log_color = self.color(self.log_colors, record.levelname)

        # Set secondary log colors
        if self.secondary_log_colors:
            for name, log_colors in self.secondary_log_colors.items():
                color = self.color(log_colors, record.levelname)
                setattr(record, name + '_log_color', color)

        # Format the message
        if sys.version_info > (2, 7):
            message = super(ColoredFormatter, self).format(record)
        else:
            message = logging.Formatter.format(self, record)

        # Add a reset code to the end of the message
        # (if it wasn't explicitly added in format str)
        if self.reset and not message.endswith(escape_codes['reset']):
            message += escape_codes['reset']

        parts = message.split('|')
        if len(parts) != 1:
            desc = parts[0] + escape_codes['reset']
            data = escape_codes['green'] + parts[1]
            message = desc + '|' + data

        return message
```

最终出来的效果如下图所示:

![ ][3]


[1]: http://hangyan.github.io/images/posts/python/experience/log-var.png "log-var"
[2]: https://camo.githubusercontent.com/b9d0e424bfc6378e79b90de33b983ef5bae2f578/687474703a2f2f692e696d6775722e636f6d2f505937714d77642e706e67 'logrus'
[3]: http://hangyan.github.io/images/posts/python/experience/log-color.png "log-color"







