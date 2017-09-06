A user friendly C++ gRPC API using Reactive Streams
----
* Author(s): Per Grön (pgron@google.com)
* Approver: a11r
* Status: Draft
* Implemented in: C++ (but could be extended to more languages)
* Last updated: 2017-09-06
* Discussion at: <google group thread> (filled after thread exists)


## Abstract

In an attempt to define an API for gRPC that is both efficient, simple to use and that covers most of gRPC's feature set, this document proposes a new C++ API for gRPC based on (a C++ interpretation of) [Reactive Streams](http://www.reactive-streams.org/).

For more information about Reactive Streams, see for example [a 2 minute introduction to Rx](https://medium.com/@andrestaltz/2-minute-introduction-to-rx-24c8ca793877) or [this one hour video presentation about RxJava](https://www.youtube.com/watch?v=_t06LRX0DV0) (one of the most widely used implementations of Reactive Streams).


## Background

gRPC currently offers two versions of its C++ API:

1. An easy to use but relatively slow synchronous version.
2. A fast but not very user friendly asynchronous version.

The async API is powerful and a good foundation, but it is too low level to be suitable for use as is by application code: It forces application code to worry about manual memory management and designing its own threading model. [This still unfinished PR that attempts to document how to use async bidirectional streaming](https://github.com/grpc/grpc/pull/8934) is an indication that the API simply is difficult to use today.

If a company would adopt C++ gRPC and use it across several teams, the author believes they would have to either build a framework that entirely wraps gRPC's API, or they would end up with code that is very inconsistent between services/teams.

This RFC hopes to make C++ gRPC easier to adopt by offering an API results in applications that are fast and that have simple code at the same time.


### Related Proposals: 

* This RFC is closely related to [#7352: Provide a simple event-processing loop for C++ async API](https://github.com/grpc/grpc/issues/7352).


## Proposal

This document proposes a new C++ API for gRPC, in addition to the current synchronous and asynchronous APIs. It is based on rather than replaces the current async API. If the C++ version is successful, this proposal could be extended to cover more languages in the future.

The proposal has two main parts: The Reactive Streams library and the new gRPC API.


### A C++ Reactive Streams library

[Reactive Streams](http://www.reactive-streams.org/) is currently defined only for Java and Javascript. This project needs a C++ version.

In preparation for this RPC the author has written [a Reactive Streams library in C++ called rs](https://github.com/per-gron/shuriken/tree/master/src/rs). rs is well documented, well tested and implements many of the [ReactiveX operators](http://reactivex.io/documentation/operators.html). Reading rs' documentation may help explain parts of this RFC.

The examples below use the syntax of this library.


### The new gRPC API

This section explains what the API could look like by showing examples. The code may be a bit confusing until you start thinking [in terms of plumbing pipes of data](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754). It's rather similar to [Google Cloud Dataflow](https://cloud.google.com/dataflow/model/programming-model)/Flume actually. Operator names are stolen from ReactiveX, which means that their [excellent](http://reactivex.io/documentation/operators/map.html) [documentation](http://rxmarbles.com/) also applies.

When reading these examples, consider what fully production ready code using the current async API would look like (with error handling, cancellation of potentially expensive upstream calls on call cancellation, respecting backpressure when appropriate, written in a uniform style etc).


#### Basic unary RPC

```proto
service TestService {
  rpc Double (TestRequest) returns (TestResponse) {}
}

message TestRequest {
  int32 data = 1;
}

message TestResponse {
  int32 data = 1;
}
```

```cpp
class TestServiceImpl : public TestService::ReactiveService {
 public:
  AnyPublisher<TestResponse> Double(TestRequest request) override {
    int answer = request.data() * 2;
    return AnyPublisher<TestResponse>(
        Just(MakeTestResponse(answer)));
  }
};
```

*Notes:*

* A Publisher is the main entity in Reactive Streams (it is called *Observable* in Rx). It is similar to a future/[promise](https://scotch.io/tutorials/javascript-promises-for-dummies), but it can emit more than one value.
* Publisher in the rs library is a [*concept*](http://en.cppreference.com/w/cpp/concept) - much like STL's "forward iterator" for example - not a concrete type. There are many different Publisher types.
* [`AnyPublisher<TestResponse>`](https://github.com/per-gron/shuriken/blob/master/src/rs/doc/reference.md#anypublisher) is a concrete Publisher type that can contain any Publisher that emits values of the type `TestResponse`. (A little bit like `boost::any` or `any_iterator`)
* `Just` takes a value and returns a Publisher that emits just that value.


#### Making requests to another service

```proto
service UpstreamService {
  rpc GetUserInfo (UserInfoRequest) returns (UserInfoResponse) {}
  rpc ResolveUsername (ResolveUsernameRequest) returns (ResolveUsernameResponse) {}
}

message UserInfoRequest {
  string userid = 1;
}

message UserInfoResponse {
  string username = 1;
  string country = 2;
}

message ResolveUsernameRequest {
  string username = 1;
}

message ResolveUsernameResponse {
  string userid = 1;
}

service TestService {
  rpc CountryByUserid (CountryByUseridRequest) returns (CountryByUseridResponse) {}
  rpc CountryByUsername (CountryByUsernameRequest) returns (CountryByUsernameResponse) {}
}

message CountryByUseridRequest {
  string userid = 1;
}

message CountryByUseridResponse {
  string country = 1;
}

message CountryByUsernameRequest {
  string username = 1;
}

message CountryByUsernameResponse {
  string country = 1;
}
```

```cpp
UserInfoRequest MakeUserInfoRequest(const std::string& userid) {
  UserInfoRequest request;
  request.set_userid(userid);
  return request;
}

class TestServiceImpl : public TestService::ReactiveService {
 public:
  /// Request handler that makes one upstream request
  AnyPublisher<CountryByUserIdResponse> CountryByUseridHandler(
      CountryByUseridRequest request) override {
    return AnyPublisher<CountryByUserIdResponse>(Pipe(
        // Make RPC call to get userinfo
        upstream_service_.GetUserInfo(
            MakeUserInfoRequest(request.userid())),
        // Convert the result to a response of the proper type
        Map([](UserInfoResponse response) {
          CountryByUseridResponse final_response;
          final_response.set_country(response.country());
          return final_response;
        })));
  }

  /// Request handler that makes two upstream requests in a row
  AnyPublisher<CountryByUsernameResponse> CountryByUsernameHandler(
      CountryByUsernameRequest request) override {
    ResolveUsernameRequest resolve_username_request;
    resolve_username_request.set_username(request.username());

    return AnyPublisher<CountryByUsernameResponse>(Pipe(
        // First, resolve the username to a user id
        upstream_service_.ResolveUsername(resolve_username_request),
        // Then resolve the user id to userinfo
        ConcatMap([this](ResolveUsernameResponse response) {
          return upstream_service_.GetUserInfo(
              MakeUserInfoRequest(response.userid()));
        }),
        // Then take the resulting userinfo and construct a response
        Map([](UserInfoResponse response) {
          CountryByUsernameResponse final_response;
          final_response.set_country(response.country());
          return final_response;
        })));
  }

 private:
  // This would have to be initialized of course. Omitted here.
  UpstreamService::ReactiveStub upstream_service_;
};
```

*Notes:*

* `Pipe` is a function in rs that lets you "pipe" a Publisher through a series of operators. The first argument is a Publisher/stream, subsequent arguments are operators on that stream.
* `Pipe(a, b, c)` is essentially the same as `c(b(a))`. Having `Pipe` like this isn't necessary. Another option is to let the operators be methods on a base Publisher type like RxJava does, but then it's impossible for user code to add their own first-class operators.
* `ConcatMap` is similar to `Map` but it lets the mapper function return a Publisher instead of a value directly. I would personally call it `FlatMap` but I've stuck to ReactiveX's naming (there is a `FlatMap` in ReactiveX, but it behaves in an unintuitive way and its implementation requires an unbounded buffer).


#### Client streaming: Sum

```proto
service TestService {
  rpc Sum (stream SumRequest) returns (SumResponse) {}
}

message SumRequest {
  int32 data = 1;
}

message SumResponse {
  int32 data = 1;
}
```

```cpp
class TestServiceImpl : public TestService::ReactiveService {
 public:
  AnyPublisher<SumResponse> Sum(AnyPublisher<SumRequest> requests) override {
    return AnyPublisher<SumResponse>(Pipe(
        requests,
        // First, convert the incoming stream of SumRequests to a stream of ints
        Map([](SumRequest request) { return request.data(); }),
        // Then compute the sum
        Sum(),
        // Finally, convert the sum (which is an int) to a SumResponse proto
        Map([](int sum) {
          SumResponse response;
          response.set_data(sum);
          return response;
        }));
  }
};
```

*Notes:*

* Because this is a client streaming RPC, the parameter to `TestServiceImpl::Sum` is an `AnyPublisher<SumRequest>` instead of a `SumRequest` directly. This is the only difference between non-streaming and client streaming RPCs for application code.
* `Sum()` as used here is a rather silly example, but it is possible for users of the API to write their own operators: [Here is the implementation of `Sum()`](https://github.com/per-gron/shuriken/blob/master/src/rs/include/rs/sum.h). Writing another operator is as easy as writing `Sum()`.


#### Server streaming: FizzBuzz

```proto
service TestService {
  rpc FizzBuzz (FizzBuzzRequest) returns (stream FizzBuzzResponse) {}
}

message FizzBuzzRequest {
  int32 upto = 1;
}

message FizzBuzzResponse {
  string data = 1;
}
```

```cpp
class TestServiceImpl : public TestService::ReactiveService {
 public:
  AnyPublisher<FizzBuzzResponse> FizzBuzz(FizzBuzzRequest request) override {
    return AnyPublisher<FizzBuzzResponse>(Pipe(
        Range(1, request.upto()),
        Map(&FizzBuzz),
        Map(&MakeFizzBuzzResponse));
  }

 private:
  static std::string FizzBuzz(int num) {
    if ((num % 3) == 0 && (num % 5) == 0) {
      return "FizzBuzz";
    } else if ((num % 3) == 0) {
      return "Fizz";
    } else if ((num % 5) == 0) {
      return "Buzz";
    } else {
      return std::to_string(num);
    }
  }

  static FizzBuzzResponse MakeFizzBuzzResponse(const std::string& data) {
    FizzBuzzResponse response;
    response.set_data(data);
    return response;
  }
};
```

*Notes:*

* There are no unbounded buffers nor any blocking I/O here; it is perfectly fine for a slow client to request a couple of billion responses.
* It is possible to split any pipeline of Publisher operations into separate functions or even separate RPCs. Backpressure, cancellation and error handling still work transparently.
* Splitting the plumbing code from logic like this can help make code more readable and testable.


#### More

More examples are possible, but this is already getting long. Please ask for more if you want to see what a particular use case would look like. Omitted examples include:

* **Bidirectional streaming:** It is just client and server streaming combined. The handler takes a Publisher and returns a Publisher that can emit more than one element.
* **Making two upstream calls in parallel:** Can be done with the [`Zip` operator](https://github.com/per-gron/shuriken/blob/master/src/rs/doc/reference.md#zippublisher).
* **Cancellation:** Most of the time no explicit code related to cancellation is needed. For example, if an RPC has made two upstream requests in parallel and one fails, the other is automatically cancelled. This works with timeouts and streams, making two calls and using just the fastest result ([`Merge()`](https://github.com/per-gron/shuriken/blob/master/src/rs/doc/reference.md#mergepublisher)+[`Take(1)`](https://github.com/per-gron/shuriken/blob/master/src/rs/doc/reference.md#takecount)) etc as well.
* **Error handling:** With errors, Publishers behave very much like futures/promises do in most languages/implementations. It is basically an async version of `try`/`catch`. Errors are automatically propagated through the chain of operators until they are caught, usually with the `Catch` operator.


#### Threading model

The default threading model for a server that uses this library is:

* The library spawns one thread/`CompletionQueue` per CPU core.
* When a request arrives, it is pegged to the `CompletionQueue` that it arrived at: If the request handler makes upstream requests, the responses are handled on the same `CompletionQueue`.
* If a request handler does heavy work, it is responsible for doing that somewhere else than on the threads created by the library; in order to be able to jump back the library provides a `Schedule` function that asynchronously executes a callback on the original `CompletionQueue` thread.

This way, an application will be able to utilize multiple threads, yet no synchronization primitives are needed in the rs library, and they are needed in application code only if it has to do slow work or if requests need to interact.


## Rationale

Alternate approaches for a C++ for gRPC include

* **The current synchronous API:** Too slow for many use cases.
* **The current asynchronous API:** Too low level for many use cases.
* **Futures, as suggested in [issue #7352](https://github.com/grpc/grpc/issues/7352):** Futures typically do not have a concept of cancellation and they do not map naturally to streams.
* **Callbacks, as suggested in [issue #7352](https://github.com/grpc/grpc/issues/7352):** Callbacks are rather low level; [Many node.js programmers would agree](http://callbackhell.com/) that it is not the nicest style of API. It is also tricky to avoid a `Send` method that never blocks and never eats all available memory.
* **Synchronous API with fibers/lightweight threads:** This mitigates some of the performance issues of the synchronous API but it does not naturally allow the user of the API to do several things at once.

The API proposed in this document has the following positive traits:

* Encourages simple application code, even for applications that are not easy.
* Fully asynchronous; efficient.
* Supports not just unary RPCs but also streaming.
* Supports transitive cancellation with little to no application code support.
* Respects backpressure: When working with streams, there are no unbounded buffers, blocking or dreop or that can block or drop data.

This API especially shines in applications that combine many async operations in complex ways:

* Techniques such as optimistic retries for long tail latency mitigation are easy to express with Reactive Streams.
* Application code that needs to do many upstream calls, some in parallel and others in sequence, some being streamed and some not, are helped greatly by Reactive Streams, because it allows the developer to not write the code otherwise required to manage the state that deals with orchestrating all of these calls.
* Reactive Streams offers a clear and composable approach to writing asynchronous functions within a server as well: An asynchronous function is one that returns a Publisher. This can greatly help API consistency within a given programming language, not just on the wire protocol level, as well as making it easy to on a type level distinguish asynchronous code from synchronous code.
* A concrete example of how code can benefit from Reactive Streams is [Netflix's Hystrix library](https://github.com/Netflix/Hystrix/wiki/How-it-Works), which helps provide latency and fault tolerance for microservices.

The main drawbacks of this proposal are:

* It is easy to use only after you have spent some time learning Reactive Streams' concepts: You must change mindset to thinking about data flows.
* There are features in gRPC that don't obviously fit Reactive Streams, for example metadata.
* It is a large change: Broad adoption will require a lot of education and a substantial change of mindset for programmers that use it. Netflix has adopted Rx at a large scale and their engineers report that it was worth it for them, but it required a fairly large educational effort.
* This proposal is based on the assumption that the current async C++ gRPC API is too low level to be user friendly for typical application development. If people do not agree with this, it is pointless.


### Why Reactive Streams?

Although Reactive Streams fits gRPC remarkably well, it can be difficult to grasp. What the proposed API achieves could be done without Reactive Streams. Why use Reactive Streams and not something tailored to gRPC?

The reason is that Reactive Streams is a well understood paradigm with a lot of momentum:

It is being used extensively by [Netflix](https://medium.com/netflix-techblog/reactive-programming-in-the-netflix-api-with-rxjava-7811c3a1496a), [Akka](http://doc.akka.io/docs/akka/2.5.4/scala/stream/stream-introduction.html), Github and many other companies. The upcoming version 5 of the widely used Spring framework [is based on Reactive Streams](https://spring.io/blog/2016/07/28/reactive-programming-with-spring-5-0-m1). A precursor to Reactive Streams is in [LINQ in C#](https://msdn.microsoft.com/en-us/library/hh242977(v=vs.103).aspx). Reactive Streams [is being standardized in Java 9](https://community.oracle.com/docs/DOC-1006738). With [ReactiveX](http://reactivex.io/languages.html) this programming model is available in lots of programming languages (albeit not always with backpressure support).

There is a lot of material on the Internet that covers Reactive Streams from many different angles. [ReactiveX](http://reactivex.io/) has lots of good documentation and [Marble diagrams](http://rxmarbles.com/) are a wonderful way to describe what Rx operators do.


## Implementation

A prototype of what is suggested in this RFC can be found [here](https://github.com/per-gron/shuriken/tree/master/src/rs-grpc). It implements the most important features, but it is still quite incomplete.

Implementation of this RFC would involve:

* Prepare the [rs](https://github.com/per-gron/shuriken/tree/master/src/rs) library so that it can be accepted as gRPC code:
  * Backport to C++11.
  * Remove usage of `std::exception_ptr` and make it possible to use `grpc::Status` instead.
* Improve the prototype library so that it can be used for experimental purposes (fix memory leaks etc).
* Write a protobuf code generator to be able to remove the most expensive C++ templates from the prototype and to make the API nicer.
* Implement missing features in the prototype: Finish cancellation support, add support for timeouts, metadata etc.
* Polish API and test.
* Evangelize.

The author could do much of this work, but it would be awesome to get help.


## Open issues

* The RPC handler methods might need to take one or more additional parameters to handle gRPC features like metadata.
* This proposal glosses over many details; the API needs to be fleshed out more.
* There are certain details in the current gRPC C++ API that seem to be missing: For example, the author could not find how get a signal in the server when a unary RPCs is cancelled.
* The rs library is a fairly minimal implementation of Reactive Streams. One thing that many Rx libraries have that rs does not is a concept for a Publisher that emits only one value. This would increase the complexity of the library a bit, but it could be worth it for the additional type safety and performance benefits it would bring.
* [rs](https://github.com/per-gron/shuriken/tree/master/src/rs) breaks Google's code style in the following ways. I am unsure about how exactly to fix this.
  * **It uses rvalue references:** These could mostly be changed to by-value instead, but I have a hard time justifying the performance penalty.
* It is not yet decided where the code for rs should be: Should it be a part of gRPC or a separate library?
