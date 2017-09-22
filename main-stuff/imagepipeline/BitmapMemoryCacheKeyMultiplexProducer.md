## BitmapMemoryCacheKeyMultiplexProducer
### &#8195;MultiplexProducer
&#8195;可能有多个DraweeView试图获得同一来源的图像，那么如果为相同的每个请求都尝试进行全部的获取工作，那么将会产生极大的性能和内存浪费，所以，需要对实际上是”同一请求“的不同图片请求进行合并。多个同一请求内容的请求实际上会合并为一次请求，并且只有当所有同类图片请求都取消时，本次合并请求才会取消，若当前没有新的结果返回时，所有同类请求都会取得最新的一份结果返回。   
&#8195;BitmapMemoryCacheKeyMultiplexProducer继承自MultiplexProducer，其业务方法produceResults()也由父类MultiplexProducer实现。   
&#8195;下面看下MultiplexProducer中图片请求合并的实现：
```
  public void produceResults(Consumer<T> consumer, ProducerContext context) {
    K key = getKey(context);
    Multiplexer multiplexer;
    boolean createdNewMultiplexer;

    do {
      createdNewMultiplexer = false;
      synchronized (this) {
        multiplexer = getExistingMultiplexer(key);
        if (multiplexer == null) {
          multiplexer = createAndPutNewMultiplexer(key);
          createdNewMultiplexer = true;
        }
      }
    } while (!multiplexer.addNewConsumer(consumer, context));

    if (createdNewMultiplexer) {
      multiplexer.startNextProducerIfHasAttachedConsumers();
    }
  }
```
&#8195;getKey()由MultiplexProducer的具体实现类实现，负责根据ProducerContext获取缓存键(CacheKey)，同一请求内容的请求应该具有相同的缓存键，并在MultiplexProducer所维护的HashMap中查询该缓存键对应的Multiplexer是否存在，如果存在，则说明已经有同类请求正在获取结果，新到来的这个请求只需要向Multiplexer注册。如果不存在，则这是一个新的图片请求，则需要创建一个MultiPlexer放入HashMap中，那么这个MultiPlexer就应该是合并的请求了，并且，这个合并请求应当维护所有对应的同类请求，当有结果返回或请求状态变化时通知所有对应的同类请求，并将请求任务交由后续流水段执行。   
![](https://github.com/icemoonlol/fresco-research-stuff/blob/master/main-stuff/resources/img/MultiplexerStructure.png)
&#8195;值得注意的是，这个Multiplexer的HashMap为MultiplexProducer的"this"锁所保护，不会出现创建多个同一CacheKey的Multiplexer的情况，并且该锁仅用来保护这个HashMap，而对于MultiPlexer的操作将由MultiPlexer的"this"锁保护，这里对锁的责任范围控制一方面细化了锁的粒度，提高了MultiplexProducer的并行性，另一方面，多个消费线程向MultiPlexer注册的过程如果受MultiplexProducer锁控制的话，那么就难以及时获取最终结果。
###&#8195;Multiplexer
&#8195;接下来看MultiPlexer是如何设计和工作的：
##### 1. addNewConsumer()：
(1).合并请求状态变化   
&#8195;创建Consumer与ProducerContext的Pair对，当在添加新的Consumer前发生了MultiPlexer的remove，那么将返回失败，否则将该Pair对保存到MultiPlexer的mConsumerContextPairs(ArraySet)中，并根据并集的原则更新当前发起下一段工作的ProducerContext的状态，即合并请求拥有最高的优先级，并且若有任何一个请求不希望进行预取则不会进行预取、若有任何一个图像请求希望收到中间结果就应当接受中间结果返回。
```
      final Pair<Consumer<T>, ProducerContext> consumerContextPair =
          Pair.create(consumer, producerContext);
      T lastIntermediateResult;
      final List<ProducerContextCallbacks> prefetchCallbacks;
      final List<ProducerContextCallbacks> priorityCallbacks;
      final List<ProducerContextCallbacks> intermediateResultsCallbacks;
      final float lastProgress;

      synchronized (Multiplexer.this) {
        if (getExistingMultiplexer(mKey) != this) {
          return false;
        }
        mConsumerContextPairs.add(consumerContextPair);
        prefetchCallbacks = updateIsPrefetch();
        priorityCallbacks = updatePriority();
        intermediateResultsCallbacks = updateIsIntermediateResultExpected();
        lastIntermediateResult = mLastIntermediateResult;
        lastProgress = mLastProgress;
      }
```
&#8195;全局预取状态变化、优先级状态和是否接收中间结果的状态变化变化会触发合并请求的状态变化回调。
```
      BaseProducerContext.callOnIsPrefetchChanged(prefetchCallbacks);
      BaseProducerContext.callOnPriorityChanged(priorityCallbacks);
      BaseProducerContext.callOnIsIntermediateResultExpectedChanged(intermediateResultsCallbacks);
```
(2).中间结果的直接获取   
&#8195;如果上次的中间结果没有发生变化，那么将直接返回已获取的中间结果(这里的同步逻辑为：可能有新的结果返回，那么前面已经将Consumer注册到MultiPlexer中了，新的结果返回可能已经触发了Consumer的回调返回了最新的结果，那么就不应该进行重复的工作)
```
synchronized (consumerContextPair) {
        // check if last result changed in the mean time. In such case we should not propagate it
        synchronized (Multiplexer.this) {
          if (lastIntermediateResult != mLastIntermediateResult) {
            lastIntermediateResult = null;
          } else if (lastIntermediateResult != null) {
            lastIntermediateResult = cloneOrNull(lastIntermediateResult);
          }
        }

        if (lastIntermediateResult != null) {
          if (lastProgress > 0) {
            consumer.onProgressUpdate(lastProgress);
          }
          consumer.onNewResult(lastIntermediateResult, false);
          closeSafely(lastIntermediateResult);
        }
      }
```   
(3).注册本次请求状态变化的回调   
```
    addCallbacks(consumerContextPair, producerContext);
```
##### 2. 注册当前请求状态变化回调
&#8195;用户可能通过ProducerContext对图片请求进行控制，如果因为Multiplexer中某个请求的取消或状态变化，就必须及时更新合并请求的状态。这类就是为到来的请求注册这样的回调以进行对应处理。   
(1).onCancellationRequested()
```
	  //...
      synchronized (Multiplexer.this) {
        pairWasRemoved = mConsumerContextPairs.remove(consumerContextPair);
        if (pairWasRemoved) {
          if (mConsumerContextPairs.isEmpty()) {
            contextToCancel = mMultiplexProducerContext;
          } else {
            isPrefetchCallbacks = updateIsPrefetch();
            priorityCallbacks = updatePriority();
            isIntermediateResultExpectedCallbacks = updateIsIntermediateResultExpected();
          }
        }
      }
      BaseProducerContext.callOnIsPrefetchChanged(isPrefetchCallbacks);
      BaseProducerContext.callOnPriorityChanged(priorityCallbacks);
      BaseProducerContext.callOnIsIntermediateResultExpectedChanged(
          isIntermediateResultExpectedCallbacks);
      if (contextToCancel != null) {
        contextToCancel.cancel();
      }
      if (pairWasRemoved) {
        consumerContextPair.first.onCancellation();
      }
```
&#8195;某个图片请求的取消首先要从mConsumerContextPairs移除对应的Pair对，并重新计算合并请求的状态，并触发合并请求状态变化的回调。此外，当这是最后一个请求时，将取消合并请求。    
(2).对于预取状态变化、中间结果接收状态变化和优先级状态变化，只是简单地重新计算各个状态。
```
        @Override
        public void onIsPrefetchChanged() {
          BaseProducerContext.callOnIsPrefetchChanged(updateIsPrefetch());
        }

        @Override
        public void onIsIntermediateResultExpectedChanged() {
          BaseProducerContext.callOnIsIntermediateResultExpectedChanged(
              updateIsIntermediateResultExpected());
        }

        @Override
        public void onPriorityChanged() {
          BaseProducerContext.callOnPriorityChanged(updatePriority());
        }
```


##### 3. startNextProducerIfHasAttachedConsumers()：   
(1).任务取消的特殊情况：
```
        Preconditions.checkArgument(mMultiplexProducerContext == null);
        Preconditions.checkArgument(mForwardingConsumer == null);

        // Cleanup if all consumers have been cancelled before this method was called
        if (mConsumerContextPairs.isEmpty()) {
          removeMultiplexer(mKey, this);
          return;
        }
```
可能在发起下一段工作之前，所有请求可能通过ProducerContext取消对图片的获取，那么合并请求也就没有必要继续了，将从HashMap中移除MultiPlexer并返回。   
(2).合并请求的创建：
```
        ProducerContext producerContext = mConsumerContextPairs.iterator().next().second;
        mMultiplexProducerContext = new BaseProducerContext(
            producerContext.getImageRequest(),
            producerContext.getId(),
            producerContext.getListener(),
            producerContext.getCallerContext(),
            producerContext.getLowestPermittedRequestLevel(),
            computeIsPrefetch(),
            computeIsIntermediateResultExpected(),
            computePriority());

        mForwardingConsumer = new ForwardingConsumer();
        multiplexProducerContext = mMultiplexProducerContext;
        forwardingConsumer = mForwardingConsumer;
      }
```
&#8195;取出MultiPlexer中第一个到来的请求所对应的ProducerContext用来构造后续工作的ProducerContext，设置合并请求的预取、中间结果、优先级状态，并构造一个用于转发获取结果和状态的ForwardingConsumer一并交由下一段工作。
##### 4. 新结果和状态的转发：
(注意内部类对象在访问外部类成员时使用的是<OuterClass>.this.<field>)
```
private class ForwardingConsumer extends BaseConsumer<T> {
      @Override
      protected void onNewResultImpl(T newResult, boolean isLast) {
        Multiplexer.this.onNextResult(this, newResult, isLast);
      }
      @Override
      protected void onFailureImpl(Throwable t) {
        Multiplexer.this.onFailure(this, t);
      }
      @Override
      protected void onCancellationImpl() {
        Multiplexer.this.onCancelled(this);
      }
      @Override
      protected void onProgressUpdateImpl(float progress) {
        Multiplexer.this.onProgressUpdate(this, progress);
      }
    }
```
##### 5. onNextResult()：新结果到来的转发
&#8195;(1).如果有新结果到来，关闭上次中介结果的引用并清除其占用的内存。注意由于涉及到对HashMap的操作，所以需要加以同步控制。
注意，在开始会检查ForwardingConsumer是否为上一次合并请求的ForwardingConsumer，即MultiPlexer只会接收最后一次发起的后续段工作返回的结果。
```
      synchronized (Multiplexer.this) {
        // check for late callbacks
        if (mForwardingConsumer != consumer) {
          return;
        }
        closeSafely(mLastIntermediateResult);
        mLastIntermediateResult = null;
```
&#8195;(2).遍历MultiPlexer中所有的Consumer进行结果地分发，并且当其为最终结果时，将从清空并从HashMap中移除该MultiPlexer，也就是说，++**MultiPlexer或者MultiplexerProducer提供的仅仅是多线程环境下图片请求的请求合并和结果分发，而并非一个持久型缓存用来存放获取结果，注意与BitmapCache的功能区分**++。
```
        iterator = mConsumerContextPairs.iterator();
        if (!isFinal) {
          mLastIntermediateResult = cloneOrNull(closeableObject);
        } else {
          mConsumerContextPairs.clear();
          removeMultiplexer(mKey, this);
        }
      }
      while (iterator.hasNext()) {
        Pair<Consumer<T>, ProducerContext> pair = iterator.next();
        synchronized (pair) {
          pair.first.onNewResult(closeableObject, isFinal);
        }
      }
    }
```
##### 6. onCancelled()：合并请求被取消
```
    public void onCancelled(final ForwardingConsumer forwardingConsumer) {
      synchronized (Multiplexer.this) {
        // check for late callbacks
        if (mForwardingConsumer != forwardingConsumer) {
          return;
        }
        mForwardingConsumer = null;
        mMultiplexProducerContext = null;
        closeSafely(mLastIntermediateResult);
        mLastIntermediateResult = null;
      }
      startNextProducerIfHasAttachedConsumers();
    }
```
onCancelled()的处理较为特殊，由于后续任务是以MultiPlexer中第一个到来的请求发起的，前面分析到，当后续任务取消后，仍然有其他请求在等待返回结果，那么就必须重新发起后续任务。


