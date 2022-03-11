Setuptools entrypoint for generating file
----
* Author(s): Patrick Lannigan
* Approver: lidizheng and gnossen
* Status: Draft
* Implemented in: Python
* Last updated: 2022-02-18
* Discussion at: https://groups.google.com/g/grpc-io/c/H617sSzoV-8

## Abstract

Implement a `setuptools` entrypoint for generating files when a Python distribution is being built.
This type of integration would be called automatically by `setuptools` and would not require and
explicit call by the project to generate the files. The entrypoint would be configured using an
established python project configuration file and would only require the `grpcio-tools` package to be
listed as a build time dependency.

## Background

Currently `grpc_tools` [provides a command][grpc_command] for integrating with `setuptools`. However,
this requires custom integration on the part of the package (example:
[grpc_health_checking][grpc_command_example_integration] and
[explicitly calling the command][grpc_command_example_call] when building the wheel).

With the introduction of [PEP 517][pep_517] and [PEP 518][pep_518], directly calling `setup.py` will
start becoming less common. Additionally, the use of custom commands will not work with projects that
use this mechanism and solely rely on [setup.cfg][declarative_config].

Custom commands are not the only integration point that is available. `setuptools` also supports
entrypoints that allows for packages to act as a plugin and provide custom behavior. This is
[documented with examples][setuptools_entrypoint] of generating files based on revision control
information. This could be used to generate source files based on `.proto` files.

[Original Feature Request][original_feature_request]

## Proposal

Add an entrypoint to `grpcio-tools` that would register as a `file_finer` hook with `setuptools`. To
simplify the implementation without introducing an additional code path, the entrypoint will target
the same `main()` function used by the `grpc_tools.protoc` CLI. A [tool table][tool_table] within the
file `pyproject.toml` will be used to accept configuration parameters. This will add a dependency on
tomli for now, but this will [not be needed][pep_680] starting in Python 3.11.

In order to prevent unintended behavior, the first thing the functionality will do is check the
configuration file to see if the section has been configured. If the file does not exist or the section
is not configured, no additional steps will be taken. `setuptools` will be provided with an empty
list to indicate that there are no additional files to include in the distribution.

## Rationale

A downside of this proposal is that it will introduce yet another way to get the Python modules that
correspond to the proto files.

The primary way generate the Python files is using the
[`grpc_tools.protoc` command line interface][protoc_cli]. The advantage of this mechanism of this
method is also its disadvantage. It is a completely standalone way to generate the source code for
the python modules. While projects are allowed to decide how to work this generated files in the way
that best suits the project, there is no integration point. This means the project **must** decide
how and when this command is executed, likely needing to write a script to execute this CLI.

For projects that use `setuptools` to publish a package, `grpcio-tools` includes a
[custom command][grpc_command]. However, it also requires a specific call like the `grpc_tools.protoc`
CLI and is not compatible with installing from a source distribution for projects that declare
`setuptools` as build system that should be used. Additionally, one of the top result for "grpc
setuptools command" is [this post][setuptools_gen_question], which suggests a custom command that
doesn't rely on the one provided by `grpcio-tools`. This suggest that there is a desire for functionality
that doesn't require a specific all and that the existing functionality is under-documented.

gRPC also provides a way to get the Python [classes at runtime][runtime_classes]
([Proposal][runtime_proposal]). However, it is still [marked as experimental][runtime_api] almost 2
years later and requires `grpcio-tools` to be available at runtime. Similar to the current `setuptools`
integration, it is not mentioned in the basic tutorial documentation.

## Implementation

- [ ] Proof of Concept with hard coded values.
- [ ] Configuration file parsing & mapping to CLI arguments.

Could be done later:

- [ ] Migrate existing `grpc*` packages to use this functionality.

I am planning work on it. I will be able to spend a few hours of each work week on this effort.

## Open issues (if applicable)

- Looking at the implementation of the [custom command][grpc_command], it looks as though only a
  sub-set of the CLI arguments supported by `grpc_tools.protoc` need to be supported. I believe this
  will become more clear after starting on the implementation. However, there could always be an
  "additional arguments" type parameter to cover this if a project has a specific need. This would
  also allow for CLI arguments to be passed to any gRPC plugins that might be used by a project.

[grpc_command]: https://github.com/grpc/grpc/blob/05e17e92390d4685f1418f535604a201a7f8e1a3/tools/distrib/python/grpcio_tools/grpc_tools/command.py#L50
[grpc_command_example_integration]: https://github.com/grpc/grpc/blob/2d4f3c56001cd1e1f85734b2f7c5ce5f2797c38a/src/python/grpcio_health_checking/health_commands.py#L48
[grpc_command_example_call]: https://github.com/grpc/grpc/blob/2d4f3c56001cd1e1f85734b2f7c5ce5f2797c38a/tools/run_tests/artifacts/build_artifact_python.sh#L196-L197
[pep_517]: https://www.python.org/dev/peps/pep-0517/
[pep_518]: https://www.python.org/dev/peps/pep-0518/
[declarative_config]: https://setuptools.pypa.io/en/latest/userguide/declarative_config.html
[setuptools_entrypoint]: https://setuptools.pypa.io/en/latest/userguide/extension.html#adding-support-for-revision-control-systems
[original_feature_request]: https://github.com/grpc/grpc/issues/28662
[tool_table]: https://www.python.org/dev/peps/pep-0518/#tool-table
[pep_680]: https://www.python.org/dev/peps/pep-0680/
[protoc_cli]: https://grpc.io/docs/languages/python/basics/#generating-client-and-server-code
[setuptools_gen_question]: https://stackoverflow.com/q/52994857
[runtime_classes]: https://github.com/grpc/grpc/blob/a72c8ebb7def13a317a1afc7c08455388d1fa2e4/src/python/grpcio/grpc/_runtime_protos.py#L1
[runtime_proposal]: ./L64-python-runtime-proto-parsing.md
[runtime_api]: https://grpc.github.io/grpc/python/grpc.html#runtime-protobuf-parsing