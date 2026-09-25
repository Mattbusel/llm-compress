# llm-compress

[![CI](https://github.com/Mattbusel/llm-compress/actions/workflows/ci.yml/badge.svg)](https://github.com/Mattbusel/llm-compress/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![C++17](https://img.shields.io/badge/C%2B%2B-17-blue.svg)
![Single header](https://img.shields.io/badge/single-header-green.svg)

Shrink an LLM conversation to fit a token budget from C++, with five strategies from simple truncation to LLM summarization.

> Part of **[llm-cpp](https://github.com/Mattbusel/llm-cpp)**, a family of 26 single-header C++ libraries for building on LLM APIs. Each one stands alone: copy one header, include it, done.

Long conversations eventually overflow the context window or get expensive. llm-compress takes a message list and a budget and returns a shorter list that keeps the system prompt, using whichever strategy suits your app. Four strategies are purely local with no dependencies; the fifth can ask a model to summarize what gets dropped.

## Features

- `TruncateHead`: drop the oldest messages after the system prompt
- `TruncateTail`: drop the newest non-system messages
- `TruncateSmart`: drop the shortest (lowest information) messages first
- `SlidingWindow{n}`: keep only the last n user/assistant turns
- `Summarize{api_key, model}`: replace the removed messages with a summary system message, written by an LLM when built with `LLM_COMPRESS_SUMMARIZE`
- System messages are preserved by default (`preserve_system`)
- Reports tokens before and after and how many messages were removed

## Quick start

Requirements: a C++17 compiler. No other dependencies, no network access.

1. Copy [`include/llm_compress.hpp`](include/llm_compress.hpp) into your project.
2. In exactly one `.cpp` file, `#define LLM_COMPRESS_IMPLEMENTATION` before including it. Other files just `#include "llm_compress.hpp"`.

```cpp
#define LLM_COMPRESS_IMPLEMENTATION
#include "llm_compress.hpp"
#include <iostream>

int main() {
    std::vector<llm::CompressMessage> history = {
        {"system", "You are a helpful assistant."},
    };
    for (int i = 0; i < 40; ++i) {
        history.push_back({"user", "Question " + std::to_string(i) + ": tell me something about C++ templates."});
        history.push_back({"assistant", "Answer " + std::to_string(i) + ": templates let the compiler generate code for each type you use."});
    }

    llm::CompressConfig cfg;
    cfg.strategy     = llm::SlidingWindow{5};  // keep the last 5 user/assistant turns
    cfg.token_budget = 1000;

    llm::CompressResult r = llm::compress_messages(history, cfg);
    std::cout << r.tokens_before << " -> " << r.tokens_after << " tokens, "
              << r.messages_removed << " messages removed\n";
}
```

Build and run:

```bash
g++ -std=c++17 -I include example.cpp -o example
./example
```

Output:

```text
1287 -> 167 tokens, 70 messages removed
```

## API

Everything lives in namespace `llm`.

| Function / type | What it does |
|---|---|
| `compress_messages(messages, cfg)` | Return a `CompressResult` with the compressed list and before/after token counts |
| `estimate_tokens(messages)` | Characters / 4 estimate for a message list |
| `CompressConfig` | `strategy` (a `std::variant` of the five strategies), `token_budget` (default 4096), `preserve_system` |

## How it works

Strategies are plain structs combined in a `std::variant`, and `compress_messages` dispatches on the active one. The local strategies remove messages until the estimate fits the budget. `Summarize` removes oldest-first like `TruncateHead`, then inserts a `Previous context summary: ...` system message. If you also `#define LLM_COMPRESS_SUMMARIZE` (and link `-lcurl`), that summary comes from an OpenAI chat completion; otherwise it is a placeholder noting how many messages were dropped.

## Examples

The [`examples/`](examples) folder has runnable programs:

- [`basic_compress.cpp`](examples/basic_compress.cpp)
- [`compress_history.cpp`](examples/compress_history.cpp)
- [`sliding_window.cpp`](examples/sliding_window.cpp)
- [`smart_truncate.cpp`](examples/smart_truncate.cpp)
- [`summarize_compress.cpp`](examples/summarize_compress.cpp)

Build the examples with CMake:

```bash
cmake -B build
cmake --build build
```

## Limitations

- Token counts are a characters/4 estimate, not a real tokenizer.
- LLM summarization needs `LLM_COMPRESS_SUMMARIZE` plus libcurl and calls the OpenAI endpoint only.

## License

MIT. See [LICENSE](LICENSE).
