# Framework Overview

This document contains a high-level description of the different components
within the ReactiveCocoa framework, and an attempt to explain how they work
together and divide responsibilities. This is meant to be a starting point for
learning about new modules and finding more specific documentation.

For examples and help understanding how to use RAC, see the [README][] or
the [Design Guidelines][].

这篇文档是描述有关 ReactiveCocoa 框架中一些高水准的组件，并且阐释它们是如何相互组合一起工作和它们各自的责任。这意味着你需要找一些特别的资料来学习和研究这个新的编程模式。

更多资料，请看 [README][] 
和 [Design Guidelines][].

## Streams

A **stream**, represented by the [RACStream][] abstract class, is any series of
object values.

Values may be available immediately or in the future, but must be retrieved
sequentially. There is no way to retrieve the second value of a stream without
evaluating or waiting for the first value.

Streams are [monads][]. Among other things, this allows complex operations to be
built on a few basic primitives (`-bind:` in particular). [RACStream][] also
implements the equivalent of the [Monoid][] and [MonadZip][] typeclasses from
[Haskell][].

[RACStream][] isn't terribly useful on its own. Most streams are treated as
[signals](#signals) or [sequences](#sequences) instead.

## Streams
Stream， 它的抽象代表类是RACStream，是一些列的对象值。
它的值在通过顺序检查之后，可以立刻生效或者在未来的当你需要它的时候才生效。在值得传递过程中，你没有办法越级操作。在第一个值没有验证完成的时候，你是没有办法拿到第二个值的。

Streams 是一个单体。除此之外，它还可以通过一些基本的语句来进行一些复杂度很高的操作。RACStream 可以理解为等价于 Haskell 里面的Monoid 和 MonadZip。

RACStream 不是一个常用的东西，本身的使用意义也不很大，一般会以signals或者sequences等这些更高层次的表现形态代替。

## Signals

A **signal**, represented by the [RACSignal][] class, is a _push-driven_
[stream](#streams).

Signals generally represent data that will be delivered in the future. As work
is performed or data is received, values are _sent_ on the signal, which pushes
them out to any subscribers. Users must [subscribe](#subscription) to a signal
in order to access its values.

Signals send three different types of events to their subscribers:

 * The **next** event provides a new value from the stream. [RACStream][]
   methods only operate on events of this type. Unlike Cocoa collections, it is
   completely valid for a signal to include `nil`.
 * The **error** event indicates that an error occurred before the signal could
   finish. The event may include an `NSError` object that indicates what went
   wrong. Errors must be handled specially – they are not included in the
   stream's values.
 * The **completed** event indicates that the signal finished successfully, and
   that no more values will be added to the stream. Completion must be handled
   specially – it is not included in the stream of values.

The lifetime of a signal consists of any number of `next` events, followed by
one `error` or `completed` event (but not both).

## Signals
Signal,它的抽象代表类是RACSignal Class ，他是一个 push-driven Stream

Signals 一般来说就是代表着数据，用来在app中各种数据的传输。推动程序去执行函数还是只是接受数据，这些的值都是通过订阅了这个的Signals来触发的。开发者想访问signal的数据必须先去订阅它。

Signals 通过三种状态来告诉订阅者它的状态：

* Next 状态是从Stream 生成一个新的值。 （RACStream 方法这能在这个状态下操作。）

* Error 状态代表的是在这个Signal 出错了。（Error 状态包含着NSError 的对象，这样可以呈现给开发者看到底是哪里出错了。Error 不包含Stream's 的值，所以要特别的进行处理。）

* Completed 状态代表着这个Signal已经成功的完成了。

Signals 的整个生命周期可以理解成，有对个Next 发起一个Value，通过Error，和Completed来结束。（对于一个Signals 来说，Error和Completed 这能存在一个）。

  



### Subscription

A **subscriber** is anything that is waiting or capable of waiting for events
from a [signal](#signals). Within RAC, a subscriber is represented as any object
that conforms to the [RACSubscriber][] protocol.

A **subscription** is created through any call to
[-subscribeNext:error:completed:][RACSignal], or one of the corresponding
convenience methods. Technically, most [RACStream][] and
[RACSignal][RACSignal+Operations] operators create subscriptions as well, but
these intermediate subscriptions are usually an implementation detail.

Subscriptions [retain their signals][Memory Management], and are automatically
disposed of when the signal completes or errors. Subscriptions can also be
[disposed of manually](#disposables).

### Subscription
Subscriber 它的所有功能就是响应一个Signals事件。只要是符合RACSubscriber 协议的，它可以代表任何的对象。

它是由 [-subscribeNext:error:completed:][RACSignal] 或者其通过一个简单地函数就可以创建出来。从技术上讲，大多数 [RACStream][] 和
[RACSignal][RACSignal+Operations] 的操作附带的都会创建一个Subscriptions的，但创建之后会立马执行。

在操作的过程中，Subsciptios 会保持着它的Signals，同时当它的Signal 状态从Next 变成 Completes 或者 Error 这两个状态其中一个的时候，Subscriptions 也会自动的销毁。当然，你也可以手动的把它销毁。

### Subjects

A **subject**, represented by the [RACSubject][] class, is a [signal](#signals)
that can be manually controlled.

Subjects can be thought of as the "mutable" variant of a signal, much like
`NSMutableArray` is for `NSArray`. They are extremely useful for bridging
non-RAC code into the world of signals.

For example, instead of handling application logic in block callbacks, the
blocks can simply send events to a shared subject instead. The subject can then
be returned as a [RACSignal][], hiding the implementation detail of the
callbacks.

Some subjects offer additional behaviors as well. In particular,
[RACReplaySubject][] can be used to buffer events for future
[subscribers](#subscription), like when a network request finishes before
anything is ready to handle the result.

### Subjects
Subjects 的抽象类代表是RACSubject, 是一个可以手动操作的信号.

Subjects 你可以理解成一个可以变化的Signal，跟OC 里面的`NSMutableArray` 是可变的， `NSArray`是不可变得概念相似。是绝好的Signals TO non-RAC code的连接器。

举个例子来说：它可以处理应用程序逻辑块的回调，而不是简单地发送一个事件。它可以以一个RACSgnal 类型作为返回值来隐藏实现细节。

Subjects 还可以提供一些额外的行为。例如 [RACReplaySubject][] 可以为未来的 Subscripbers 开辟一个缓存区。就像在发送一个网络请求的时候，在完成整个请求之前，这里会返回请求情况。



### Commands

A **command**, represented by the [RACCommand][] class, creates and subscribes
to a signal in response to some action. This makes it easy to perform
side-effecting work as the user interacts with the app.

Usually the action triggering a command is UI-driven, like when a button is
clicked. Commands can also be automatically disabled based on a signal, and this
disabled state can be represented in a UI by disabling any controls associated
with the command.

On OS X, RAC adds a `rac_command` property to
[NSButton][NSButton+RACCommandSupport] for setting up these behaviors
automatically.

### Connections

A **connection**, represented by the [RACMulticastConnection][] class, is
a [subscription](#subscription) that is shared between any number of
subscribers.

[Signals](#signals) are _cold_ by default, meaning that they start doing work
_each_ time a new subscription is added. This behavior is usually desirable,
because it means that data will be freshly recalculated for each subscriber, but
it can be problematic if the signal has side effects or the work is expensive
(for example, sending a network request).

A connection is created through the `-publish` or `-multicast:` methods on
[RACSignal][RACSignal+Operations], and ensures that only one underlying
subscription is created, no matter how many times the connection is subscribed
to. Once connected, the connection's signal is said to be _hot_, and the
underlying subscription will remain active until _all_ subscriptions to the
connection are [disposed](#disposables).

## Sequences

A **sequence**, represented by the [RACSequence][] class, is a _pull-driven_
[stream](#streams).

Sequences are a kind of collection, similar in purpose to `NSArray`. Unlike
an array, the values in a sequence are evaluated _lazily_ (i.e., only when they
are needed) by default, potentially improving performance if only part of
a sequence is used. Just like Cocoa collections, sequences cannot contain `nil`.

Sequences are similar to [Clojure's sequences][seq] ([lazy-seq][] in particular), or
the [List][] type in [Haskell][].

RAC adds a `-rac_sequence` method to most of Cocoa's collection classes,
allowing them to be used as [RACSequences][RACSequence] instead.

## Disposables

The **[RACDisposable][]** class is used for cancellation and resource cleanup.

Disposables are most commonly used to unsubscribe from a [signal](#signals).
When a [subscription](#subscription) is disposed, the corresponding subscriber
will not receive _any_ further events from the signal. Additionally, any work
associated with the subscription (background processing, network requests, etc.)
will be cancelled, since the results are no longer needed.

For more information about cancellation, see the RAC [Design Guidelines][].

## Schedulers

A **scheduler**, represented by the [RACScheduler][] class, is a serial
execution queue for [signals](#signals) to perform work or deliver their results upon.

Schedulers are similar to Grand Central Dispatch queues, but schedulers support
cancellation (via [disposables](#disposables)), and always execute serially.
With the exception of the [+immediateScheduler][RACScheduler], schedulers do not
offer synchronous execution. This helps avoid deadlocks, and encourages the use
of [signal operators][RACSignal+Operations] instead of blocking work.

[RACScheduler][] is also somewhat similar to `NSOperationQueue`, but schedulers
do not allow tasks to be reordered or depend on one another.

## Value types

RAC offers a few miscellaneous classes for conveniently representing values in
a [stream](#streams):

 * **[RACTuple][]** is a small, constant-sized collection that can contain
   `nil` (represented by `RACTupleNil`). It is generally used to represent
   the combined values of multiple streams.
 * **[RACUnit][]** is a singleton "empty" value. It is used as a value in
   a stream for those times when more meaningful data doesn't exist.
 * **[RACEvent][]** represents any [signal event](#signals) as a single value.
   It is primarily used by the `-materialize` method of
   [RACSignal][RACSignal+Operations].

## Asynchronous Backtraces

Because RAC-based code often involves asynchronous work and queue-hopping, the
framework supports [capturing asynchronous backtraces][RACBacktrace] to make debugging
easier.

On OS X, backtraces can be automatically captured from any code, including
system libraries.

On iOS, only queue hops from within RAC and your project will be captured (but
the information is still valuable).

[Design Guidelines]: DesignGuidelines.md
[Haskell]: http://www.haskell.org
[lazy-seq]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/lazy-seq
[List]: http://www.haskell.org/ghc/docs/latest/html/libraries/base-4.6.0.1/Data-List.html
[Memory Management]: MemoryManagement.md
[monads]: http://en.wikipedia.org/wiki/Monad_(functional_programming)
[Monoid]: http://www.haskell.org/ghc/docs/latest/html/libraries/base-4.6.0.1/Data-Monoid.html#t:Monoid
[MonadZip]: http://www.haskell.org/ghc/docs/latest/html/libraries/base-4.6.0.1/Control-Monad-Zip.html#t:MonadZip
[NSButton+RACCommandSupport]: ../ReactiveCocoaFramework/ReactiveCocoa/NSButton+RACCommandSupport.h
[RACBacktrace]: ../ReactiveCocoaFramework/ReactiveCocoa/RACBacktrace.h
[RACCommand]: ../ReactiveCocoaFramework/ReactiveCocoa/RACCommand.h
[RACDisposable]: ../ReactiveCocoaFramework/ReactiveCocoa/RACDisposable.h
[RACEvent]: ../ReactiveCocoaFramework/ReactiveCocoa/RACEvent.h
[RACMulticastConnection]: ../ReactiveCocoaFramework/ReactiveCocoa/RACMulticastConnection.h
[RACReplaySubject]: ../ReactiveCocoaFramework/ReactiveCocoa/RACReplaySubject.h
[RACScheduler]: ../ReactiveCocoaFramework/ReactiveCocoa/RACScheduler.h
[RACSequence]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSequence.h
[RACSignal]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSignal.h
[RACSignal+Operations]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSignal+Operations.h
[RACStream]: ../ReactiveCocoaFramework/ReactiveCocoa/RACStream.h
[RACSubject]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSubject.h
[RACSubscriber]: ../ReactiveCocoaFramework/ReactiveCocoa/RACSubscriber.h
[RACTuple]: ../ReactiveCocoaFramework/ReactiveCocoa/RACTuple.h
[RACUnit]: ../ReactiveCocoaFramework/ReactiveCocoa/RACUnit.h
[README]: ../README.md
[seq]: http://clojure.org/sequences
