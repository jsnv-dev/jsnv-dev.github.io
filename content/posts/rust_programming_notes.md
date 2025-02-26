---
title: Rust Programming Notes
date: 2025-02-19
draft: false
tags:
  - programming
  - learning
categories:
  - course
keywords:
  - programming
  - learning
---
These are my notes from the [`Rust: The Complete Developer's Guide`](https://www.udemy.com/course/rust-the-complete-developers-guide/) course. You can find the actual codes at my [`GitHub repo`](https://github.com/jsnv-dev/learning_rust_2).

## Core Concepts: The Building Blocks of Rust

### Representing Data with Structs

Structs in Rust are used to create custom data types that group related values together. Unlike classes in object-oriented languages, structs primarily focus on data representation.

A struct is defined using the `struct` keyword followed by a name and a list of fields enclosed in curly braces:

```rust
struct Deck {
    cards: Vec<String>,
}
```

This code defines a struct named `Deck` with a single field named `cards`. The field is of type `Vec<String>`, which is a vector (a growable array) of strings.

To create an instance of a struct, the struct name is used followed by curly braces containing the field values:

```rust
let deck: Deck = Deck { cards: vec![] };
```

In this example:
- `let deck: Deck` declares a variable named `deck` of type `Deck`
- `Deck { cards: vec![] }` creates a new instance of the `Deck` struct with an empty vector for the `cards` field
- `vec![]` is a macro that creates an empty vector

For debugging purposes, adding the `#[derive(Debug)]` attribute above a struct definition enables printing the struct with the `{:?}` or `{:#?}` format specifiers:

```rust
#[derive(Debug)]
struct Deck {
    cards: Vec<String>,
}

// Later in code
println!("Here's your deck: {:#?}", deck);
```

The `#[derive(Debug)]` attribute instructs the Rust compiler to automatically implement the `Debug` trait, which provides a standardized way to format the struct for debugging output.

### Arrays vs Vectors

Rust provides two primary collection types for storing sequences of values: arrays and vectors.

**Arrays** have a fixed size that cannot change after creation. They're defined using square brackets with the type and size specified:

```rust
let suits = ["Hearts", "Spades", "Diamonds"];
let values = ["Ace", "Two", "Three"];
```

**Vectors** are dynamically sized and can grow or shrink at runtime. They're created using the `vec!` macro or `Vec::new()` function:

```rust
let mut cards = vec![];
// or
let mut cards = Vec::new();
```

The key differences between arrays and vectors are:

1. **Size flexibility**: Arrays have a fixed size, while vectors can grow or shrink
2. **Performance**: Arrays have a slight performance advantage since their size is known at compile time
3. **Usage intent**: Arrays signal to other developers that the collection's size won't change

When choosing between arrays and vectors, it should be considered whether the collection's size might need to change during program execution.

### Mutable vs Immutable Bindings

In Rust, variables (called "bindings") are immutable by default. This means once a value is assigned to a variable, it cannot be changed. To create a mutable variable, the `mut` keyword is used:

```rust
// Immutable binding
let cards = vec![];
// cards.push("card"); // Error! Cannot modify immutable binding

// Mutable binding
let mut cards = vec![];
cards.push("card"); // Works!
```

This immutability-by-default design helps prevent unexpected changes to data and is a core aspect of Rust's safety guarantees. When a binding is declared as mutable, it means:

1. The value it points to can be modified
2. The binding itself can be reassigned to point to a different value

```rust
let mut x = 5;
x = 6; // Reassignment works with mutable bindings
```

When working with vectors or other collections, a mutable binding is needed to add or remove elements:

```rust
let mut cards = vec![];
for suit in suits {
    for value in values {
        let card = format!("{} of {}", value, suit);
        cards.push(card); // Requires `cards` to be mutable
    }
}
```

It should always be considered whether a binding truly needs to be mutable. Using immutable bindings where possible helps prevent bugs and makes code easier to reason about.

### Implementations and Methods

Structs can have associated functionality through implementation blocks. An implementation block is created using the `impl` keyword:

```rust
impl Deck {
    fn new() -> Self {
        let suits = ["Hearts", "Spades", "Diamonds"];
        let values = ["Ace", "Two", "Three"];
        let mut cards = vec![];

        for suit in suits {
            for value in values {
                let card = format!("{} of {}", value, suit);
                cards.push(card);
            }
        }
        Deck { cards }
    }
    
    fn shuffle(&mut self) {
        // Implementation of shuffle...
    }
}
```

There are two types of functions that can be defined within an implementation block:

1. **Associated functions**: Functions that don't take `self` as their first parameter
   - Called using the syntax `Deck::new()`
   - Similar to "static methods" in other languages
   - Typically used for constructors or utility functions

2. **Methods**: Functions that take `self` as their first parameter
   - Called using the syntax `deck.shuffle()`
   - Operate on a specific instance of the struct
   - First parameter is one of:
     - `&self` (immutable reference to the instance)
     - `&mut self` (mutable reference to the instance)
     - `self` (takes ownership of the instance)

Methods are used when access to or modification of the data within a struct instance is needed:

```rust
impl Deck {
    fn shuffle(&mut self) {
        let mut rng = thread_rng();
        self.cards.shuffle(&mut rng);
    }
    
    fn deal(&mut self, num_cards: usize) -> Vec<String> {
        self.cards.split_off(self.cards.len() - num_cards)
    }
}
```

The `shuffle` method takes `&mut self` because it needs to modify the cards within the deck. The `deal` method also takes `&mut self` and returns a vector of strings representing the dealt cards.

### Implicit Returns

Rust functions return the value of their last expression if it doesn't end with a semicolon. This is called an implicit return:

```rust
// Explicit return
fn new() -> Self {
    let deck = Deck { cards };
    return deck;
}

// Implicit return (preferred style)
fn new() -> Self {
    Deck { cards }
}
```

Similarly, in conditional expressions:

```rust
fn is_even(num: i32) -> bool {
    if num % 2 == 0 {
        true // No semicolon here - this value is returned
    } else {
        false // No semicolon here - this value is returned
    }
}

// Or more concisely:
fn is_even(num: i32) -> bool {
    num % 2 == 0 // No semicolon - this expression's value is returned
}
```

Implicit returns make the code more concise and are the preferred style in Rust. It should be remembered that adding a semicolon turns an expression into a statement, which doesn't return a value.

### Installing External Crates

Rust uses packages called "crates" for code reuse. The standard library is automatically included, but external crates must be explicitly added.

To add an external crate:

1. The `cargo add` command can be used:
   ```
   cargo add rand
   ```

2. Or `Cargo.toml` can be manually edited:
   ```toml
   [dependencies]
   rand = "0.9.0"
   ```

External crates are listed on [crates.io](https://crates.io), and documentation is often available on [docs.rs](https://docs.rs).

### Using Code from Crates

After adding an external crate, its functionality must be brought into scope using the `use` statement:

```rust
use rand::{seq::SliceRandom, thread_rng};

impl Deck {
    fn shuffle(&mut self) {
        let mut rng = thread_rng();
        self.cards.shuffle(&mut rng);
    }
}
```

The `use` statement employs path syntax:
- `rand` is the crate name
- `seq::SliceRandom` is a path inside the crate
- Using `{}` allows importing multiple items from the same crate

When a crate contains multiple modules, navigation through them may be necessary:

```rust
// Import individual items
use rand::seq::SliceRandom;
use rand::thread_rng;

// Or use nested paths
use rand::{seq::SliceRandom, thread_rng};
```

This improves code readability by avoiding repeated references to the same crate.

## Ownership and Borrowing: Rust's Unique Memory System

### The Basics of Ownership

Ownership is one of Rust's most distinctive features. The ownership system manages memory without a garbage collector while ensuring memory safety at compile time.

The core rules of ownership are:

1. **Every value has a single owner**: At any given time, a value is owned by exactly one variable.
2. **When the owner goes out of scope, the value is dropped**: Rust automatically frees the memory when the owner variable is no longer valid.
3. **When a value is assigned to another variable, ownership is transferred (moved)**.

This example demonstrates ownership transfer:

```rust
fn main() {
    let bank = Bank::new();
    let other_bank = bank;
    
    println!("{:#?}", bank); // Error: use of moved value: 'bank'
}
```

When `bank` is assigned to `other_bank`, the value is moved. After this point, `bank` no longer owns any value, so attempting to use it results in a compile-time error.

The ownership system prevents multiple variables from having write access to the same data, eliminating entire categories of bugs like use-after-free and double-free errors.

### Visualizing Ownership and Moves

Understanding when values are moved is crucial for working with Rust. Here are several examples:

1. **Assigning to another variable**:
   ```rust
   let account = Account::new(1, String::from("me"));
   let other_account = account;
   // account is now invalid
   ```

2. **Passing to a function**:
   ```rust
   fn print_account(account: Account) {
       println!("{:#?}", account);
   }
   
   let account = Account::new(1, String::from("me"));
   print_account(account);
   // account is now invalid
   ```

3. **Storing in a collection**:
   ```rust
   let account = Account::new(1, String::from("me"));
   let list_of_accounts = vec![account];
   // account is now invalid
   ```

4. **Accessing fields after partial moves**:
   ```rust
   fn print_holder(holder: String) {
       println!("{}", holder);
   }
   
   let account = Account::new(1, String::from("me"));
   print_holder(account.holder);
   // account.holder is now invalid, but other fields remain
   ```

These examples show how ownership is moved in different contexts. Understanding these patterns helps prevent common errors in Rust programming.

### Writing Useful Code with Ownership

The ownership system might initially seem restrictive, but Rust provides mechanisms to work within these constraints. One approach is to return values that were passed in:

```rust
fn print_account(account: Account) -> Account {
    println!("{:#?}", account);
    account // Return the account so the caller regains ownership
}

fn main() {
    let mut account = Account::new(1, String::from("me"));
    account = print_account(account); // Regain ownership
    account = print_account(account); // Can use it again
}
```

While this pattern works, it can become cumbersome for complex functions. This leads to Rust's borrowing system, which provides a more elegant solution.

### Introducing the Borrow System

Borrowing allows referencing a value without taking ownership of it. This is done using references, created with the `&` operator:

```rust
fn print_account(account: &Account) {
    println!("{:#?}", account);
}

fn main() {
    let account = Account::new(1, String::from("me"));
    let account_ref = &account;
    
    print_account(account_ref);
    println!("{:#?}", account); // Still valid because ownership wasn't transferred
}
```

References are like pointers in other languages but with safety guarantees. They ensure the referenced data remains valid for the reference's lifetime and prevent data races.

By default, references are immutable (read-only). The key benefits of references include:

1. The original owner retains ownership
2. Multiple references can access the same data simultaneously
3. The data remains available after the function call

This allows for more flexible and ergonomic code while maintaining Rust's safety guarantees.

### Immutable References

Immutable references (also called "shared references") allow read-only access to data. Key rules for immutable references:

1. Any number of immutable references to a value can exist simultaneously
2. While immutable references exist, the original owner cannot modify the value
3. References must not outlive the data they refer to

```rust
fn main() {
    let account = Account::new(1, String::from("me"));
    let account_ref1 = &account;
    let account_ref2 = &account;
    
    print_account(account_ref1);
    print_account(account_ref2);
    println!("{:#?}", account); // Still valid
}
```

The ability to have multiple immutable references stems from Rust's commitment to preventing data races. Since these references are read-only, multiple parts of code can safely access the data simultaneously.

This pattern is particularly useful for:
- Passing large structures to functions without copying them
- Sharing access to data across multiple functions
- Implementing algorithms that need to examine but not modify data

### Mutable References

When modification of borrowed data is needed, mutable references can be used:

```rust
fn change_account(account: &mut Account) {
    account.balance = 10;
}

fn main() {
    let mut account = Account::new(1, String::from("me"));
    change_account(&mut account);
    println!("{:#?}", account); // Shows modified account
}
```

Mutable references have stricter rules:

1. Only one mutable reference to a particular piece of data can exist at a time
2. Both mutable and immutable references to the same data cannot exist simultaneously
3. The owner cannot modify or access the data while a mutable reference exists

These restrictions prevent data races at compile time. A data race occurs when:
- Two or more pointers access the same data simultaneously
- At least one of them is writing
- There's no synchronization

By enforcing these rules, Rust eliminates an entire category of concurrency bugs without runtime overhead.

### Copy-able Values

Some simple types in Rust are automatically copied instead of moved when assigned or passed to functions:

```rust
fn main() {
    let num = 5;
    let other_num = num; // num is copied, not moved
    
    println!("{} {}", num, other_num); // Both valid
}
```

Types that implement the `Copy` trait behave this way. These include:
- All integer types (`i32`, `u64`, etc.)
- Boolean (`bool`)
- Floating point types (`f32`, `f64`)
- Characters (`char`)
- Tuples if all elements are `Copy` (e.g., `(i32, bool)`)
- Arrays of fixed size if the elements are `Copy`
- References (`&T`)

Understanding which types are `Copy` helps predict when values will be copied rather than moved. This behavior can initially seem inconsistent but becomes intuitive with practice.


## Enums Unleashed: Pattern Matching and Options

### Defining Enums

Enums (enumerations) in Rust allow defining a type that can be one of several variants. Unlike enums in many other languages, Rust enums can contain data within each variant:

```rust
#[derive(Debug)]
enum Media {
    Book { title: String, author: String },
    Movie { title: String, director: String },
    Audiobook { title: String },
}
```

This enum defines a `Media` type that can be either a `Book`, `Movie`, or `Audiobook`, each containing different data fields. This makes enums in Rust much more powerful than simple enumerations in most languages.

The `#[derive(Debug)]` attribute enables printing enum instances for debugging purposes.

### Declaring Enum Values

To create an instance of an enum, the variant is specified and the required data is provided:

```rust
let audiobook = Media::Audiobook {
    title: String::from("An Audiobook"),
};

let good_movie = Media::Movie {
    title: String::from("Good Movie"),
    director: String::from("Good Director"),
};

let bad_book = Media::Book {
    title: String::from("Bad Book"),
    author: String::from("Bad Author"),
};
```

Each variant uses the `::` syntax to indicate it's part of the `Media` enum. The fields are provided using struct-like syntax within curly braces.

### Adding Implementations to Enums

Like structs, enums can have methods through implementation blocks:

```rust
impl Media {
    fn description(&self) -> String {
        if let Media::Book { title, author } = self {
            format!("Book: {} {}", title, author)
        } else if let Media::Movie { title, director } = self {
            format!("Movie: {} {}", title, director)
        } else if let Media::Audiobook { title } = self {
            format!("Audiobook {}", title)
        } else {
            String::from("Media description")
        }
    }
}
```

This implementation adds a `description` method to all `Media` values. Inside the method, pattern matching is used to determine which variant the `self` value is and extract its fields.

### Pattern Matching with Enums

Pattern matching is a powerful feature in Rust that works particularly well with enums. The `match` statement provides a concise way to handle different enum variants:

```rust
impl Media {
    fn description(&self) -> String {
        match self {
            Media::Book { title, author } => {
                format!("Book: {} {}", title, author)
            }
            Media::Movie { title, director } => {
                format!("Movie: {} {}", title, director)
            }
            Media::Audiobook { title } => {
                format!("Audiobook: {}", title)
            }
            Media::Podcast(id) => {
                format!("Podcast: {}", id)
            }
            Media::Placeholder => {
                format!("Placeholder")
            }
        }
    }
}
```

The `match` expression:
1. Takes a value to match against (`self` in this case)
2. Lists patterns to compare against the value
3. Executes the code associated with the first matching pattern
4. Is exhaustive - all possible values must be covered

Pattern matching can also destructure the data within enum variants, allowing direct access to the fields.

### When to Use Structs vs Enums

The choice between structs and enums is based on how data is organized:

- **Enums are used when**: A fixed set of variants exists where each variant might have different fields but shares the same set of methods. Enums are ideal for representing data that can be one of several distinct types.

- **Structs are used when**: A single concept with a fixed set of fields exists. Structs are better for representing a single data type with consistent attributes.

A good rule of thumb: If all variants need the same methods with the same signatures, an enum is appropriate. If different variants need different methods, separate structs might be better.

### Unlabeled Fields

Enum variants can contain data in different ways:

1. **Named fields** (like structs):
   ```rust
   Book { title: String, author: String }
   ```

2. **Unnamed fields** (tuple-like):
   ```rust
   Podcast(u32)
   ```

3. **No data** (unit-like):
   ```rust
   Placeholder
   ```

Unnamed fields are accessed by position rather than by name:

```rust
match media {
    Media::Podcast(episode_number) => {
        format!("Podcast: {}", episode_number)
    }
    // Other cases...
}
```

Unit-like variants are useful for simple flags or states that don't need associated data.

### The Option Enum

`Option` is a built-in enum in Rust's standard library that represents a value that might be present or absent:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`Option<T>` is generic over type `T`, meaning it can wrap any type. It's used instead of null references found in many other languages.

```rust
fn get_by_index(index: usize) -> Option<&Media> {
    if index < self.items.len() {
        Some(&self.items[index])
    } else {
        None
    }
}
```

To use an `Option`, both the `Some` and `None` cases must be handled:

```rust
match catalog.get_by_index(0) {
    Some(value) => {
        println!("Item: {:#?}", value);
    }
    None => {
        println!("Nothing at that index");
    }
}
```

This forces explicit handling of the case where a value might be missing, preventing null pointer exceptions at runtime.

### Option From Another Perspective

To understand `Option` better, a similar enum can be implemented:

```rust
enum MightHaveAValue<'a> {
    ThereIsAValue(&'a Media),
    NoValueAvailable,
}

fn get_by_index(&self, index: usize) -> MightHaveAValue {
    if index < self.items.len() {
        MightHaveAValue::ThereIsAValue(&self.items[index])
    } else {
        MightHaveAValue::NoValueAvailable
    }
}
```

This custom enum serves the same purpose as `Option` - representing a value that might not exist. The standard `Option` enum is preferred in practice, but building a custom one helps understand the concept.

### Replacing a Custom Enum with Option

Refactoring to use the standard `Option` enum is straightforward:

```rust
fn get_by_index(&self, index: usize) -> Option<&Media> {
    if index < self.items.len() {
        Some(&self.items[index])
    } else {
        None
    }
}

// Usage
match catalog.get_by_index(index) {
    Some(value) => {
        println!("Item in pattern match: {:#?}", value);
    }
    None => {
        println!("No value here!!!");
    }
}
```

Using `Option` brings several benefits:
1. Standard, widely recognized API
2. Access to many built-in methods for working with optional values
3. Compiler optimizations specific to this common enum

### Other Ways of Handling Options

Beyond `match` statements, `Option` provides several methods for extracting or working with values:

1. **`unwrap()`** - Get the value or panic if `None`:
   ```rust
   let item = catalog.get_by_index(0).unwrap(); // Panics if None
   ```

2. **`expect()`** - Like `unwrap()` but with a custom error message:
   ```rust
   let item = catalog.get_by_index(0).expect("Expected there to be an item here");
   ```

3. **`unwrap_or()`** - Get the value or a default if `None`:
   ```rust
   let item = catalog.get_by_index(40).unwrap_or(&placeholder);
   ```

These methods should be used carefully:
- `unwrap()` and `expect()` are appropriate for prototyping or when certainty about a value's existence is established
- `unwrap_or()` is safer for production code as it handles the `None` case gracefully

For robust error handling, `match` or `if let` statements are usually preferred.

## Project Architecture: Mastering Modules in Rust

### Modules Overview

Modules in Rust organize code into logical units, improving maintainability and readability. They allow:

1. Grouping related code together
2. Controlling the visibility of items (public/private)
3. Creating clear namespaces

There are three main ways to create modules:

1. **Inline modules** within a file:
   ```rust
   mod content {
       pub enum Media { /*...*/ }
       pub struct Catalog { /*...*/ }
   }
   ```

2. **File-based modules** (one module per file):
   ```rust
   // content.rs
   pub enum Media { /*...*/ }
   pub struct Catalog { /*...*/ }
   
   // main.rs
   mod content;
   ```

3. **Directory-based modules** with submodules:
   ```
   src/
   ├── main.rs
   └── content/
       ├── mod.rs
       ├── media.rs
       └── catalog.rs
   ```

Each approach has different organizational implications and visibility rules.

### Rules of Modules

Understanding module visibility rules is essential:

1. **Items are private by default** - The `pub` keyword is used to make items visible outside their module
2. **Child modules see parent modules** - Code in child modules can access items in parent modules
3. **Parent modules don't automatically see child modules** - Parent modules need explicit `pub` declarations to access child items
4. **Siblings don't see each other** - Sibling modules need `pub` declarations to access each other's items

For directory-based modules:
1. Each directory creates a module named after the directory
2. Each directory must contain a `mod.rs` file that serves as the module's root
3. A directory's `mod.rs` must explicitly declare submodules with `pub mod filename;`

This structure creates a module hierarchy that matches the file system.

### Refactoring with Multiple Modules

A practical example of refactoring code into modules:

```rust
// Before: Everything in main.rs
enum Media { /*...*/ }
impl Media { /*...*/ }
struct Catalog { /*...*/ }
impl Catalog { /*...*/ }

// After refactoring:
// src/main.rs
mod content;
use content::media::Media;
use content::catalog::Catalog;

// src/content/mod.rs
pub mod media;
pub mod catalog;

// src/content/media.rs
#[derive(Debug)]
pub enum Media { /*...*/ }
impl Media { /*...*/ }

// src/content/catalog.rs
use super::media::Media;
#[derive(Debug)]
pub struct Catalog {
    items: Vec<Media>,
}
impl Catalog { /*...*/ }
```

This refactoring separates concerns and creates a clear module hierarchy.

Key points:
1. `mod content;` in `main.rs` tells Rust to look for a module named "content"
2. `pub mod media;` in `content/mod.rs` re-exports the media module
3. `use super::media::Media;` in `catalog.rs` references a module from a parent directory using `super`
4. The `pub` keyword is used strategically to control visibility

The refactored code is more maintainable and better organized, with clear separation between components.

## Handling the Unexpected: Errors and Results

### Project Overview

In this section, an application for processing log files will be built. The primary focus is on handling errors effectively when working with file operations in Rust.

The application will:
1. Read content from a `logs.txt` file
2. Extract and filter error lines
3. Write these errors to a new file
4. Handle any potential errors that may occur during these operations

This project serves as an introduction to Rust's error handling mechanisms, particularly the `Result` enum.

### Reading a File

The first task is to read a file's contents. In Rust, file operations are handled through the standard library's `fs` module.

```rust
use std::fs;

fn main() {
    let text = fs::read_to_string("logs.txt");
    
    println!("{:#?}", text);
}
```

When running this code, something interesting appears in the console output - there's an `Ok` wrapping the file content. This is the first encounter with Rust's `Result` enum, which will be explored in depth.

### The Result Enum

The `Result` enum is a fundamental part of error handling in Rust. Similar to the `Option` enum (which represents a value that might be present or absent), the `Result` enum represents an operation that might succeed or fail.

The `Result` enum has two variants:
1. `Ok(T)` - Contains a success value of type `T`
2. `Err(E)` - Contains an error value of type `E`

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

To understand this concept better, a simple `divide` function can be implemented:

```rust
use std::io::Error;

fn divide(a: f64, b: f64) -> Result<f64, Error> {
    if b == 0.0 {
        Err(Error::other("Can't divide by 0"))
    } else {
        Ok(a / b)
    }
}
```

This function returns a `Result` with:
- `Ok` variant containing the division result when successful
- `Err` variant containing an error when division by zero is attempted

The `Result` type is used whenever an operation might fail, making errors explicit and ensuring they are handled appropriately.

### The Result Enum in Action

To use the `divide` function and handle potential errors, a `match` statement can be used to check which variant was received:

```rust
fn main() {
    match divide(5.0, 0.0) {
        Ok(result_of_division) => {
            println!("{}", result_of_division);
        }
        Err(what_went_wrong) => {
            println!("{}", what_went_wrong);
        }
    }
}
```

The `match` statement allows explicit handling of both success and failure cases. When calling `divide(5.0, 0.0)`, the function returns an `Err` variant, and the second arm of the `match` statement executes.

### Types of Errors

In the `divide` function, `Error` from the `std::io` module was used. This is just one of many error types available in Rust.

```rust
use std::io::Error;

fn divide(a: f64, b: f64) -> Result<f64, Error> {
    if b == 0.0 {
        Err(Error::other("Can't divide by 0"))
    } else {
        Ok(a / b)
    }
}
```

The standard library contains various error types for different modules:
- `std::io::Error` for input/output operations
- `std::fmt::Error` for formatting operations
- `std::num::ParseIntError` for integer parsing errors

Different functions across the standard library return these specific error types to provide context about what went wrong.

### Empty OK Variants

Sometimes, operations don't have a meaningful value to return upon success but can still fail. For these cases, Rust uses a special type called the unit type, written as `()` (empty parentheses).

For example, a simple email validation function can be created:

```rust
fn validate_email(email: String) -> Result<(), Error> {
    if email.contains("@") {
        Ok(())
    } else {
        Err(Error::other("emails must have an @"))
    }
}
```

When matching on a `Result` with an empty `Ok` variant, `Ok(..)` can be used to acknowledge the value without binding it:

```rust
match validate_email(String::from("asdf@asdf.com")) {
    Ok(..) => println!("email is valid"),
    Err(reason_this_failed_validation) => {
        println!("{}", reason_this_failed_validation)
    }
}
```

The `(..)` syntax indicates recognition that there's a value in the `Ok` variant, but it doesn't need to be used.

### Using a Result When Reading Files

With an understanding of the `Result` enum, the file reading example can be revisited and potential errors handled:

```rust
use std::fs;

fn main() {
    match fs::read_to_string("logs.txt") {
        Ok(text_that_was_read) => {
            println!("{}", text_that_was_read.len());
        }
        Err(why_this_failed) => {
            println!("Failed to read file: {}", why_this_failed)
        }
    }
}
```

This pattern explicitly handles both success and failure cases. If the file exists and is readable, its length will be seen. If not, an error message explaining why the operation failed will be displayed.

### Tricky Strings

Before proceeding with the log processing application, understanding how Rust handles strings is necessary. Unlike many other languages, Rust has several string types that serve different purposes.

Three common string types can be examined:

```rust
fn string_test(a: String, b: &String, c: &str) {}

fn main() {
    string_test(
        "red".to_string(),
        &String::from("red"),
        String::from("red").as_str(),
    );
}
```

1. `String` - An owned, growable string (on heap)
2. `&String` - A reference to a `String`
3. `&str` - A string slice, which is a reference to string data owned by another value

### The Stack and Heap

To understand these string types better, understanding how Rust uses memory is necessary:

1. **Stack**: Fast but limited size memory (2-8MB). Stores values with known, fixed sizes.
2. **Heap**: Slower but can grow to store large amounts of data.
3. **Data segment**: Stores literal values written directly into source code.

When a vector or a `String` is created, Rust uses a common pattern:
- Store metadata (length, capacity, pointer) on the stack
- Store the actual data on the heap

```rust
let numbers = vec![1, 2, 3, 4, 5];
```

In this example:
- The stack stores a `Vec` struct with length (5), capacity (5), and a pointer
- The heap stores the actual numbers [1, 2, 3, 4, 5]

This approach prevents stack overflow when data grows large while maintaining efficient access.

### Strings, String Refs, and String Slices

Now the application of these memory concepts to the three string types can be examined:

1. `String` (e.g., `let color = String::from("blue");`)
   - Stack: Contains length, capacity, and pointer
   - Heap: Contains the actual text data "blue"

2. `&String` (e.g., `let color_ref = &color;`)
   - A reference to the `String` struct on the stack
   - Points to the metadata, not directly to the text

3. `&str` (e.g., `let color_slice = "blue";`)
   - Stack: Contains pointer and length
   - Points directly to text data (either in the data segment for literals or on the heap for slices of owned strings)

A key advantage of `&str` is efficiency when working with string literals or portions of existing strings. Unlike creating a new `String`, a string slice doesn't require new heap allocations.

### When to Use Which String

Based on the above distinctions, here's when to use each string type:

1. `String` - Used when:
   - Ownership of text data is needed
   - A string that can grow or shrink is needed
   - Strings need to be stored in a collection

2. `&String` - Rarely used explicitly, as Rust automatically converts `&String` to `&str` in most contexts

3. `&str` - Used when:
   - Ownership is not needed
   - Reference to all or part of an existing string is desired
   - Working with string literals

### Finding Error Logs

Now the function to extract error logs from file content can be implemented:

```rust
fn extract_errors(text: &str) -> Vec<String> {
    let split_text = text.split("\n");
    
    let mut results = vec![];
    
    for line in split_text {
        if line.starts_with("ERROR") {
            results.push(line.to_string());
        }
    }
    
    results
}
```

This function:
1. Takes a string slice (`&str`) as input
2. Splits it by newline characters
3. Checks each line to see if it starts with "ERROR"
4. Collects matching lines into a vector

In the main function, this can now be used to extract errors:

```rust
fn main() {
    match fs::read_to_string("logs.txt") {
        Ok(text_that_was_read) => {
            let error_logs = extract_errors(text_that_was_read.as_str());
            println!("{:#?}", error_logs)
        }
        Err(why_this_failed) => {
            println!("Failed to read file: {}", why_this_failed)
        }
    }
}
```

### Understanding the Issue

When modifying the code to declare `error_logs` outside the match statement, a lifetime error is encountered:

```rust
fn main() {
    let mut error_logs = vec![];
    
    match fs::read_to_string("logs.txt") {
        Ok(text_that_was_read) => {
            error_logs = extract_errors(text_that_was_read.as_str());
        }
        Err(why_this_failed) => {
            println!("Failed to read file: {}", why_this_failed)
        }
    }
    
    println!("{:#?}", error_logs);
}
```

This error occurs because:
1. `text_that_was_read` is a `String` that owns its data
2. `text_that_was_read.as_str()` creates a string slice that borrows from that `String`
3. The `extract_errors` function returns a vector of string slices that borrow from `text_that_was_read`
4. When `text_that_was_read` goes out of scope at the end of the match arm, those borrowed slices become invalid

### Fixing Errors Around String Slices

To fix this issue, the `extract_errors` function needs to be modified to take ownership of the strings it returns:

```rust
fn extract_errors(text: &str) -> Vec<String> {
    let split_text = text.split("\n");
    
    let mut results = vec![];
    
    for line in split_text {
        if line.starts_with("ERROR") {
            results.push(line.to_string());
        }
    }
    
    results
}
```

By changing the return type from `Vec<&str>` to `Vec<String>` and converting each line to an owned `String` with `to_string()`, the returned values don't depend on the lifetime of the input text.

### Writing Data to a File

Now that the error logs have been obtained, the final step is to write them to a file:

```rust
fn main() {
    match fs::read_to_string("logs.txt") {
        Ok(text_that_was_read) => {
            let error_logs = extract_errors(text_that_was_read.as_str());
            
            match fs::write("errors.txt", error_logs.join("\n")) {
                Ok(..) => println!("Wrote errors.txt"),
                Err(reason_write_failed) => {
                    println!("Writing of errors.txt failed: {}", reason_write_failed);
                }
            }
        }
        Err(why_this_failed) => {
            println!("Failed to read file: {}", why_this_failed)
        }
    }
}
```

Here, `fs::write` is used to write the joined error logs to a file. This function also returns a `Result`, which is handled with another match statement.

The nested match statements work but make the code harder to read.

### Alternatives to Nested Matches

To simplify error handling, methods provided by the `Result` enum can be used:

```rust
fn main() {
    let text = fs::read_to_string("logs.txt").expect("failed to read logs.txt");
    let error_logs = extract_errors(text.as_str());
    fs::write("errors.txt", error_logs.join("\n")).expect("failed to write errors.txt");
}
```

The `expect` method:
1. Unwraps the value from an `Ok` variant
2. Causes a panic with the provided message if the variant is `Err`

This approach is more concise but will terminate the program if an error occurs.

### The Try Operator

Rust provides an even more elegant solution with the `?` operator (try operator):

```rust
use std::io::Error;

fn main() -> Result<(), Error> {
    let text = fs::read_to_string("logs.txt")?;
    let error_logs = extract_errors(text.as_str());
    fs::write("errors.txt", error_logs.join("\n"))?;

    Ok(())
}
```

The `?` operator:
1. Unwraps the value from an `Ok` variant
2. Returns early from the function with the error if the variant is `Err`

This approach requires the function to return a `Result` type.

### When to Use Each Technique

Each error handling technique has its appropriate use case:

1. **Match statements** - Used when there is a meaningful way to handle the error in the current function
   ```rust
   match fs::read_to_string("config.txt") {
       Ok(config) => parse_config(config),
       Err(_) => use_default_config()
   }
   ```

2. **Expect/unwrap** - Used for quick prototyping or when an error would indicate a programming mistake
   ```rust
   let config = fs::read_to_string("config.txt").expect("Config file must exist");
   ```

3. **Try operator (`?`)** - Used when errors should be propagated to the calling function
   ```rust
   fn read_config() -> Result<Config, Error> {
       let text = fs::read_to_string("config.txt")?;
       Ok(parse_config(text))
   }
   ```

The recommended approach for production code is to use the `?` operator in functions that return `Result`, allowing errors to bubble up to a point where they can be meaningfully handled.

## Iterator Deep Dive: Efficient Data Processing

### Project Overview

In this section, iterators, a powerful feature in Rust for processing collections of data, will be explored. Iterators facilitate clean, efficient code when working with sequences of elements.

A small project that performs various operations on vectors of strings will be built, using iterators to:
1. Print elements
2. Shorten strings
3. Convert strings to uppercase
4. Move elements between collections
5. Split strings into characters
6. Find elements with fallbacks

### Basics of Iterators

An iterator in Rust is an object that allows traversal through elements in a collection. Iterators are the foundation of many looping constructs in Rust.

Starting with a basic vector and exploring iterators:

```rust
fn main() {
    let colors = vec![
        String::from("red"),
        String::from("green"),
        String::from("blue"),
    ];

    let mut colors_iter = colors.iter();

    println!("{:#?}", colors_iter.next());
    println!("{:#?}", colors_iter.next());
    println!("{:#?}", colors_iter.next());
    println!("{:#?}", colors_iter.next());
}
```

When executed, this code will output:
```
Some("red")
Some("green")
Some("blue")
None
```

An iterator is created with the `iter()` method. The key things to understand about iterators:

1. Iterators are separate from the collections they iterate over
2. The `next()` method advances the iterator and returns the next element
3. Elements are wrapped in `Some` until exhausted, then `None` is returned
4. Iterators are mutable because they maintain internal state about their position

Behind the scenes, an iterator maintains:
- A reference to the collection
- A position pointer to the current element
- A pointer to the end of the collection

### Using For Loops with Iterators

For everyday use, calling `next()` manually is rare. Instead, `for` loops are used, which handle iterators automatically:

```rust
fn print_elements(elements: &Vec<String>) {
    for element in elements {
        println!("{}", element);
    }
}
```

When a `for` loop is used, Rust:
1. Creates an iterator from the collection
2. Repeatedly calls `next()` on that iterator
3. Unwraps the `Some` values automatically
4. Stops the loop when `None` is encountered

This makes iterating much cleaner and less error-prone.

### Iterator Consumers

Beyond `for` loops, Rust provides higher-level functions for working with iterators. These functions are called "consumers" because they consume the iterator, producing a final result:

```rust
fn print_elements(elements: &Vec<String>) {
    elements
        .iter()
        .for_each(|element| {
            println!("{}", element);
        });
}
```

The `for_each` method is an iterator consumer. It:
1. Takes a closure (anonymous function)
2. Applies that closure to each element
3. Automatically handles calling `next()` until exhaustion

Important: Iterators in Rust are "lazy", meaning they don't do any work until a consumer is called. Creating an iterator with `iter()` doesn't start iteration - calling a consumer like `for_each()` does.

### Iterator Adaptors

Iterator adaptors transform elements as they flow through the iterator pipeline. Unlike consumers, adaptors don't cause iteration to occur - they're intermediate operations:

```rust
fn print_elements(elements: &Vec<String>) {
    elements
        .iter()
        .map(|element| format!("{} {}", element, element))
        .for_each(|element| {
            println!("{}", element);
        });
}
```

In this example:
1. `iter()` creates the iterator
2. `map()` is an adaptor that transforms each element
3. `for_each()` is a consumer that causes iteration to happen

If the consumer is removed:
```rust
elements.iter().map(|element| format!("{} {}", element, element));
```
Nothing happens! Rust will warn: "iterators are lazy and do nothing unless consumed."

### Vector Slices

When writing functions that take collections, it's often beneficial to use slices instead of specific collection types:

```rust
// Less flexible - requires a full vector
fn print_elements(elements: &Vec<String>) { ... }

// More flexible - works with full vectors or portions of vectors
fn print_elements(elements: &[String]) { ... }
```

A slice (`&[T]`) is a view into a contiguous sequence of elements. Benefits include:
1. More flexible function signatures
2. Works with any contiguous sequence, not just vectors
3. Allows working with portions of collections without copying

For example, with vector slices:
```rust
print_elements(&colors);         // Use the whole vector
print_elements(&colors[1..3]);   // Use just elements 1 and 2
```

### Reminder on Ownership and Borrowing

Before implementing more iterator functions, it's important to review ownership and borrowing rules:

1. If an argument needs to be **stored** → Take ownership
2. If an argument needs to be **read** → Take a reference (`&T`)
3. If an argument needs to be **modified** → Take a mutable reference (`&mut T`)

For a string-shortening function, the strings need to be modified in place:

```rust
fn shorten_strings(elements: &mut [String]) {
    // Implementation coming soon
}
```

### Iterators with Mutable References

To modify elements in a collection, an iterator that provides mutable references is needed. This is done with `iter_mut()`:

```rust
fn shorten_strings(elements: &mut [String]) {
    elements
        .iter_mut()
        .for_each(|element| {
            element.truncate(1);
        });
}
```

There are three main iterator methods in Rust:
1. `iter()` - Returns immutable references (`&T`)
2. `iter_mut()` - Returns mutable references (`&mut T`)
3. `into_iter()` - Takes ownership of elements (behavior varies)

Using `iter()` in the function above would cause an error because the closure would receive immutable references that it can't modify.

### Collecting Elements from an Iterator

Often transformation of elements and collection into a new collection is desired. The `collect()` method does this:

```rust
fn to_uppercase(elements: &[String]) -> Vec<String> {
    elements
        .iter()
        .map(|element| element.to_uppercase())
        .collect()
}
```

Here:
1. An iterator is created with `iter()`
2. Each element is transformed with `map()`
3. The transformed elements are collected into a new collection with `collect()`

The `collect()` method is special because it can create different collection types.

### How Collect Works

The `collect()` method needs to know what type of collection to create. There are three ways to specify this:

1. Return type inference:
```rust
fn to_uppercase(elements: &[String]) -> Vec<String> {
    elements.iter().map(|e| e.to_uppercase()).collect()
}
```

1. Variable type annotation:
```rust
let uppercase: Vec<String> = elements.iter().map(|e| e.to_uppercase()).collect();
```

1. Turbofish syntax:
```rust
elements.iter().map(|e| e.to_uppercase()).collect::<Vec<String>>()
```

The turbofish syntax is often preferred because it makes the target collection type explicit at the point of collection.

### Moving Ownership with into_iter()

When ownership of elements is needed (rather than just borrowing them), `into_iter()` is used:

```rust
fn move_elements(vec_a: Vec<String>, vec_b: &mut Vec<String>) {
    vec_a
        .into_iter()
        .for_each(|element| {
            vec_b.push(element);
        });
}
```

The behavior of `into_iter()` depends on what it's called on:
1. On a value (`vec_a.into_iter()`): Returns owned values
2. On a reference (`(&vec_a).into_iter()`): Returns immutable references 
3. On a mutable reference (`(&mut vec_a).into_iter()`): Returns mutable references

This function takes ownership of `vec_a` and moves its elements into `vec_b`.

### Inner Maps

Iterators can be used for complex transformations, including nested operations:

```rust
fn explode(elements: &[String]) -> Vec<Vec<String>> {
    elements
        .iter()
        .map(|element| {
            element
                .chars()
                .map(|c| c.to_string())
                .collect::<Vec<String>>()
        })
        .collect::<Vec<Vec<String>>>()
}
```

This function:
1. Takes each string in the input vector
2. Converts it to a vector of single-character strings
3. Returns a vector of these character vectors

Notice how iterators are chained: the outer `map` operates on strings, while the inner `map` operates on characters.

### Reminder on Lifetimes

When working with references in iterators, lifetimes come into play. Consider this function:

```rust
fn find_color_or<'a>(elements: &'a [String], search: &str, fallback: &'a str) -> &'a str {
    elements
        .iter()
        .find(|element| element.contains(search))
        .map_or(fallback, |element| element.as_str())
}
```

The `'a` lifetime annotations ensure that:
1. The returned string slice doesn't outlive the collection it might come from
2. The fallback string lives at least as long as the returned reference

### Iterators Wrapup

The final function demonstrates several advanced iterator concepts:

```rust
fn find_color_or(elements: &[String], search: &str, fallback: &str) -> String {
    elements
        .iter()
        .find(|element| element.contains(search))
        .map_or_else(
            || String::from(fallback),
            |element| element.to_string()
        )
}
```

This function:
1. Uses `find()` to locate the first element containing the search string
2. Uses `map_or_else()` to handle both the found and not-found cases
3. Returns an owned `String` to avoid lifetime complications

The `find()` method is an iterator consumer that returns the first element matching a predicate, wrapped in an `Option`.

## Advanced Lifetimes: Mastering Rust's Memory Model

### Lifetime Annotations

Lifetime annotations are a unique feature of Rust that help ensure memory safety when working with references. They explicitly define the relationship between references' lifetimes.

Starting with a function that finds the next language in a list after a specified current language:

```rust
fn next_language(languages: &[String], current: &str) -> &str {
    let mut found = false;

    for lang in languages {
        if found {
            return lang;
        }

        if lang == current {
            found = true;
        }
    }

    languages.last().unwrap()
}
```

This function receives two references (`&[String]` and `&str`) and returns a reference (`&str`). When compiled, an error about missing lifetime specifiers occurs.

### A Review of Borrowing Rules

Before diving into lifetime annotations, a review of borrowing rules is helpful:

1. Every variable has a lifetime - from when it's created until it goes out of scope or its value is moved
2. References cannot outlive the values they refer to
3. If a reference goes out of scope before the referenced value, that's fine

Consider this code that would cause problems:

```rust
fn main() {
    let result;
    {
        let languages = vec![
            String::from("rust"),
            String::from("go"),
            String::from("typescript"),
        ];
        result = next_language(&languages, "go");
    } // languages is dropped here
    
    println!("{}", result); // ERROR: uses a reference to a dropped value
}
```

This fails because `result` contains a reference to data within `languages`, but `languages` is dropped before `result` is used.

### What Lifetime Annotations Are All About

Lifetime annotations tell the compiler about the relationship between references' lifetimes. When a function:
1. Takes multiple references as input
2. Returns a reference

The compiler needs to know which input reference's lifetime is connected to the output reference's lifetime.

For the function, lifetime annotations are added like this:

```rust
fn next_language<'a>(languages: &'a [String], current: &str) -> &'a str {
    // Same implementation as before
}
```

These annotations tell the compiler:
- There is a lifetime `'a`
- Both `languages` and the return value share this lifetime
- The return value's lifetime is connected to `languages`, not to `current`

Lifetime annotations don't change how long references live; they describe relationships between lifetimes.

### Lifetime Elision

For simple cases, Rust can automatically determine lifetimes without explicit annotations. This feature is called "lifetime elision." The rules are:

1. Each input reference gets its own lifetime parameter
2. If there's exactly one input lifetime, it's assigned to all output lifetimes
3. If there are multiple input lifetimes but one is `&self` or `&mut self`, the lifetime of `self` is assigned to all output lifetimes

Functions like this don't need explicit annotations:

```rust
fn last_language(languages: &[String]) -> &str {
    languages.last().unwrap()
}
```

Because there's only one reference parameter, Rust automatically assigns its lifetime to the return value.

### Common Lifetimes

One frequently used function demonstrates lifetime annotations clearly:

```rust
fn longest_language<'a>(lang_a: &'a str, lang_b: &'a str) -> &'a str {
    if lang_a.len() >= lang_b.len() {
        lang_a
    } else {
        lang_b
    }
}
```

Here, both input references and the return value share the same lifetime `'a`, meaning:
1. The returned reference will be valid as long as BOTH input references are valid
2. The function can return either input reference safely

This is necessary because the function could return either reference, depending on their lengths, and the compiler must ensure safety regardless of which path is taken.

## Generics and Traits: Writing Flexible, Reusable Code

### Project Setup

Setting up a project to explore generics and traits begins with creating a new Rust project and adding necessary dependencies. This forms the foundation for learning these important Rust concepts.

```rust
// First, create a new project
// $ cargo new generics

// Add required dependencies in Cargo.toml
[dependencies]
num-traits = "0.2.19"
```

The `num-traits` crate provides useful traits for numeric types, which will be essential for demonstrating generic programming with numbers. This crate extends functionality for various numeric types through traits.

With the project structure in place, a simple initial implementation can be started:

```rust
fn solve(a: f64, b: f64) -> f64 {
    (a.powi(2) + b.powi(2)).sqrt()
}

fn main() {
    let a = 3.0;
    let b = 4.0;

    println!("{}", solve(a, b));
}
```

This function implements the Pythagorean theorem to calculate the hypotenuse of a right triangle. While simple, it will serve as the foundation for exploring generics, as it currently works only with `f64` values.

### Issues with Number Types

Rust's strong type system becomes apparent when working with numeric types. Unlike some languages that automatically convert between number types, Rust requires explicit type handling.

Consider this modification to the previous code:

```rust
use num_traits::ToPrimitive;

fn solve(a: f64, b: f64) -> f64 {
    (a.powi(2) + b.powi(2)).sqrt()
}

fn main() {
    let a: f32 = 3.0;
    let b: f64 = 4.0;

    let a_f64 = a.to_f64().unwrap();

    println!("{}", solve(a_f64, b));
}
```

This code demonstrates two important points about Rust's numeric types:

1. **No automatic type conversion**: Trying to pass an `f32` to a function expecting an `f64` results in a compilation error.
2. **Explicit conversion required**: The `to_f64()` method from `num_traits` is used to convert `f32` to `f64`.

The need for explicit conversion becomes tedious when working with different numeric types. For example:

```rust
// This won't compile: can't add f32 and f64 directly
let a: f32 = 3.0;
let b: f64 = 4.0;
let c = a + b; // Error!
```

This strict type handling prevents subtle bugs but creates a challenge: how to write functions that work with different numeric types without excessive conversion code?

### The Basics of Generics

Generics provide a solution to the numeric type conversion problem. They allow writing code that works with multiple types while maintaining type safety.

Here's how the `solve` function can be modified to use generics:

```rust
use num_traits::{Float, ToPrimitive};

fn solve<T: Float>(a: T, b: T) -> f64 {
    let a_f64 = a.to_f64().unwrap();
    let b_f64 = b.to_f64().unwrap();

    (a_f64.powi(2) + b_f64.powi(2)).sqrt()
}

fn main() {
    let a: f32 = 3.0;
    let b: f32 = 4.0;

    println!("{}", solve::<f32>(a, b));
}
```

Breaking down the generic function:

1. `fn solve<T: Float>`: Declares a generic function with type parameter `T` that must implement the `Float` trait
2. `(a: T, b: T)`: Both parameters must be of the same type `T`
3. `-> f64`: The function returns an `f64` regardless of input types
4. Inside the function, both values are converted to `f64` for calculation

The generic type parameter `T` with the `Float` trait bound ensures that:
- The function accepts any floating-point type
- The type must implement methods required by the `Float` trait
- Type safety is maintained (can't mix different types)

When calling the function with `solve::<f32>(a, b)`, the type parameter is explicitly specified, though Rust can often infer it. This tells the compiler to use the `f32` version of the generic function.

Generic functions follow these key principles:
1. Type parameters are declared within angle brackets `<T>`
2. Trait bounds constrain what types can be used
3. The same type parameter can be used multiple times to ensure type consistency
4. Type parameters can be explicitly specified or inferred

### Trait Bounds

Trait bounds specify what capabilities a type must have to be used with a generic function. They act as constraints that ensure type safety while maintaining flexibility.

In the previous example, `Float` is a trait bound:

```rust
fn solve<T: Float>(a: T, b: T) -> f64 {
    // ...
}
```

This bound ensures that:
1. Type `T` must implement the `Float` trait
2. Methods defined in the `Float` trait (like `to_f64()`) can be called on values of type `T`
3. Only appropriate types (floating-point numbers) can be used with this function

Without this bound, the function would fail because it couldn't call methods like `to_f64()` on arbitrary types.

If an inappropriate type is used:

```rust
let a: i32 = 3;  // Integer, not a float
let b: i32 = 4;
println!("{}", solve::<i32>(a, b));  // Compilation error!
```

The compiler produces an error: "the trait bound `i32: Float` is not satisfied". This demonstrates how trait bounds provide compile-time guarantees rather than runtime errors.

Trait bounds can be thought of as a contract:
- The function promises to only use methods defined in the trait
- The type promises to implement all methods required by the trait
- The compiler enforces this contract

This mechanism enables writing flexible, reusable code while maintaining Rust's safety guarantees.

### Multiple Generic Types

Sometimes functions need to work with multiple different types simultaneously. Rust supports this through multiple generic type parameters.

The `solve` function can be modified to accept different types for each parameter:

```rust
use num_traits::{Float, ToPrimitive};

fn solve<T: Float, U: Float>(a: T, b: U) -> f64 {
    let a_f64 = a.to_f64().unwrap();
    let b_f64 = b.to_f64().unwrap();

    (a_f64.powi(2) + b_f64.powi(2)).sqrt()
}

fn main() {
    let a: f32 = 3.0;
    let b: f64 = 4.0;

    println!("{}", solve(a, b));
}
```

Key changes in this version:
1. Two type parameters: `T` and `U`, both with the `Float` trait bound
2. Parameters with different types: `a: T` and `b: U`
3. Type inference works: `solve(a, b)` without explicit type parameters

This function now accepts any combination of floating-point types. The compiler generates the appropriate code based on the actual types used at the call site.

When multiple generic types are used:
1. Each type parameter is declared separately: `<T: Float, U: Float>`
2. Type parameters can have the same or different trait bounds
3. Type inference works as long as the compiler can determine concrete types
4. Each type parameter can be used independently in the function signature

The choice between single vs. multiple generic types depends on requirements:
- Single type parameter (`<T>`) enforces that all uses must be the same type
- Multiple parameters (`<T, U>`) allow different types to be used together
- Type parameters follow naming conventions: typically starting with `T`, then `U`, `K`, etc.

### Super Solve Flexibility

To make the `solve` function work with any numeric type, not just floating-point numbers, a more general trait bound is needed. The `ToPrimitive` trait provides this flexibility:

```rust
use num_traits::ToPrimitive;

fn solve<T: ToPrimitive, U: ToPrimitive>(a: T, b: U) -> f64 {
    let a_f64 = a.to_f64().unwrap();
    let b_f64 = b.to_f64().unwrap();

    (a_f64.powi(2) + b_f64.powi(2)).sqrt()
}

fn main() {
    let a: u8 = 3;
    let b: f64 = 4.0;

    println!("{}", solve(a, b));
}
```

The significant changes are:
1. Replacing `Float` with `ToPrimitive` trait bound
2. Using an integer type (`u8`) for one parameter
3. The function still works with this diverse combination of types

This demonstrates the power of generics combined with appropriate trait bounds:
- The function now works with integers, floating-point numbers, or any type implementing `ToPrimitive`
- Type safety is maintained through the trait bounds
- The function body remains unchanged
- All type checking happens at compile time with no runtime overhead

This approach allows maximum flexibility while ensuring that only appropriate types are used. The trait bound `ToPrimitive` acts as a capability requirement - any type that can be converted to primitive numeric types will work.

The key insight is choosing the right trait bound:
- More specific bounds (like `Float`) restrict to fewer types but provide more capabilities
- More general bounds (like `ToPrimitive`) allow more types but provide fewer guaranteed capabilities
- The ideal bound is the most general one that still provides all needed functionality

### App Overview

With a solid understanding of generics and trait bounds, a more complex application can be developed. This application will demonstrate these concepts in a practical context.

The application will consist of:
1. A `Container` trait defining common container operations
2. A `Basket` implementation that can store a single item
3. A `Stack` implementation that can store multiple items
4. Generic implementations allowing these containers to work with any type

The `Container` trait will define these operations:
- `get` - Retrieve an item from the container
- `put` - Store an item in the container
- `is_empty` - Check if the container has any items

Both `Basket` and `Stack` will implement this trait, but with different internal logic. This demonstrates how traits enable polymorphism - using different types through a common interface.

The architecture looks like this:

```
Container (trait)
├── Basket<T> (struct)
└── Stack<T> (struct)
```

This design enables writing generic functions that work with any container, regardless of its specific implementation. For example, a function could add an item to any container that implements the `Container` trait:

```rust
fn add_to_container<T, C: Container<T>>(container: &mut C, item: T) {
    container.put(item);
}
```

This approach showcases the true power of generics and traits: code that works with any type that satisfies specific behavioral requirements.

### Building the Basket

The first container implementation is the `Basket` - a simple container that holds at most one item. Initially, a non-generic version is implemented:

```rust
// basket.rs
#[derive(Debug)]
pub struct Basket {
    item: Option<String>,
}

impl Basket {
    pub fn new(item: String) -> Self {
        Basket { item: Some(item) }
    }

    pub fn get(&mut self) -> Option<String> {
        self.item.take()
    }

    pub fn put(&mut self, item: String) {
        self.item = Some(item)
    }

    pub fn is_empty(&self) -> bool {
        self.item.is_none()
    }
}
```

Key design decisions in this implementation:
1. Using `Option<String>` to represent an item that might not be present
2. The `get` method uses `take()` to remove the item from the basket
3. The `put` method replaces any existing item
4. `is_empty` checks if the option is `None`

In the main file, the basket can be used:

```rust
// main.rs
mod basket;

use basket::Basket;

fn main() {
    let b1 = Basket::new(String::from("Hi there"));
    // Can use the basket methods here
}
```

This initial implementation is limited to storing strings only. The next step is to make it generic to store any type.

### Generic Structs

To make the `Basket` work with any type, it needs to be converted to a generic struct:

```rust
// basket.rs
#[derive(Debug)]
pub struct Basket<T> {
    item: Option<T>,
}

impl<T> Basket<T> {
    pub fn new(item: T) -> Self {
        Basket { item: Some(item) }
    }

    pub fn get(&mut self) -> Option<T> {
        self.item.take()
    }

    pub fn put(&mut self, item: T) {
        self.item = Some(item)
    }

    pub fn is_empty(&self) -> bool {
        self.item.is_none()
    }
}
```

The key changes are:
1. Adding a type parameter `T` to the struct definition: `Basket<T>`
2. Using `Option<T>` instead of `Option<String>`
3. Adding `<T>` to the implementation block: `impl<T> Basket<T>`
4. Updating all method signatures to use type `T`

This allows creating baskets for any type:

```rust
// main.rs
fn main() {
    let b1 = Basket::new(String::from("Hi there"));
    let b2 = Basket::new(10);
    let b3 = Basket::new(true);
}
```

The syntax of generic structs follows these patterns:
1. The type parameter `T` is declared after the struct name: `struct Basket<T>`
2. Implementation blocks must redeclare the type parameter: `impl<T> Basket<T>`
3. Once a `Basket<T>` is created with a specific type, it can only store values of that type
4. Different instances can use different types: `Basket<String>`, `Basket<i32>`, etc.

The generic implementation maintains type safety while providing flexibility. Each basket instance is specialized for a particular type, enforced by the compiler.

### More on Generic Structs

Next, a `Stack` container is implemented to store multiple items of the same type:

```rust
// stack.rs
pub struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    pub fn new(items: Vec<T>) -> Self {
        Stack { items }
    }

    pub fn get(&mut self) -> Option<T> {
        self.items.pop()
    }

    pub fn put(&mut self, item: T) {
        self.items.push(item);
    }

    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}
```

Like `Basket`, `Stack` is implemented as a generic struct. The key differences are:
1. `Stack` uses a `Vec<T>` to store multiple items
2. `get` returns and removes the last item using `pop()`
3. `put` adds an item to the end using `push()`
4. `is_empty` delegates to the vector's method

Using both generic structs together:

```rust
// main.rs
mod basket;
mod stack;

use basket::Basket;
use stack::Stack;

fn main() {
    let b1 = Basket::new(String::from("Hi there"));
    let b2 = Basket::new(10);
    let b3 = Basket::new(true);

    let s1 = Stack::new(vec![String::from("hi")]);
    let s2 = Stack::new(vec![1, 2, 3]);
}
```

Both container types demonstrate important aspects of generic structs:
1. Type parameters enable creating type-safe containers
2. Each instance is specialized for a specific type
3. The same implementation works for any type
4. The compiler prevents type mismatches

While `Basket` and `Stack` have similar methods, they're completely different types with no shared interface. This limitation will be addressed using traits.

### Implementing a trait

To create a shared interface for different container types, a `Container` trait is defined:

```rust
// container.rs
pub trait Container<T> {
    fn get(&mut self) -> Option<T>;
    fn put(&mut self, item: T);
    fn is_empty(&self) -> bool;
}
```

This trait defines a common interface that any container must implement. It's generic over type `T`, allowing containers to work with any type while maintaining type safety.

Now `Basket` can implement this trait:

```rust
// basket.rs
use super::container::Container;

impl<T> Container<T> for Basket<T> {
    fn get(&mut self) -> Option<T> {
        self.item.take()
    }

    fn put(&mut self, item: T) {
        self.item = Some(item)
    }

    fn is_empty(&self) -> bool {
        self.item.is_none()
    }
}
```

And similarly for `Stack`:

```rust
// stack.rs
use super::container::Container;

impl<T> Container<T> for Stack<T> {
    fn get(&mut self) -> Option<T> {
        self.items.pop()
    }

    fn put(&mut self, item: T) {
        self.items.push(item);
    }

    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}
```

Key points about trait implementation:
1. The implementation uses the syntax: `impl<T> Container<T> for Basket<T>`
2. The methods in the trait don't need the `pub` keyword
3. Method implementations must match the signatures defined in the trait
4. Existing methods can be moved to the trait implementation

In `main.rs`, both modules need to be imported:

```rust
mod basket;
mod container;
mod stack;

use basket::Basket;
use container::Container;
use stack::Stack;
```

With the trait in place, it becomes possible to write functions that work with any container, regardless of its specific implementation.

### Generic Trait Bounds

With the `Container` trait implemented, generic functions can be written that work with any container implementation:

```rust
fn add_string<T: Container<String>>(c: &mut T, s: String) {
    c.put(s);
}

fn main() {
    let mut b1 = Basket::new(String::from("Hi there"));
    let mut s1 = Stack::new(vec![String::from("hi")]);

    add_string(&mut b1, String::from("hi"));
    add_string(&mut s1, String::from("hi"));
}
```

The `add_string` function demonstrates several powerful concepts:
1. Generic type parameter `T` with a trait bound: `T: Container<String>`
2. The bound specifies that `T` must implement `Container` for `String` types
3. The function works with any type implementing this trait
4. Both `Basket<String>` and `Stack<String>` satisfy this bound

Generic trait bounds provide a powerful abstraction mechanism:
1. **Polymorphism**: Different types can be used interchangeably
2. **Type safety**: Compiler ensures type compatibility
3. **Code reuse**: Functions work with any conforming type
4. **Zero runtime cost**: All checking happens at compile time

This pattern enables writing highly flexible, reusable code while maintaining Rust's strong type safety. The compiler generates specialized code for each concrete type, eliminating any runtime overhead from abstraction.

For example, these would fail to compile:
```rust
let mut b2 = Basket::new(10);
add_string(&mut b2, String::from("hi")); // Error! b2 is Basket<i32>, not Container<String>
```

The flexibility comes from focusing on behavior (defined by traits) rather than specific types. This enables writing truly generic algorithms and data structures.

Generic trait bounds represent the culmination of Rust's type system: they combine the flexibility of generics with the abstraction capabilities of traits, all while maintaining type safety and performance.
