# client-go under the hood

[client-go](https://github.com/kubernetes/client-go/)包含了一系列可用的构件为我们开发我们自己的controller,这些构件定义在[tools/cache folder](https://github.com/kubernetes/client-go/tree/master/tools/cache)

下图展示了这些构件如何与我们自己开发的controller在代码层面的交互

<p align="center">
  <img src="images/client-go-controller-interaction.jpeg" height="600" width="700"/>
</p>

## client-go components

* Reflector: 定义在[type *Reflector* inside package *cache*](https://github.com/kubernetes/client-go/blob/master/tools/cache/reflector.go),
通过Kubernetes API监控特定资源,资源可以为kubernetes内建资源也可以是自定义资源.
当reflector收到来自API的资源更新或创建通知,它将通过相应的API创建一个对象,并将它推入Delta Fifo队列.

* Informer: 定义在[base controller inside package *cache*](https://github.com/kubernetes/client-go/blob/master/tools/cache/controller.go),
它从Delta Fifo队列弹出一个对象保存下来,同时调用我们的controller并传递该对象.

* Indexer: 定义在[type *Indexer* inside package *cache*](https://github.com/kubernetes/client-go/blob/master/tools/cache/index.go).
为对象提供索引功能,一个典型的使用场景是基于对象的labels创建索引.
Indexer维护索引通过一系列的索引函数,并使用一个线程安全的池子保存对象和它们的key.


## Custom Controller components

* Informer reference: Informer实例引用,
需要我们在我们的controller代码中自己创建.

* Indexer reference: Indexer实例引用,
需要我们在我们的controller代码中自己创建,我们在执行处理过程中通过它来取回对象.

在client-go中提供了*NewIndexerInformer*函数来创建Informer和Indexer.
在我们代码中可以[直接调用](https://github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go#L174)该函数或者使用[工厂方法](https://github.com/kubernetes/sample-controller/blob/master/main.go#L61)

* Resource Event Handlers: 一系列调函数,通过它们Informer传递对象给我们的controller.
一个典型的模式为该回调函数获得对象的key后将对象的key推入workqueue为接下来的处理.

* Work queue: 解耦对象传递和对象处理过程的队列.

* Process Item: 对象具体的处理过程.
其中一般会使用Indexer reference或者Listing包装函数通过对象key取回对象.