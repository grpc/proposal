L117: C-core: Replace gpr Logging with absl Logging
----

* Author(s): @tanvi-jagtap
* Approver: @markdroth , @ctiller
* Status: Final
* Implemented in: gRPC C++
* Last updated: 2024-05-07.
* Discussion at: https://groups.google.com/g/grpc-io/c/Jg7bvHqAyCU

## Abstract

Replacing gpr logging and asserts with Abseil logging and asserts

## Background

gRPC is currently maintaining its own version of the logging and assert APIs. The Abseil library released their logs and asserts too. We want to leverage the Abseil library and avoid maintaining our own logging code. Also, the current implementation of gpr_log uses format strings which are not type safe and have exploit potential. Moving to absl logging will eliminate this security risk.

### References

* [gpr logging header](https://github.com/grpc/grpc/blame/83a17ff4684dc1fb3493a151ac0b655b1c55e766/include/grpc/support/log.h)
* [absl logging header](https://github.com/abseil/abseil-cpp/blob/master/absl/log/log.h)
* [absl assert header](https://github.com/abseil/abseil-cpp/blob/master/absl/log/check.h)

## Related Proposals:

None.

## Proposal

We are proposing to remove all instances of gpr logging and asserts and replace them with their absl equivalents.

### Functions that will be removed, and their replacements
* `gpr_log`
	* `gpr_log(GPR_DEBUG,...)` with `VLOG(2) << ...`
	* `gpr_log(GPR_INFO,...)` with `LOG(INFO) << ...`
	* `gpr_log(GPR_ERROR,...)` with `LOG(ERROR) << ...`
* `gpr_log_message`
	* `gpr_log_message(GPR_DEBUG,...)` with `VLOG(2) << ...`
	* `gpr_log_message(GPR_INFO,...)` with `LOG(INFO) << ...`
	* `gpr_log_message(GPR_ERROR,...)` with `LOG(ERROR) << ...`
* Asserts
	* `GPR_ASSERT(...)` with `CHECK(...)`
	* `GPR_DEBUG_ASSERT(...)` with `DCHECK(...)`
* `gpr_assertion_failed` with `CHECK(...)`
* `gpr_set_log_function` - This function will be removed. Teams can consider usage of [LogSink](https://github.com/abseil/abseil-cpp/blob/fa57bfc573453d57a38552eedcce894b0e2d9f5e/absl/log/log_sink.h) and [AddLogSink](https://github.com/abseil/abseil-cpp/blob/fa57bfc573453d57a38552eedcce894b0e2d9f5e/absl/log/log_sink_registry.h).
* `gpr_set_log_verbosity` will be removed. 

### Flags and Functions that will be removed 
* `gpr_log_severity_string` - This wont be needed anymore. 
* `gpr_should_log` - This wont be needed anymore. 
* `GRPC_STACKTRACE_MINLOGLEVEL` - This will not be needed anymore.

### Will work similar to before
* `GRPC_VERBOSITY` will work as follows

```
void SomeInitFunctionCalledByGrpcInit() {
  absl::optional<std::string> verbosity = grpc_core::GetEnv("GRPC_VERBOSITY");
  if (verbosity.has_value()) {
    VLOG(2) << "Log verbosity: " << verbosity.value();
    if (absl::EqualsIgnoreCase(verbosity.value(), "INFO")) {
      absl::SetVLogLevel("*grpc*/*", -1);
      absl::SetMinLogLevel(absl::LogSeverityAtLeast::kInfo);
    } else if (absl::EqualsIgnoreCase(verbosity.value(), "ERROR")) {
      absl::SetVLogLevel("*grpc*/*", -1);
      absl::SetMinLogLevel(absl::LogSeverityAtLeast::kError);
    } else if (absl::EqualsIgnoreCase(verbosity.value(), "WARNING")) {
      absl::SetVLogLevel("*grpc*/*", -1);
      absl::SetMinLogLevel(absl::LogSeverityAtLeast::kWarning);
    } else if (absl::EqualsIgnoreCase(verbosity.value(), "DEBUG")) {
      absl::SetGlobalVLogLevel(2);
      absl::SetMinLogLevel(absl::LogSeverityAtLeast::kInfo);
    } else {
      // Do not alter absl settings if GRPC_VERBOSITY flag is not set.
    }
  } else {
    VLOG(2) << "No verbosity set.";
  }
}
```

### Temporary Experimental Functions

* The following functions will be added. These are very simple wrappers around absl logging.
  * These are needed because PHP and RUBY cannot use absl directly. Absl logging cannot be used from `C` code. These simple wrapper functions will be exported so that they can be used from `C` code.
  * We want to avoid usage of format specifiers that RUBY and PHP are currently using.
  * As soon as we are able to use C++ APIs from Ruby and PHP, these experimental API wrappers will be deleted.

```
// Equivalent to ABSL_LOG(severity) << message_str;
GPRAPI void grpc_absl_log(const char* file, int line,
                          gpr_log_severity severity,
                          const char* message_str);

// Equivalent to ABSL_LOG(severity) << message_str << num;
GPRAPI void grpc_absl_log_int(const char* file, int line,
                              gpr_log_severity severity,
                              const char* message_str, intptr_t num);

// Equivalent to ABSL_LOG(severity) << message_str1 << message_str2;
GPRAPI void grpc_absl_log_str(const char* file, int line,
                              gpr_log_severity severity,
                              const char* message_str1,
                              const char* message_str2);
```

## Rationale

### Advantages 

*	Format specifiers are not type-safe. We want to avoid these.
*	We want to avoid maintaining platform specific gpr APIs when absl is providing them.
*	Abseil provides a range of APIs such as PLOG, VLOG, DVLOG with specific logging frequencies such as LOG_IF, LOG_EVERY_N, LOG_EVERY_POW_2 etc.

### Action Items

*	Coding using the above APIs will need to migrate to the newer APIs.
*	The application will need to call absl::InitializeLog() itself, or else there will be a log message indicating that logging is happening before absl logging is initialized. If in the future absl fixes [#1656](https://github.com/abseil/abseil-cpp/issues/1656), then we could consider initializing absl logging in gRPC init.

## Implementation

*	[GPR_ASSERT and GPR_DEBUG_ASSERT removal](https://github.com/grpc/grpc/pulls?q=%22%5Bgrpc%5D%5BGpr_To_Absl_Logging%5D+Migrating+from+gpr+to+absl+logging+GPR_ASSERT%22+)
*	[gpr_log removal](https://github.com/grpc/grpc/pulls?q=%22%5Bgrpc%5D%5BGpr_To_Absl_Logging%5D+Migrating+from+gpr+to+absl+logging+-+gpr_log%22+)
