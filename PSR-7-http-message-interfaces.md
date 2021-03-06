HTTP消息接口
=====================

本文档描述了在[RFC 7230][]和[RFC 7231][]中被描述来代表HTTP消息通用接口，以及在[RFC 3986][]中规定的URIs语法。

[RFC 7230]:http://tools.ietf.org/html/rfc7230
[RFC 7231]:http://tools.ietf.org/html/rfc7231
[RFC 3986]:http://tools.ietf.org/html/rfc3986
[RFC 2119]: http://www.ietf.org/rfc/rfc2119.txt

HTTP消息是Web开发的基础。Web浏览器和HTTP客户端之间，例如使用cURL创建一个发送到Web服务器的HTTP请求，Web服务器会返回一个HTTP响应。服务器端代码收到一个HTTP请求消息，并返回一个HTTP响应消息。

HTTP消息通常对与最终的消费者用户来说是抽象的，但作为开发者，我们通常需要知道它们是如何构建以及如何访问和操纵它们，以执行我们的任务，是否可能被创建为到HTTP API的请求或处理传入请求。

每一个HTTP请求消息有一个具体形式：
```php
POST /path HTTP/1.1
Host: example.com

foo=bar&baz=bat
```
一个请求的第一行是“请求行”，按照顺序包含了HTTP请求方法，请求的目标地址（通常是一个绝对URI或者服务器上的路径）以及HTTP协议的版本。接着是一个或多个HTTP头，一个空行，以及消息主体。

HTTP响应信息具有类似的结构：
```php
HTTP/1.1 200 OK
Content-Type: text/plain

This is the response body
```
第一行是“状态行”，按照顺序，依次包含了HTTP协议的版本，HTTP状态码，以及一个“原因分析”，也就是一个对人类友好可读的状态码的描述。像请求消息一样，随后一个或多个HTTP头，一个空行，以及消息主体。

本文档中描述的接口都是围绕HTTP消息的抽象和构成它们的元素。

关键词 “必须”("MUST")、“一定不可/一定不能”("MUST NOT")、“需要”("REQUIRED")、
“将会”("SHALL")、“不会”("SHALL NOT")、“应该”("SHOULD")、“不该”("SHOULD NOT")、
“推荐”("RECOMMENDED")、“可以”("MAY")和”可选“("OPTIONAL")的详细描述可参见 [RFC 2119][] 。

### 参考

- [RFC 2119][]
- [RFC 3986][]
- [RFC 7230][]
- [RFC 7231][]


1. 规范
--------

### 1.1. 消息

HTTP消息是从客户机到服务器的请求或从服务器到客户端的响应。本规范分别为其定义了对于HTTP消息的接口 `Psr\Http\Message\RequestInterface` 和 `Psr\Http\Message\ResponseInterface` 。

`Psr\Http\Message\RequestInterface` 和     `Psr\Http\Message\ResponseInterface` 都继承自 `Psr\Http\Message\MessageInterface` 。而 `Psr\Http\Message\MessageInterface` **可以** 被直接实现，实现者 **应该** 实现 `Psr\Http\Message\RequestInterface` 和     `Psr\Http\Message\ResponseInterface` 。

从这里开始，之后的描述中命名空间 `Psr\Http\Message` 在提到这个接口的时候将会被省略。

### 1.2. HTTP头

#### 不区分大小写的头字段名

HTTP消息包括大小写不敏感的头字段名。头是从实现了 `MessageInterface ` 接口的类中以不区分大小写的方式获取的。例如，获取 `foo` 头与获取 `FoO` 头的返回结果是相同的。同样，设置 `Foo` 头将覆写之前的设置的 `foo` 头的值。

```php
$message = $message->withHeader('foo', 'bar');

echo $message->getHeaderLine('foo');
// Outputs: bar

echo $message->getHeaderLine('FOO');
// Outputs: bar

$message = $message->withHeader('fOO', 'baz');
echo $message->getHeaderLine('foo');
// Outputs: baz
```
尽管头能被不区分大小写地获取，但原有的大小写规范 **必须** 被保留，尤其是使用 `getHeaders()` 函数来获取头的时候。

不符合要求的HTTP应用程序可能依赖于一定的大小写规范，所以对于一个用户能够在创建一个请求或响应时控制HTTP报头的大小写的情况下是非常有用的。

#### 带有多个值的报头

为了能够容纳具有多个值且依然能够方便地以字符串形式传输的报头， `MessageInterface` 接口的实例能够以数组或字符串的形式来获取报头。使用 `getHeaderLine()` 方法来获取特定名称的报头的值，其形式为不区分大小写并用逗号连接的字符串，包含了所有的报头值。使用 `getHeader()` 来获取特定名称的所有报头值，其以数组形式返回结果，且不区分大小写。

```php
$message = $message
    ->withHeader('foo', 'bar')
    ->withAddedHeader('foo', 'baz');

$header = $message->getHeaderLine('foo');
// $header contains: 'bar, baz'

$header = $message->getHeader('foo');
// ['bar', 'baz']
```
注意：并非所有的报头值都可以用逗号连接起来（例如： `Set-Cookie` ）。当需要获取此类报头值时，基于 `MessageInterface` 的消费者类 **应该** 使用 `getHeader()` 方法来获取这类多值型报头。

#### 主机头

在请求中，主机头通常反映了URI的主机组成部分，以及建立TCP连接时使用的主机。然而HTTP规范允许以上两种主机头之间是不相同的。

在构造头的过程中，如果没有提供主机头的情况下，实现者 **必须** 尝试从提供的URI设置主机头。

默认情况下， `RequestInterface::withUri()` 方法将使用传递给 `UriInterface` 接口的与主机组件匹配的主机头来替换返回的请求中主机头。

你也可以通关传递第二个参数值（ `$preserveHost` ）为 `true` 来保留原有的主机头状态。当这个参数设置为 `true` 时，返回的请求将不会更新返回的消息的主机头 - 除非该消息不包含主机头。

下面的表格表明了将 `$preserveHost` 参数设置为 `true` 的情况下 `withUri()` 方法返回的请求中使用 `getHeaderLine('Host')` 方法在多个初始请求和URL中返回的结果。

|请求头1  | 请求主机组件2  | URI主机组件3 |     结果   |
|:-----: | :----------: | :--------:  | :--------:|
|''      |  ''          | ''          | ''        |
|''      |  foo.com     | ''          | foo.com   | 
|''      |  foo.com     | bar.com     | foo.com   | 
|foo.com |  ''          | bar.com	  | foo.com   | 
|foo.com |  foo.com     | baz.com     | foo.com   | 

1. `Host` 头值始终在其余操作之前；
2. URI的主机组件在操作请求之前组成；
3. URI的主机组件通过 `withUri()` 来注入。

### 1.3 流

HTTP消息有一个启动线，头和消息主体组成。消息主题可以很小也能很大。如果试图以字符串的形式来表达消息体，将会很轻易就消耗比预期更多的内存，因为消息体式必须被全部存储在内存中的。（ Attempting to store the body of a request or response in memory would preclude the use of that implementation from being able to work with large message bodies.）*** 试图在内存中存储请求或者响应的消息体能够排除与很大的消息体之间良好工作的实现。 *** 

