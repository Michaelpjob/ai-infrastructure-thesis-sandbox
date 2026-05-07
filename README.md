# AI Infrastructure Thesis Sandbox

A local-first thesis-tracking sandbox for hypothetical AI infrastructure allocations.

Live: https://michaelpjob.github.io/ai-infrastructure-thesis-sandbox/

Open `index.html` in a browser (or visit the live link above), adjust weights, lock a scenario to capture baseline prices, then revisit after quote refreshes to see cumulative hypothetical performance versus equal-weight and SPY benchmarks.

## Notes

- Data is stored in `localStorage` under `ai-infra-tracker-v1`.
- Quote refresh supports Stooq by default, plus Finnhub and Alpha Vantage with an API key.
- Tracking is hypothetical. Prices are end-of-day or delayed. This is not a brokerage.
