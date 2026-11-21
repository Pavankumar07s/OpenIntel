# [FEATURE]: Speed up base64 encoding of binary values (serialize is about 12× slower than it needs to be on large buffers)

### Feature and its Use Cases

`uint8ArrayToBase64` (`src/serialization/index.ts:16-22`) builds the binary string one byte at a time before calling `btoa`:

```ts
let binary = '';
for (let i = 0; i < bytes.length; i++) {
  binary += String.fromCharCode(bytes[i]!);
}
return btoa(binary);
```

Every `Uint8Array`, `ArrayBuffer`, other typed array and `DataView` in a backup goes through this function (lines 159, 170, 177). For apps that store binary blobs (images, files, encrypted payloads), it is the slowest part of `exportDB`.

**Measurement** on `main` (4b90eca): `serialize()` vs. the same encoding done in 32 KiB chunks. Node 24.22, Vitest 5.0.1, happy-dom environment, as in the test suite. The output was byte-for-byte identical in every case.

| Buffer | current | chunked | `deserialize` (unchanged) |
| --- | --- | --- | --- |
| 1 MiB | 83 ms | 7 ms | 4 ms |
| 4 MiB | 374 ms | 39 ms | 14 ms |
| 16 MiB | 1827 ms | 146 ms | 54 ms |

**Proposed change:**

```ts
function uint8ArrayToBase64(bytes: Uint8Array): string {
  let binary = '';
  const CHUNK = 0x8000; // stays well below engine argument-count limits
  for (let i = 0; i < bytes.length; i += CHUNK) {
    binary += String.fromCharCode.apply(
      null,
      bytes.subarray(i, i + CHUNK) as unknown as number[],
    );
  }
  return btoa(binary);
}
```

- The output format is unchanged, so existing backups stay compatible.
- Optionally, use the native `Uint8Array.prototype.toBase64()` / `Uint8Array.fromBase64()` when the runtime has them, and keep the chunked version as the fallback.
- Add a round-trip test with a buffer larger than one chunk (for example 100 000 bytes), so the chunk boundaries are covered.

### Additional Context

- Pure performance change in one function. No API change.
- No existing issue or PR covers it. PR #127 touches the call sites but not this function.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
