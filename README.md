# Flow Adapters (STL. Адаптеры)

Библиотека адаптеров данных на C++ для декларативной, потоковой и ленивой обработки контейнеров, алгоритмов и файлов через конвейерный оператор `|` (pipe syntax).

Библиотека реализует функциональный стиль потоковой обработки данных без использования `<ranges>`, с соблюдением требования константного расхода памяти $O(1)$ для всех ленивых адаптеров (за исключением аккумулирующих: `AggregateByKey` и `Join`).

---

## Возможности и реализованные адаптеры

### Источники данных (Sources / Generators)
- **`Dir(path, recursive = false)`** — обход директории (и опционально рекурсивный обход поддиректорий) с генерацией путей `std::filesystem::path`.
- **`AsDataFlow(container)`** — преобразование STL-контейнера в ленивый поток данных.

### Ленивые адаптеры ($O(1)$ по дополнительной памяти)
- **`OpenFiles()`** — открытие файловых потоков (`std::ifstream`) для каждого переданного пути.
- **`Split(delimiters)`** — разбиение текстового потока на токены по набору символов-разделителей.
- **`Filter(predicate)`** — ленивая фильтрация элементов по предикату.
- **`Transform(func)`** — преобразование каждого элемента входной последовательности.
- **`DropNullopt()`** — отсеивание `std::nullopt` из потока `std::optional<T>`.
- **`SplitExpected(on_success, on_unexpected)`** — разделение потока `std::expected<T, E>` на два конвейера обработки (для ожидаемых значений и ошибок).

### Агрегации и объединения
- **`AggregateByKey(init_val, aggregator, key_extractor)`** — неленивая агрегация значений по заданному ключу.
- **`Join(other_flow, key_extractor)`** — объединение двух потоков данных по ключу (аналог SQL `LEFT JOIN`).
- **`KV`** / **`JoinResult`** — структуры для пар «ключ-значение» и результатов объединения.

### Терминальные операции (Sinks)
- **`Out(stream)`** — потоковый вывод элементов в `std::ostream`.
- **`Write(stream, delimiter)`** — запись элементов в поток с разделителем между ними.
- **`AsVector()`** — материализация ленивого потока в `std::vector<T>`.

---

## Структура проекта

```text
flow_adapters/
├── bin/                      # Пример использования / точка входа
│   ├── CMakeLists.txt
│   └── main.cpp
├── lib/                      # Заголовочные файлы библиотеки
│   └── processing.h          # Реализация адаптеров и оператора |
├── tests/                    # Юнит-тесты на Google Test
│   ├── CMakeLists.txt
│   ├── coverage.cmake        # Конфигурация расчета покрытия кода
│   ├── aggregate_by_key_ut.cpp
│   ├── drop_nullopt_ut.cpp
│   ├── filter_ut.cpp
│   ├── flow_convertions_ut.cpp
│   ├── join_ut.cpp
│   ├── split_expected_ut.cpp
│   ├── split_ut.cpp
│   ├── transform_ut.cpp
│   └── write_ut.cpp
├── example_texts/            # Пример текстовых данных
│   └── sample.txt
└── CMakeLists.txt
```

---

## Пример использования

Подсчет частотности слов среди всех текстовых файлов в директории:

```cpp
#include "processing.h"
#include <iostream>
#include <filesystem>
#include <algorithm>
#include <format>

using namespace flow;

int main(int argc, char* argv[]) {
    if (argc < 2) return 1;

    Dir(argv[1], /*recursive=*/true)
        | Filter([](const std::filesystem::path& p) {
            return p.extension() == ".txt";
          })
        | OpenFiles()
        | Split("\n ,.;")
        | Transform([](std::string token) {
            std::transform(token.begin(), token.end(), token.begin(),
                           [](unsigned char c) { return std::tolower(c); });
            return token;
          })
        | AggregateByKey(
            0uz,
            [](const std::string&, size_t& count) { ++count; },
            [](const std::string& token) { return token; }
          )
        | Transform([](const auto& stat) {
            return std::format("{} - {}", stat.first, stat.second);
          })
        | Out(std::cout);

    return 0;
}
```

---

## Сборка и тестирование

### Требования
- Компилятор с поддержкой **C++20** (или выше)
- **CMake** версии 3.16+
- **Google Test** (загружается автоматически через CMake `FetchContent`)

### Сборка

```bash
cmake -B build -S .
cmake --build build
```

### Запуск тестов

```bash
ctest --test-dir build --output-on-failure
```

Или запуск конкретного теста:
```bash
./build/tests/aggregate_by_key_ut
./build/tests/split_expected_ut
```