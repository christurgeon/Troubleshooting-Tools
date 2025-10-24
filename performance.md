# Branchless Programming Techniques in Rust

We can still branch logic **without using `if`** by leveraging techniques like:

- **Match expressions** (pattern matching is often faster / cleaner)
- **Lookup tables** (arrays, hashmaps, function pointers)
- **Traits and polymorphism** to pick behavior at compile time
- **Branchless programming** with arithmetic or bitwise operations

---

## Branchless Code Example

Branchless command dispatcher using `match` and function pointers:

```rust
fn operation_add(a: i32, b: i32) -> i32 { a + b }
fn operation_sub(a: i32, b: i32) -> i32 { a - b }
fn operation_mul(a: i32, b: i32) -> i32 { a * b }
fn operation_div(a: i32, b: i32) -> i32 { a / b }

fn main() {
    let operations: [fn(i32, i32) -> i32; 4] = [
        operation_add,
        operation_sub,
        operation_mul,
        operation_div,
    ];

    let opcode = 2; // 0=add, 1=sub, 2=mul, 3=div
    let a = 10;
    let b = 5;

    // No if needed — direct table lookup
    let result = operations[opcode](a, b);

    println!("Result: {}", result); // 50
}
```

## Branchless Computation Using Arithmetic

A common trick is to replace branching with bitwise masking:

```rust
/// Returns `a` if cond is true, else returns `b`
fn select(cond: bool, a: i32, b: i32) -> i32 {
    let mask = -(cond as i32); // cond=true => -1 (all bits 1), cond=false => 0
    (a & mask) | (b & !mask)
}

fn main() {
    let x = select(true, 42, 99); // 42
    let y = select(false, 42, 99); // 99
    println!("{}, {}", x, y);
}
```
