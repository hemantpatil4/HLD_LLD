# Binary, Bits, Bitwise Operators & Snowflake IDs — System Design Foundation

## 1. Why learn this before Snowflake IDs?

Snowflake-style IDs are built by **packing multiple pieces of information into one integer**.

To understand that, you need to be comfortable with:

1. Bits and bytes
2. Binary representation
3. Powers of 2
4. Bitwise operators
5. Bit masks
6. Extracting individual bits
7. Shifting values into different positions
8. Combining fields into one number
9. Reading/unpacking those fields again
10. Applying the same idea to a distributed ID generator

The central idea is:

```text
multiple values
     ↓
position their bits
     ↓
combine their bits
     ↓
one integer
```

---

# 2. Bits and Bytes

## 2.1 What is a bit?

A **bit** is the smallest unit of information.

It can contain only:

```text
0
or
1
```

Think of it as a switch:

```text
0 = OFF
1 = ON
```

---

## 2.2 What is a byte?

A byte contains **8 bits**.

```text
1 byte = 8 bits
```

Example:

```text
10110100
```

This is one byte.

Because every bit has 2 possibilities:

```text
8 bits → 2^8 possibilities → 256
```

For an unsigned 8-bit value:

```text
00000000 = 0
11111111 = 255
```

So one byte can represent:

```text
0 → 255
```

---

# 3. Common Sizes

| Bits | Bytes | Number of combinations |
|---:|---:|---:|
| 1 | 0.125 | 2 |
| 8 | 1 | 2^8 = 256 |
| 16 | 2 | 2^16 = 65,536 |
| 32 | 4 | 2^32 = 4,294,967,296 |
| 64 | 8 | 2^64 |
| 128 | 16 | 2^128 |

Important conversions:

```text
8 bits   = 1 byte
32 bits  = 4 bytes
64 bits  = 8 bytes
128 bits = 16 bytes
```

---

# 4. Common C# Data Types

Typical sizes:

| C# Type | Size |
|---|---:|
| `byte` | 1 byte |
| `short` | 2 bytes |
| `int` | 4 bytes |
| `long` | 8 bytes |
| `float` | 4 bytes |
| `double` | 8 bytes |
| `decimal` | 16 bytes |
| `Guid` | 16 bytes |

A `Guid`/UUID is:

```text
128 bits
= 16 bytes
```

The familiar text:

```text
550e8400-e29b-41d4-a716-446655440000
```

is a textual representation of the 128-bit value. It is not 16 characters.

---

# 5. Binary Numbers

Binary uses powers of 2.

For four bits:

```text
Bit position:  3   2   1   0
               ↓   ↓   ↓   ↓

Place value:   8   4   2   1
```

Example:

```text
1101
```

means:

```text
1×8 + 1×4 + 0×2 + 1×1

= 8 + 4 + 1

= 13
```

Therefore:

```text
1101 = 13
```

Another example:

```text
1010

= 8 + 2

= 10
```

---

# 6. The `2^N` Rule

If you have `N` bits, you have:

```text
2^N
```

possible combinations.

Examples:

```text
1 bit   → 2^1   = 2
2 bits  → 2^2   = 4
3 bits  → 2^3   = 8
8 bits  → 2^8   = 256
32 bits → 2^32
64 bits → 2^64
128 bits → 2^128
```

This is extremely important for system design.

For example, if a pattern is:

```text
101010????1011??01?
```

there are:

```text
4 + 2 + 1 = 7
```

unknown bits.

Therefore:

```text
2^7 = 128
```

possible combinations.

Each `?` independently has:

```text
0
or
1
```

---

# 7. Bitwise Operators

The main bitwise operators in C# are:

```text
&    AND
|    OR
^    XOR
~    NOT
<<   Left shift
>>   Right shift
```

---

# 8. Bitwise AND `&`

AND compares corresponding bits.

Truth table:

| A | B | A & B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

Rule:

> AND produces `1` only when both bits are `1`.

Example:

```text
13 = 1101
 9 = 1001

1101
1001
----
1001
```

Therefore:

```text
13 & 9 = 9
```

---

# 9. Bitwise OR `|`

OR produces `1` when either corresponding bit is `1`.

| A | B | A \| B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 1 |

Example:

```text
13 = 1101
10 = 1010

1101
1010
----
1111
```

Therefore:

```text
13 | 10 = 15
```

OR is commonly used to **combine already-positioned bit fields**.

---

# 10. Bitwise XOR `^`

XOR means:

> Different bits produce `1`; equal bits produce `0`.

| A | B | A ^ B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

Example:

```text
13 = 1101
10 = 1010

1101
1010
----
0111
```

Therefore:

```text
13 ^ 10 = 7
```

Useful properties:

```text
A ^ A = 0
A ^ 0 = A
A ^ B ^ B = A
```

This is why XOR can solve some "every number appears twice except one" problems.

---

# 11. Bitwise NOT `~`

NOT flips every bit:

```text
0 → 1
1 → 0
```

For an 8-bit illustration:

```text
13 = 00001101

~13 = 11110010
```

C# `int` is 32 bits, so the real representation is 32 bits.

For signed integers, two's-complement representation means:

```text
~x = -(x + 1)
```

Therefore:

```text
~13 = -14
```

---

# 12. Left Shift `<<`

The left shift operator moves bits to the left.

Example:

```text
5 = 0101
```

Now:

```text
5 << 1
```

Move every bit one position left:

```text
0101
 ↓↓↓
1010
```

`1010 = 10`.

Therefore:

```text
5 << 1 = 10
```

Generally, when there is no overflow:

```text
x << 1 ≈ x × 2
x << 2 ≈ x × 4
x << 3 ≈ x × 8
```

Mathematically:

```text
x << n ≈ x × 2^n
```

But for system design, the more useful mental model is:

> **Left shift moves a value into higher bit positions.**

This is how we position fields when packing multiple values into one integer.

---

# 13. Right Shift `>>`

Right shift moves bits to the right.

Example:

```text
20 = 10100
```

Then:

```text
20 >> 1
```

gives:

```text
10100
   ↓
01010
```

`01010 = 10`.

Therefore:

```text
20 >> 1 = 10
```

For non-negative integers:

```text
x >> 1 ≈ x / 2
x >> 2 ≈ x / 4
x >> 3 ≈ x / 8
```

The important system-design mental model:

> **Right shift moves a field toward the least-significant/right side so that we can inspect or extract it.**

---

# 14. Quick Bitwise Cheat Sheet

| Operator | Meaning | Common purpose |
|---|---|---|
| `&` | AND | Check/extract bits |
| `|` | OR | Set/combine bits |
| `^` | XOR | Toggle bits |
| `~` | NOT | Invert bits |
| `<<` | Left shift | Move/pack fields |
| `>>` | Right shift | Move/extract fields |

---

# 15. What is a Bit Mask?

A **mask** is a bit pattern used to focus on particular bits.

Think of it as a filter.

Suppose:

```text
value = 13
```

Binary:

```text
1101
```

Maybe we only care about bit position 2:

```text
Bit position:  3 2 1 0
               ↓ ↓ ↓ ↓
value:         1 1 0 1
                 ↑
              interested
```

Create a mask:

```text
0100
```

Decimal:

```text
0100 = 4
```

Now:

```text
1101
0100
----
0100
```

The result is non-zero.

Therefore, bit 2 is ON.

---

# 16. Why Masking?

Masking does not necessarily modify the original value.

It means:

> "I only want to look at these particular bits."

Example:

```csharp
int value = 13; // 1101
int mask = 4;   // 0100

if ((value & mask) != 0)
{
    // bit is ON
}
```

Calculation:

```text
value = 1101
mask  = 0100

1101
0100
----
0100
```

Result:

```text
0100 = 4
```

So:

```text
4 != 0
```

is true.

The bit is ON.

---

# 17. What if the Bit is OFF?

Take:

```text
value = 9
```

Binary:

```text
1001
```

Same mask:

```text
0100
```

AND:

```text
1001
0100
----
0000
```

Result is zero.

Therefore:

```csharp
(value & mask) != 0
```

is false.

So the bit is OFF.

---

# 18. Real Example: Permissions

Suppose an application uses one integer to store permissions.

```text
Bit 0 → READ
Bit 1 → WRITE
Bit 2 → DELETE
Bit 3 → ADMIN
```

Define:

```csharp
const int READ   = 1; // 0001
const int WRITE  = 2; // 0010
const int DELETE = 4; // 0100
const int ADMIN  = 8; // 1000
```

Suppose:

```text
permissions = 13
```

Binary:

```text
1101
```

Therefore:

```text
READ   = ON
WRITE  = OFF
DELETE = ON
ADMIN  = ON
```

Check DELETE:

```csharp
if ((permissions & DELETE) != 0)
{
    Console.WriteLine("Can delete");
}
```

---

# 19. Set a Bit ON

To turn a bit ON:

```csharp
value |= mask;
```

Example:

```text
value = 1001
mask  = 0100

1001
0100
----
1101
```

The bit is now ON.

---

# 20. Clear a Bit

To turn a bit OFF:

```csharp
value &= ~mask;
```

Example:

```text
value = 1101
mask  = 0100

~mask:
1011

1101
1011
----
1001
```

The selected bit is cleared.

---

# 21. Toggle a Bit

To flip a bit:

```csharp
value ^= mask;
```

XOR does:

```text
0 → 1
1 → 0
```

Example:

```text
value = 1001
mask  = 0100

1001
0100
----
1101
```

Run it again:

```text
1101
0100
----
1001
```

---

# 22. Four Patterns Worth Memorizing

```csharp
// Check
(value & mask) != 0

// Set ON
value |= mask;

// Clear/OFF
value &= ~mask;

// Toggle
value ^= mask;
```

---

# 23. Extracting a Particular Bit

One of the most useful expressions:

```csharp
int bit = (number >> position) & 1;
```

Meaning:

> Extract the bit at the specified position.

Example:

```text
number = 5

5 = 101
```

Get bit position 2:

```text
(5 >> 2) & 1
```

First:

```text
101 >> 2
= 001
```

Then:

```text
001
001
---
001
```

Result:

```text
1
```

So bit 2 is ON.

Get bit position 1:

```text
(5 >> 1) & 1
```

```text
101 >> 1
= 010

010
001
---
000
```

Result:

```text
0
```

So bit 1 is OFF.

---

# 24. Generate All Bit Combinations

Suppose:

```text
???
```

There are 3 unknown bits.

Number of possibilities:

```text
2^3 = 8
```

Those possibilities are exactly the binary representations of:

```text
0 → 7
```

```text
000
001
010
011
100
101
110
111
```

C#:

```csharp
int numberOfBits = 3;
int total = 1 << numberOfBits;

for (int combination = 0;
     combination < total;
     combination++)
{
    // combination represents one unique bit pattern
}
```

Because:

```text
1 << 3 = 8
```

---

# 25. Example: Pattern `1??0`

Pattern:

```text
1??0
```

There are 2 unknown bits:

```text
2^2 = 4
```

Possible combinations:

```text
1000
1010
1100
1110
```

Using bitwise operations:

```csharp
static List<string> GenerateAll(string pattern)
{
    List<string> result = new();

    int questionMarks = 0;

    foreach (char c in pattern)
    {
        if (c == '?')
            questionMarks++;
    }

    int total = 1 << questionMarks;

    for (int combination = 0;
         combination < total;
         combination++)
    {
        char[] current = pattern.ToCharArray();

        int bitPosition = 0;

        for (int i = 0; i < current.Length; i++)
        {
            if (current[i] == '?')
            {
                int bit =
                    (combination >> bitPosition) & 1;

                current[i] =
                    bit == 0 ? '0' : '1';

                bitPosition++;
            }
        }

        result.Add(new string(current));
    }

    return result;
}
```

The critical expression is:

```csharp
(combination >> bitPosition) & 1
```

It means:

```text
move the required bit to the right
             ↓
          >> position
             ↓
keep only the rightmost bit
             ↓
            & 1
```

---

# 26. The Connection to Bit Packing

Now we can use the same operators in the opposite direction.

Suppose we have:

```text
Time     = 5
Worker   = 2
Sequence = 3
```

And decide to create an 8-bit ID:

```text
┌──────────┬────────┬──────────┐
│ Time     │ Worker │ Sequence │
│ 4 bits   │ 2 bits │ 2 bits   │
└──────────┴────────┴──────────┘
```

Binary:

```text
Time     = 0101
Worker   = 10
Sequence = 11
```

Target:

```text
0101 10 11
```

---

# 27. Position the Sequence

Sequence already belongs in the last 2 bits:

```text
Sequence = 11
```

No shift required.

As 8 bits:

```text
00000011
```

---

# 28. Position the Worker

Worker:

```text
10
```

Sequence occupies 2 bits.

Therefore move Worker left by 2:

```csharp
worker << 2
```

```text
00000010
      ↓↓
00001000
```

Therefore:

```text
2 << 2 = 8
```

The Worker is now positioned correctly:

```text
00001000
```

---

# 29. Combine Worker and Sequence

Use OR:

```text
00001000
00000011
--------
00001011
```

Now we have:

```text
Worker + Sequence

10 11
```

---

# 30. Position Time

Time:

```text
0101
```

Worker + Sequence occupy:

```text
2 + 2 = 4 bits
```

Therefore:

```csharp
time << 4
```

```text
00000101
       ↓↓↓↓
01010000
```

---

# 31. Combine All Three

```text
Time     = 01010000
Worker   = 00001000
Sequence = 00000011
```

OR:

```text
01010000
00001000
00000011
--------
01011011
```

Final packed value:

```text
01011011
```

Decimal:

```text
64 + 16 + 8 + 2 + 1 = 91
```

Therefore:

```text
ID = 91
```

---

# 32. What Does `91` Represent?

Internally:

```text
91 = 01011011
```

We deliberately structured those bits:

```text
┌────┬──┬──┐
│0101│10│11│
│ 5  │ 2│ 3│
└────┴──┴──┘
```

So one integer contains:

```text
Time     = 5
Worker   = 2
Sequence = 3
```

This is called **bit packing**.

---

# 33. General Bit-Packing Formula

Suppose the layout is:

```text
┌──────────┬────────┬──────────┐
│ Field A  │ Field B│ Field C  │
│ A bits   │ B bits │ C bits   │
└──────────┴────────┴──────────┘
```

Then:

```text
Field C → no shift

Field B → shift C bits

Field A → shift B + C bits
```

Formula:

```csharp
id =
    (fieldA << (B + C))
    |
    (fieldB << C)
    |
    fieldC;
```

---

# 34. Unpacking / Reading the Fields Back

Suppose:

```text
ID = 91
```

Binary:

```text
01011011
```

We want to recover:

```text
Time
Worker
Sequence
```

---

## 34.1 Extract Sequence

Sequence is the final 2 bits.

Mask:

```text
00000011
```

Therefore:

```csharp
int sequence = id & 3;
```

Calculation:

```text
01011011
00000011
--------
00000011
```

Result:

```text
Sequence = 3
```

---

## 34.2 Extract Worker

Worker is the 2 bits before sequence.

First shift right by 2:

```csharp
id >> 2
```

```text
01011011
   ↓↓
00010110
```

Now mask the final 2 bits:

```csharp
int worker = (id >> 2) & 3;
```

```text
00010110
00000011
--------
00000010
```

Result:

```text
Worker = 2
```

---

## 34.3 Extract Time

Time is everything above the lower 4 bits.

Shift right by 4:

```csharp
int time = id >> 4;
```

```text
01011011 >> 4
= 00000101
```

Result:

```text
Time = 5
```

---

# 35. The Most Important Packing/Unpacking Pattern

Packing:

```csharp
id =
    (time << 4)
    | (worker << 2)
    | sequence;
```

Unpacking:

```csharp
sequence = id & 3;

worker = (id >> 2) & 3;

time = id >> 4;
```

Conceptually:

```text
PACK

values
  ↓
shift into positions
  ↓
OR together
  ↓
one integer


UNPACK

one integer
  ↓
shift field toward right
  ↓
AND with mask
  ↓
original value
```

---

# 36. Now Apply This to Snowflake IDs

A common Snowflake-style layout is:

```text
64-bit ID

┌─────────────────────┬────────────┬────────────┐
│ Timestamp           │ Worker ID  │ Sequence   │
│ 41 bits             │ 10 bits    │ 12 bits    │
└─────────────────────┴────────────┴────────────┘
```

Useful bits:

```text
41 + 10 + 12 = 63 bits
```

The remaining bit is typically kept clear to keep the value positive in a signed 64-bit integer representation.

---

# 37. Why These Three Fields?

## Timestamp — 41 bits

Represents when the ID was generated.

Usually:

```text
currentTime - customEpoch
```

rather than an absolute timestamp.

This allows the ID to be roughly time ordered.

---

## Worker ID — 10 bits

Identifies which generator/server produced the ID.

Number of possible workers:

```text
2^10 = 1024
```

So:

```text
Worker IDs: 0 → 1023
```

---

## Sequence — 12 bits

Identifies the ID within the same timestamp unit, commonly the same millisecond, on a particular worker.

Number of possible sequence values:

```text
2^12 = 4096
```

So:

```text
0 → 4095
```

A worker can therefore generate up to 4096 IDs during one millisecond before needing to move to the next millisecond or apply an overflow strategy.

The theoretical rate is:

```text
4096 IDs/ms
× 1000 ms/sec
= 4,096,000 IDs/sec
```

per worker, subject to implementation/runtime constraints.

---

# 38. Why 64 Bits?

A Snowflake-style ID wants to be:

- compact
- numeric
- sortable by creation time
- unique across workers
- generated without a central database counter

Using 64 bits gives a convenient `long`-sized identifier.

Compared with a UUID:

```text
UUID  → 128 bits → 16 bytes
Snowflake-style ID → 64 bits → 8 bytes
```

This can be useful for database indexes, network payloads, and storage.

---

# 39. Real Snowflake Packing Formula

With:

```text
Timestamp = 41 bits
Worker     = 10 bits
Sequence   = 12 bits
```

Sequence occupies the last:

```text
12 bits
```

Therefore Worker must shift:

```text
workerId << 12
```

Timestamp must move past:

```text
10 + 12 = 22 bits
```

Therefore:

```text
timestamp << 22
```

The packed ID is conceptually:

```csharp
long id =
    (timestamp << 22)
    | (workerId << 12)
    | sequence;
```

This is the same exact operation as our 8-bit example.

---

# 40. Why `<< 22`?

Because:

```text
Timestamp
    ↓
must move past

Worker = 10 bits
Sequence = 12 bits

10 + 12 = 22
```

Therefore:

```text
timestamp << 22
```

---

# 41. Why `<< 12`?

Because:

```text
Worker
   ↓
must move past

Sequence = 12 bits
```

Therefore:

```text
workerId << 12
```

Sequence stays where it is:

```text
sequence
```

Final layout:

```text
┌──────────────────────────────┬───────────┬────────────┐
│ Timestamp                   │ Worker    │ Sequence   │
│ 41 bits                     │ 10 bits   │ 12 bits    │
└──────────────────────────────┴───────────┴────────────┘
                ↑                    ↑           ↑
              << 22                << 12       << 0
```

---

# 42. Why OR `|`?

Because after shifting, the fields occupy different bit positions.

Conceptually:

```text
Timestamp:
TTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTT000000000000000000000000

Worker:
000000000000000000000000000000000000000WWWWWWWWWW000000000000

Sequence:
0000000000000000000000000000000000000000000000000000SSSSSSSSSSSS
```

They do not overlap.

OR combines them:

```text
Timestamp positioned
        |
        OR
        |
Worker positioned
        |
        OR
        |
Sequence
        ↓
One 64-bit ID
```

---

# 43. Why This Is Useful in Distributed Systems

Consider:

```text
                  Load Balancer
                 /      |      \
                ↓       ↓       ↓
             Worker A Worker B Worker C
```

Each worker can generate IDs locally.

Worker A:

```text
Timestamp + WorkerA + Sequence
```

Worker B:

```text
Timestamp + WorkerB + Sequence
```

Worker C:

```text
Timestamp + WorkerC + Sequence
```

Because the Worker ID is different, their IDs don't collide under correct assignment and clock/sequence rules.

No central:

```text
SELECT NEXT VALUE FOR ...
```

operation is required.

---

# 44. UUID vs Auto Increment vs Snowflake

| Approach | Size | Distributed generation | Time sortable | Central coordination |
|---|---:|---|---|---|
| DB Auto Increment | Usually 4/8 bytes | Poor across independent generators | Usually yes | Usually required |
| UUID v4 | 16 bytes | Excellent | No | No |
| Snowflake-style | 8 bytes | Excellent | Yes/roughly | No central ID counter |

The choice depends on the system.

Snowflake is attractive when you want:

```text
compact
+
distributed generation
+
rough time ordering
+
high throughput
```

---

# 45. Important Snowflake Caveat: Worker ID Assignment

The Worker ID must itself be unique among active generators.

For example:

```text
Worker A → ID 1
Worker B → ID 2
Worker C → ID 3
```

If two active workers accidentally use:

```text
Worker A → ID 1
Worker B → ID 1
```

they can generate conflicting IDs when timestamp and sequence also match.

So worker-ID assignment is a real distributed-system concern.

Possible approaches include:

- static configuration
- orchestrator-assigned identity
- database/coordination service
- service discovery
- leased worker IDs

---

# 46. Sequence Handling

Suppose:

```text
sequence = 4095
```

because 12 bits can represent:

```text
0 → 4095
```

and the same worker needs another ID in the same millisecond.

There is no unused 12-bit sequence value.

A typical implementation:

1. Wait until the next millisecond.
2. Reset sequence to `0`.
3. Generate the next ID.

Conceptually:

```text
10:30:15.100
sequence 0
sequence 1
sequence 2
...
sequence 4095

10:30:15.101
sequence 0
```

---

# 47. Timestamp and Custom Epoch

Instead of storing a full Unix timestamp, a Snowflake implementation commonly chooses a custom epoch.

Example:

```text
customEpoch = 2020-01-01
```

Then:

```text
timestamp =
    currentTime - customEpoch
```

For milliseconds:

```csharp
long timestamp =
    DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
    - customEpoch;
```

The actual Snowflake implementation must define the exact epoch and unit consistently.

---

# 48. Simplified C# Snowflake Generator

A simplified educational implementation:

```csharp
public class SnowflakeIdGenerator
{
    private const long CustomEpoch = 1577836800000L; // Example epoch

    private const int WorkerIdBits = 10;
    private const int SequenceBits = 12;

    private const long MaxWorkerId =
        (1L << WorkerIdBits) - 1;

    private const long MaxSequence =
        (1L << SequenceBits) - 1;

    private const int WorkerIdShift = SequenceBits;
    private const int TimestampShift =
        SequenceBits + WorkerIdBits;

    private readonly long workerId;

    private long sequence = 0;
    private long lastTimestamp = -1;

    private readonly object sync = new();

    public SnowflakeIdGenerator(long workerId)
    {
        if (workerId < 0 || workerId > MaxWorkerId)
            throw new ArgumentException("Invalid worker ID.");

        this.workerId = workerId;
    }

    public long NextId()
    {
        lock (sync)
        {
            long timestamp = GetCurrentTimestamp();

            if (timestamp < lastTimestamp)
            {
                throw new InvalidOperationException(
                    "Clock moved backwards.");
            }

            if (timestamp == lastTimestamp)
            {
                sequence =
                    (sequence + 1) & MaxSequence;

                if (sequence == 0)
                {
                    timestamp =
                        WaitForNextMillisecond(lastTimestamp);
                }
            }
            else
            {
                sequence = 0;
            }

            lastTimestamp = timestamp;

            return
                (timestamp << TimestampShift)
                | (workerId << WorkerIdShift)
                | sequence;
        }
    }

    private static long GetCurrentTimestamp()
    {
        return DateTimeOffset.UtcNow
            .ToUnixTimeMilliseconds()
            - CustomEpoch;
    }

    private static long WaitForNextMillisecond(
        long lastTimestamp)
    {
        long timestamp;

        do
        {
            timestamp = GetCurrentTimestamp();
        }
        while (timestamp <= lastTimestamp);

        return timestamp;
    }
}
```

This is an educational implementation. Production systems need careful consideration of worker identity, clock behavior, overflow handling, persistence/restart behavior, and concurrency.

---

# 49. What the C# Code Is Doing

The important line is:

```csharp
return
    (timestamp << TimestampShift)
    | (workerId << WorkerIdShift)
    | sequence;
```

With:

```text
TimestampShift = 22
WorkerIdShift = 12
```

so:

```csharp
return
    (timestamp << 22)
    | (workerId << 12)
    | sequence;
```

This is exactly the tiny 8-bit example scaled up.

---

# 50. Why `& MaxSequence`?

We have:

```text
MaxSequence = 4095
```

which is:

```text
2^12 - 1
```

Binary:

```text
111111111111
```

So:

```csharp
sequence = (sequence + 1) & MaxSequence;
```

keeps only the lower 12 bits.

Conceptually:

```text
sequence
    ↓
limit to 12 bits
    ↓
000000000000 → 111111111111
```

However, the implementation must separately detect the wraparound and wait for the next timestamp.

---

# 51. Decoding a Snowflake-style ID

If:

```text
id =
    timestamp
    + worker
    + sequence
```

was packed as:

```csharp
id =
    (timestamp << 22)
    | (workerId << 12)
    | sequence;
```

then the fields can be extracted conceptually as:

```csharp
long sequence =
    id & ((1L << 12) - 1);

long workerId =
    (id >> 12) & ((1L << 10) - 1);

long timestamp =
    id >> 22;
```

This follows the exact same pattern:

```text
extract:
(id >> offset) & mask
```

except timestamp is already the upper field, so after shifting it right by 22, no additional mask is necessary.

---

# 52. The Whole Concept in One Diagram

```text
             GENERATION

Timestamp ────────┐
                   │ << 22
                   ↓
            TTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTT000000000000000000000000

Worker ID ────────┐
                   │ << 12
                   ↓
            000000000000000000000000000000000000000WWWWWWWWWW000000000000

Sequence ─────────────────────────────────────────────────SSSSSSSSSSSS

                   │
                   │ OR
                   ↓

            ┌─────────────────────────────────────────────────────────┐
            │ TTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTTWWWWWWWWWWSSSSSSSSSSSS │
            └─────────────────────────────────────────────────────────┘
                              ONE 64-BIT ID
```

---

# 53. Generation vs Extraction

## Generate

```text
timestamp
    ↓
<< 22

worker
    ↓
<< 12

sequence
    ↓
no shift

        ↓

OR everything

        ↓

Snowflake ID
```

## Extract

```text
Snowflake ID
     ↓
sequence = id & sequenceMask

worker =
    (id >> 12) & workerMask

timestamp =
    id >> 22
```

The same bitwise concepts work in both directions.

---

# 54. The Core Mental Model

If you remember only one thing, remember this:

```text
<<
```

means:

> **Move this field left to its assigned position.**

```text
>>
```

means:

> **Move a field right so we can inspect/extract it.**

```text
&
```

means:

> **Keep only the bits I care about.**

```text
|
```

means:

> **Combine fields that have already been positioned.**

So:

```text
PACKING:

(field << position) | (field << position) | field


EXTRACTION:

(id >> position) & mask
```

---

# 55. The Complete Learning Chain

You can now see the progression:

```text
BIT
 ↓
0 / 1
 ↓
BYTE
 ↓
8 bits
 ↓
BINARY NUMBER
 ↓
2^N POSSIBILITIES
 ↓
BITWISE OPERATORS
 ↓
&  |  ^  ~  <<  >>
 ↓
MASKING
 ↓
EXTRACT A BIT
 ↓
BIT PACKING
 ↓
PACK MULTIPLE FIELDS INTO ONE NUMBER
 ↓
SNOWFLAKE ID
 ↓
DISTRIBUTED UNIQUE ID GENERATION
```

That is why we studied the low-level concepts first.

---

# 56. Interview Cheat Sheet

### Bits

```text
1 byte = 8 bits
N bits = 2^N possible combinations
```

### Bitwise

```text
&   AND
|   OR
^   XOR
~   NOT
<<  left shift
>>  right shift
```

### Masks

```csharp
// Check
(value & mask) != 0

// Set
value |= mask;

// Clear
value &= ~mask;

// Toggle
value ^= mask;
```

### Extract bit

```csharp
(number >> position) & 1
```

### Pack

```csharp
(field << position) | otherField
```

### Snowflake

```text
41 bits timestamp
10 bits worker
12 bits sequence
```

Total useful bits:

```text
41 + 10 + 12 = 63
```

Common shifts:

```text
timestamp << 22
workerId << 12
sequence << 0
```

Packed:

```csharp
id =
    (timestamp << 22)
    | (workerId << 12)
    | sequence;
```

---

# 57. The Most Important Understanding

Do not memorize:

```text
41 + 10 + 12
<< 22
<< 12
```

Instead derive them.

If the layout is:

```text
[ Timestamp ][ Worker ][ Sequence ]
```

and:

```text
Worker = 10 bits
Sequence = 12 bits
```

then:

```text
Worker must move past 12 bits
        ↓
Worker << 12

Timestamp must move past:
10 + 12 = 22 bits
        ↓
Timestamp << 22
```

That reasoning is what the interviewer cares about.

---

# 58. Final Tiny Example to Remember

Given:

```text
Time     = 5
Worker   = 2
Sequence = 3
```

Layout:

```text
[ Time: 4 ][ Worker: 2 ][ Seq: 2 ]
```

Binary:

```text
Time     = 0101
Worker   = 10
Sequence = 11
```

Pack:

```csharp
int id =
    (5 << 4)
    | (2 << 2)
    | 3;
```

Result:

```text
01010000
00001000
00000011
--------
01011011
```

Therefore:

```text
ID = 91
```

And:

```text
91
 ↓
01011011
 ↓
0101 | 10 | 11
 ↓
  5  |  2 |  3
```

**One integer successfully contains three separate values.**

That is the fundamental idea behind Snowflake-style IDs.
