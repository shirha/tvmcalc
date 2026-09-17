# tvmcalc

### `futureValue(PV, PMT, annualRate, n, m)`
\[
FV = -\Bigl(PV\cdot(1+i)^n + PMT\cdot\frac{(1+i)^n-1}{i}\Bigr)
\]

### `rate(PV, PMT, FV, n, m)`
Solves for the periodic rate `i` using **bisection** on the residual function

\[
f(i) = -(PV\cdot(1+i)^n + PMT\cdot\frac{(1+i)^n-1}{i}) - FV
\]
