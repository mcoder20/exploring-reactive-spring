## Reactive Spring - 

#### Main question, how do we handle more users/ how do we handle more api requests - 
  - reactive programming is a way to make more efficient use of threads .
	
	* could we do better with our existing nodes?? say in a microservices placed behind load balancer and scaled out but could we do better with our existing nodes?
		* how to handle more request per node? where are we being bottlenecked and why???
			* though it depends on different things in different techstack but for JVM developers the answer is threads
			* lot of things we do in apps today deal with IO so even if app goes out of thread the cpu usage is still low, the traditional impl with IO streams does not lend itself well
			to this since they are blocking and the thread is sitting idle on waiting state doing nothing so that is what we are trying to solve with reactive programming model
			a way to say let me know once this IO task is done instead of the main request thread waiting for it to complete.
	
	* question is if we have had these concepts of NIO since java 1.4 then why aren't we using it as a default if we already know most of the time JVM is idle waiting for some IO response - 
		* people have tendencies to fall for the fallacies/misbelief of distributed computing like - 
			- misconceptions like assuming data in n/w is infinitely fast and it will never fail, assuming next byte is right around the corner, once the data is flowing in then we will get all data and nothing is going to stop it from coming.
		* using java nio is not easy or natural
		* we dont work at that level, we don't write code working with IO streams or NIO directly. we work in terms of higher abstraction
	
	* no. of organisations have tried to create api that helps us describe asynchronous computational processing/async multithreaded processing.
	* in 2015 few major players got together and decided to extract out for simple interfaces that are part of reactive streams specifications that describe
	async unbounded streams of data.(reactive-streams.org)
		* reactive stream spec provides us foundational types that we can depend upon like common currency that can use to describe asynchronous streams of data
		potentially unbounded latent streams of data.
			* benefit of reactive stream types is that they give us a way to say "hey i have got a thing that's gonna produce some data and i want to subscribe to
			that data when it is avaialable (async by default)"
			* since we have registered the intent to recieve the data when it is avaialable we dont have to wait for that data and someone else can use that thread.
			* but that is all we get from reactive streams there is no plumbing involved on how we can process streams of data ,that is all up to us ( we generally
			dont work at this level so some toolkit can be used for this purposes like in spring we can use project reactor. )
		
		* web sockets in spring 4.0 was built over project reactor
		* project reactor is a toolkit that allows us to work with reactive stream types and allows us to process data coming from this reactive streams.
		* spring reactive web brings in Spring WebFlux and not MVC, webflux is a new reactive web runtime that allows us to build reactive web services in spring ecosystem.
		
#### Basics of Reactive -  
  * The 4 main types reactive types are - 
    - Publisher - it broadcasts data to subscribers 
    - Processor - its a bridge.(a publisher and a subscriber)
    - Subscriber - it consumes the data. (in onNext method). When subscriber first subscribes to a publisher a new subscription is created. 
    - Subscription - this is the thing that we use to request more data from the publisher. i.e. it drives the rate of consumption. so publisher cannot overwhelm 
    subscriber with data since we are managing how much data we want. (this flow control of taking as many as we can and cancelling if we cannot consume more is called backpressure)
    
  * we don't directly implement these interfaces and in order to do seemless use of rate limiting or other things on streams of data, for this we can use tools like project reactor,
  it builds on top of reactive stream specification and provides 2 other types .
    * **Flux** - it is a thing that produces zero or more values. (its ultimately Publisher but supports high level operations such 
    as map or flatmap or filter etc.)
    * **Mono** - a thing that produces at most one value. It is like a CompletableFuture.
    
  * mongodb has a concept of `tailable query` if db is started in replicated mode. This is particularly useful when working with reactive application.
    *  since mongo can be distributed db they prefer UUID for id field over monotonically incrementing numbers.
    
    
  * **Transactions in reactive world** - 
    * since we are no more staying on the same thread during a transaction we cannot rely on the traditional approach of 
    ThreadLocal(map of key value stored on a current thread, with key being currentthread name and value being any value) that spring and everything until now used.
    
    * so we need some other way to propagate that state associated with  current ongoing transaction (and we cannot rely on threadlocal now with reactive)
    so reactive team created `Context`. its like a dictionary that gets propagated for the entirety of the pipeline.
    *  anywhere in the pipeline we can get access to the context say using `doOnEach(signal -> signal.getContext()`
    * spring uses this Context heavily in security while propagating the Authentication principal, in distributed tracing,
    transaction propagation (we can also use it for MDC)
    
  * **Http service** -
    * in spring reactive REST api can be designed in 2 ways - MVC style and Functional reactive style 
    * apart from creating normal REST api that we were also building with spring MVC , ***what reactive spring helps us in cases we 
    are at a risk of monopolizing a thread*** . e.g. whereever we want to keep a socket open(using traditional java i/o)
      * instead of returning one since value from a REST api, suppose we want to get access to latest data as quickly as possible.
      * this is a common scenario and a common implementation for such a case is say we might have something that uses a server socket that just keeps that socket open
      and we keep reading from that socket all the time (that way we dont have to connect again, or latency of setting up network call with acking, connecting, teardown etc.).
      * i.e. in above approach the socket remains open and a thread is assigned to that socket so that we can process new updates as soon as they arrive instead of repolling or reinvoking REST api.
      *  use cases for such kind of requirement or style - chat messaging, stock ticker(high speed low latency trading), status/presence notification
      * to do this in Http we can build a ServerSendEvent endpoint, this endpoint will give us a infinite stream of values.
        * `Server-Sent event` is a protocol that is based on Http, but when the client knows about the special content type it does not disconnect and keep pointing to that socket live.
    * Spring uses Scheduler behind the scenes for doing everything in reactive which is essentially a threadpool and Clock.
      * the threadpool by default has threads equal to number of cores and they are all individual event loop.
      * if there is any blocking call that hurts reactive api heaving hence in that case blocking code should be moved to different scheduler.
      * use project blockinghound to verify there is no blocking code and use `Schedulers.elastic()` if there is blocking IO that cannot be avoided . 