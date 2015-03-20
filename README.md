# Gherkin 3

[![Build Status](https://travis-ci.org/cucumber/gherkin3.png)](https://travis-ci.org/cucumber/gherkin3)

Gherkin 3 is a parser and compiler for the Gherkin language.

It is intended to replace [Gherkin 2](https://github.com/cucumber/gherkin) and be used by
all [Cucumber](https://cukes.info) implementations to parse `.feature` files.

Gherkin 3 is currently implemented for the following platforms:

* .NET (C#)
* JVM (Java)
* JavaScript (Browser or Node.js/IO.js)
* Ruby (MRI, JRuby or any other Ruby implementation)

See `TODO.md` for what's remaining before we're ready to roll it out and refactor
the Cucumber implementations to use it.

See `CONTRIBUTING.md` if you want to contribute a parser for a new language.
Our wish-list is (in no particular order):

* C
* Go
* PHP
* Python
* Rust
* Swift

## Example

```java
// Java
Parser<Feature> parser = new Parser<>();
Feature feature = parser.parse(gherkinDoc);
```

```csharp
// C#
var parser = new Parser();
var feature = parser.Parse(gherkinDoc);
```

```ruby
# Ruby
parser = Parser.new
feature = parser.parse(gherkin_doc)
```

```javascript
// JavaScript
var parser = new Parser();
var feature = parser.parse(gherkinDoc);
```

## Why Gherkin 3?

I wrote up a summary [here](https://groups.google.com/d/msg/cukes/YLKsqbBMBoI/DYhfFx8GBegJ).

## Architecture

The following diagram outlines the architecture:

    +-----------+   +-------+   +------+   +---+
    |Gherkin doc|-->|Scanner|-->|Parser|-->|AST|
    +-----------+   +-------+   +------+   +---+

The scanner reads a gherkin doc (typically read from a `.feature` file) and creates
a *token* for each line. The tokens are passed to the parser, which outputs an AST
(Abstract Syntax Tree).

If the scanner sees a `# language` header, it will reconfigure itself dynamically
to look for Gherkin keywords for the associated language. The keywords are defined in
`dialects.json`.

The scanner is hand-written, but the parser is generated by the [Berp](https://github.com/gasparnagy/berp)
parser generator as part of the build process.

Berp takes a grammar file (`gherkin.berp`) and a template file (`gherkin-X.razor`) as input
and outputs a parser in language *X*:

    +------------+   +-----+   +---------------+
    |gherkin.berp|-->| berp|<--|gherkin-X.razor|
    +------------+   +--+--+   +---------------+
                        |
                        V
                    +--------+
                    |Parser.x|
                    +--------+

Also see the [wiki](https://github.com/cucumber/gherkin3/wiki) for some early
design docs (which might be a little outdated, but mostly OK).

### AST

The AST produced by the parser can be described with the following class diagram:

![](https://github.com/cucumber/gherkin3/blob/master/docs/ast.png)

Every class represents a node in the AST. Every node has a `Location` that describes
the line number and column number in the input file. These numbers are 1-indexed.

All fields on nodes are strings (except for `Location.line` and `Location.column`).

The implementation is simple objects without behaviour, only data. It's up to
the implementation to decide whether to use classes or just basic collections,
but the AST *must* have a JSON representation (this is used for testing).

You can see some examples in the
[tesdata/good](https://github.com/cucumber/gherkin3/tree/master/testdata/good)
directory.

### Compiler

(Work in progress)

The compiler compiles the AST produced by the parser
into a simpler form - *Test cases*. The rationale is to provide a simpler
data structure to Cucumber, which simplifies the internals of Cucumber.

    +------------+   +-------+   +------+   +--------+   +----------+
    |Feature file|-->|Scanner|-->|Parser|-->|Compiler|-->|Test cases|
    +------------+   +-------+   +------+   +--------+   +----------+

Each `Scenario` will be compiled into a `TestCase`, as will `Examples` rows under
a `Scenario Outline`. Any `Background` steps will also be compiled into each
`TestCase`. Tags will be compiled into the `TestCase` as well.

Example:

```gherkin
Feature:
  Background:
    Given a

  Scenario: b
    Given c

  Scenario Outline: c
    Given <x>
    When y

    Examples:
      | x |
      | d |
      | e |
```

This will be compiled into several `TestCase` objects (here represented as YAML
for simplicity):

```
- steps:
  - name: a
  - name: c
- steps:
  - name: a
  - name: d
  - name: y
- steps:
  - name: a
  - name: e
  - name: y
```

Each `TestCase` will also keep a reference back to the original AST nodes for
rendering and error reporting (stack traces).

Cucumber will further transform this list of `TestCase` to add various hooks and
link Step Definitions.

## Building Gherkin 3

See `CONTRIBUTING.md`
