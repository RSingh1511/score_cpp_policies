# score_cpp_policies

Centralized C++ quality tool policies for Eclipse S-CORE, providing sanitizer configurations reusable across all S-CORE modules (logging, communication, baselibs, etc.).

Planned: clang-format, code coverage policies.

## What This Provides

- **Sanitizer Bazel feature flags** — ASan, UBSan, LSan, TSan as Bazel build features
- **`//sanitizers:wrapper`** — shell script that sets all sanitizer runtime options centrally
- **`sanitizers/sanitizers.bazelrc`** — canonical config that consumers import or copy
- **Suppression files** — per-sanitizer suppression lists for known false positives (GoogleTest, etc.)
- **Constraint system** — `target_compatible_with` settings for sanitizer-incompatible targets
- **`//clang_tidy:defs.bzl`** — `make_clang_tidy_aspect` / `make_clang_tidy_test` macros wrapping `aspect_rules_lint`
- **`clang_tidy/.clang-tidy`** — default S-CORE check set (conservative baseline, tailorable per module)
- **`clang_tidy/clang_tidy.bazelrc`** — canonical bazelrc snippet consumers import

## Available Sanitizer Configurations

| Config | Sanitizers | Notes |
|--------|-----------|-------|
| `--config=asan_ubsan_lsan` | ASan + UBSan + LSan | **Recommended** — catches memory errors, UB, and leaks |
| `--config=asan` | AddressSanitizer | Alias for `asan_ubsan_lsan` |
| `--config=ubsan` | UndefinedBehaviorSanitizer | Alias for `asan_ubsan_lsan` |
| `--config=lsan` | LeakSanitizer | Alias for `asan_ubsan_lsan` |
| `--config=tsan` | ThreadSanitizer | Cannot be combined with ASan |

## Usage

### Add Dependency

```python
bazel_dep(name = "score_cpp_policies", version = "0.0.0")
```

### Configure Sanitizers

Copy the sanitizer configs from `sanitizers/sanitizers.bazelrc` into your `.bazelrc` and adapt the paths:

```bazelrc
# ASan + UBSan + LSan (Combined)
build:asan_ubsan_lsan --features=asan
build:asan_ubsan_lsan --features=ubsan
build:asan_ubsan_lsan --features=lsan
build:asan_ubsan_lsan --platform_suffix=asan_ubsan_lsan
test:asan_ubsan_lsan --config=with_debug_symbols
test:asan_ubsan_lsan --test_tag_filters=-no-asan,-no-lsan,-no-ubsan
test:asan_ubsan_lsan --@score_cpp_policies//sanitizers/flags:sanitizer=asan_ubsan_lsan
test:asan_ubsan_lsan --run_under=@score_cpp_policies//sanitizers:wrapper

# Shortcuts
build:asan  --config=asan_ubsan_lsan
test:asan  --test_tag_filters=-no-asan
build:ubsan --config=asan_ubsan_lsan
test:ubsan --test_tag_filters=-no-ubsan
build:lsan  --config=asan_ubsan_lsan
test:lsan  --test_tag_filters=-no-lsan

# ThreadSanitizer
build:tsan --features=tsan
build:tsan --platform_suffix=tsan
test:tsan --config=with_debug_symbols
test:tsan --cxxopt=-O1
test:tsan --test_tag_filters=-no-tsan
test:tsan --@score_cpp_policies//sanitizers/flags:sanitizer=tsan
test:tsan --run_under=@score_cpp_policies//sanitizers:wrapper
```

> **Note**: The `--run_under=@score_cpp_policies//sanitizers:wrapper` automatically loads runtime options and suppressions from the module.

### Run Tests

```bash
# ASan + UBSan + LSan (recommended)
bazel test --config=asan_ubsan_lsan //...

# ThreadSanitizer (separate run — cannot combine with ASan)
bazel test --config=tsan //...
```

## Tagging Tests for Sanitizer Compatibility

```python
cc_test(
    name = "my_test",
    srcs = ["my_test.cpp"],
    tags = ["no-tsan"],  # skip when running --config=tsan
    deps = ["@googletest//:gtest_main"],
)
```

| Tag | Skipped when using |
|-----|--------------------|
| `no-tsan` | `--config=tsan` |
| `no-asan` | `--config=asan` or `asan_ubsan_lsan` |
| `no-lsan` | `--config=lsan` or `asan_ubsan_lsan` |
| `no-ubsan` | `--config=ubsan` or `asan_ubsan_lsan` |

## Suppression Files

Default suppressions for common third-party libraries are included:

| File | Sanitizer | Current Suppressions |
|------|-----------|---------------------|
| `sanitizers/suppressions/asan.supp` | ASan | *(empty)* |
| `sanitizers/suppressions/lsan.supp` | LSan | GoogleTest static initialization leaks |
| `sanitizers/suppressions/tsan.supp` | TSan | stdlib false positives, Rust test suppressions |
| `sanitizers/suppressions/ubsan.supp` | UBSan | *(empty)* |

**Adding project-specific suppressions:** Currently, the wrapper loads suppressions from this module only. For project-specific suppressions, you'll need to create a custom wrapper or extend the environment variables in your `.bazelrc`. This is a known limitation being tracked for future improvements.

> **Runtime Options**: See [`sanitizers/templates/`](sanitizers/templates/) for detailed documentation of all sanitizer options configured by the wrapper.

## Constraint System

Use constraints to mark targets as incompatible with specific sanitizers:

```python
cc_library(
    name = "legacy_lib",
    target_compatible_with = [
        "@score_cpp_policies//sanitizers/constraints:no_tsan",
    ],
)
```

| Constraint | Effect |
|-----------|--------|
| `@score_cpp_policies//sanitizers/constraints:no_tsan` | Skip when `--config=tsan` |
| `@score_cpp_policies//sanitizers/constraints:no_asan_ubsan_lsan` | Skip when `--config=asan_ubsan_lsan` |
| `@score_cpp_policies//sanitizers/constraints:any_sanitizer` | Only builds with a sanitizer enabled |

## Clang-Tidy Integration

### Add Dependency

```python
bazel_dep(name = "score_cpp_policies", version = "0.0.0")
bazel_dep(name = "toolchains_llvm", version = "1.5.0")
```

Set up `@llvm_toolchain` via the `llvm` extension (see `tests/MODULE.bazel` for an example).

### Import the Bazelrc Snippet

In your `.bazelrc`:

```bazelrc
import %workspace%/path/to/clang_tidy.bazelrc
test:clang-tidy --aspects=//tools/lint:linters.bzl%clang_tidy_aspect
```

### Create `tools/lint/linters.bzl`

```python
load("@score_cpp_policies//clang_tidy:defs.bzl", "make_clang_tidy_aspect", "make_clang_tidy_test")

clang_tidy_aspect = make_clang_tidy_aspect(
    binary = Label("@llvm_toolchain//:clang-tidy"),
    configs = [Label("//:.clang-tidy")],
)

clang_tidy_test = make_clang_tidy_test(aspect = clang_tidy_aspect)
```

### Add a `.clang-tidy` Config

Place a `.clang-tidy` at your repo root. Use
[`@score_cpp_policies//clang_tidy:.clang-tidy`](clang_tidy/.clang-tidy) as the starting point.

### Run

```bash
# Lint all C++ targets
bazel test --config=clang-tidy //...
```

Reports are written to `bazel-out/.../rules_lint_report/` as text files.

---

## Testing This Repository

```bash
cd tests

bazel test --config=asan_ubsan_lsan //...
bazel test --config=tsan //...
bazel test --config=clang-tidy //...
```

## Contributing

See [CONTRIBUTION.md](CONTRIBUTION.md) for guidelines. All commits must follow [Eclipse Foundation commit rules](https://www.eclipse.org/projects/handbook/#resources-commit). Contributors must sign the ECA and DCO.

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
