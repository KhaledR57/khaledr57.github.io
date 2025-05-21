---
title: "GSoC 2025: Implementing Expression Evaluation in MariaDB's mysqltest Framework"
description: "How I built a powerful expression parser for MariaDB's testing tool during Google Summer of Code."
pubDate: "Aug 29 2025"
heroImage: "/gsoc25-mariadb.png"
tags: ["gsoc", "mariadb", "database", "parser", "c++", "open-source", "testing"]
---

During Google Summer of Code 2025, I had the opportunity to work with MariaDB on a project that would transform their testing infrastructure. My task was to implement expression evaluation capabilities in **mysqltest**, MariaDB's test framework, by emulating a subset of the server's expression functionality.

What started as "just add some basic arithmetic" quickly became a journey into language design and parser implementation 😅. I found myself learning the same concepts that database developers use when building SQL parsers.

By the end, I had built a complete expression parser and executor combined, supporting arithmetic, comparison, logical operators, and 30+ built-in functions.

# The Problem

Before my project, writing tests in `mysqltest` was harder than it needed to be. The tool had some serious limits that forced developers to use complex SQL queries for simple tasks.

For example, `mysqltest` couldn't:

- Handle basic arithmetic and logic opertions.
- String manipulations without complex SQL queries.
- Complex conditional logic in `if`/`while` statements  
  
```bash
# Required complex SQL queries for simple calculations
SET @var = 10 * 5 + 2;
let $result = `SELECT @var`;

# String operations also required SQL
SET @var = SUBSTRING_INDEX('a,b,c', ',', 2);
let $result=`SELECT @var`;

# Limited to simple checks without logical operators
if ($count > 0)

```

I fixed this by implementing a recursive descent parser and executor to handle arithmetic, comparisons, logical operators, and built-in functions in mysqltest expressions. Now, developers can write better, more readable tests that are still compatible with the old ones.

# My Contribution

## 1. Recursive Descent Parser and Executor

Built a combined parser and executor using recursive descent technique, emulating a subset of MariaDB server's expression functionality but with a simpler approach than the bison/yacc-generated parser used in the main server. The implementation follows MariaDB's [operator precedence](https://mariadb.com/docs/server/reference/sql-structure/operators/operator-precedence?q=date):

```scala
expression     ::= logical_or
logical_or     ::= logical_and { "||" logical_and }
logical_and    ::= equality { "&&" equality }
equality       ::= comparison { ("==" | "!=") comparison }
comparison     ::= bitwise_or { ("<" | ">" | "<=" | ">=") bitwise_or }
bitwise_or     ::= bitwise_and { "|" bitwise_and }
bitwise_and    ::= bitwise_shift { "&" bitwise_shift }
bitwise_shift  ::= term { ("<<" | ">>") term }
term           ::= factor { ("+" | "-") factor }
factor         ::= bitwise_xor { ("*" | "/" | "%") bitwise_xor }
bitwise_xor    ::= unary { "^" unary }
unary          ::= ("!" | "-" | "~") unary | primary
primary        ::= number | string | function | "(" expression ")"
```

## 2. Data Representation

For the type system design, I needed to handle the different data types mysqltest needs to work with. I followed the classic approach described in [GNU Guile's Simple Representation](https://www.gnu.org/software/guile/manual/html_node/A-Simple-Representation.html), using a structure with a type indicator and a union for the actual values. My solution supported:

- **Integers**: Decimal, hexadecimal (`0xFF`), binary (`0b1010`)
- **Booleans**: boolean values (true, false)
- **Strings**: quoted or unquoted string literals
- **NULL**: null values

I chose this straightforward tagged union approach rather than implementing advanced [pointer tagging techniques](https://www.npopov.com/2012/02/02/Pointer-magic-for-efficient-dynamic-value-representations.html) because I wanted to keep things simple and focused on getting the core functionality working correctly first.

```cpp
struct Expression_value {
    Expression_value_type type;
    unsigned long long int_val;
    bool is_numeric;
    bool is_unsigned;
    My_string str_val;
    
    void set_int(long long value);
    void set_string(const char *value, size_t len);
    void set_bool(bool value);
};
```

## 3. Backward Compatible Syntax

One of the key challenges was maintaining backward compatibility. How do you add powerful new features to a tool that thousands of existing tests depend on? Any breaking change would require updating countless test files.

The solution came during week 2 when I realized I shouldn't make the parser smarter about guessing intent, but instead make the syntax explicit.<br>
Introduced the `$()` syntax that:

```bash
# Preserves 100% backward compatibility
let $old = 5 + 2;               # Still stores string "5 + 2"

# Enables new expression evaluation  
let $new = $(5 + 2);            # Evaluates to integer 7

# Supports string interpolation
let $msg = "Result: $(5 + 2)";  # Produces "Result: 7"
```

This elegant solution ensured **zero breaking changes** to existing test suites while providing powerful new capabilities.

## 4. Rich Function Library

Added support for 30+ built-in functions to handle common operations that tests need, making tests more readable and maintainable and avoid the need to write complex SQL queries.

**String Functions:**
```bash
let $name = $(concat("Hello", " ", "World"));  # "Hello World"
let $len  = $(length($name));                  # 11
let $sub  = $(substr($name, 1, 5));            # "Hello"
```

**Numeric Functions:**
```bash
let $hex = $(hex(255));     # "FF"
let $bin = $(bin(10));      # "1010"
let $abs = $(abs(-42));     # 42
```

**Conditional Functions:**
```bash
let $result  = $(ifnull($maybe_null, "default"));     # "default" if $maybe_null is NULL
let $default = $(coalesce($a, $b, $c, "fallback"));   # first non-NULL value
```

## 5. Control Flow Integration

Enhanced `if` and `while` statements to support more complex expressions:

```bash
# Full expression evaluation
if ($(length($table_name) > 10 && $row_count < 1000))
while ($($counter < $max_iterations && $status == "active"))
```

# What I Learned

## 1. Parser Theory in Practice
Building a recursive descent parser from scratch was like learning a new language (and it actually was 😁). Reading through [Crafting Interpreters](https://craftinginterpreters.com/parsing-expressions.html) helped me understand how to:
- Turn formal grammar rules into working code
- Handle operator precedence correctly
- Deal with tricky syntax through careful grammar design

The "aha!" moment came when I realized that each grammar rule could become a function, and the parsing process was just functions calling each other.

## 2. Large Codebase Integration
Working with MariaDB's large codebase (15,000+ lines in mysqltest.cc alone) was challenging at first. I remember staring at the file structure for hours, trying to understand how everything connected. But gradually, patterns emerged:
- Debug and trace existing code to understand complex system behavior
- The importance of maintaining backward compatibility in production software

## 3. Open Source Development
The experience taught me:
- How to write clear, comprehensive documentation
- Working in large projects with established codebase and conventions
- Professional Git workflow and pull request management

# Current State & Impact

### **Completed Features**
- Full expression evaluation with proper precedence
- 30+ built-in functions (string, numeric, conditional)
- Integration with existing control flow (`if`/`while`) and in `let`  
- Comprehensive test coverage
- Complete backward compatibility


### **Performance Benefits**
- **Reduced SQL dependencies**: Tests no longer need database queries for simple calculations
- **Improved readability**: Complex test logic is now clear and maintainable

# Future Enhancements

While my GSoC project is complete, there are exciting possibilities for future development:

### 1. **Floating-Point Arithmetic**
```bash
let $precise = $(3.14159 * $radius * $radius);
```

### 2. **Date/Time Type Support**
```bash
let $date       = $('2025-12-31');
let $datetime   = $('2025-12-31 23:59:59');
let $time       = $('14:30:00');
let $formatted  = $(date_format($date, '%Y-%m-%d'));
```

<!-- ### 3. **User-Defined Functions**
```bash
function factorial($n) {
    return $($n <= 1 ? 1 : $n * factorial($($n - 1)));
}
``` -->

# Acknowledgments

Huge thanks to my amazing mentors Oleksandr Byelkin and Lena Startseva! 🙏 I really really appreciate their effort and guidance. Their support made this project possible! 😊

Working on MariaDB's testing framework was a great experience. I'm proud that my contribution will help database developers write better tests for years to come.

# Project Details

**Project Title**: MDEV-36107, MDEV-36108: Add expression support to mysqltest  
**Organization**: MariaDB Foundation  
**Duration**: 3 months (June - August 2025)  
**Pull Request**: [#4114](https://github.com/MariaDB/server/pull/4114)  
**Status**: In Review  
**JIRA Tickets**: [MDEV-36107](https://jira.mariadb.org/browse/MDEV-36107), [MDEV-36108](https://jira.mariadb.org/browse/MDEV-36108)  


# Resources & References

- [Crafting Interpreters](https://craftinginterpreters.com/contents.html) - Essential reading for understanding parser implementation
- [Value Representation in Language Implementations](https://wingolog.org/archives/2011/05/18/value-representation-in-javascript-implementations) - Inspired my type system design
- [GNU Guile's Simple Representation](https://www.gnu.org/software/guile/manual/html_node/A-Simple-Representation.html) - Classic approach to dynamic typing
- [Pointer Magic for Efficient Dynamic Value Representations](https://www.npopov.com/2012/02/02/Pointer-magic-for-efficient-dynamic-value-representations.html) - Advanced optimization techniques


---
