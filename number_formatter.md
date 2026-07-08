# ICU4X Number Formatter Design

This document describes the design and architecture of number formatting components in ICU4X.

## Overview

ICU4X number formatting is designed to be highly modular, performant, and zero-copy. It covers basic decimal formatting and localized currency formatting.

The formatting pipeline follows a layered architecture where the currency formatter builds upon the decimal formatter, sharing common data structures and formatting traits.

```mermaid
graph TD
    FD[FixedDecimal] --> DF[DecimalFormatter]
    DF --> CF["CurrencyFormatter<br/>&lt;Decimal&gt;"]
    DF --> CSF["CurrencyFormatter<br/>&lt;Scientific&gt;"]
    DF --> CCF["CurrencyFormatter<br/>&lt;Compact&gt;"]
```

## Unified Architecture: Decimal Formatter as the Engine

*Related PRs & Proposals: [#8182](https://github.com/unicode-org/icu4x/pull/8182) (Initial generic design), [#8189](https://github.com/unicode-org/icu4x/pull/8189) (Unification under AbstractFormatter), and [CLDR-19617](https://unicode-org.atlassian.net/browse/CLDR-19617) (Compact Currency / Units pattern gluing).*

To minimize code duplication, reduce binary size, and simplify configuration, ICU4X adopts an architecture where the decimal formatter acts as the foundational engine that dominates and powers all higher-level dimension formatters (such as currency formatting, and in the future, unit formatting).

Instead of each dimension formatter implementing its own number formatting logic or storing redundant data structures (such as grouping rules, symbols, and plural rules), higher-level formatters wrap an underlying numeric engine that implements the `AbstractFormatter` trait.

```mermaid
graph TD
    subgraph Numeric Engine ["Numeric Engine (icu_decimal)"]
        AF["trait AbstractFormatter (Sealed)"]
        DF["DecimalFormatter"] ---|implements| AF
        CDF["CompactDecimalFormatter"] ---|implements| AF
        SDF["ScientificDecimalFormatter (Future)"] ---|implements| AF
    end

    subgraph Dimension Formatters ["Dimension Formatters"]
        CF["CurrencyFormatter&lt;V: AbstractFormatter&gt;"]
        UF["UnitsFormatter&lt;V: AbstractFormatter&gt; (Future)"]
    end

    AF -->|powers| CF
    AF -->|powers| UF
```

### Benefits of the Engine Approach
1. **Zero Duplication**: `CurrencyFormatter` and `UnitsFormatter` do not store or reimplement compact decimal rules, scientific notation algorithms, or standard grouping formatting. They delegate 100% of numeric value formatting to the underlying `AbstractFormatter`.
2. **Alignment with CLDR-19617**: This architecture directly prepares ICU4X for [CLDR-19617](https://unicode-org.atlassian.net/browse/CLDR-19617) (Compact Currency / Units pattern gluing). Under CLDR-19617, compact numbers are formatted independently and glued into currency or unit placeholder patterns, which maps 1:1 with delegating value formatting to `CompactDecimalFormatter` and interpolating the result into the dimension's placeholder pattern.
3. **Future Extensibility (Units Formatter)**: When building `UnitsFormatter`, no new compact or decimal formatting logic needs to be written. It will simply be defined as `pub struct UnitsFormatter<V: AbstractFormatter> { value_formatter: V, units_data: UnitsData }`.
4. **SemVer & API Stability**: `AbstractFormatter` uses the **Sealed Trait Pattern**, preventing downstream crates from implementing it. This allows the ICU4X team to evolve internal numeric helper methods without breaking external code.

## Currency Format

This section discusses the design options for currency formatting, including the requirements and the proposed designs to address them.

### Requirements

1. **Format length**: There are multiple formatting shapes and lengths. For example:
   - **Long**: `1 US dollar` / `2 US dollars`
   - **Short**: `1 USD`
   - **Narrow**: `$1`
2. **Value representation**: The value can be represented in different notations, such as:
   - **Decimal**: Standard decimal representation (e.g., `100,000.00 USD`).
   - **Scientific**: Scientific notation (e.g., `1E+5 USD`).
   - **Compact**: Compact notation, short or long (e.g., `100K USD`, `100 thousand USD`).
3. **Accounting format**: An option in the configuration to format negative numbers using accounting convention (e.g., `($10.00)` instead of `-$10.00`).

### Designs

#### Option 1: Single struct with generic value representation

In this option, we use a single `CurrencyFormatter<T>` struct where `T` represents the value representation (Decimal, Compact, or Scientific). This provides a unified type while allowing constructors to be partitioned by capability using trait bounds and concrete implementations.

```mermaid
graph TD
    CF["CurrencyFormatter<br/>&lt;T&gt;"] --> Dec["T = Decimal"]
    CF --> Comp["T = Compact"]
    CF --> Sci["T = Scientific"]

    Dec --> DecCons["Constructors:<br>- try_new_long()<br>- try_new_short()<br>- try_new_narrow()"]
    Comp --> CompCons["Constructors:<br>- try_new_long_compact()<br>- try_new_short_compact()<br>- try_new_narrow_compact()<br>- try_new_long_verbose()"]
    Sci --> SciCons["Constructors:<br>- try_new_long_scientific()<br>- try_new_short_scientific()<br>- try_new_narrow_scientific()"]
```

```rust
pub trait ValueRepresentation {}

pub struct Decimal;
impl ValueRepresentation for Decimal {}

pub struct Compact;
impl ValueRepresentation for Compact {}

pub struct Scientific;
impl ValueRepresentation for Scientific {}

pub struct CurrencyFormatter<T: ValueRepresentation> {
    // ...
    _marker: core::marker::PhantomData<T>,
}

impl CurrencyFormatter<Decimal> {
    /// Creates a currency formatter for long formatting.
    pub fn try_new_long(...) -> Result<Self, DataError>;

    /// Creates a currency formatter for short formatting.
    pub fn try_new_short(...) -> Result<Self, DataError>;

    /// Creates a currency formatter for narrow formatting.
    pub fn try_new_narrow(...) -> Result<Self, DataError>;
}

impl CurrencyFormatter<Scientific> {
    /// Creates a currency formatter for long scientific formatting.
    pub fn try_new_long_scientific(...) -> Result<Self, DataError>;

    /// Creates a currency formatter for short scientific formatting.
    pub fn try_new_short_scientific(...) -> Result<Self, DataError>;

    /// Creates a currency formatter for narrow scientific formatting.
    pub fn try_new_narrow_scientific(...) -> Result<Self, DataError>;
}

impl CurrencyFormatter<Compact> {
    /// Creates a currency formatter for long compact formatting.
    pub fn try_new_long_compact(...) -> Result<Self, DataError>;

    /// Creates a currency formatter for short compact formatting.
    pub fn try_new_short_compact(...) -> Result<Self, DataError>;

    /// Creates a currency formatter for narrow compact formatting.
    pub fn try_new_narrow_compact(...) -> Result<Self, DataError>;

    /// Creates a currency formatter for long compact formatting with verbose names (future).
    pub fn try_new_long_verbose(...) -> Result<Self, DataError>;
}
```

#### Option 2: Separate structs for each formatting style

In this option, we define separate structs for each major formatting style. This allows each struct to only load the data it strictly needs, and provides a clearer separation of concerns.

- **`CurrencyFormatter`**: For standard formatting with short or narrow symbols (e.g., `$10.00`, `US$10.00`).
- **`LongCurrencyFormatter`**: For formatting with long currency names (e.g., `10.00 US dollars`).
- **`CompactCurrencyFormatter`**: For compact formatting with short or narrow symbols (e.g., `$12K`).
- **`LongCompactCurrencyFormatter`**: For compact formatting with long currency names (e.g., `12 thousand US dollars`).
- **`ScientificCurrencyFormatter`**: For scientific formatting with short or narrow symbols (e.g., `1E+5 USD`).
- **`LongScientificCurrencyFormatter`**: For scientific formatting with long currency names (e.g., `1E+5 US dollars`).

```mermaid
graph TD
    CF[CurrencyFormatter] --> CF_Desc["Standard short/narrow symbols<br>(e.g., $10.00, US$10.00)"]
    LCF[LongCurrencyFormatter] --> LCF_Desc["Long currency names<br>(e.g., 10.00 US dollars)"]
    CCF[CompactCurrencyFormatter] --> CCF_Desc["Compact with short/narrow symbols<br>(e.g., $12K)"]
    LCCF[LongCompactCurrencyFormatter] --> LCCF_Desc["Compact with long names<br>(e.g., 12 thousand US dollars)"]
    SCF[ScientificCurrencyFormatter] --> SCF_Desc["Scientific with short/narrow symbols<br>(e.g., 1E+5 USD)"]
    LSCF[LongScientificCurrencyFormatter] --> LSCF_Desc["Scientific with long names<br>(e.g., 1E+5 US dollars)"]
```

```rust
// Standard short/narrow formatter
pub struct CurrencyFormatter;

// Long name formatter
pub struct LongCurrencyFormatter;

// Compact short/narrow formatter
pub struct CompactCurrencyFormatter;

// Compact long name formatter
pub struct LongCompactCurrencyFormatter;

// Scientific short/narrow formatter
pub struct ScientificCurrencyFormatter;

// Scientific long name formatter
pub struct LongScientificCurrencyFormatter;
```

#### Option 3: Unified CurrencyFormatter<V: AbstractFormatter> (Selected)

*Proposed and implemented in PRs [#8182](https://github.com/unicode-org/icu4x/pull/8182) and [#8189](https://github.com/unicode-org/icu4x/pull/8189), in alignment with [CLDR-19617](https://unicode-org.atlassian.net/browse/CLDR-19617).*

In this selected design, we merge all currency formatters into a single generic type `CurrencyFormatter<V>`, where `V` is constrained by `AbstractFormatter` from `icu_decimal`. This replaces marker traits with actual functional formatter engines.

```mermaid
graph TD
    CF["CurrencyFormatter&lt;V: AbstractFormatter&gt;"] --> Standard["V = DecimalFormatter"]
    CF --> Compact["V = CompactDecimalFormatter"]

    Standard --> StdCons["Standard Constructors:<br>- try_new_short()<br>- try_new_narrow()<br>- try_new_long()"]
    Compact --> CompCons["Compact Constructors:<br>- try_new_compact_short()<br>- try_new_compact_narrow()<br>- try_new_compact_long()"]
```

```rust
use icu_decimal::{AbstractFormatter, CompactDecimalFormatter, DecimalFormatter};

/// Unified currency formatter powered by an underlying AbstractFormatter.
pub struct CurrencyFormatter<V: AbstractFormatter> {
    value_formatter: V,
    currency_data: CurrencyFormatterData,
}

// Standard currency formatting (wraps DecimalFormatter)
impl CurrencyFormatter<DecimalFormatter> {
    pub fn try_new_short(...) -> Result<Self, DataError>;
    pub fn try_new_narrow(...) -> Result<Self, DataError>;
    pub fn try_new_long(...) -> Result<Self, DataError>;
}

// Compact currency formatting (wraps CompactDecimalFormatter)
impl CurrencyFormatter<CompactDecimalFormatter> {
    pub fn try_new_compact_short(...) -> Result<Self, DataError>;
    pub fn try_new_compact_narrow(...) -> Result<Self, DataError>;
    pub fn try_new_compact_long(...) -> Result<Self, DataError>;
}
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbNjg2NjQxMTEyXX0=
-->