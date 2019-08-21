gRPC Objective-C Bazel Build Support
----
* Author: tonyzhehaolu
* Approver: mxyan
* Status: In Review
* Implemented in: Bazel and Starlark
* Last updated: Jul 26, 2019
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/p6Z8kfc1koQ

## Abstract

Proposes a set of Bazel rules for building iOS applications with gRPC Objective-C library.


## Background

The gRPC Objective-C library so far only supports installation via Cocoapods. Requests for Bazel support are continually raised.

In addition to the available native rules (`objc_library` and `proto_library`), `objc_grpc_library` (with the same name and similar usage as the one in Google3) needs to be created because the native `objc_proto_library` is actually [not usable](https://github.com/bazelbuild/bazel/issues/7348). Some other rules are also needed in order to compile the actual library from `.proto` files.

### Related Proposals
`objc_proto_grpc_library` and `objc_grpc_library` are built upon the implementation of the `generate_cc` rule defined in [generate_cc.bzl](https://github.com/grpc/grpc/blob/bazel_test/bazel/generate_cc.bzl). There doesn't seem to be a proposal for that, though.


## Proposal

For now, assume that the `WORKSPACE` root is the gRPC repository.

### Dependency Graph

According to the dependencies in an iOS application that uses gRPC Objc library (shown below), we created two `objc_library` targets in the `src/objective-c` package within the `com_github_grpc_grpc` workspace: `grpc_objc_client` and `proto_objc_rpc`.

![gRPC Objective-C Library Dependency](L56_graphics/dependency.png)

* Target `//src/objective-c:grpc_objc_client` compiles all the files in `src/objective-c/GRPCClient/`, dependent on `//:grpc` which compiles core gRPC and `//src/objective-c:rx_library` which compiles `src/objective-c/RxLibrary`. It is publicly visible so that any application-specific Objective-C code can depend on it. It is only necessary when the app does not use protocol buffers (which means the service stub libraries, plus all that they are dependent on, are not included) - it is a rare case.
* `//src/objective-c:proto_objc_rpc` does the `src/objective-c/ProtoRPC/` directory. It is also made publicly visible so that the generated service stubs can be compiled depending on this rule. Users do not need to manually add this label to `deps`, though.
* The Objective-C stubs are generated *and compiled* into native Bazel `objc_library` targets via `objc_proto_grpc_library` and `objc_grpc_library`. Details about these two custom rules are discussed in the upcoming sections.
* Although "app-specific resources" depend on multiple libraries in the graph, users only need to add the `objc_proto_grpc_library` and (or) `objc_grpc_library` targets they defined. This is because the dependency on gRPC-protoRPC and gRPC-ObjC-client are carried along by those two rules.
* All the necessary external dependencies are loaded with `grpc_deps()` in `//bazel:grpc_deps.bzl` and are hidden from users.

### Rules for Compiling `.proto` Files

The gist of these custom rules is to run protobuf compiler and Objective-C plugin on provided `.proto` files with `ctx.action.run`. Those executables are available from `@com_google_protobuf//:protoc` and `@com_github_grpc_grpc//:grpc_objective_c_plugin`.

We use the native `proto_library` rule as a manager for `.proto` files (i.e. their package paths and dependencies). They wrap the `.proto` files and are passed into `objc_proto_grpc_library` and `objc_grpc_library` as `deps`.

`objc_grpc_library`, takes in as `deps` a list of `proto_library` targets and geneates the message stubs (excluding the service ones) for these targets and all their transitively dependent protos. In addition, it takes in a list of labels of `.proto` files as `srcs`. If the `.proto` file is in the same package, users can use its relative path. The list of `.proto` files should all contain service stubs; otherwise Bazel will complain about certain `.pbrpc.{h,m}` files not being generated. `objc_grpc_library`, generates services stubs for a `.proto` file if and only if the `.proto` file is listed in `srcs`.

As a result of compiling every `.proto` files in the dependency chain, the app-specific code only needs to depend on the one or the few `objc_grpc_library`'s at the bottom of the dependency graph.

For `objc_grpc_library`, it is also possible to tell that [well known protos](https://github.com/protocolbuffers/protobuf/tree/master/src/google/protobuf) are required in the dependency, by passing `True` for the field `use_well_known_protos`.

In terms of the execution of `protoc`, the command is similar to that in a [podspec](https://github.com/grpc/grpc/blob/0803c79411597f58eae0b12b4eb272c506b8cdbb/examples/objective-c/helloworld/HelloWorld.podspec) (all `.proto` targets in `deps`, including their transitive dependencies, are provided as inputs, and their directory from the `WORKSPACE` root are added to `-I` flags programmatically). The output directory is set to `//bazel-out/<*CPU architecture*>/bin/<package name>/_generated_protos/` so that bazel is able to locate the generated files.

Generated files in the directory above follow the same hierarchy as the `.proto` files. Consider this project where we are building from package `//A`:

![hierarchy Example](L56_graphics/hierarchy1.png)

The resulting structure in `bin` will be:

![hierarchy Result Example](L56_graphics/hierarchy2.png)

Lastly, three other targets are created to split the files into `hdrs`, `srcs` (potentially empty, since there might be no service stubs), and `non_arc_srcs`. They will be fed into a `objc_library` rule.

When `#import`-ing `.proto` and `.pb*.{h,m}` files, always use their *absolute paths* from the `WORKSPACE` root.


### Compatibility with Google3

In order to smoothen the transition to Google3, we defined a function in `//bazel/grpc_build_system.bzl` that is loaded in `src/objective-c/BUILD` - `grpc_objc_library`. In the open-source version, this function solely passes the attributes to a `native.objc_library`. In Google3, however, the implementation of this function is different; it passes a subset of the attributes to `native.objc_library` and injects some dependencies and generates other targets according to circumstances.

In Google3, `//src/objective-c:grpc_objc_client` does not depend directly on `//:grpc`. Instead, it depends on a different `//:grpc_objc` target. Switching it to depend directly on `//:grpc` causes duplicated symbols so it's not practical. Therefore, in `//bazel:grpc_build_system.bzl`'s `grpc_generate_one_off_target()`, I generated an alias from `//:grpc_objc` to `//:grpc`.


### Example

Consider the hierarchy from the above section and further suppose that `world.proto` imports both `hello.proto` and `grpc.proto`. Also suppose that `grpc.proto` and `world.proto` have service stubs defined which we would like to use.

Note that the correct import statements in `world.proto` should be:
```
#import "A/protos/library/hello.proto"
#import "B/D/grpc.proto"
```

Configure `WORKSPACE` as shown below. Load `grpc_deps` for binding external git repositories such as `@com_google_protobuf` and other iOS-related dependencies:
```
# The choice of name here is significant, because some bzl scripts are directly dependent on the name @com_github_grpc_grpc
git_repository(
    name = "com_github_grpc_grpc",
    remote = "https://github.com/grpc/grpc.git",
    branch = "master"
)

load("@com_github_grpc_grpc//bazel:grpc_deps.bzl", "grpc_deps")

grpc_deps()

load("@build_bazel_rules_apple//apple:repositories.bzl", "apple_rules_dependencies")
apple_rules_dependencies()

load("@build_bazel_apple_support//lib:repositories.bzl", "apple_support_dependencies")
apple_support_dependencies()
```

The `BUILD` file for this sample project can be written similar to the following snippet. Assume that this is the `BUILD` file for package `//A` and there is a `proto_library` target defined in package `//B` for `grpc.proto`, named `grpc_proto`.
```
load("@build_bazel_rules_apple//apple:ios.bzl", "ios_application")
load("@com_github_grpc_grpc//bazel:grpc_objc_library.bzl", "objc_grpc_library")

proto_library(
    name = "world_proto",
    srcs = ["protos/world.proto"],
    deps = [
        ":hello_proto",
        "//B:grpc_proto"
    ]
)
proto_library(
    name = "hello_proto",
    srcs = ["protos/library/hello.proto"]
)
objc_grpc_library(
    name = "world_grpc_objc",
    srcs = [
        "protos/world.proto",
        "//B/D:grpc.proto", # since we need the service stubs from these two files
    ],
    deps = [":world_proto"]
)

# app-specific library below
objc_library(
    name = "exampleObjCLibrary",
    ...
    deps = [":world_grpc_objc"]
)
ios_application(
    ...
    deps = [":exampleObjCLibrary"]
)
```
Again, import the geneated stubs in the app-specific source files as:
```
#import "A/proto/Hello.pbrpc.h"
#import "B/D/Grpc.pbrpc.h"
```

## Migrating Tests and Examples to Bazel

With Bazel basically up and running, some of the unit tests of Objective-C library are being migrated to Bazel for shorter test durations. The migration is already completed to the greated extent as for the current stage. Updated runner scripts are available in `src/objective-c/tests`.

Different from the tests in `UnitTests`, other existing tests utilizes the property of an abstract base class and inheritance. To elaborate on that, we had defined a base class for `InteropTests` and `MacTests`, and other test classes that inherit the base class while implementing different setups, thereby invoking the same set of test methods under various circumstances. The base classes are not meant to be executed. With Xcode, previously, we just needed to disable the tests in the base class. With Bazel, however, there is currently no such feature.

In order to prevent the test cases from the base class being executed, the `defaultTestSuite` property is overriden. The property returns an empty test suite if it sees the test instance is exactly the base class; otherwise, it returns the default test suite, which is all the tests being inherited. For example:

In `InteropTests.h`:
```
@property(class, readonly) XCTestSuite *defaultTestSuite;
```
In `InteropTests.m`:
```
+ (XCTestSuite *)defaultTestSuite {
  if (self == [InteropTests class]) {
    return [XCTestSuite testSuiteWithName:@"InteropTestsEmptySuite"];
  } else {
    return super.defaultTestSuite;
  }
}
```

### Test Target - `grpc_objc_client_internal_testing`

Source files in `internal_testing` are meant to be used for logging patch data of each gRPC call, in order to provide some metrics in the test environment. In addition to that, there are a few lines in the source code that is disabled in the production environment - `GRPCOpBatchLog` and its refereneces. These lines are enabled only during testing as well.

With Cocoapods, it is allowed to "inject" preprocessor definitions to any targets by modifying `post_install` in a Podfile. In contrast, due to the nature of Bazel, preprocessor definitions can only be passed down the dependency chain. There is no way to define preprocessors (unless from the command line for the whole project) for the targets that the current target depends on. Therefore, we created a target - `grpc_objc_client_internal_testing` that recompiled the entire library again with `GRPC_TEST_OBJC=1`. 

`grpc_objc_client_internal_testing` includes all the source files previously in `grpc_objc_client` and `proto_objc_rpc`, along with `internal_testing/*`.

### Local Version of `objc_grpc_library`

The `objc_grpc_library` defined in `//bazel/objc_grpc_library.bzl` for external use creates duplicate symbol problem when used with local source files. That is because `objc_grpc_library` specifies the dependency to the gRPC library as an external repository which will create another identical set of static libraries in `bazel-out`. Therefore, a new rule `local_objc_grpc_library` is defined in `//src/objective-c:grpc_objc_internal_library.bzl`. Instead of `@com_github_grpc_grpc//src/objective-c:grpc_objc_client` and `proto_objc_rpc`, it depends on `//src/objective-c:grpc_objc_client_internal_testing`.

Other than that, it works identically as `objc_grpc_library`.

### Shared Library and Wrapper Rules

For convenience and future imports to Google3, we defined another wrapper rule - `grpc_objc_testing_library` - for `src/objective-c/tests` package only. We created a target called `TestConfigs` which is basically an `objc_library` rule that contains shared headers, common preprocessor definitions, and the certificate bundle. Each test target is created with the wrapper rule which can append `TestConfigs` to every target. Meaningless repetitions of adding the shared configuration to `deps` is avoided, in consequence.

`proto_library_objc_wrapper` is a temporary workaround for importing the test targets to Google3. Its open-source version does nothing other than passing the arguments to `native.proto_library`.

## Implementation

The implementation is done by tonyzhehaolu.


## Open Issues

For the time being, `objc_grpc_library` is unable to detect if a label in `srcs` crosses package boundaries. Namely, if a the `grpc.proto` (as in the example above) is referred to as `//B:D/grpc.proto` instead of `//B/D:grpc.proto`, it is still accepted.

`tvos_unit_test` is not ready for use, so are `tvos_application` and `watchos_application`. Related issue: [here](https://github.com/bazelbuild/rules_apple/issues/523).

By Aug 20, the `objc_proto_library` was already removed from Bazel as a native rule. It will probably be remove officially in 0.29. Therefore, we will need to split `objc_grpc_library` into two in the near future in order to stick with the convension in Google3. This will not be discussed here as it's related to how the rules are implemented in Google3.