# noble-ed25519

[Fastest](#speed) 4KB JS implementation of ed25519 [EDDSA](https://en.wikipedia.org/wiki/EdDSA)
signatures compliant with [RFC8032](https://tools.ietf.org/html/rfc8032),
FIPS 186-5 & [ZIP215](https://zips.z.cash/zip-0215).

If you're looking for additional features,
check out [noble-curves](https://github.com/paulmillr/noble-curves):
a drop-in replacement with common.js, ristretto255, X25519 / curve25519, ed25519ph and ed25519ctx.

Has SUF-CMA (strong unforgeability under chosen message attacks)
and, unlike many other libraries, non-repudiation
(SBS / Strongly Binding Signatures aka exclusive ownership).
See `verify` docs for details.

Check out [Upgrading](#upgrading) section for v1 to v2 transition instructions
and [the online demo](https://paulmillr.com/noble/).

### This library belongs to _noble_ crypto

> **noble-crypto** — high-security, easily auditable set of contained cryptographic libraries and tools.

- No dependencies, protection against supply chain attacks
- Auditable TypeScript / JS code
- Supported in all major browsers and stable node.js versions
- All releases are signed with PGP keys
- Check out [homepage](https://paulmillr.com/noble/) & all libraries:
  [curves](https://github.com/paulmillr/noble-curves)
  (4kb versions [secp256k1](https://github.com/paulmillr/noble-secp256k1),
  [ed25519](https://github.com/paulmillr/noble-ed25519)),
  [hashes](https://github.com/paulmillr/noble-hashes)

## Usage

> npm install @noble/ed25519

We support all major platforms and runtimes. For node.js <= 18 and React Native, additional polyfills are needed: see below.

```js
import * as ed from '@noble/ed25519';
// import * as ed from "https://deno.land/x/ed25519/mod.ts"; // Deno
// import * as ed from "https://unpkg.com/@noble/ed25519"; // Unpkg
(async () => {
  // keys, messages & other inputs can be Uint8Arrays or hex strings
  // Uint8Array.from([0xde, 0xad, 0xbe, 0xef]) === 'deadbeef'
  const privKey = ed.utils.randomPrivateKey(); // Secure random private key
  const message = Uint8Array.from([0xab, 0xbc, 0xcd, 0xde]);
  const pubKey = await ed.getPublicKeyAsync(privKey); // Sync methods below
  const signature = await ed.signAsync(message, privKey);
  const isValid = await ed.verifyAsync(signature, message, pubKey);
})();
```

Additional steps needed for some environments:

```ts
// 1. Enable synchronous methods.
// Only async methods are available by default, to keep the library dependency-free.
import { sha512 } from '@noble/hashes/sha512';
ed.etc.sha512Sync = (...m) => sha512(ed.etc.concatBytes(...m));
// Sync methods can be used now:
// ed.getPublicKey(privKey); ed.sign(msg, privKey); ed.verify(signature, msg, pubKey);

// 2. node.js 18 and earlier,  needs globalThis.crypto polyfill
import { webcrypto } from 'node:crypto';
// @ts-ignore
if (!globalThis.crypto) globalThis.crypto = webcrypto;

// 3. React Native needs crypto.getRandomValues polyfill and sha512
import 'react-native-get-random-values';
import { sha512 } from '@noble/hashes/sha512';
ed.etc.sha512Sync = (...m) => sha512(ed.etc.concatBytes(...m));
ed.etc.sha512Async = (...m) => Promise.resolve(ed.etc.sha512Sync(...m));
```

## API

There are 3 main methods: `getPublicKey(privateKey)`, `sign(message, privateKey)`
and `verify(signature, message, publicKey)`. We accept Hex type everywhere:

```ts
type Hex = Uint8Array | string
```

### getPublicKey

```typescript
function getPublicKey(privateKey: Hex): Uint8Array;
function getPublicKeyAsync(privateKey: Hex): Promise<Uint8Array>;
```

Generates 32-byte public key from 32-byte private key.
- Some libraries have 64-byte private keys. Don't worry, those are just
  priv+pub concatenated. Slice it: `priv64b.slice(0, 32)`
- Use `Point.fromPrivateKey(privateKey)` if you want `Point` instance instead
- Use `Point.fromHex(publicKey)` if you want to convert hex / bytes into Point.
  It will use decompression algorithm 5.1.3 of RFC 8032.
- Use `utils.getExtendedPublicKey` if you need full SHA512 hash of seed

### sign

```ts
function sign(
  message: Hex, // message which would be signed
  privateKey: Hex // 32-byte private key
): Uint8Array;
function signAsync(message: Hex, privateKey: Hex): Promise<Uint8Array>;
```

Generates EdDSA signature. Always deterministic.

Assumes unhashed `message`: it would be hashed by ed25519 internally.
For prehashed ed25519ph, switch to noble-curves.

### verify

```ts
function verify(
  signature: Hex, // returned by the `sign` function
  message: Hex, // message that needs to be verified
  publicKey: Hex // public (not private) key,
  options = { zip215: true } // ZIP215 or RFC8032 verification type
): boolean;
function verifyAsync(signature: Hex, message: Hex, publicKey: Hex): Promise<boolean>;
```

Verifies EdDSA signature. Has SUF-CMA (strong unforgeability under chosen message attacks).
By default, follows ZIP215 [1] and can be used in consensus-critical apps [2].
`zip215: false` option switches verification criteria to strict
RFC8032 / FIPS 186-5 and provides non-repudiation with SBS (Strongly Binding Signatures) [3].

[1]: https://zips.z.cash/zip-0215
[2]: https://hdevalence.ca/blog/2020-10-04-its-25519am
[3]: https://eprint.iacr.org/2020/1244

### utils

A bunch of useful **utilities** are also exposed:

```typescript
const etc: {
  bytesToHex: (b: Bytes) => string;
  hexToBytes: (hex: string) => Bytes;
  concatBytes: (...arrs: Bytes[]) => Uint8Array;
  mod: (a: bigint, b?: bigint) => bigint;
  invert: (num: bigint, md?: bigint) => bigint;
  randomBytes: (len: number) => Bytes;
  sha512Async: (...messages: Bytes[]) => Promise<Bytes>;
  sha512Sync: Sha512FnSync;
};
const utils: {
  getExtendedPublicKeyAsync: (priv: Hex) => Promise<ExtK>;
  getExtendedPublicKey: (priv: Hex) => ExtK;
  precompute(p: Point, w?: number): Point;
  randomPrivateKey: () => Bytes; // Uses CSPRNG https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues
};

class ExtendedPoint { // Elliptic curve point in Extended (x, y, z, t) coordinates.
  constructor(ex: bigint, ey: bigint, ez: bigint, et: bigint);
  static readonly BASE: Point;
  static readonly ZERO: Point;
  static fromAffine(point: AffinePoint): ExtendedPoint;
  static fromHex(hash: string);
  get x(): bigint;
  get y(): bigint;
  // Note: It does not check whether the `other` point is valid point on curve.
  add(other: ExtendedPoint): ExtendedPoint;
  equals(other: ExtendedPoint): boolean;
  isTorsionFree(): boolean; // Multiplies the point by curve order
  multiply(scalar: bigint): ExtendedPoint;
  subtract(other: ExtendedPoint): ExtendedPoint;
  toAffine(): Point;
  toRawBytes(): Uint8Array;
  toHex(): string; // Compact representation of a Point
}
// Curve params
ed25519.CURVE.p // 2 ** 255 - 19
ed25519.CURVE.n // 2 ** 252 + 27742317777372353535851937790883648493
ed25519.ExtendedPoint.BASE // new ed25519.Point(Gx, Gy) where
// Gx=15112221349535400772501151409588531511454012693041857206046113283949847762202n
// Gy=46316835694926478169428394003475163141307993866256225615783033603165251855960n;
```

## Security

The module is production-ready.
It is cross-tested against [noble-curves](https://github.com/paulmillr/noble-curves),
and has similar security.

1. The current version is rewrite of v1, which has been audited by cure53:
[PDF](https://cure53.de/pentest-report_ed25519.pdf).
2. It's being fuzzed by [Guido Vranken's cryptofuzz](https://github.com/guidovranken/cryptofuzz):
run the fuzzer by yourself to check.

Our EC multiplication is hardened to be algorithmically constant time.
We're using built-in JS `BigInt`, which is potentially vulnerable to
[timing attacks](https://en.wikipedia.org/wiki/Timing_attack) as
[per MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt#cryptography).
But, _JIT-compiler_ and _Garbage Collector_ make "constant time" extremely hard
to achieve in a scripting language. Which means _any other JS library doesn't
use constant-time bigints_. Including bn.js or anything else.
Even statically typed Rust, a language without GC,
[makes it harder to achieve constant-time](https://www.chosenplaintext.ca/open-source/rust-timing-shield/security)
for some cases. If your goal is absolute security, don't use any JS lib —
including bindings to native ones. Use low-level libraries & languages.

We consider infrastructure attacks like rogue NPM modules very important;
that's why it's crucial to minimize the amount of 3rd-party dependencies & native
bindings. If your app uses 500 dependencies, any dep could get hacked and you'll
be downloading malware with every `npm install`. Our goal is to minimize this attack vector.

As for key generation, we're deferring to built-in
[crypto.getRandomValues](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues)
which is considered cryptographically secure (CSPRNG).

## Speed

Benchmarks done with Apple M2 on macOS 13 with Node.js 20.

    getPublicKey(utils.randomPrivateKey()) x 9,173 ops/sec @ 109μs/op
    sign x 4,567 ops/sec @ 218μs/op
    verify x 994 ops/sec @ 1ms/op
    Point.fromHex decompression x 16,164 ops/sec @ 61μs/op

Compare to alternative implementations:

    tweetnacl@1.0.3 getPublicKey x 1,808 ops/sec @ 552μs/op ± 1.64%
    tweetnacl@1.0.3 sign x 651 ops/sec @ 1ms/op
    ristretto255@0.1.2 getPublicKey x 640 ops/sec @ 1ms/op ± 1.59%
    sodium-native#sign x 83,654 ops/sec @ 11μs/op

## Contributing

1. Clone the repository
2. `npm install` to install build dependencies like TypeScript
3. `npm run build` to compile TypeScript code
4. `npm run test` to run tests

## Upgrading

noble-ed25519 v2 features improved security and smaller attack surface.
The goal of v2 is to provide minimum possible JS library which is safe and fast.

That means the library was reduced 4x, to just over 300 lines. In order to
achieve the goal, **some features were moved** to
[noble-curves](https://github.com/paulmillr/noble-curves), which is
even safer and faster drop-in replacement library with same API.
Switch to curves if you intend to keep using these features:

- x25519 / curve25519 / getSharedSecret
- ristretto255 / RistrettoPoint
- Using `utils.precompute()` for non-base point
- Support for environments which don't support bigint literals
- Common.js support
- Support for node.js 18 and older without [shim](#usage)

Other changes for upgrading from @noble/ed25519 1.7 to 2.0:

- Methods are now sync by default; use `getPublicKeyAsync`, `signAsync`, `verifyAsync` for async versions
- `bigint` is no longer allowed in `getPublicKey`, `sign`, `verify`. Reason: ed25519 is LE, can lead to bugs
- `Point` (2d xy) has been changed to `ExtendedPoint` (xyzt)
- `Signature` was removed: just use raw bytes or hex now
- `utils` were split into `utils` (same api as in noble-curves) and
  `etc` (`sha512Sync` and others)

## License

MIT (c) 2019 Paul Miller [(https://paulmillr.com)](https://paulmillr.com), see LICENSE file.
