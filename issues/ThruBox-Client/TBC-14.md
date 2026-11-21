# [FEATURE]: Test the uncovered branches in `src/client.ts`

### Feature and its Use Cases

`npm run coverage` on `main` (9afc707) passes all 30 tests, but several error-handling paths in `src/client.ts`, the only file with logic, never run:

```
File       | % Stmts | % Branch | % Funcs | % Lines
All files  |   91.91 |    83.78 |   94.11 |    93.7
 client.ts |   90.98 |    83.78 |    90.9 |   92.92
```

Paths with no test:

| Line(s) | Path |
|---|---|
| 256 | The real timeout: the `setTimeout(() => controller.abort(), this.timeout)` callback never fires. The existing "timeout" test (`tests/client.test.ts:198-203`) only makes `fetch` reject with an `AbortError` directly. |
| 268-269 | 2xx with status 204 returns `undefined` |
| 291 (false branch) | 429 without a `Retry-After` header falls back to 60 |
| 295-298 | `Retry-After` given as an HTTP date |
| 281, 308 | 404 / other 4xx with an empty body falls back to "Not found" / "Request failed" |
| 332-333 | `fetch` rejects with something that is not an `Error` ("Unknown error") |
| 328-329 | the `error instanceof RelayError` branch for status 0 or 5xx. Nothing inside the `try` throws such an error today, so this looks unreachable. Either test it with a mocked rejection or remove it. |
| 158-159 | `poll()` rejects an invalid `intervalMs` |
| 170, 185 | errors and polls after `stop()` are ignored |
| 179-180 | `poll()` without `onError` logs through `console.error` |

Also, no test on `main` checks that `send()` is **not** retried on a 5xx, which the README and the `send()` JSDoc rely on (`maxRetries = 0` for POST, `src/client.ts:250`). Open PR #83 adds such a test ("should not retry by default").

`BestPracticesChecklist.md` lists `test_most` as "Not measured".

**Proposed change**

- Add one focused test per row above. Use `vi.useFakeTimers()` with a `fetch` mock that never resolves (but rejects when its `signal` aborts) for the real timeout, and `vi.setSystemTime()` for the HTTP-date `Retry-After`.
- Add a test that `send()` makes exactly one `fetch` call on a 503 and on a network error, if #83 has not landed by then.
- Tests only: no change to `vitest.config.ts` and no behaviour change in `src/`. The only possible edit there is removing the unreachable branch at 328-329, if the maintainers prefer that. Bugs found while writing the tests should go in separate issues.

### Additional Context

- Fixes for the open bugs (#75, #80/#81, #82/#83) will add tests to the same file. This issue only covers behaviour that is already correct but untested, to avoid overlapping with those PRs.
- Coverage thresholds in `vitest.config.ts` are left to the CI/test-infrastructure work, so they can be set once these tests have landed.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
