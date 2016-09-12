# ES-Study
首先从https://github.com/elastic/elasticsearch上下载源码，我下载的是版本号为2.2的。
源码下载之后迫不及待的想知道程序的入口在哪里，那我们是不是可以通过启动脚本入口。然后进入es的目录/bin/找到es的启动脚本elasticsearch.sh。
在脚本的最后我们发现org.elasticsearch.bootstrap.Elasticsearch有这行代码，就推测这是ES的入口方法，然后我们代开下载好的源码进行查看。
发现Elasticsearch这个类很简单，在main函数中调用Bootstrap.init(args)。那我们接下来查看Bootstrap这个类相应的方法。
