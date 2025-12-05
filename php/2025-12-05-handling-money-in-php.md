---
title: "Handling Money in PHP: Stop Using Floats and Start Using Minor Units"
excerpt: "Stop risking financial data corruption with floating-point math. Learn the robust, architectural approach to handling money in PHP using minor units and a strict separation of storage and display logic."
slug: "2025-12-05-handling-money-in-php"
published_at: 2025-12-05
author: "gocanto"
categories: "php"
tags: ["floating-point", "php", "money", "risk", "finance"]
---

If you’ve been building web applications for any length of time, you’ve likely encountered the infamous floating-point math problem.

You try something simple in PHP:

```php
echo 0.1 + 0.2; // Result: 0.30000000000000004
```
    
In most applications, that tiny fraction doesn't matter. In e-commerce, fintech, or payroll—areas where I've spent years integrating payment rails, a tiny fraction gets you fired.

The golden rule of handling money in software development is simple: **Never use floating-point numbers for currency calculations.**

Instead, we use integers representing the smallest unit of currency (often called "minor units") and use the standard [moneyphp/money](https://github.com/moneyphp/money "null") library to handle the heavy lifting.

But even when using the right library, developers often stumble on the crucial steps of getting data _out_ of the money object for two distinct purposes: **Database Storage** and **User Display**.

Let's look at the best, safest way to parse monetary amounts in PHP using a robust, static utility class.


### The Terminology Shift: Forget "Cents"

Before we look at the code, we need to fix a common naming mistake.

When asking for the integer value of money, developers often name methods things like `getAmountInCents()`.

This is dangerous because it assumes every currency works like the US Dollar or Euro (where one central unit = 100 minor units).

- **USD ($10.00):** Minor unit is cents. Stored as `1000`.
- **JPY (¥1000):** The Japanese Yen has no decimal places. The major unit is the minor unit. Stored as `1000`.
- **BHD (BD 1.000):** The Bahraini Dinar has three decimal places. Stored as `1000`.
    

If you hardcode `/ 100` math into your display logic, your application cannot handle international currencies correctly.

We need a solution that handles **Storage** (integers) and **Display** (strings) independently, respecting the rules of the specific currency being used.


### The Solution: A Static Utility Class

Instead of binding this logic to specific models via Traits, we can create a dedicated, static utility class. This allows us to use these conversion methods anywhere in our application—Controllers, Jobs, DTOs, or Listeners—without strict dependencies.

Here is a modern PHP class that handles both scenarios correctly.

```php
<?php

declare(strict_types=1);

namespace App\Support;

use Money\Currencies\ISOCurrencies;
use Money\Formatter\DecimalMoneyFormatter;
use Money\Money;
use OverflowException;

class ConvertsMoney
{
    /**
     * Use this for DATABASE STORAGE.
     * Returns the integer minor unit (e.g., 1050 for $10.50, 1050 for ¥1050).
     * Safe against integer overflow.
     */
    public static function toStorageFormat(Money $money): int
    {
        // MoneyPHP stores amounts internally as strings to support massive numbers.
        $amountStr = $money->getAmount();

        // We must ensure this string actually fits into a PHP integer before casting.
        // We use BC Math (bccomp) to compare the string values safely.

        // 1. Check if it's too big
        if (bccomp($amountStr, (string) PHP_INT_MAX) === 1) {
            throw new OverflowException("Money amount '{$amountStr}' exceeds current system's PHP_INT_MAX");
        }

        // 2. Check if it's too small (negative amounts for refunds/debts)
        if (bccomp($amountStr, (string) PHP_INT_MIN) === -1) {
            throw new OverflowException("Money amount '{$amountStr}' is below current system's PHP_INT_MIN");
        }

        // It is now safe to cast to int.
        return (int) $amountStr;
    }

    /**
     * Use this for FRONTEND DISPLAY.
     * Returns the human-readable string in major units (e.g., "10.50" or "1050").
     * Automatically handles currencies with different decimal counts.
     */
    public static function toDisplayFormat(Money $money): string
    {
        // Load standard ISO currencies rules (USD=2 decimals, JPY=0, BHD=3, etc.)
        $currencies = new ISOCurrencies();
        
        // This formatter knows where to put the decimal point based on the currency.
        $moneyFormatter = new DecimalMoneyFormatter($currencies);

        return $moneyFormatter->format($money);
    }
}
```


### Why this approach is better

Let's break down why using this class is preferred over naive implementations.

#### 1\. Safety First: The Storage Method

The `toStorageFormat` method addresses a hidden danger. The `moneyphp` library stores its internal value as a string. It does this so it can handle quadrillions of dollars without losing precision.

However, your database's `BIGINT` column and PHP's `int` type have limits (`PHP_INT_MAX`).

If you blindly cast a considerable Money value to `(int)`, PHP might silently wrap the number around to negative values, causing catastrophic financial data corruption.

This class uses `bccomp` (Binary Calculator string comparison) to ensure the monetary amount actually fits inside your system's integer limits _before_ casting it.

#### 2\. Accuracy Second: The Display Method

The `toDisplayFormat` method completely avoids doing math.

Do not do this:

```php
// BAD: Assuming everything has two decimals
echo $money->getAmount() / 100; 
```

Instead, we use the `DecimalMoneyFormatter`. It looks up the currency code in the Money object (e.g., 'USD' or 'JPY'), consults the ISO rules for that currency, and determines exactly where the decimal point should go.


### Seeing it in action

Here is how you might use this class in a typical workflow, such as processing a transaction.

```php
use Money\Money;
use App\Support\ConvertsMoney;

class TransactionProcessor
{
    public function process(Money $total): void
    {
        // --- 1. Storage ---
        // We need the raw integer for the database 'amount' column.
        // We call the static method directly.
        $dbValue = ConvertsMoney::toStorageFormat($total);
        $currencyCode = $total->getCurrency()->getCode();
        
        // simulated DB save: 
        // INSERT INTO transactions (amount, currency) VALUES ($dbValue, $currencyCode);
        echo "Saving integer {$dbValue} ({$currencyCode}) to database...\n";


        // --- 2. Display ---
        // We need a pretty string to show the user on the receipt.
        $uiValue = ConvertsMoney::toDisplayFormat($total);
        
        echo "Sending receipt for amount: {$uiValue}\n";
    }
}

// --- Examples ---

$processor = new TransactionProcessor();

echo "--- Processing USD ---\n";
// $10.50
$usd = Money::USD(1050); 
$processor->process($usd);
// Output:
// Saving integer 1050 (USD) to database...
// Sending receipt for amount: 10.50


echo "\n--- Processing JPY ---\n";
// ¥1050 (Yen has no decimals)
$jpy = Money::JPY(1050); 
$processor->process($jpy);
// Output:
// Saving integer 1050 (JPY) to database...
// Sending receipt for amount: 1050
```

By separating storage concerns from display concerns into a dedicated utility, you ensure your application’s financial data is both accurate in the database and understandable to your users, regardless of the currency they use.

#php #money #architecture #fintech



