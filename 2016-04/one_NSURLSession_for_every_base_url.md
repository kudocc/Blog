# Use one NSURLSession for every base url

我们项目中使用了AFNetworking，然后做了一层封装，对于每个请求都新建了一个`AFHTTPSessionManager`，我第一次意识到这个有问题是发现了内存泄漏，因为其内部组合了`NSURLSession`，而`NSURLSession`的delegate是retain的，而这个delegate就是`AFHTTPSessionManager`实例，所以`AFHTTPSessionManager`是有循环引用的，这并不是一个很大的问题，因为一般应用会使用单例，I hope so.

还有一个问题是今早我想到的。我们工程基本上就是一个base url，照理说可以使用HTTP的persistent connections特性的，不过我们工程中每次请求都创建了一个新的NSURLSession对象，我有点怀疑是不是每次发送请求都会重新建立tcp么？我就尝试着抓了一下包，果然，每次发送请求都要进行tcp的三次握手，浪费流量、延迟又高。于是我修改代码，使每次请求都用一个`NSURLSession`，再次抓包，发现之后的请求不会重新建立tcp连接了。Done！

所以对于同一个域名发送的请求，要尽量使用一个NSURLSession实例，HTTP默认使用头`Connection: keep-alive`，这样就可以尽量使多个请求使用同一条tcp连接，保证了低延迟。不过，即使这样也未必会保证两个请求使用同一条tcp连接，这也依赖于服务端http服务器的设置，就我所知，apache可以设置keep-alive的超时时间和同一条连接最多可以被多少个请求使用。在服务端response headers中可能会看到：`Keep-Alive: timeout=15, max=100`。这里我参考了[这个](http://stackoverflow.com/questions/19155201/http-keep-alive-timeout)和[这个](http://stackoverflow.com/questions/12095533/proper-use-of-keepalive-in-apache-htaccess)。由于自己对服务端不十分了解，可能有错误的地方，忘指正。
