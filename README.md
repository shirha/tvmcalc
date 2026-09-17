# tvmcalc

### `futureValue(PV, PMT, annualRate, n, m)`
\[
FV = -\Bigl(PV\cdot(1+i)^n + PMT\cdot\frac{(1+i)^n-1}{i}\Bigr)
\]

### `presentValue(PMT, FV, annualRate, n, m)`
\[
PV = -\frac{FV + PMT\cdot\frac{(1+i)^n-1}{i}}{(1+i)^n}
\]
Special case when `i ≈ 0`: \( PV = -(FV + PMT\cdot n) \).

### `payment(PV, FV, annualRate, n, m)`
\[
PMT = -\frac{(PV\cdot(1+i)^n + FV)\cdot i}{(1+i)^n - 1}
\]
Special case when `i ≈ 0`: \( PMT = -(PV + FV)/n \).

### `periods(PV, PMT, FV, annualRate, m)`
\[
n = \frac{\ln\bigl(\frac{PMT - FV\cdot i}{PV\cdot i + PMT}\bigr)}{\ln(1+i)}
\]
Special case when `i ≈ 0`: \( n = -(PV + FV)/PMT \).

### `rate(PV, PMT, FV, n, m)`
Solves for the periodic rate `i` using **bisection** on the residual function

\[
f(i) = -(PV\cdot(1+i)^n + PMT\cdot\frac{(1+i)^n-1}{i}) - FV
\]

[tvmcalc](https://shirha.github.io/tvmcalc/tvmcalc.html) [MD](https://github.com/shirha/tvmcalc/blob/main/tvmcalc.md)
