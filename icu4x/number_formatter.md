# ICU4X Number Formatter Design

This document describes the design and architecture of number formatting components in ICU4X.

## Overview

ICU4X number formatting is designed to be highly modular, performant, and zero-copy. It ranges from formatting simple decimal numbers to complex dimensions such as currencies and measurement units.

The formatting pipeline follows a layered architecture where complex formatters build upon simpler ones, sharing common data structures and formatting traits.

```mermaid
graph TD
    FD[FixedDecimal] --> DF[DecimalFormatter]
    DF --> CDF[CompactDecimalFormatter]
    DF --> CF[CurrencyFormatter]
    DF --> LCF[LongCurrencyFormatter]
    DF --> CCF[CompactCurrencyFormatter]
    DF --> UF[UnitsFormatter / CategorizedFormatter]
```

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

## Core Principles

- **Zero-Copy**: Data is loaded from the data provider with minimal overhead and no unnecessary copies.
- **Zero-Allocation**: Formatting operations utilize the `Writeable` trait to write directly to a sink (e.g., a string buffer or a stream) without intermediate allocations.
- **Separation of Concerns**: Logic is separated from data. Data is loaded dynamically via `DataProvider`.
- **Type Safety**: Strong types are used to prevent errors (e.g., `CurrencyCode`, `MeasureUnitCategory`).

## Components

### 1. Input: `FixedDecimal`

Instead of formatting floating-point numbers directly (which can introduce rounding errors), ICU4X uses `FixedDecimal` (from the `fixed_decimal` crate) as the primary input for formatting.

- Represents a decimal number with fixed precision.
- Keeps track of sign, integer digits, and fractional digits.
- Supports operations like shifting decimal points without losing precision.

### 2. Decimal Formatting (`icu_decimal`)

The `icu_decimal` crate provides the foundation for all number formatting.

#### `DecimalFormatter`
- Formats `FixedDecimal` into localized representations.
- **Data Dependencies**:
  - `DecimalSymbolsV1`: Contains localized symbols (decimal separator, grouping separator, plus/minus signs).
  - `DecimalDigitsV1`: Contains localized digit characters (e.g., Latin digits, Bangla digits).
- **Features**:
  - Grouping sizes and separators based on locale.
  - Numbering system resolution (e.g., `-u-nu-thai`).

#### `CompactDecimalFormatter`
- Formats numbers in compact notation (e.g., `1.2M` instead of `1,200,000`).
- Uses plural rules to select correct patterns based on the magnitude of the number.

### 3. Currency Formatting (`icu_experimental::dimension::currency`)

Currency formatting is located in the experimental area and supports various widths and styles.

#### `CurrencyFormatter`
- Used for formatting monetary values with short or narrow symbols (e.g., `$10.00`, `US$10.00`).
- **Internal Mechanism**:
  - Resolves the currency symbol and pattern (e.g., `¤{0}` or `{0} ¤`) from `CurrencyEssentialsV1` data.
  - Uses `DecimalFormatter` to format the numeric value.
  - Interpolates the formatted number and currency symbol into the pattern.

#### `LongCurrencyFormatter`
- Used for formatting monetary values with long names (e.g., `10.00 US dollars`).
- **Internal Mechanism**:
  - Requires `PluralRules` to select the correct plural form of the currency name (e.g., "dollar" vs "dollars").
  - Interpolates the formatted number and localized currency name.

#### `CompactCurrencyFormatter`
- Combines compact decimal formatting with currency formatting (e.g., `$12K`).
- **Internal Mechanism**:
  - Obtains the compact decimal pattern and significand.
  - Formats the significand.
  - Interpolates the compact representation into the currency pattern.

### 4. Unit Formatting (`icu_experimental::dimension::units`)

Unit formatting is transitioning to a more type-safe "categorized" approach.

#### `UnitsFormatter` (Legacy)
- A generic formatter for unit values.
- Being deprecated in favor of `CategorizedFormatter`.

#### `CategorizedFormatter`
- A type-safe formatter generic over `MeasureUnitCategory` (e.g., `Area`, `Duration`).
- **Features**:
  - Prevents mixing units of different categories.
  - Uses `UnitsDisplayNamesV1` to load localized unit names.
  - Uses `PluralRules` to select correct patterns based on the value (e.g., "1 meter" vs "2 meters").

## Key Formatting Trait: `Writeable`

All formatters return intermediate structures that implement the `writeable::Writeable` trait.

- Enables formatting directly to a `PartsWrite` sink.
- Supports "parts" annotation, allowing the caller to identify which parts of the output string correspond to the currency symbol, digits, grouping separators, etc. (useful for UI styling).

```rust
pub trait Writeable {
    fn write_to_parts<S: PartsWrite + ?Sized>(&self, sink: &mut S) -> Result<(), Error>;
    // ...
}
```

## Data Loading and Providers

All constructors support both compiled data (`try_new`) and dynamic data loading via `DataProvider` (`try_new_unstable`).

- `try_new_unstable` allows for tree-shaking and dynamic data loading.
- Markers are used to identify the required data (e.g., `DecimalSymbolsV1::MARKER`).
