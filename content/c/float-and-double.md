+++
title = "`float` and `double`"
description = "An overview of the `float` and `double` data types, including their memory sizes, precision, range, and how they are represented in binary."
date = 2026-10-06T04:37:33+00:00

[taxonomies]
tags = ["float", "double"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## `float` and `double` comparison

Here is how a double data type directly compares to a float data type under the standard IEEE 754 specifications:

| Feature           | float (Single Precision)    | double (Double Precision)     |
| ----------------- | --------------------------- | ----------------------------- |
| Size in Memory    | 4 bytes (32 bits)           | 8 bytes (64 bits)             |
| Sign Bit          | 1 bit                       | 1 bit                         |
| Exponent Width    | 8 bits                      | 11 bits                       |
| Mantissa Width    | 23 bits (24 effective)      | 52 bits (53 effective)        |
| Decimal Precision | ~7 digits of accuracy       | ~15 to 17 digits of accuracy  |
| Approximate Range | ±1.4 × 10⁻⁴⁵ to ±3.4 × 10³⁸ | ±5.0 × 10⁻³²⁴ to ±1.7 × 10³⁰⁸ |

### Key Differences Explained

- **Precision**: A double provides more than twice the decimal precision of a float. If you calculate `1.0 / 3.0`, a float cuts off around `0.3333333`, while a double carries out to roughly `0.3333333333333333`.
- **Range**: A double can handle drastically larger (and smaller fractionally) numbers because its exponent component has 3 extra bits, allowing it to scale up to `10³⁰⁸` compared to the float's limit of `10³⁸`.
- **Performance & Memory**: A float takes up half the memory space. In applications processing millions of numbers (like graphics programming or audio buffers), using float saves substantial memory and can run faster on hardware optimized for it.

---

## How are the mantissa bits used in float?

In a float (32-bit), the 23 mantissa bits (also called the significand or fraction) are used to store the actual precision digits of a number in binary scientific notation.
The IEEE 754 standard uses a clever trick called the implicit leading bit to get 24 bits of precision out of only 23 bits of physical storage. Here is exactly how it works:

### 1. The Normal Form: The Implicit "1." Trick

In standard base-10 scientific notation, you always format numbers so there is exactly one non-zero digit before the decimal point (e.g., 4.51 × 10³).
Binary works the same way. In binary, the only non-zero digit is 1. Therefore, every normalized binary scientific number always looks like:
$$1.xxxxxxxxxxxxxxxxxxxxxxx_2 \times 2^{\text{exponent}}$$
Because the digit before the binary point is always 1, the hardware engineers realized they don't actually need to waste memory storing it.

- The 23 bits in memory only store the fractional part (the digits after the binary point).
- When the CPU performs calculations, it automatically re-attaches the 1. to the front.
- This gives you 24 bits of precision using only 23 bits of space.

#### Visual Example:

If you want to store the number 9.0 in a float:

1.  9.0 in binary is 1001.0
2.  Move the binary point to normalize it: 1.001000... × 2³
3.  Drop the leading 1..
4.  The 23 mantissa bits stored in memory will look like this: 00100000000000000000000

### 2. How the Mantissa maps to Decimal Values

Each of the 23 bits represents a negative power of 2, starting right after the binary point:
$$\text{Value} = 1 + (b_1 \times 2^{-1}) + (b_2 \times 2^{-2}) + (b_3 \times 2^{-3}) + \dots + (b_{23} \times 2^{-23})$$
Where b₁ is the first mantissa bit, b₂ is the second, and so on:

- Bit 1 = 2⁻¹ = 0.5
- Bit 2 = 2⁻² = 0.25
- Bit 3 = 2⁻³ = 0.125
- Bit 23 = 2⁻²³ = 0.0000001192...

This is why a float has ~7 decimal digits of precision. The smallest step change the mantissa can make is 2⁻²³, which is roughly 1.19 × 10⁻⁷.

### 3. The Exception: Subnormal Numbers (Stored Exponent = 0)

When a number gets so tiny that the exponent cannot go any lower, the float switches to subnormal mode to prevent dropping straight to zero.
When this happens:

- The implicit leading bit changes from 1. to 0..
- The number is evaluated as: 0.xxxxxxxxxxxxxxxxxxxxxxx₂ × 2⁻¹²⁶.
- As the number gets smaller, more leading zeros creep into the 23 mantissa bits, causing you to steadily lose precision until no bits are left.

---

## How is the range `±1.4 × 10⁻⁴⁵ to ±3.4 × 10³⁸` of `float` computed?

The numbers $3.4 \times 10^{38}$ and $1.4 \times 10^{-45}$ are computed by converting the largest and smallest possible binary values of a float into base-10 (decimal) numbers.
To understand the math, we first look at how the 8 exponent bits work under the [IEEE 754 standard](https://en.wikipedia.org/wiki/IEEE_754), and then factor in the 23 mantissa bits.

### Step 1: The 8-Bit Exponent Range and Bias

An 8-bit binary number can represent integers from 0 to 255. However, IEEE 754 reserves two of these values for special purposes:

- 0 is reserved for zero and subnormal numbers.
- 255 is reserved for Infinity and NaN (Not a Number).

This leaves a usable range of 1 to 254. To allow for negative exponents (tiny fractions), a bias of 127 is subtracted from the stored exponent.
$$\text{True Exponent} = \text{Stored Exponent} - 127$$

- Maximum Normal Exponent: $254 - 127 =$ $+127$
- Minimum Normal Exponent: $1 - 127 =$ $-126$

### Step 2: Computing the Max Range ($3.4 \times 10^{38}$)

To get the absolute largest number, we maximize both the exponent and the mantissa:

1.  Max Exponent: $2^{127}$
2.  Max Mantissa: All 23 bits are set to 1. In binary scientific notation, this represents $1.1111..._2$, which is effectively just a fraction below $2$ (specifically, $2 - 2^{-23} \approx 1.99999988$).

Now, we multiply them together:
$$\text{Max Value} \approx 2 \times 2^{127} = 2^{128}$$
Using logarithms to convert $2^{128}$ to base-10:
$$2^{128} = 10^{128 \times \log_{10}(2)} \approx 10^{128 \times 0.30103} \approx 10^{38.53}$$
The exact math yields $3.4028235 \times 10^{38}$.

### Step 3: Computing the Min Range ($1.4 \times 10^{-45}$)

The value $1.4 \times 10^{-45}$ represents the absolute smallest non-zero positive number a float can hold. To achieve this, the system shifts from a "normal" number to a subnormal number:

1.  Min Exponent: When the stored exponent is 0, the true exponent is fixed at $-126$.
2.  Min Mantissa: Instead of the usual implicit leading 1, the implicit leading digit becomes 0. We then set only the very last bit of the 23-bit mantissa to 1. This gives a value of $2^{-23}$.

Now, we multiply them together:
$$\text{Min Value} = 2^{-23} \times 2^{-126} = 2^{-149}$$
Using logarithms to convert $2^{-149}$ to base-10:
$$2^{-149} = 10^{-149 \times \log_{10}(2)} \approx 10^{-149 \times 0.30103} \approx 10^{-44.85}$$
The exact math yields $1.4012985 \times 10^{-45}$.

---

## The classic `0.1 + 0.2 != 0.3` problem

The reason 0.1 + 0.2 equals 0.30000000000000004 comes down to a fundamental limitation of computers: they think in base-2 (binary), while humans think in base-10 (decimal).
Because computers have a fixed size (usually a 64-bit double in languages like JavaScript, Python, and Java), they cannot accurately represent fractions that repeat infinitely in binary.
Here is the step-by-step breakdown of how this math error happens under the hood.

### 1. The Repeating Fraction Problem

In base-10, you cannot cleanly write $1/3$ as a decimal. It becomes an infinite repeating fraction: 0.333333.... No matter how many 3s you write down on a piece of paper, you will eventually run out of space and be forced to round it.

Binary has the exact same issue, but with different numbers. A fraction can only be written cleanly in binary if its denominator's prime factors are powers of 2.

- Clean in decimal: $1/10$ (0.1) and $2/10$ (0.2)
- Infinite in binary: Both 0.1 and 0.2 become infinite repeating binary fractions!

$$\text{0.1 in binary} = 0.00011001100110011..._2 \text{ (repeating forever)}$$
$$\text{0.2 in binary} = 0.00110011001100110..._2 \text{ (repeating forever)}$$

### 2. Computers Cut Off and Round

Since a 64-bit double only allocates 52 bits for the mantissa (precision), the computer has to chop off the infinite tail of these numbers and round them to the nearest possible binary value.
When you type 0.1 and 0.2, the computer actually stores slightly inaccurate values:

- Stored 0.1 $\approx$ 0.10000000000000000555111512312578...
- Stored 0.2 $\approx$ 0.20000000000000001110223024625156...

### 3. The Math and the "4" at the End

When the CPU adds these two rounded binary numbers together, the small rounding errors stack on top of each other.
If we look at the exact underlying decimal math the computer is doing:

```
  0.100000000000000005551115... (Stored 0.1)
+ 0.200000000000000011102230... (Stored 0.2)
---------------------------------
  0.300000000000000016653345... (Exact sum)
```

The exact sum ends in ...16653345.... When the computer tries to fit this result back into a 64-bit double, it rounds it to the closest valid floating-point number it can possibly represent.

That closest number happens to be exactly 0.30000000000000004440892098500626....

When your programming language prints it out, it truncates the view to 17 digits, leaving you with the famous 0.30000000000000004.

### How to Fix This in Code

If you are dealing with everyday rounding or money, you don't want these errors messing up your software. Here are the common solutions:

1.  **Format/Round for display**: Keep the math as-is, but round the string output when showing it to users (e.g., result.toFixed(2) in JavaScript).
2.  **Work in Cents (Integers)**: Instead of using decimals for money, multiply by 100 and work strictly with whole numbers (integers), since 10 + 20 = 30 is always exact.
3.  **Use Arbitrary-Precision Libraries**: Use built-in types designed for exact decimal math, such as BigDecimal in Java, decimal in Python or C#, or libraries like big.js in JavaScript.
