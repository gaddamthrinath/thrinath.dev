---
title: "Stop Guessing How JavaScript Parses Numbers Here's What Actually Happens"
description: "A deep dive into JavaScript's Number(), parseInt(), and parseFloat() functions, explaining how they handle strings, whitespace, and the special NaN value."
category: "notes"
tags: ["javascript", "numbers", "parsing", "nan", "typescript"]
publishedAt: 2026-06-24
slug: "stop-guessing-how-javascript-parses-numbers-heres-what-actually-happens"


---
A few months back, a teammate was debugging a bulk pricing import. Store managers would upload a spreadsheet of products, and the system calculated each line total by multiplying the price column by the quantity column. Every so often, an order total on the dashboard would show up as `NaN` instead of a number, and nobody could figure out why, since the validation step was supposed to catch exactly that kind of bad row before it ever got that far.

The validation looked completely reasonable:

```js
const price = Number(priceCell);

if (price === NaN) {
  return rejectRow("Invalid price");
}

return price * quantity;
```

The cause turned out to be a single row where the price cell had been left as `"N/A"` instead of a number, which happens constantly with real spreadsheets. `Number("N/A")` correctly produces `NaN`. The check should have caught it right there. But it never does, because of one very specific fact about JavaScript: `NaN` is never equal to anything, including itself. `price === NaN` is dead code. It always evaluates to `false`, no matter what `price` actually is, so the bad row never gets rejected. Instead, it falls straight through to `price * quantity`, which is `NaN * quantity`, which is `NaN`. That value then gets stored and displayed as if it were a real number, which is exactly the dashboard symptom that started the investigation.

That bug is what this post is really about. Not the trivia, but the handful of habits in how JavaScript converts strings to numbers that look fine in a code review and then misbehave in production. We will go through `Number()`, `parseInt()`, and `parseFloat()`, what `NaN` actually is, and the specific pitfalls — like the one above — that show up again and again in real codebases.

## JavaScript only has one number type

Most languages give you separate types depending on what kind of number you are storing:

```java
int age = 25;
float price = 99.99;
```

JavaScript skips that distinction entirely. An integer and a decimal are both just `Number`.

```js
typeof 10;    // "number"
typeof 3.14;  // "number"
```

Internally, every JavaScript number — whole or decimal — is stored using the IEEE-754 double-precision floating-point format. That single fact explains several things that will come up later in this post, including why `NaN` counts as a "number" at all, and why adding two simple decimals can produce a result that looks wrong.

## Three tools for turning text into numbers

Almost everything you build ends up needing to convert text into a number at some point. A price typed into a form, a quantity column from a spreadsheet, a pixel value read off a CSS style — all of it starts out as a string, and your code needs an actual number before it can do any math with it.

JavaScript gives you three different functions for this: `Number()`, `parseInt()`, and `parseFloat()`. At first glance that looks like overkill — why would one language need three ways to do the same conversion? The reason is that they are not actually solving the same problem. Each one makes a different assumption about how messy the input is allowed to be, and that difference is exactly what causes the confusing, seemingly inconsistent results people run into.

`Number()` assumes the string is either a clean number or it isn't, and refuses to guess. `parseInt()` and `parseFloat()` assume the string probably has a number sitting in it somewhere, possibly with extra junk attached, and they try to pull that number out anyway. Knowing which assumption you actually want is most of the battle. We will go through each one in turn, starting with `Number()`, since it is the strictest of the three and the easiest to reason about.

## Number(): converts the whole string, or nothing at all

Say you are building a simple price calculator. A customer types a number into a field, and your code needs to turn that text into something you can do math with. The obvious tool is `Number()`:

```js
Number("19.99"); // 19.99
Number("-50");   // -50
```

That works exactly as expected, until a customer's input is not quite clean. Maybe there is a stray space, or maybe there is a typo. This is where `Number()`'s actual rule matters: it only succeeds if the entire string is a valid number. Not most of it. All of it.

A few cases that catch people off guard:

**Trailing zeros get dropped, because they don't change the value.** Picture a price that was typed with extra zeros, like `"19.00"`:

```js
Number("19.00"); // 19
```

JavaScript reads the whole string, sees a valid decimal number, and stores the value it represents — which is just `19`. The extra zeros after the decimal point don't change what number is being described, so they don't survive the conversion.

**A number can start with just a decimal point.** If a customer types `.99` instead of `0.99`, it still works:

```js
Number(".99"); // 0.99
```

JavaScript treats a leading decimal point as shorthand for a zero in front of it, so `.99` is read the same way as `0.99` would be, and that's the value that gets returned.

**Whitespace around the edges gets trimmed before anything else happens.** If your form input has accidental leading or trailing spaces, this saves you:

```js
Number("  19.99  "); // 19.99
```

Before JavaScript even tries to parse this as a number, it strips the whitespace from both ends, leaving `"19.99"`. That cleaned-up string is what actually gets converted, which is why the spaces have no effect on the final result.

**But anything invalid breaks the whole conversion, even with valid characters around it.** Imagine the price field accidentally has a currency symbol pasted in, or a stray character from a copy-paste:

```js
Number("19.99 /mo"); // NaN
```

JavaScript reads `19.99` just fine, but then runs into ` /mo`, and because the entire string has to be valid, that trailing text is enough to fail the whole thing. `Number()` does not give you partial credit for getting most of the string right.

This is the core idea to hold onto: `Number()` is a strict, all-or-nothing converter. That makes it the right tool when you are validating input and need to know whether what someone typed is actually usable as a number.

## parseInt(): pulls a number out, ignores what's left

Now imagine a different situation. You are reading a CSS value off a computed style, something like `"250px"`, and you just need the number out of it. `Number()` would reject this outright, because `"px"` is not part of a valid number:

```js
Number("250px"); // NaN
```

This is exactly the situation `parseInt()` was built for. Instead of validating the whole string, it reads digits from the beginning and simply stops the moment it hits something that does not fit:

```js
parseInt("250px"); // 250
```

JavaScript reads `2`, `5`, `0`, hits `p`, and stops there. It does not care what happens after that point. It just hands back what it managed to read before things broke.

The same stop-at-the-first-problem behavior applies to decimal points, since a decimal point is not part of a whole number:

```js
parseInt("123.45"); // 123
```

Trace through it the same way: JavaScript reads `1`, then `2`, then `3`. The next character is `.`, and a decimal point cannot be part of an integer, so reading stops right there. The `45` after it never even gets looked at. `parseInt()` just returns the `123` it already had, which becomes important later, because it means you can silently lose the decimal part of a number without any warning.

### The radix argument, and why you'll see it everywhere

`parseInt()` takes a second argument that tells it what numeric base to read in:

```js
parseInt("250px", 10); // 250, explicitly base 10
```

In older JavaScript engines, leaving this out could cause a string like `"08"` to be read as octal and produce an unexpected result. Modern engines no longer do this, so `parseInt("08")` reliably gives you `8` today. Even so, you will see `parseInt(x, 10)` all over production codebases and style guides. It costs nothing to add, it removes any ambiguity for the next person reading your code, and it has simply stuck around as a habit worth keeping.

If you genuinely want to read a different base, this is what makes it possible:

```js
parseInt("1010", 2); // 10, reading "1010" as binary
```

## parseFloat(): the same idea, but it keeps the decimal

Here's where a lot of explanations stop short, and it is exactly where the real bugs show up. Go back to the CSS example, but this time the value has a decimal point, like a value pulled from a `rem` or `em` unit:

```js
parseFloat("3.14rem"); // 3.14
```

`parseFloat()` works exactly like `parseInt()` — start at the beginning, read until something breaks — except it understands decimal points and keeps reading through them. Compare it directly against `parseInt()` on the same value to see why this matters:

```js
parseInt("3.14rem");   // 3, stops at the decimal point
parseFloat("3.14rem"); // 3.14, reads straight through it
```

If you needed the actual measurement and used `parseInt()` out of habit, you would have just silently truncated it to `3`. This is a genuinely common bug. Someone reaches for `parseInt()` because it is the more familiar name, the code runs without errors, and the decimal portion of every value quietly disappears.

`parseFloat()` does not take a radix argument the way `parseInt()` does. There is no equivalent of binary or hexadecimal for a decimal number in this context, so the option simply does not exist.

## The same messy string, three different answers

The clearest way to see how these three functions differ is to run the exact same input through all of them. Picture a value scraped from a webpage that you need to turn into a usable number: `"123.45kg"`.

```js
Number("123.45kg");     // NaN, the entire string isn't a valid number
parseInt("123.45kg");   // 123, stops at the decimal point
parseFloat("123.45kg"); // 123.45, reads through the decimal, stops at "k"
```

Three different results from one input, and all three are correct, because none of these functions is trying to answer the same question. If you remember nothing else from this section, remember this:

```
Number()      asks: is the entire string a valid number?
parseInt()    asks: what whole number can I read from the start?
parseFloat()  asks: what decimal number can I read from the start?
```

## What NaN actually is, and why it behaves so strangely

When `Number()` cannot make sense of what you gave it, it does not throw an error. It hands back a special value called `NaN`, short for "Not a Number." That name makes it sound straightforward, but `NaN` has two specific behaviors that catch people off guard, and both of them have caused real bugs in production code.

### typeof NaN is "number"

Try this in a console and look at what comes back:

```js
typeof NaN; // "number"
```

That looks like a contradiction. Something literally named "Not a Number" reports its own type as `"number"`. The confusion comes from reading `NaN` as a description of what it *is*, when really it is just the name of one specific value, the same way `42` is the name of a specific value.

Go back to the very first idea in this post: JavaScript only has one numeric type, and every number-like value lives inside it, including the unusual ones. `Infinity` is not a number you can point to on a number line either, and it is still typed as `"number"`:

```js
typeof 42;       // "number"
typeof Infinity; // "number"
typeof NaN;      // "number"
```

`NaN` is not a separate category of thing sitting outside the number system. It is a value that lives inside JavaScript's `Number` type and represents "this calculation did not produce a usable result" — closer to an error code than to the absence of a number.

### NaN is never equal to itself, and this is what breaks real code

This is the behavior that actually causes bugs, so it is worth tracing through slowly, one concrete value at a time, rather than just stating the rule.

Go back to the spreadsheet import from the start of this post. Picture the exact row that caused the problem: the price cell contains the text `"N/A"`, and the quantity column says `5`. Here is the validation code again, with those two real values dropped in:

```js
const priceCell = "N/A";
const quantity = 5;

const price = Number(priceCell);
// price is now NaN

if (price === NaN) {
  return rejectRow("Invalid price");
}

return price * quantity;
```

Walk through exactly what happens, line by line, with these actual values:

1. `Number("N/A")` runs, and produces `NaN`. So `price` is now `NaN`.
2. The code checks `price === NaN`, which is really asking: does `NaN === NaN`?
3. That comparison is `false`. Always. `NaN` is the one value in JavaScript that is never equal to anything, not even to another `NaN`:

```js
NaN === NaN; // false
```

4. Because the check evaluated to `false`, the `if` block never runs. The function does not return early. It falls straight through to the last line instead.
5. That last line runs `price * quantity`, which with these values is `NaN * 5`, which is `NaN`.
6. The function returns `NaN` as if it were a normal line total. No error gets thrown anywhere in this entire sequence. Nothing logs a warning. The bad row just keeps moving through the system as a number-shaped value, until it eventually shows up somewhere downstream, like a total on a dashboard, which is exactly the symptom that started the investigation.

This is not a JavaScript quirk invented out of nowhere. It comes from the IEEE-754 floating-point standard, the same standard mentioned earlier that JavaScript uses to store every number. `NaN` represents an unknown or invalid numeric result, and the standard defines anything unknown as not equal to anything else, including itself. JavaScript just inherited that rule, along with every other language built on the same standard.

The fix is a one-function swap, and tracing the same `"N/A"` row through it shows exactly why it works:

```js
const priceCell = "N/A";
const quantity = 5;

const price = Number(priceCell);
// price is still NaN, same as before

if (Number.isNaN(price)) {
  return rejectRow("Invalid price");
}
```

`Number.isNaN(price)` asks a much simpler question than `price === NaN` ever could: is this value actually `NaN`? Since `price` genuinely is `NaN`, the check correctly returns `true`, the `if` block runs, and the function returns `"Invalid price"` before it ever reaches the multiplication. The row gets rejected the way it was always supposed to.

## isNaN() vs Number.isNaN(): two different questions wearing similar names

This pair causes almost as much confusion as `NaN === NaN` does, because the two function names look nearly identical but check completely different things.

**`Number.isNaN()` asks: is this value literally `NaN`, with no conversion involved?**

```js
Number.isNaN(NaN);   // true
Number.isNaN("abc"); // false
```

`"abc"` returns `false` here because it is a string, not the actual `NaN` value. `Number.isNaN()` never tries to convert anything. It just checks exactly what you handed it.

**`isNaN()` asks: if I convert this to a number first, would the result be `NaN`?**

```js
isNaN("abc"); // true
```

This is the part that trips people up. `isNaN()` quietly runs `Number("abc")` behind the scenes, gets `NaN` back, and then checks whether that result is `NaN`. It is, so the function returns `true`, even though the original value was a plain string that was never `NaN` to begin with. The hidden conversion is doing all the work, and it is easy to forget it is happening.

You can see the difference clearly by walking through what each one does step by step.

```js
isNaN("abc");
// Internally: Number("abc") runs first -> NaN
// Then: is that result NaN? -> true
```

```js
Number.isNaN("abc");
// No conversion happens at all.
// Is "abc" literally NaN? No, it's a string.
// -> false
```

Here's the comparison laid out in full:

| Value | isNaN() | Number.isNaN() |
|---|---|---|
| NaN | true | true |
| "abc" | true | false |
| "123" | false | false |
| 123 | false | false |
| undefined | true | false |
| null | false | false |

Look closely at the `undefined` row, because it shows exactly how the hidden conversion can bite you. `isNaN(undefined)` returns `true`, because `Number(undefined)` produces `NaN`. But `undefined` usually means "this value was never set," not "someone typed something invalid." If your code uses `isNaN()` to validate a form field, an untouched field and a genuinely broken field both get treated the same way, which is rarely what you actually want.

This is why most production code defaults to `Number.isNaN()`. It never performs a conversion behind your back, so it never classifies something as `NaN` unless it genuinely already is. The standard pattern looks like this:

```js
const quantity = Number(userInput);

if (Number.isNaN(quantity)) {
  console.log("Please enter a valid quantity");
}
```

You convert once, deliberately, with `Number()`. Then you check the actual result with `Number.isNaN()`, which does exactly what it says and nothing more.

## Number.isFinite(): catches the case Number.isNaN() misses

`Number.isNaN()` solves the spreadsheet import bug, but it does not solve every input problem. Imagine a quantity column on the same import. A store manager should never be able to upload a row with `Infinity` units of something, but `Number.isNaN()` would not catch that case at all, because `Infinity` is not `NaN`:

```js
Number.isNaN(Infinity); // false
```

`Number.isFinite()` closes this gap. It rejects `NaN`, `Infinity`, and `-Infinity` all at once, while still accepting any ordinary number:

```js
Number.isFinite(5);        // true
Number.isFinite(NaN);      // false
Number.isFinite(Infinity); // false
```

A safer version of the quantity check looks like this:

```js
const quantity = Number(userInput);

if (!Number.isFinite(quantity)) {
  console.log("Please enter a valid quantity");
}
```

If you are validating anything a user could type a number into — a price, a quantity, an age — `Number.isFinite()` is usually the safer default over `Number.isNaN()` alone, simply because it covers more of the ways a number can go wrong.

## Common pitfalls

Everything above explains why these functions behave the way they do. This section is the part worth bookmarking — the specific traps that show up repeatedly in real codebases and in interviews.

**Reaching for `parseInt()` when you actually need strict validation.** `parseInt()` is forgiving by design, which is exactly the wrong behavior when you are trying to reject bad input rather than extract a number from it. Picture an age field on a signup form:

```js
parseInt("25years old"); // 25
```

The invalid part gets silently thrown away, and a clearly malformed input gets treated as a valid age. If you are validating rather than extracting, `Number()` is the right tool, because it rejects the whole thing:

```js
Number("25years old"); // NaN
```

**Losing the decimal portion of a number by using `parseInt()` out of habit.** This was covered above with the CSS example, but it deserves repeating, because it is one of the most common silent bugs around: any time the value might have a decimal component, `parseInt()` will quietly drop it.

```js
parseInt("99.99"); // 99
```

**Passing a number instead of a string into `parseInt()` can silently mangle it.** This one is rare, but it has a genuinely surprising mechanism worth knowing. `parseInt()` only knows how to read strings, so if you hand it an actual number, JavaScript converts it to a string first, behind the scenes, before `parseInt()` ever sees it. Most of the time that conversion is harmless. But very small or very large numbers convert to scientific notation when stringified, and that is where things go wrong:

```js
(0.0000003).toString(); // "3e-7"
parseInt(0.0000003);    // 3
```

Trace through exactly what happens with this input, `0.0000003`:

1. You call `parseInt(0.0000003)`, passing in a number, not a string.
2. Before `parseInt()` can do anything, JavaScript converts that number to a string, the same way `(0.0000003).toString()` would. The result of that conversion is `"3e-7"`, JavaScript's scientific-notation way of writing `0.0000003`.
3. `parseInt()` only ever sees the string `"3e-7"`. It has no idea what the original number was.
4. It reads the first character, `3`, which is a valid digit, so it keeps that.
5. The next character is `e`, which is not a valid digit, so reading stops immediately.
6. `parseInt()` returns `3`, the only digit it managed to read before hitting the letter.

The result, `3`, has nothing to do with the original value of `0.0000003`. It is just the first digit of the scientific-notation string that got created in step 2. The same thing happens with extremely large numbers, since those also stringify into scientific notation. This is uncommon in everyday code, since you would usually be parsing user input as a string to begin with, but it is a real trap if a number ever ends up passed into `parseInt()` directly instead of through a string field, and it is a fairly well-known one in interview settings precisely because the result looks so disconnected from the input.

**Empty strings convert to `0`, not `NaN`.** This one surprises almost everyone the first time they hit it. Picture a quantity field that gets left blank, so the value coming in is an empty string, `""`:

```js
Number("");  // 0
```

This feels like it should fail, but it doesn't. JavaScript treats an empty or whitespace-only string as "no value provided, default to zero," not as invalid input, so the result is `0`, not `NaN`. The same thing happens with a string that's just spaces:

```js
Number(" "); // 0
```

If you are checking whether a field was actually filled in, `Number.isNaN()` on the result will not catch this case, because the result genuinely is not `NaN` — it's a valid-looking `0`. You need a separate check for an empty string before you even get to the number conversion.

Now run the exact same empty string through `parseInt()` instead:

```js
parseInt(""); // NaN
```

Here, `parseInt()` tries to read a number starting from the beginning of the string, finds nothing there at all, and has nothing to return — so it gives back `NaN`. The same blank input, `""`, produces `0` from one function and `NaN` from the other. That's exactly the kind of inconsistency that causes bugs if a codebase mixes both functions without realizing they disagree on this case.

**Zero is falsy, and that breaks naive validation checks.** This is a classic, and it looks completely reasonable until you trace through it:

```js
const quantity = Number("0");

if (!quantity) {
  console.log("Please enter a quantity");
}
```

Walk through it with this exact input, `"0"`:

1. `Number("0")` runs, and correctly returns the number `0`. The conversion is not the problem — this step works exactly as it should.
2. `quantity` is now `0`.
3. The check is `!quantity`, which means "the opposite of whatever `quantity` is, treated as true or false." In JavaScript, `0` counts as a falsy value, so `!0` evaluates to `true`.
4. Because the condition is `true`, the `if` block runs, and `"Please enter a quantity"` gets logged.

The message logs even though `0` is a perfectly valid quantity to reject an order for, or to represent in plenty of other contexts. The bug is not in the conversion. It's in step 3 — `!quantity` treats `0` the same way it would treat an empty string or `undefined`, as something to reject. Use `Number.isNaN(quantity)` instead of `!quantity` any time zero is a value you need to accept, since `Number.isNaN(0)` correctly returns `false`.

**Thousands separators are not understood by either function, and they fail in different ways.** Picture a price pasted in from a spreadsheet, formatted as `"1,250.50"`:

```js
Number("1,250.50");    // NaN
parseFloat("1,250.50"); // 1
```

Trace through both with this same input. `Number("1,250.50")` checks whether the entire string is a valid number. It reads `1`, then immediately hits a comma, which is not a valid character anywhere in a JavaScript number literal. Since the whole string has to be valid, the comma invalidates everything, and the result is `NaN`.

`parseFloat("1,250.50")` works differently. It reads `1`, then also hits the comma — but instead of failing the whole thing, it just stops reading right there and returns the `1` it already had. The `,250.50` after the comma is simply ignored.

Same input, two completely different failure modes: one rejects everything, the other quietly returns a number that is wildly wrong. Neither function strips commas for you. If you are handling formatted numbers from spreadsheets or user input, you need to remove the separators yourself before conversion ever happens.

**Currency symbols cause the same kind of failure.** A price field that still has the dollar sign attached, maybe from a copy-paste:

```js
Number("$19.99"); // NaN
```

The `$` is not part of any valid number, so the entire string gets rejected. Strip the symbol out first.

**Floating-point arithmetic is not exact, and this is not a JavaScript bug specifically.**

```js
0.1 + 0.2; // 0.30000000000000004
```

This happens because of how the IEEE-754 standard stores decimal fractions in binary, and it shows up in nearly every language built on that same standard, not just JavaScript. If you are doing anything involving money, do not store amounts as raw floating-point numbers. Work in integer cents internally, or use a dedicated decimal library, and only format back to a decimal display value at the very end.

**`typeof NaN` returning `"number"` is a recurring interview question for a reason.** It was covered earlier in this post, but it is worth listing here too, since it is exactly the kind of detail that gets asked specifically because it feels like it should be wrong:

```js
typeof NaN; // "number"
```

## When to reach for each one

A short, practical summary, based on the scenarios covered above:

Use `Number()` any time you are validating input and need the entire value to be clean, with nothing left over — a price field, an age field, anything where a partially valid string should count as completely invalid.

Use `parseInt()` when you specifically want a whole number and expect there might be extra characters after it, like reading a pixel value out of a computed style. Pass `10` as the radix as a habit, even though modern engines rarely need it.

Use `parseFloat()` when you want that same forgiving behavior, but the value might have a decimal portion you cannot afford to lose, like a measurement in `rem` or `em`.

## The mental model worth keeping

If you take one thing away from this post, take this:

```
Number()        validates the entire string, all or nothing
parseInt()      reads a whole number from the start, stops at the first break
parseFloat()    reads a decimal number from the start, stops at the first break
isNaN()         converts first, then checks
Number.isNaN()  checks only, no conversion
```

Every example in this post, including the spreadsheet import bug at the very top, traces back to one of these five ideas. The bug was not exotic. It was one line of code that looked completely reasonable, sitting on top of a fact about `NaN` that almost nobody is taught explicitly until it costs them a debugging session.

If you want to keep pulling on this thread, the next natural stop is truthy and falsy values, and `==` versus `===`. The falsy-zero pitfall above is a small preview of how much of JavaScript's "weird" behavior comes down to the same kind of thing: not a bug in the language, just a rule that nobody mentioned until it mattered.