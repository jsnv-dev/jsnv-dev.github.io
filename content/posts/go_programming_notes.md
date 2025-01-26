---
title: "Go Programming Notes"
date: 2025-01-26T00:00:00Z
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
# Go Programming Notes

These are my notes from the [`Go: The Complete Developer's Guide (Golang)`](https://www.udemy.com/course/go-the-complete-developers-guide/) course. You can find the actual codes at my [`GitHub repo`](https://github.com/jsnv-dev/learning_go).

## Getting Started with Go

### Five Important Questions
Even a simple Go program reveals five fundamental aspects of the language:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hi there")
}
```

Looking at this program raises five essential questions that form the foundation of understanding Go:

1. How do programmers run code in a Go project?
2. What does `package main` signify?
3. What purpose does `import "fmt"` serve?
4. What is the `func` keyword's role?
5. How should code be organized in Go files?

The answers to these questions reveal how Go programs are structured and executed.

### Go Packages
Every Go file must declare its package membership. Two distinct types of packages exist in Go:

1. Executable Packages:
   ```go
   package main
   
   func main() {
       // Entry point for executable programs
   }
   ```
   The word "main" is special - it tells Go to create a program that can be run directly. An executable package must contain a `main()` function.

2. Reusable Packages:
   ```go
   package cards
   
   func newDeck() {
       // Helper code for other packages
   }
   ```
   Any package name besides "main" creates a reusable package, meant to provide helper code and functionality to other packages.

### Import Statements
Imports give access to code from other packages. For example:

```go
import "fmt"
import "io"
```

Or using the grouped syntax:
```go
import (
    "fmt"
    "io"
)
```

Each import makes that package's functionality available in the current file. The `fmt` package, short for "format", provides functions for formatting and printing output.

### File Organization
Go enforces a specific file structure:
1. Package declaration at the top
2. Import statements
3. Code/function declarations

This organization isn't merely conventional - it's required by the Go compiler:

```go
// Required order
package main

import "fmt"

func main() {
    // Code here
}
```

## Working with Variables

### Variable Declarations
Go provides multiple ways to declare variables, each serving different purposes:

```go
// Long form with explicit type
var card string = "Ace of Spades"

// Let Go infer the type
var card = "Ace of Spades"

// Short declaration syntax
card := "Ace of Spades"
```

The `:=` operator both declares and initializes a variable, but it can only be used inside functions. The var keyword works anywhere. Go assigns zero values to variables that aren't explicitly initialized:
- string `""`
- int `0`
- bool `false`

### Functions and Return Types
Functions in Go must declare both their parameter types and return types:

```go
func newCard() string {
    return "Five of Diamonds"
}

// Multiple return values
func deal(d deck, handSize int) (deck, deck) {
    return d[:handSize], d[handSize:]
}
```

The return type appears after the parameter list. If a function declares a return type, the Go compiler ensures it always returns that type of value.

### Slices and For Loops
Slices provide dynamic arrays in Go. The for loop with range makes iteration straightforward:

```go
cards := []string{"Ace of Diamonds", "Four of Clubs"}
cards = append(cards, "Six of Spades")

for i, card := range cards {
    fmt.Println(i, card)
}
```

The range keyword provides both the index and value of each element. Using `_` ignores values you don't need:

```go
for _, card := range cards {
    fmt.Println(card)
}
```

## Object-Oriented Programming vs Go Approach

When creating a deck of cards, an object-oriented language might use a class with properties and methods:

```go
// Theoretical OO approach (not valid Go)
class Deck {
    private cards []string
    
    public shuffle() {
        // shuffle the cards
    }
}
```

Go takes a fundamentally different approach. Instead of creating classes with properties and methods, Go extends existing types and adds behavior to them:

```go
// Go's approach
type deck []string

func (d deck) shuffle() {
    // shuffle the cards
}
```

This difference is significant. Go favors composition over inheritance, and its type system reflects this philosophy. Rather than creating complex hierarchies of classes, Go programmers combine simple types to build more complex ones.

### Custom Type Declarations
Creating a new type in Go involves extending an existing type:

```go
type deck []string
```

This line creates a new type called `deck` that extends the behavior of a slice of strings. This new type inherits all the capabilities of a string slice but can also have its own specific methods. This approach demonstrates Go's focus on simplicity and explicit behavior.

### Receiver Functions
Receiver functions add methods to custom types:

```go
func (d deck) print() {
    for i, card := range d {
        fmt.Println(i, card)
    }
}
```

The `(d deck)` before the function name is called a receiver. It means any variable of type `deck` now has access to the print method. This can be called like:

```go
cards := newDeck()
cards.print()
```

By convention, the receiver variable uses a one or two letter abbreviation that matches the type name (d for deck). This keeps the code concise while maintaining readability.

## Creating a New Deck

### Creating a New Deck
The newDeck function demonstrates how to create and initialize a custom type:

```go
func newDeck() deck {
    cards := deck{}
    
    cardSuits := []string{"Spades", "Diamonds", "Hearts", "Clubs"}
    cardValues := []string{"Ace", "Two", "Three", "Four"}
    
    for _, suit := range cardSuits {
        for _, value := range cardValues {
            cards = append(cards, value+" of "+suit)
        }
    }
    
    return cards
}
```

This function shows several important Go concepts:
- Creating an empty deck using the custom type
- Using nested loops to generate combinations
- Building strings by concatenation
- Returning the custom type

### Slice Range Syntax
Go provides special syntax for working with parts of a slice:

```go
cards := newDeck()
hand, remainingCards := cards[0:5], cards[5:]
```

The range syntax `[low:high]` creates a slice from index `low` up to (but not including) index `high`. Omitting a number means "start" or "end" respectively:
- `cards[0:5]` - first 5 cards
- `cards[5:]` - everything from index 5 onwards
- `cards[:5]` - first 5 cards (same as [0:5])

### Multiple Return Values
Functions in Go can return multiple values:

```go
func deal(d deck, handSize int) (deck, deck) {
    return d[:handSize], d[handSize:]
}
```

This enables functions to return related pieces of data together. When calling such functions, you must receive all return values:

```go
hand, remainingDeck := deal(cards, 5)
```

This feature eliminates the need for special container objects or complex return types, making the code more straightforward and explicit.

## Working with Files and Data

### Byte Slices
Working with files in Go requires understanding byte slices. A byte slice represents a sequence of raw binary data:

```go
[]byte("Hi there!")
```

Converting between strings and byte slices is common when working with files because file operations deal with raw binary data. The conversion is explicit, reflecting Go's philosophy of making operations visible and clear.

### Deck to String
Before saving a deck to a file, it needs to be converted to a string:

```go
func (d deck) toString() string {
    return strings.Join([]string(d), ",")
}
```

This method:
1. Converts the deck to a slice of strings using type conversion
2. Joins all cards with commas between them
3. Returns a single string representation

### Saving Data to Hard Drive
Writing data to files uses the `ioutil` package:

```go
func (d deck) saveToFile(filename string) error {
    return ioutil.WriteFile(filename, []byte(d.toString()), 0666)
}
```

The process involves:
1. Converting the deck to a string
2. Converting the string to a byte slice
3. Writing the bytes to a file
4. Using permissions (0666) to set file access rights

## Error Handling

In Go, error handling is explicit and straightforward. When reading from files or performing operations that might fail, Go requires direct handling of potential errors. Consider this example from the cards project:

```go
func newDeckFromFile(filename string) deck {
    bs, err := ioutil.ReadFile(filename)
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }
    
    s := strings.Split(string(bs), ",")
    return deck(s)
}
```

This error handling pattern appears throughout Go programs. The `if err != nil` check is crucial because it allows programs to respond appropriately when things go wrong. In this case, if the file can't be read, the program prints the error and exits rather than continuing with invalid data.

## Random Number Generation and Shuffling

Implementing the shuffle functionality for our deck reveals important aspects of Go's random number generation. A naive implementation might look like this:

```go
func (d deck) shuffle() {
    for i := range d {
        newPosition := rand.Intn(len(d))
        d[i], d[newPosition] = d[newPosition], d[i]
    }
}
```

However, this implementation has a subtle problem. Go's random number generator needs a seed value to generate truly random numbers. Without setting a seed, the same sequence of random numbers appears every time. Here's the correct implementation:

```go
func (d deck) shuffle() {
    source := rand.NewSource(time.Now().UnixNano())
    r := rand.New(source)
    
    for i := range d {
        newPosition := r.Intn(len(d))
        d[i], d[newPosition] = d[newPosition], d[i]
    }
}
```

This version uses the current time as a seed, ensuring different random sequences on each program run. The difference illustrates how Go encourages explicit handling of program state.

## Testing 

Go includes a testing framework in its standard library. Test files are identified by the `_test.go` suffix. Here's an example testing the deck creation:

```go
func TestNewDeck(t *testing.T) {
    d := newDeck()
    
    if len(d) != 16 {
        t.Errorf("Expected deck length of 16, but got %v", len(d))
    }
    
    if d[0] != "Ace of Spades" {
        t.Errorf("Expected first card of Ace of Spades, but got %v", d[0])
    }
    
    if d[len(d)-1] != "Four of Clubs" {
        t.Errorf("Expected last card of Four of Clubs, but got %v", d[len(d)-1])
    }
}
```

This test demonstrates several testing principles in Go:
1. Test functions start with "Test"
2. They take a single parameter of type `*testing.T`
3. Errors are reported using the `t.Errorf` function
4. Tests verify specific, expected behaviors

Testing file operations requires cleanup to ensure tests remain reliable:

```go
func TestSaveToDeckAndNewDeckFromFile(t *testing.T) {
    os.Remove("_decktesting")
    
    deck := newDeck()
    deck.saveToFile("_decktesting")
    
    loadedDeck := newDeckFromFile("_decktesting")
    
    if len(loadedDeck) != 16 {
        t.Errorf("Expected 16 cards in deck, got %v", len(loadedDeck))
    }
    
    os.Remove("_decktesting")
}
```

This test demonstrates proper cleanup by removing test files both before and after the test runs. This practice prevents test files from accumulating and ensures each test starts with a clean state.

## Understanding Structs 

Structs provide a way to create composite types that group related data together. Unlike the deck type, which extended a slice, structs combine different types of data:

```go
type contactInfo struct {
    email   string
    zipCode int
}

type person struct {
    firstName string
    lastName  string
    contact   contactInfo
}
```

This structure shows how Go allows composition of types. Instead of inheritance, Go encourages building complex types by combining simpler ones. When creating instances of structs, Go provides several syntax options:

```go
// With field names (most clear)
jim := person{
    firstName: "Jim",
    lastName: "Party",
    contact: contactInfo{
        email: "jim@gmail.com",
        zipCode: 94000,
    },
}

// Zero value initialization
var alex person
```

The second form demonstrates Go's zero value concept - each field gets a default value appropriate to its type. This predictable initialization is a key feature of Go's design.

## Maps

Maps provide a way to create collections of key-value pairs. Unlike structs, which have a fixed set of fields determined at compile time, maps allow dynamic key-value associations:

```go
colors := map[string]string{
    "red":   "#ff0000",
    "green": "#4bf745",
}
```

The syntax `map[keyType]valueType` specifies that all keys must be of one type and all values must be of another type. This type consistency is enforced by the compiler, preventing type-related errors at runtime.

Maps can also be created using the make function:

```go
colors := make(map[string]string)
```

This approach creates an empty map ready for use. Unlike structs, maps must be initialized before use - the zero value of a map is nil, and attempting to add to a nil map causes a runtime panic.

Adding and removing values from maps uses straightforward syntax:

```go
// Adding key-value pairs
colors["white"] = "#ffffff"

// Removing entries
delete(colors, "white")
```

Maps differ from structs in several key ways:
1. Maps require all keys/values to be the same type
2. Maps are reference types (passed by reference)
3. Map keys are indexed and can be iterated over
4. Map keys don't need to be known at compile time

## Interfaces 

Interfaces solve a common problem in programming: how to make code reusable across different types. 

```go
type bot interface {
    getGreeting() string
}

type englishBot struct{}
type spanishBot struct{}

func (englishBot) getGreeting() string {
    return "Hi there!"
}

func (spanishBot) getGreeting() string {
    return "Hola!"
}
```

Instead of writing separate functions to handle each type of bot, interfaces allow writing a single function that works with any type implementing the required methods:

```go
func printGreeting(b bot) {
    fmt.Println(b.getGreeting())
}
```

This demonstrates a fundamental principle in Go: interface satisfaction is implicit. Any type that implements the required methods automatically satisfies the interface - no explicit declaration is needed.

The HTTP package demonstrates practical interface usage through the Reader and Writer interfaces:

```go
resp, err := http.Get("http://google.com")
if err != nil {
    fmt.Println("Error:", err)
    os.Exit(1)
}
```

The response body implements the Reader interface, which specifies a Read method:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

This standard interface allows different types (files, network connections, strings) to provide data in a consistent way. The Writer interface serves a similar purpose for output:

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

These interfaces enable powerful composition, as seen in the io.Copy function:

```go
io.Copy(os.Stdout, resp.Body)
```

This function works with any Reader as input and any Writer as output, demonstrating Go's approach to code reuse through interfaces.

## Concurrency 

Go provides powerful tools for concurrent programming through goroutines and channels. A goroutine is a lightweight thread managed by Go's runtime:

```go
go checkLink(link, c)
```

The `go` keyword launches a function in a new goroutine. The Go scheduler manages these goroutines, multiplexing them onto operating system threads. This makes concurrent programming more manageable than direct thread manipulation.

Channels provide safe communication between goroutines:

```go
c := make(chan string)

// Send value through channel
c <- "message"

// Receive value from channel
msg := <-c
```

The channel operations block until both sender and receiver are ready, providing synchronization without explicit locks. This blocking behavior helps prevent race conditions and makes concurrent code easier to reason about.

### Function Literal

Function literals (anonymous functions) are often used with goroutines:

```go
go func(link string) {
    time.Sleep(5 * time.Second)
    checkLink(link, c)
}(l)
```

When using function literals with goroutines, variables should be passed as parameters rather than accessed from the enclosing scope. This prevents issues with shared variable access:

```go
// Problematic: l is shared
for l := range c {
    go func() {
        checkLink(l, c)  // l may change
    }()
}

// Correct: l is passed as parameter
for l := range c {
    go func(link string) {
        checkLink(link, c)  // link is local copy
    }(l)
}
```

Channel iteration provides a clean way to process a stream of values:

```go
for l := range c {
    go func(link string) {
        time.Sleep(5 * time.Second)
        checkLink(link, c)
    }(l)
}
```

This pattern creates continuous monitoring, with each check launching after a delay. The for-range loop continues until the channel is closed, and each iteration spawns a new goroutine to handle the check independently.

Go's concurrency features work together to enable clear, efficient concurrent programs. Goroutines provide lightweight concurrency, channels ensure safe communication, and the Go runtime manages the actual execution efficiently.