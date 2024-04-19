L116: Replacing gpr logging with Abseil logging
----

* Author(s): tjagtap@google.com
* Approver: roth@google.com , ctiller@google.com
* Status: In Review 
* Implemented in: gRPC C++
* Last updated: April 18, 2024.
* Discussion at: (filled after thread exists)

## Abstract

Replacing gpr logging and asserts with Abseil logging and asserts

## Background & Rationale

gRPC is currently maintaining its own version of the logging and assert APIs. The Abseil library released their logs and asserts too. We want to leverage the Abseil library and avoid maintaining our own logging code. Also, the current implementation of gpr_log uses format strings which are not type safe and have exploit potential. Moving to absl logging will eliminate this security risk.

## Related Proposals:

None.

## Proposal

We are proposing to remove all instances of gpr logging and asserts and replace them with their absl equivalents.

## Implementation

### References
* [gpr logging header](https://github.com/grpc/grpc/blame/83a17ff4684dc1fb3493a151ac0b655b1c55e766/include/grpc/support/log.h)
* [absl logging header](https://github.com/abseil/abseil-cpp/blob/master/absl/log/log.h)
* [absl assert header](https://github.com/abseil/abseil-cpp/blob/master/absl/log/check.h)

### Functions that will be removed, and their replacements
* `gpr_log`
	* `gpr_log(GPR_DEBUG,...)` with `VLOG(2) << ...`
	* `gpr_log(GPR_INFO,...)` with `LOG(INFO) < ...`
	* `gpr_log(GPR_ERROR,...)` with `LOG(ERROR) < ...`
* `gpr_log_message`
	* `gpr_log_message(GPR_DEBUG,...)` with `VLOG(2) << ...`
	* `gpr_log_message(GPR_INFO,...)` with `LOG(INFO) < ...`
	* `gpr_log_message(GPR_ERROR,...)` with `LOG(ERROR) < ...`
* Asserts
	* `GPR_ASSERT(...)` with `CHECK(...)`
	* `GPR_DEBUG_ASSERT(...)` with `DCHECK(...)`
* `gpr_assertion_failed` with `CHECK(...)`
* `gpr_set_log_function` - This function will be removed. Teams can consider usage of [LogSink](https://github.com/abseil/abseil-cpp/blob/fa57bfc573453d57a38552eedcce894b0e2d9f5e/absl/log/log_sink.h) and [AddLogSink](https://github.com/abseil/abseil-cpp/blob/fa57bfc573453d57a38552eedcce894b0e2d9f5e/absl/log/log_sink_registry.h).

### Functions that will be removed 
* `gpr_log_severity_string` - This wont be needed anymore. 
* `gpr_should_log` - This wont be needed anymore. 

### Will work similar to before
* `gpr_set_log_verbosity` and GRPC_VERBOSITY will work as follows

```
void SomeInitFunctionCalledByGrpcInit() {
	optional<string> verbosity = GetEnv("GRPC_VERBOSITY");
	if (verbosity.has_value()) {
		if (verbosity == "GPR_INFO" || verbosity == "INFO") {
			SetMinLogLevel(INFO);
		}
		else if (verbosity == "GPR_ERROR" || verbosity == "ERROR") {
			SetMinLogLevel(ERROR);
		}
		else if (verbosity == "WARNING") {
			SetMinLogLevel(WARNING);
		}
		else if (verbosity == "GPR_DEBUG") {
			SetGlobalVLogLevel(2);
			SetMinLogLevel(INFO);
		}
		else {
			LOG(ERROR) << "Unknown log verbosity: " << verbosity;
		}
	}
}

void gpr_set_log_verbosity(gpr_log_severity verbosity) {
		if (verbosity == GPR_INFO) {
			SetMinLogLevel(INFO);
		}
		else if (verbosity == GPR_ERROR) {
			SetMinLogLevel(ERROR);
		}
		else if (verbosity == GPR_DEBUG) {
			SetGlobalVLogLevel(2);
			SetMinLogLevel(INFO);
		}
		else {
			LOG(ERROR) << "Unknown log verbosity: " << verbosity;
		}
}
```