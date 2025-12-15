---
title: "Money in Go, done properly"
excerpt: "Money bugs rarely crash your app. They quietly show up as one-cent mismatches, failed refunds, or totals that do not reconcile."
slug: "2025-12-15-money-in-Go-done-properly"
published_at: 2025-12-15
author: "gocanto"
categories: "go"
tags: ["engineering", "go", "money"]
---

![money](https://github.com/user-attachments/assets/6cc1810c-9d29-4160-a848-514353758bfe)


# Money in Go, done properly

If you have ever seen `0.1 + 0.2` turn into `0.30000000000000004`, you already know why money and floats do not mix. The `gocanto/money` Go module is a practical implementation of Martin Fowler’s Money pattern: treat money as a 
first-class value object, keep currency attached to the amount, and make rounding rules explicit so you do not lose cents over time. [GitHub](https://github.com/gocanto/money)

This post focuses on the new Go package (not the [PHP side](https://oullin.io/post/2025-12-05-handling-money-in-php)). Still, it follows the same core principles: store money in minor units as integers, separate storage from display, 
and make invalid operations impossible, or at least make them loud.

## What `gocanto/money` is

`gocanto/money` is a Go implementation of Fowler’s Money pattern, inspired by the `moneyphp/money` [library](https://github.com/moneyphp/money), but shaped into a Go-idiomatic API.

At a glance, it gives you:

-   Integer-based money arithmetic (no float math in core operations).
-   ISO 4217 currency dataset plus support for custom currencies.
-   Formatting and parsing for human-entered amounts, including separator handling.
-   Aggregation helpers (sum, min, max, avg).
-   Currency conversion via an exchange abstraction.
-   JSON marshalling and database support via `sql.Scanner` and `driver.Valuer`.
    

## Install and requirements

Install:

```bash
go get github.com/gocanto/money
```

The module targets Go `1.25.5` (per its `go.mod`).


## The mental model: Money is amount + currency

The library treats `Money` as a value that always carries:

-   an `amount` stored in minor units (like cents)
-   a `currency` definition (code, fraction, symbol, separators, and more)
    

That means “$10.00” is stored as `1000` minor units for USD (2 decimals). The key win is that you stop passing naked numbers around your codebase.


## Quick start

Here is the simplest happy path:

```go
mm := money.NewManager()

price := mm.Create(10000, currency.USD) // $100.00
tax   := mm.Create(850, currency.USD)   // $8.50

total, _ := mm.Add(price, tax)

display, _ := total.Display()
fmt.Println(display) // $108.50
```

This “minor units everywhere” approach is the package's default stance.


## Doing arithmetic safely

In this library, you do operations through `money.Manager`. That makes it easy to keep validation rules consistent (like “same currency required”):

```go
mm := money.NewManager()

a := mm.Create(10000, currency.USD)
b := mm.Create(2500, currency.USD)

sum, _ := mm.Add(a, b)        // $125.00
diff, _ := mm.Subtract(a, b)  // $75.00
prod, _ := mm.Multiply(a, 2)  // $200.00
```

All of these avoid float math in the core calculations.

If you accidentally try to add different currencies, you get an error (and sentinel errors are available):

```go
usd := mm.Create(100, currency.USD)
eur := mm.Create(100, currency.EUR)

_, err := mm.Add(usd, eur) // currencies don't match
```


## Comparisons are currency-aware too

Money comparisons include currency checks, plus useful helpers:

```go
a := mm.Create(10000, currency.USD)
b := mm.Create(5000, currency.USD)

gt, _ := a.GreaterThan(b) // true
cmp, _ := a.Compare(b)    // 1
```


## Splitting and allocation without losing cents

This is where many “simple” implementations silently go wrong. If you split `$100.00` into three parts, you must decide where the extra cent goes.

The library gives you built-in split and ratio allocation:


```go
m := mm.Create(10000, currency.USD)

parts, _ := mm.Split(m, 3)
// $33.34, $33.33, $33.33

parts2, _ := mm.Allocate(m, 50, 30, 20)
// $50.00, $30.00, $20.00
```


## Display and formatting

You can render a money value in a currency-aware format:

```go
m := mm.Create(123456, currency.USD)

s, _ := m.Display() // "$1,234.56"
```

You can also get “major units”, but treat that as a display or UI concern, not a storage one:

```go
major, _ := m.AsMajorUnits() // 1234.56
```


## Parsing user input without reinventing the mess

User input is where things get messy fast: symbols, codes, commas, dots, and regional formats.

`parser.Parser` handles common patterns:

```go
p := parser.NewParser()  val, cur, _ := p.ParseAmount("$1,234.56") // 1234.56, "USD" val, cur, _ = p.ParseAmount("EUR 10,50")  // see note below
```

By default, comma-only inputs are treated as thousands separators, which can surprise you when dealing with European decimal formats. The library calls this out and provides comma-decimal helpers:

```go
p.ParseAmountWithDecimalComma("€10,50") // parses as 10.50 EUR p.ParseDecimalWithComma("10,50")        // parses as 10.50
```


## Aggregation and exchange

You can aggregate lists of money values through an `Aggregator`:

```go
agg := money.NewAggregator(mm)

total, _ := agg.Sum(prices...)
min, _ := agg.Min(prices...)
avg, _ := agg.Avg(prices...)
```

For conversion, you plug in an exchange rate source via `exchange.Exchange`:

```go
currencies := currency.NewManager()
ex := exchange.NewExchange()

_ = ex.AddRate(currency.USD, currency.EUR, 0.85)

converter, _ := money.NewConverter(currencies, ex)

usd := mm.Create(10000, currency.USD)
eur, _ := converter.Convert(usd, currency.EUR) // €85.00
```


## JSON and database support

If you are building APIs, it is helpful that `Money` marshals cleanly:

```json
{"amount":2999, "currency" :"USD"}
```

For databases, `Money` implements `sql.Scanner` and `driver.Valuer`, storing values as a delimited string, such as `amount|currency_code` (separator configurable).

That design is pragmatic: you can place it in a single column and still keep the currency attached to the amount.


## What’s different vs MoneyPHP

A few intentional Go-specific choices stand out:

-   Amounts are stored as `int64` (minor units), not strings.
-   A Go-idiomatic API, including helper constructors like `money.FromUSD()` and a `money.Manager` for operations.
-   Built-in DB support with `sql.Scanner` and `driver.Valuer`.
-   Aggregation and a simple in-memory exchange system are included.
    

## Best practices

If you only take a few rules into your codebase, take these:

1.  **Store minor units as integers everywhere** (DB, events, internal structs).
2.  **Use exact decimal strings for user-provided decimals when you can**, and parse them once at the boundary.
3.  **Use Split/Allocate for divisions** so rounding is handled consistently.
4.  **Keep currency mismatches as errors, not edge cases**. That is the whole point of using a Money type.


## Integration guide in a real Go service

### 1) Boundary rule: parse at the edge, store minor units inside

-   **From HTTP/UI**: prefer `CreateFromString("99.99", "USD")` for exact user input, and use floats only if you must.
-   **Inside the domain**: keep `*money.Money` (or `amount int64 + currency string`) in your structs and do arithmetic via `money.Manager`.
    

Example request DTO and mapping:
```go
type CreateInvoiceReq struct {
	Amount   string `json:"amount"`   // "99.99"
	Currency string `json:"currency"` // "USD"
}

func (s *Service) CreateInvoice(req CreateInvoiceReq) (*Invoice, error) {
	mm := money.NewManager()

	price, err := mm.CreateFromString(req.Amount, req.Currency)
	if err != nil {
		return nil, err
	}

	//Perform all operations in mm.Add/Subtract/Split/Allocate, etc.
	return &Invoice{Total: price}, nil
}
```

`parser.Parser` is proper when you accept human-formatted strings like `$1,234.56` or `EUR 10,50`, and it has explicit helpers for decimal-comma inputs.


### 2) JSON APIs: stable representation

The README shows JSON like:

```json
{"amount":2999, "currency":"USD"}
```

So a clean API approach is:

-   **External**: send `amount` as minor units (int) plus currency code.
-   **UI**: format with `.Display()` on the server or client as needed.


### 3) Database storage options

**Option A (simple): single column using Scanner/Valuer**  
The package supports storing money as a delimited string, such as `2999|USD`, via `sql.Scanner` and `driver.Valuer`.

```go
type Product struct {
	ID    int
	Price *money.Money
}

// INSERT will store as: 2999|USD
_, _ = db.Exec(`INSERT INTO products (price) VALUES (?)`, p.Price)
```
**Option B (more queryable): two columns**  
If you need to query ranges and aggregate in SQL, consider:

-   `price_amount BIGINT NOT NULL` (minor units)
-   `price_currency CHAR(3) NOT NULL`
    

Then reconstruct `Money` in Go:
```go
m := money.NewManager().Create(priceAmount, priceCurrency)
```
Both approaches still follow the best practice of storing minor units.


### 4) Currency conversion: keep it explicit

Conversion is done via a `Converter` that depends on a currency manager and an exchange rate source. The README includes an in-memory exchange (`exchange.NewExchange`) where you add rates and then convert.

In production, you can wrap your FX provider behind a small interface that feeds rates into the converter, and you can decide:

-   When to snapshot rates.
-   How to audit which rate was used for each transaction.


5) Common service patterns that avoid bugs

- Use Split() / Allocate() for proration (discounts, revenue share) so rounding is consistent.
- Treat currency mismatch as a fundamental error, not a special case. Sentinel errors exist (example: exception.ErrCurrencyMismatch). 

In tests, assert on minor units and currency code. Keep display-formatting tests separate, since formatting varies by currency rules.

## Outro

Money bugs are rarely loud. They show up later as one-cent mismatches, invoice totals that do not reconcile, or “impossible” refunds that only fail in production. The point of `gocanto/money` is to make those mistakes harder to write in the first place by keeping currency attached to every amount, using integers for storage and arithmetic, and giving you clear tools for rounding, splitting, allocation, parsing, and conversion.

If you are building anything that charges, refunds, prorates, splits revenue, or reports financial totals, this package is a solid baseline. Start by parsing at the edges, store minor units in the database, and keep Money as a first-class type throughout your domain. That alone removes a whole category of “we will fix it later” accounting surprises.












