# Test Framework Detection and Round-Trip Stub Templates

For each language, this file specifies how to detect the project's test framework and how to write a runnable round-trip test stub. Stubs use real implementations on both sides of the seam — the whole point of the skill.

When generating a stub, replace the `<...>` placeholders with concrete values from the analysis. The stub should fail on first run (because the boundary is currently unverified) and pass once the producer/consumer agreement is restored.

---

## TypeScript / JavaScript

### Detection (in this order — first match wins)

1. `vitest` in `package.json` devDeps OR `vitest.config.*` present → **Vitest**.
2. `jest` in devDeps OR `jest.config.*` OR `"jest"` key in `package.json` → **Jest**.
3. `mocha` in devDeps OR `.mocharc.*` → **Mocha** (default assertion library: Chai if present, else `node:assert`).
4. `node:test` imported anywhere → **node:test** (Node 20+ built-in).
5. `ava` in devDeps → **AVA**.
6. Fallback: ask once, cache in `.boundary-bug-hunter.json`.

### Test file location

- `*.test.ts` / `*.test.js` next to source (Vitest, Jest default).
- `tests/` or `__tests__/` directory.
- For integration tests specifically: `tests/integration/` is the canonical location across frameworks. If absent, create it.

### Stub: Vitest

```ts
import { describe, it, expect } from 'vitest';
import { <producerFn> } from '<producer-import-path>';
import { <consumerFn> } from '<consumer-import-path>';

describe('boundary: <producer> → <consumer>', () => {
  it('<verb-phrase describing the round-trip>', async () => {
    const produced = await <producerFn>(<realistic-input>);
    const consumed = await <consumerFn>(produced);
    expect(consumed).toMatchObject({
      <field>: <expected-value-derived-from-input>,
    });
  });
});
```

### Stub: Jest

Identical to Vitest but `from 'vitest'` becomes implicit globals (or `from '@jest/globals'` if `injectGlobals: false`).

### Stub: Mocha + Chai

```ts
import { expect } from 'chai';
import { <producerFn> } from '<producer-import-path>';
import { <consumerFn> } from '<consumer-import-path>';

describe('boundary: <producer> → <consumer>', () => {
  it('<verb-phrase>', async () => {
    const produced = await <producerFn>(<realistic-input>);
    const consumed = await <consumerFn>(produced);
    expect(consumed).to.deep.include({ <field>: <expected> });
  });
});
```

### Stub: node:test

```ts
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { <producerFn> } from '<producer-import-path>';
import { <consumerFn> } from '<consumer-import-path>';

test('boundary: <producer> → <consumer> — <verb-phrase>', async () => {
  const produced = await <producerFn>(<realistic-input>);
  const consumed = await <consumerFn>(produced);
  assert.partialDeepStrictEqual(consumed, { <field>: <expected> });
});
```

---

## Python

### Detection

1. `pytest` in `pyproject.toml`/`setup.py` deps OR `pytest.ini`/`pyproject.toml [tool.pytest.ini_options]`/`conftest.py` → **pytest**.
2. `tests/` files derive from `unittest.TestCase` → **unittest**.
3. `nose2` / `nose` in deps → **nose2** (rare, but possible).
4. Fallback: pytest is the safe default for new tests in modern projects.

### Test file location

- `tests/` at repo root (pytest default).
- `<package>/tests/` adjacent to source.
- Integration tests: `tests/integration/test_<feature>.py`.

### Stub: pytest

```python
import pytest

from <producer_module> import <producer_fn>
from <consumer_module> import <consumer_fn>


def test_boundary_<producer>_to_<consumer>():
    """<verb-phrase describing the round-trip>."""
    produced = <producer_fn>(<realistic_input>)
    consumed = <consumer_fn>(produced)
    assert consumed.<field> == <expected_value_derived_from_input>
```

For async producers/consumers, add `pytest-asyncio` and:

```python
import pytest

@pytest.mark.asyncio
async def test_boundary_<producer>_to_<consumer>():
    produced = await <producer_fn>(<realistic_input>)
    consumed = await <consumer_fn>(produced)
    assert consumed.<field> == <expected_value>
```

### Stub: unittest

```python
import unittest

from <producer_module> import <producer_fn>
from <consumer_module> import <consumer_fn>


class TestBoundary<Producer>To<Consumer>(unittest.TestCase):
    def test_<verb_phrase>(self):
        produced = <producer_fn>(<realistic_input>)
        consumed = <consumer_fn>(produced)
        self.assertEqual(consumed.<field>, <expected_value>)
```

---

## Go

### Detection

- Standard library `testing` is universal. `*_test.go` files with `func TestXxx(t *testing.T)`.
- If `github.com/stretchr/testify` is in `go.mod`, prefer testify assertions for clarity.
- If `github.com/onsi/ginkgo` is present → use Ginkgo (rare for boundary tests; ginkgo is BDD-style).

### Test file location

- Adjacent to source: `<file>.go` → `<file>_test.go`.
- For integration: `internal/integration/` or `tests/integration/` (no strict convention; follow repo style).

### Stub: stdlib testing

```go
package <package_name>

import (
	"testing"

	<producer_import>
	<consumer_import>
)

func TestBoundary_<Producer>_To_<Consumer>(t *testing.T) {
	produced, err := <ProducerFn>(<realisticInput>)
	if err != nil {
		t.Fatalf("producer failed: %v", err)
	}

	consumed, err := <ConsumerFn>(produced)
	if err != nil {
		t.Fatalf("consumer failed: %v", err)
	}

	if consumed.<Field> != <expectedValue> {
		t.Errorf("round-trip mismatch: got %v, want %v", consumed.<Field>, <expectedValue>)
	}
}
```

### Stub: testify

```go
package <package_name>

import (
	"testing"

	"github.com/stretchr/testify/require"
)

func TestBoundary_<Producer>_To_<Consumer>(t *testing.T) {
	produced, err := <ProducerFn>(<realisticInput>)
	require.NoError(t, err)

	consumed, err := <ConsumerFn>(produced)
	require.NoError(t, err)

	require.Equal(t, <expectedValue>, consumed.<Field>)
}
```

---

## Rust

### Detection

- Built-in `#[test]` attribute is the default. Integration tests live in `tests/` at the package root.
- If `proptest`, `quickcheck`, or `rstest` is in `dev-dependencies`, prefer the project's existing pattern.

### Stub: built-in

For unit-level seam (same crate):

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use <consumer_module>::<consumer_fn>;

    #[test]
    fn boundary_<producer>_to_<consumer>() {
        let produced = <producer_fn>(<realistic_input>).expect("producer failed");
        let consumed = <consumer_fn>(produced).expect("consumer failed");
        assert_eq!(consumed.<field>, <expected_value>);
    }
}
```

For integration (in `tests/`):

```rust
// tests/boundary_<producer>_to_<consumer>.rs
use <crate_name>::{<producer_fn>, <consumer_fn>};

#[test]
fn round_trip_preserves_<field>() {
    let produced = <producer_fn>(<realistic_input>).expect("producer failed");
    let consumed = <consumer_fn>(produced).expect("consumer failed");
    assert_eq!(consumed.<field>, <expected_value>);
}
```

For async (with `tokio::test`):

```rust
#[tokio::test]
async fn round_trip_preserves_<field>() {
    let produced = <producer_fn>(<input>).await.unwrap();
    let consumed = <consumer_fn>(produced).await.unwrap();
    assert_eq!(consumed.<field>, <expected>);
}
```

---

## Ruby

### Detection

1. `rspec` gem in Gemfile OR `spec/` dir present → **RSpec**.
2. `minitest` gem OR `test/` dir with `*_test.rb` → **minitest**.

### Test file location

- RSpec: `spec/integration/<feature>_spec.rb`.
- minitest: `test/integration/<feature>_test.rb`.

### Stub: RSpec

```ruby
require 'spec_helper'
require '<producer_path>'
require '<consumer_path>'

RSpec.describe 'boundary: <producer> → <consumer>' do
  it '<verb phrase>' do
    produced = <ProducerClass>.new.<producer_method>(<realistic_input>)
    consumed = <ConsumerClass>.new.<consumer_method>(produced)
    expect(consumed.<field>).to eq(<expected_value>)
  end
end
```

### Stub: minitest

```ruby
require 'minitest/autorun'
require_relative '<producer_path>'
require_relative '<consumer_path>'

class BoundaryProducerToConsumerTest < Minitest::Test
  def test_<verb_phrase>
    produced = <ProducerClass>.new.<producer_method>(<realistic_input>)
    consumed = <ConsumerClass>.new.<consumer_method>(produced)
    assert_equal <expected_value>, consumed.<field>
  end
end
```

---

## Java / Kotlin

### Detection

1. `junit-jupiter` (JUnit 5) in dependencies → **JUnit 5** (preferred).
2. `junit:junit` (JUnit 4) → **JUnit 4**.
3. `kotest` in Kotlin projects → **Kotest**.
4. `spock` in Groovy/mixed projects → **Spock**.

### Test file location

- Maven/Gradle: `src/test/java/...` or `src/test/kotlin/...` mirroring the source package.
- Integration tests: `src/integrationTest/...` (Gradle) or a separate Maven module.

### Stub: JUnit 5 (Java)

```java
package <package>;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;

class Boundary<Producer>To<Consumer>Test {
    @Test
    void <verbPhrase>() {
        var produced = new <ProducerClass>().<producerMethod>(<realisticInput>);
        var consumed = new <ConsumerClass>().<consumerMethod>(produced);
        assertEquals(<expectedValue>, consumed.<field>);
    }
}
```

### Stub: JUnit 5 (Kotlin)

```kotlin
package <package>

import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.assertEquals

class Boundary<Producer>To<Consumer>Test {
    @Test
    fun `<verb phrase>`() {
        val produced = <ProducerClass>().<producerMethod>(<realisticInput>)
        val consumed = <ConsumerClass>().<consumerMethod>(produced)
        assertEquals(<expectedValue>, consumed.<field>)
    }
}
```

---

## .NET (C#)

### Detection

1. `xunit` package referenced → **xUnit**.
2. `NUnit` package referenced → **NUnit**.
3. `MSTest.TestFramework` referenced → **MSTest**.

### Test file location

Separate test project: `<MainProject>.Tests/` with the same folder structure as the main project.

### Stub: xUnit

```csharp
using Xunit;

namespace <Namespace>.Tests;

public class Boundary<Producer>To<Consumer>Tests
{
    [Fact]
    public async Task <VerbPhrase>()
    {
        var producer = new <ProducerClass>();
        var consumer = new <ConsumerClass>();

        var produced = await producer.<ProducerMethod>(<realisticInput>);
        var consumed = await consumer.<ConsumerMethod>(produced);

        Assert.Equal(<expectedValue>, consumed.<Field>);
    }
}
```

### Stub: NUnit

```csharp
using NUnit.Framework;

namespace <Namespace>.Tests;

[TestFixture]
public class Boundary<Producer>To<Consumer>Tests
{
    [Test]
    public async Task <VerbPhrase>()
    {
        var producer = new <ProducerClass>();
        var consumer = new <ConsumerClass>();

        var produced = await producer.<ProducerMethod>(<realisticInput>);
        var consumed = await consumer.<ConsumerMethod>(produced);

        Assert.That(consumed.<Field>, Is.EqualTo(<expectedValue>));
    }
}
```

---

## PHP

### Detection

- `phpunit/phpunit` in `composer.json` devDeps → **PHPUnit**.
- `pestphp/pest` → **Pest** (Laravel ecosystem common).

### Test file location

- `tests/` directory at project root, mirroring the source structure.
- `tests/Feature/` for integration-style tests in Laravel projects.

### Stub: PHPUnit

```php
<?php

namespace Tests\Integration;

use PHPUnit\Framework\TestCase;
use <ProducerNamespace>\<ProducerClass>;
use <ConsumerNamespace>\<ConsumerClass>;

class Boundary<Producer>To<Consumer>Test extends TestCase
{
    public function test<VerbPhrase>(): void
    {
        $producer = new <ProducerClass>();
        $consumer = new <ConsumerClass>();

        $produced = $producer-><producerMethod>(<realisticInput>);
        $consumed = $consumer-><consumerMethod>($produced);

        $this->assertSame(<expectedValue>, $consumed-><field>);
    }
}
```

---

## Elixir

### Detection

- ExUnit is the standard. `test/` directory with `*_test.exs`.

### Stub: ExUnit

```elixir
defmodule <App>.Boundary<Producer>To<Consumer>Test do
  use ExUnit.Case, async: true

  alias <Producer.Module>
  alias <Consumer.Module>

  test "<verb phrase>" do
    {:ok, produced} = <Producer.Module>.<producer_fn>(<realistic_input>)
    {:ok, consumed} = <Consumer.Module>.<consumer_fn>(produced)
    assert consumed.<field> == <expected_value>
  end
end
```

---

## Dart / Flutter

### Detection

- `test` package OR `flutter_test` in `pubspec.yaml` devDeps.

### Stub

```dart
import 'package:test/test.dart';
import 'package:<app>/<producer_path>.dart';
import 'package:<app>/<consumer_path>.dart';

void main() {
  group('boundary: <producer> → <consumer>', () {
    test('<verb phrase>', () async {
      final produced = await <producerFn>(<realisticInput>);
      final consumed = await <consumerFn>(produced);
      expect(consumed.<field>, equals(<expectedValue>));
    });
  });
}
```

---

## Stub-writing rules (apply across all languages)

1. **Use real implementations on both sides.** No mocks of the producer or consumer themselves. If the producer requires a database, use the project's existing integration-test fixtures, not a fresh in-memory mock.
2. **Realistic input.** Pull a representative input from the diff or from existing fixtures. Don't pass `{}` or `null` and call it tested.
3. **Assert the contract.** The assertion must reference the specific fields/invariants the consumer depends on. Asserting "no exception thrown" is not a round-trip test.
4. **Name the seam in the test name.** `boundary_<producer>_to_<consumer>` makes the intent obvious in failure output.
5. **Async-safe.** If either side is async, the test must await both. Synchronous assertions on async results are silent passes.
6. **No new mocks introduced.** If the project's existing integration suite needs a mock for a third-party service, reuse it. Never introduce a new mock to "make the round-trip easier" — that defeats the test.
