# TVM Calculator – Detailed Documentation

**TVM Calculator** is a single-page web application for solving Time Value of Money problems.  
It allows the user to enter any four of the five classic TVM variables and solve for the fifth.

**Variables**
| Variable | Meaning                          | Unit / Notes                     |
|----------|----------------------------------|----------------------------------|
| PV       | Present Value                    | Currency (can be negative)       |
| PMT      | Periodic Payment                 | Currency (can be negative)       |
| FV       | Future Value                     | Currency (should be positive)    |
| RATE     | Annual interest rate             | Percentage (e.g. 20 = 20 %)      |
| PERIODS  | Number of compounding periods    | Integer or decimal               |

**Sign convention**  
- Money received or loaned **to you** → positive  
- Money paid or invested **by you** → negative  

**Rate types**  
- **Eff** (Effective) – true annual effective rate  
- **Nom** (Nominal) – annual rate that is divided by the compounding frequency  

**Compounding frequencies**  
1 (Annual), 2 (Semiannual), 4 (Quarterly), 12 (Monthly), 52 (Weekly), 365 (Daily)

---

## 1. Global State & DOM References

```js
const fields = ["PV", "PMT", "FV", "RATE", "PERIODS"];
const currencyFields = ["PV", "PMT", "FV"];
const els = { PV:…, PMT:…, FV:…, RATE:…, PERIODS:… };
```

- `fields` – ordered list of the five TVM variables.  
- `currencyFields` – subset that receive currency formatting (commas).  
- `els` – quick lookup object mapping variable name → `<input>` element.  
- Other important elements: `frequency` (select), `rateTypeButtons` (radio), `solveButtons` (PV/PMT/FV/RATE/PERIODS), `PmtxPer` (read-only total-payments display), `minus`, `clear`, `guide`, `help`.

Mutable state variables (inside the IIFE):
- `solveFor` – currently selected variable to solve for (default `"FV"`).
- `rateType` – `"effective"` or `"nominal"` (default `"effective"`).
- `trgt` – last focused input (used by the minus button).

---

## 2. Number Cleaning & Formatting Helpers

### `cleanNumber(str)`
Removes everything except digits, a single leading minus sign, and a single decimal point.  
Used on every keystroke and before parsing.

### `getValue(id)`
1. Calls `cleanNumber` on the input’s current value.  
2. Parses with `parseFloat`.  
3. Returns the number or `NaN` if invalid.

### `setValue(id, value)`
Writes a calculated result back into the corresponding input:
- Clears the field if the value is not finite.
- Forces values with absolute value < 0.005 to zero.
- Currency fields → `formatCurrency`.
- RATE / PERIODS → rounded to 2 decimal places.

### `formatCurrency(value)`
Uses `toLocaleString("en-US", { maximumFractionDigits: 2 })` so the user sees commas (e.g. `-6,000.00`).

### `formatAllCurrencyFields()`
Convenience wrapper that re-formats PV, PMT and FV after loading from the URL.

---

## 3. Core TVM Mathematics

All formulas use the classic ordinary annuity equations (payments at the **end** of each period).

### `periodicRate(annualRate, m)`
Converts the annual rate into a periodic rate `i`.

- Effective: $\( i = (1 + r)^{1/m} - 1 \)$
- Nominal:   $\( i = r / m \)$

Returns `NaN` if `1 + annualRate ≤ 0` (effective case).

### `annualRateFromPeriodic(i, m)`
Inverse of the above – converts a solved periodic rate back to an annual percentage for display.

### `futureValue(PV, PMT, annualRate, n, m)`

$$
FV = -\Bigl(PV\cdot(1+i)^n + PMT\cdot\frac{(1+i)^n-1}{i}\Bigr)
$$

Special case when `i ≈ 0`: \( FV = -(PV + PMT\cdot n) \).

### `presentValue(PMT, FV, annualRate, n, m)`

$$
PV = -\frac{FV + PMT\cdot\frac{(1+i)^n-1}{i}}{(1+i)^n}
$$

Special case when `i ≈ 0`: \( PV = -(FV + PMT\cdot n) \).

### `payment(PV, FV, annualRate, n, m)`

$$
PMT = -\frac{(PV\cdot(1+i)^n + FV)\cdot i}{(1+i)^n - 1}
$$

Special case when `i ≈ 0`: \( PMT = -(PV + FV)/n \).

### `periods(PV, PMT, FV, annualRate, m)`

$$
n = \frac{\ln\bigl(\frac{PMT - FV\cdot i}{PV\cdot i + PMT}\bigr)}{\ln(1+i)}
$$

Special case when `i ≈ 0`: \( n = -(PV + FV)/PMT \).

**Important edge case**  
When the numerator and denominator are both zero (or extremely close to zero) the formula returns `NaN`.  
This occurs, for example, with:

```
PV = -6000, PMT = 100, FV = 6000, RATE = 20 %, Nominal, Monthly
```

because the payment exactly equals the interest on the balance, making the number of periods mathematically indeterminate.

### `rate(PV, PMT, FV, n, m)`
Solves for the periodic rate `i` using **bisection** on the residual function

$$
f(i) = -(PV\cdot(1+i)^n + PMT\cdot\frac{(1+i)^n-1}{i}) - FV
$$

- Search interval starts at `[-0.999999, 1]` and expands the upper bound if necessary.  
- 200 iterations or until the residual is smaller than `1e-7`.  
- Returns the midpoint of the final bracket (or `NaN` if no root is found).

After obtaining the periodic rate the function converts it back to an annual percentage with `annualRateFromPeriodic`.

---

## 4. UI / State Management Functions

### `clear()`
- Empties all five inputs.  
- Resets frequency to Monthly (12).  
- Resets rate type to Effective.  
- Sets `solveFor = "FV"`.  
- Updates button highlighting.  
- Removes the query string from the URL.

### `loadUrl()`
Parses the query string of the form:

```
?PV,PMT,FV,RATE,PERIODS,frequency,solveFor,rateType
```

Example:
```
?-6000,100,6000,20,600,12,PMT,Nom
```

- Populates the five fields (empty string → blank).  
- Formats currency fields.  
- Sets frequency, `solveFor` and `rateType`.  
- Returns `true` if data was present, `false` otherwise.

### `updateUrl()`
Writes the current values (clean numbers, no commas) plus frequency / solveFor / rateType back into the address bar using `history.replaceState`.  
This makes every calculation bookmarkable / shareable.

### `updateAppearance()`
Adds / removes the CSS class `solve` on the five variable buttons so the currently selected variable is visually highlighted.

### `calculate(flash = true)`
Main calculation orchestrator:

1. Highlights the active solve button.  
2. Reads all five values.  
3. Checks that the four *non-solved* variables are valid numbers.  
   - If any are missing and `flash === true`, adds the `invalid` CSS class (pink pulse animation) for 1.5 s.  
4. Calls the appropriate math function according to `solveFor`.  
5. Writes the result with `setValue`.  
6. When solving for PERIODS, also fills the “Pmt × Per” read-only field with the absolute total of all payments.  
7. Updates the URL.

---

## 5. Event Listeners

| Element / Event              | Action |
|------------------------------|--------|
| Variable buttons (`.key`)    | Set `solveFor` and call `calculate()` |
| Rate-type radios             | Update `rateType` and recalculate (no flash) |
| Frequency `<select>`         | Recalculate (no flash) |
| Currency inputs – focus      | Strip commas so the user can edit the raw number |
| Currency inputs – blur       | Re-apply currency formatting |
| All numeric inputs – input   | Live cleaning of illegal characters |
| Minus button (`#minus`)      | Toggles a leading minus on the last focused field (only PV or PMT) |
| Clear button                 | Calls `clear()` |
| Guide link                   | Toggles visibility of the help panel |
| `focusin` (document)         | Keeps track of the last focused element for the minus button |

---

## 6. Initialization Sequence

```js
const hasUrlData = loadUrl();
if (hasUrlData) calculate();
```

- Loads any values present in the query string.  
- Highlights the correct solve button.  
- Automatically performs a calculation if the URL contained data.  
- Displays client width / height / devicePixelRatio in the help panel (debug info).

A temporary debug object is also exposed on `window.tvm` for console experimentation.

---

## 7. URL Parameter Format (Shareable Links)

```
?PV,PMT,FV,RATE,PERIODS,frequency,solveFor,rateType
```

- Empty fields are represented by consecutive commas.  
- `solveFor` must be one of `PV|PMT|FV|RATE|PERIODS` (case-insensitive).  
- `rateType` accepts `Eff` / `effective` / `e` or `Nom` / `nominal` / `n`.  
- The page automatically recalculates on load when a valid query string is present.

Examples are embedded in the help panel (S&P 500 401k, Bitcoin Roth, mortgage, credit-card scenarios, Michael Saylor cases, etc.).

---

## 8. Known Edge Cases & Limitations

1. **Singular periods formula**  
   When \( PMT = -PV \cdot i \) and \( FV = -PV \) (or equivalent) both numerator and denominator become zero → `NaN` → blank field.  
   Most visible with Nominal rate + Monthly compounding and the classic “payment equals interest” situation.

2. **Rate solver**  
   The bisection method is robust for most practical cases but can fail (return `NaN`) for extreme or inconsistent cash-flow signs.

3. **Zero interest rate**  
   All four closed-form formulas contain special-case handling for `i ≈ 0`.

4. **Floating-point precision**  
   Very large numbers of periods or extreme rates can produce `Infinity` or loss of precision; the code does not currently guard against them beyond the built-in `Number.isFinite` checks.

5. **No beginning-of-period (annuity-due) mode**  
   All calculations assume ordinary annuities (end-of-period payments).

---

## 9. CSS Highlights

- `.key.solve` – light-blue background on the currently selected variable button.  
- `.invalid` – pink pulse animation when a required field is empty.  
- Responsive layout fixed at 300 px width for a mobile-friendly calculator feel.  
- Hidden scrollbar, clean minimal design.

---

This document covers every function, the mathematical model, the UI interaction model, and the shareable-URL mechanism of the TVM Calculator.
