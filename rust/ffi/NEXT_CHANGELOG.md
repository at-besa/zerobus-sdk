# NEXT CHANGELOG

## Release v1.5.0

### Major Changes

### New Features and Improvements

- Build the FFI library for Linux musl targets (`x86_64-unknown-linux-musl`, `aarch64-unknown-linux-musl`), enabling C/C++ and Go-on-Alpine consumers to link `libzerobus_ffi.a` / `libzerobus_ffi.so` on musl-based (Alpine) containers. Artifacts ship in the `linux-musl-x86_64` / `linux-musl-aarch64` directories.
- Expose the core ack callback through the C FFI. `CStreamConfigurationOptions` gains `ack_on_ack` (`void (*)(int64_t offset_id, void *user_data)`), `ack_on_error` (`void (*)(int64_t offset_id, const char *error_message, void *user_data)`), and `ack_user_data`. When either pointer is non-null, an `AckCallback` is registered on the stream so acks and errors are delivered asynchronously instead of only via `wait_for_offset` / `flush`. Both create paths (`zerobus_sdk_create_stream` and `zerobus_sdk_create_stream_with_headers_provider`) read the new fields. Callbacks fire on a background task (serialized, never re-entered concurrently), so the callback and its `user_data` must outlive the stream and synchronize any shared state; panics are contained at the boundary and logged. `zerobus.h` documents the ack semantics (per-record, monotonic), error delivery, and lifetime contract.
- Add callback-based async overloads for all previously blocking stream operations: stream creation (`zerobus_sdk_create_stream_async`, `zerobus_sdk_create_stream_with_headers_provider_async`), stream recreation (`zerobus_sdk_recreate_stream_async`), offset-returning ingest calls (`zerobus_stream_ingest_proto_record_async`, `zerobus_stream_ingest_json_record_async`, `zerobus_stream_ingest_proto_records_async`, `zerobus_stream_ingest_json_records_async`), completion methods (`zerobus_stream_wait_for_offset_async`, `zerobus_stream_flush_async`, `zerobus_stream_close_async`), and unacked-record retrieval (`zerobus_stream_get_unacked_records_async`). These APIs return immediately after validation/scheduling and complete via callbacks; caller-owned string/descriptor/config inputs are copied before return, and SDK/stream handles must remain valid until callback completion.

### Bug Fixes

### Documentation

### Internal Changes

- Move the header-callback helpers `zerobus_alloc_header_array`, `zerobus_alloc_cstring`, and `zerobus_free_headers` from `arrow.rs` into the shared `common.rs` module, next to the `CHeader`/`CHeaders` types they serve. No behavior, signature, or ABI change; their declarations move ahead of the Arrow functions in the generated `zerobus.h`.

### Behavior Changes

### Breaking Changes

### Deprecations

### API Changes

- `CStreamConfigurationOptions` (in `zerobus.h`) gains three trailing fields: `ack_on_ack`, `ack_on_error`, and `ack_user_data`. This is an additive struct-layout change — existing fields keep their order and offsets, and `zerobus_get_default_config()` zero-initializes the new fields (no callback). Consumers that hand-mirror the struct (e.g. the Go cgo preamble) must add the matching trailing fields to keep the layout byte-identical.
- Add `zerobus_alloc_header_array` and `zerobus_alloc_cstring` so a non-Rust headers callback can allocate the `CHeaders` it returns through this library instead of its own allocator. The buffers are then freed by `zerobus_free_headers` with the matching allocator, keeping each allocate/free pair inside one library. This removes a cross-allocator free that could corrupt the heap on Windows when the consumer and this statically linked library resolve to different CRT heaps. Additive only — existing functions and `zerobus_free_headers` are unchanged.
- Add `CreateStreamAsyncCallback`, `OffsetAsyncCallback`, `BoolAsyncCallback`, and `RecordArrayAsyncCallback` plus the full async stream API set (`*_async` overloads for create/recreate/ingest/wait/flush/get_unacked_records/close) to `zerobus.h`. Callback `const CResult *` values are valid only for the duration of each callback; any error text must be copied during the call.
