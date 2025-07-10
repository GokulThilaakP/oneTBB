# Custom Assertion Handler

## Introduction

oneTBB provides assertion macros for detecting programming errors and invalid states during development and testing. Currently, these assertions use a fixed behavior: they print error messages to stderr and terminate the program with `std::abort()`. While this approach works for many scenarios, it limits flexibility in different deployment environments and use cases.

This proposal introduces a mechanism for customizing assertion failure handling through a `set_assertion_handler` API. The motivation includes several important use cases:

**Improved debugging experience**: Developers can integrate custom logging systems, breakpoint handling, or specialized error reporting instead of relying on stderr output and program termination.

**Enhanced testing support**: Unit test frameworks can capture assertion failures as exceptions rather than terminating the test process, enabling better test coverage and validation of error conditions.

**Production deployment flexibility**: Applications can implement graceful degradation or recovery strategies rather than experiencing hard crashes in production environments.

**Integration with existing error infrastructure**: Applications can route oneTBB assertion failures through their established error reporting and monitoring systems.

## Proposal

### Public API

The proposed extension adds the following interface to the oneTBB library:

```cpp
namespace tbb {
    // Function pointer type for assertion handlers
    using assertion_handler_type = void(*)(const char* location, int line,
                                           const char* expression, const char* comment);

    // Set custom assertion handler, returns previous handler
    assertion_handler_type set_assertion_handler(assertion_handler_type new_handler);
}
```

### Proposed Implementation Details

#### 1. Handler Storage and Management
The implementation will use a global static variable to store the current assertion handler. The `set_assertion_handler` function will return the previously installed handler to enable handler restoration. When the assertion handler is `nullptr`, the default implementation (stderr output followed by `std::abort()`) will be used.

#### 2. Assertion Execution Flow
The assertion execution will first check if a custom handler is installed. If present, the custom handler will be invoked with the assertion details. If no custom handler is set, the default behavior will be executed.

#### 3. New Library Entry Point
A new library entry point will be added to support the `set_assertion_handler` function, ensuring it can be used both within the oneTBB library implementation and by external applications.

## Examples

### Custom Logging Integration

Applications can integrate assertion failures with their logging infrastructure:

```cpp
tbb::set_assertion_handler([](const char *location, int line, const char *expression, const char *comment) {
    my_logger.error("TBB assertion failed in {} at line {}: {}", location, line, expression);
    if (comment) {
        my_logger.error("Details: {}", comment);
    }
});
```

### Exception-Based Error Handling

Applications preferring exception-based error handling can convert assertions to exceptions:

```cpp
tbb::set_assertion_handler([](const char *location, int line, const char *expression, const char *comment) {
    throw std::runtime_error(std::string("Assertion failed: ") + expression + " in " + location);
});
```

### Unit Testing Support

Unit test frameworks can capture assertion failures and handle them as exceptions or test failure macro calls
instead of terminating the entire test process, allowing tests to continue running.

## Design Considerations

### Performance Implications

The implementation is designed for minimal performance overhead. Handler lookup involves a simple pointer check with no heap allocation. The use of C-style function pointers avoids C++ runtime dependencies that could impact performance in memory allocator contexts.

### API and ABI Backward Compatibility

**API and ABI Stability**: The proposed interface uses C-style function pointers rather than `std::function` to ensure ABI stability across compiler versions and standard library implementations.

**Backward Compatibility**: Existing code will continue to work without modification. The default behavior remains unchanged when no custom handler is installed.

### Dependency Constraints

**Minimal Dependencies**: The implementation avoids C++ runtime dependencies, making it suitable for use within tbbmalloc and other low-level components.

**Build Configuration Independence**: The mechanism works in both debug builds (where assertions are enabled by default) and release builds (when compiled with `TBB_USE_ASSERT`).

## Alternative Approaches Considered

### `std::function` vs Function Pointers
**Proposed**: Function pointers
- **Pros**: No C++ runtime dependency, ABI stable, minimal overhead
- **Cons**: Less flexible than `std::function`

**Alternative**: `std::function`
- **Pros**: More flexible, supports lambda captures, type-safe
- **Cons**: C++ runtime dependency, ABI instability, higher overhead

### Thread-Local vs Global Handler
**Proposed**: Global handler
- **Pros**: Simple implementation, consistent behavior across threads
- **Cons**: Cannot customize per-thread behavior

**Alternative**: Thread-local handlers
- **Pros**: Per-thread customization possible
- **Cons**: Complex implementation, unclear semantics for assertion origins

### Handler Chaining vs Single Handler
**Proposed**: Single handler replacement
- **Pros**: Simple, predictable behavior
- **Cons**: Cannot compose multiple handlers

**Alternative**: Handler chaining
- **Pros**: Composable behavior, multiple observers
- **Cons**: Complex implementation, unclear failure handling

## Testing Aspects

The assertion handler mechanism supports comprehensive testing:

```cpp
// Test helper to verify assertion behavior
class AssertionCapture {
    std::string captured_message;
    assertion_handler_type old_handler;

public:
    AssertionCapture() {
        old_handler = tbb::set_assertion_handler(
            [this](const char *location, int line, const char *expression, const char *comment) {
                captured_message = std::string(expression) + " in " + location;
                throw std::runtime_error("captured assertion");
            });
    }
    ~AssertionCapture() { tbb::set_assertion_handler(old_handler); }

    const std::string &get_message() const { return captured_message; }
};
```

## Open Questions

1. **Handler Composition**: Should the API support chaining multiple handlers for different use cases?

2. **Handler Scoping**: Should there be mechanisms for scoped handler installation using RAII-style classes to automatically restore previous handlers?

3. **Thread Safety**: Should the handler installation and invocation be thread-safe, or should applications be responsible for synchronization?
