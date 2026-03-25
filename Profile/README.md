<img src="https://repository-images.githubusercontent.com/1182806847/6f5f3db0-f5c5-40ab-ac26-8c4feb1ef6ba">

# 📜 OpenClaiming Protocol (OCP)

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Protocol](https://img.shields.io/badge/protocol-OCP%20v1-green.svg)
![Status](https://img.shields.io/badge/status-draft-orange.svg)
![Spec](https://img.shields.io/badge/spec-open-red.svg)
![Chains](https://img.shields.io/badge/chains-8%20mainnets-purple.svg)

Welcome to the **OpenClaiming Protocol (OCP)** — an open standard for typed,
cryptographically signed authorization claims that anyone can verify, across
systems, organizations, and blockchains.

---

## ✨ What is an OpenClaim?

An **OpenClaim** is a signed JSON document that states what an agent, user,
or system is **authorized to do** — signed by whoever grants that authority,
verifiable by anyone.

```json
{
  "ocp": 1,
  "iss": "example.com/alice",
  "stm": { "role": "admin" },
  "key": ["data:key/es256;base64,MFkw..."],
  "sig": ["BASE64_SIGNATURE"]
}
```

The signatures prove **authorized parties made the statement**.
`key[]` and `sig[]` are parallel arrays, always sorted lexicographically —
deterministic regardless of signing order.

Learn more at **https://openclaiming.org**.

---

## 🚀 Design Goals

OpenClaiming was designed to be:

- **Simple** — one JSON object, one primitive
- **Human-readable** — no binary encoding required
- **Decentralized** — no central registry or identity provider
- **Implementation-friendly** — reference implementations in JS and PHP
- **Blockchain-agnostic** — EVM, Solana, Cosmos, TON, Bitcoin all supported
- **Identity-provider-agnostic** — works with any key type
- **Easy to publish anywhere** — `.well-known/` convention
- **Extensible** — standardized extensions for payments, actions, delegation

The protocol intentionally focuses on one primitive:

> A signed claim. Everything else is built on top of this.

---

## 🔐 Multisignature Model

OpenClaiming natively supports multisig across **different key formats** in
a single claim:

```json
{
  "key": [
    "data:key/eip712,0xabc...",
    "data:key/es256;base64,MFkw..."
  ],
  "sig": [
    "BASE64_65BYTES_EVM_SIG",
    "BASE64_64BYTES_P256_SIG"
  ]
}
```

- Each signature corresponds to its key at the same index
- Threshold is defined by the application or smart contract
- EVM (secp256k1) and P-256 (ES256) signers can coexist in one claim

---

## 🧩 Extensions

OpenClaiming supports **standardized extensions** for common use cases:

### 💸 Payment Extension (EVM)

Authorizes a token transfer — signed as EIP-712, enforced on-chain via
`ecrecover` + `transferFrom`. The contract executes the transfer forwarded
as the payer, with no proxy pre-approval required.

```json
{
  "ocp": 1,
  "payer":      "0xabc...",
  "token":      "0xusdc...",
  "recipients": ["0xdef..."],
  "max":        "1000000",
  "line":       42,
  "nbf":        1700000000,
  "exp":        1700003600,
  "chainId":    8453,
  "contract":   "0x9999...9999"
}
```

### ⚡ Authorization Extension

Grants typed, constrained authority from one principal to a subject:

```json
{
  "ocp": 1,
  "authority":   "0xabc...",
  "subject":     "0xdef...",
  "roles":       ["editor"],
  "actions":     ["publish", "delete"],
  "constraints": [{"key": "budget", "op": "lte", "value": "500"}],
  "nbf":         1700000000,
  "exp":         1700003600
}
```

➡️ [Extensions Overview](https://github.com/OpenClaiming/Documentation/blob/main/docs/extensions.md)

---

## ⛓ Chain Support

OpenClaiming is blockchain-agnostic. The claim schema is identical across
chains — only canonicalization and signature algorithm vary:

| Chain   | Canonicalization        | Algorithm  | On-chain verification |
|---------|------------------------|------------|-----------------------|
| EVM     | EIP-712 + keccak256    | secp256k1  | `ecrecover`           |
| Solana  | Borsh + SHA-512        | Ed25519    | `ed25519_program`     |
| Cosmos  | Amino + SHA-256        | secp256k1  | `verify_signature`    |
| TON     | TL-B                   | Ed25519    | `check_signature`     |
| Bitcoin | SHA-256d               | secp256k1  | script verify         |
| ES256   | RFC 8785 / JCS         | P-256      | off-chain / WASM      |

EVM smart contracts are **deployed on 8 mainnets**.

➡️ [EVM Integration](https://github.com/OpenClaiming/Documentation/blob/main/docs/evm.md)

---

## 🏗 Where OpenClaiming fits in the agentic stack

OCP is the **authorization payload layer** — it sits above key management
and below on-chain execution:

```
OWS (Open Wallet Standard)   — key management and signing
x402 / MPP                   — HTTP payment transport
OpenClaiming Protocol        — typed authorization payload  ← here
Smart contracts              — on-chain enforcement
```

**OWS** (MoonPay, PayPal, Ethereum Foundation, Coinbase, Solana Foundation
and 15+ others) defines how agents hold keys and produce signatures.
It deliberately says nothing about what is being signed.
**OpenClaiming fills that gap.**

An OWS wallet signing an OpenClaiming Payment claim needs zero changes
to either spec. OWS signs the EIP-712 digest. The signature slots into
`sig[]`. The contract verifies with `ecrecover`. No adapters, no wrappers.

---

## 🔒 Hardware-Grade Security with SafeBox

Standard software key management — including OWS's current in-process model
— cannot fully protect against a privileged attacker who can inspect process
memory during the signing window. OWS's own spec acknowledges this gap.

**SafeBox** provides the next level: attested AMI execution environments
built on AWS Nitro Enclaves.

- **Hardware attestation** of the exact binary running (byte-identical,
  reproducible builds)
- **Process memory physically inaccessible** to the host OS
- **M-of-N governance** for enclave policy changes
- OpenClaiming claims signed inside SafeBox are attestation-bound —
  the signature proves not just who signed, but *where* and *what binary*

This is the TEE-grade signing model that OWS's roadmap points toward.

➡️ **safebots.ai** for architecture details

---

## ⚙ Example Usage

```javascript
import OpenClaim from "openclaiming";

const claim = {
  ocp: 1,
  iss: "example.com/alice",
  stm: { role: "admin" }
};

const signed = await OpenClaim.sign(claim, secret);
const valid  = await OpenClaim.verify(signed);
console.log(valid); // true
```

**EVM Payment signing:**

```javascript
const signed = await OpenClaim.EVM.sign({
  ocp:        1,
  payer:      "0xabc...",
  token:      "0xusdc...",
  recipients: ["0xdef..."],
  max:        "1000000",
  line:       42,
  exp:        Math.floor(Date.now() / 1000) + 3600,
  chainId:    8453,
  contract:   "0x9999...9999"
}, secret);
```

---

## 🌐 Publishing Claims

Claims are published at a well-known path:

```
.well-known/openclaiming/<domain>/<identity>.json
```

Example:

```
example.com/.well-known/openclaiming/example.com/alice.json
```

---

## 📚 Documentation

### Core Protocol

- 📘 [Introduction](https://github.com/OpenClaiming/Documentation/blob/main/docs/introduction.md)
- 🧠 [Concepts](https://github.com/OpenClaiming/Documentation/blob/main/docs/concepts.md)
- 📄 [OpenClaim Format](https://github.com/OpenClaiming/Documentation/blob/main/docs/openclaim_format.md)
- 🔐 [Signatures](https://github.com/OpenClaiming/Documentation/blob/main/docs/signatures.md)
- 🧾 [Canonicalization](https://github.com/OpenClaiming/Documentation/blob/main/docs/canonicalization.md)
- 🌐 [Publishing Claims](https://github.com/OpenClaiming/Documentation/blob/main/docs/publishing_claims.md)

### Extensions

- 🧩 [Extensions Overview](https://github.com/OpenClaiming/Documentation/blob/main/docs/extensions.md)
- 💸 [Payments](https://github.com/OpenClaiming/Documentation/blob/main/docs/payments.md)
- ⚡ [Actions / Invocations](https://github.com/OpenClaiming/Documentation/blob/main/docs/actions.md)

### Identity & Authorization

- 🔗 [Identity Linking](https://github.com/OpenClaiming/Documentation/blob/main/docs/identity_linking.md)
- 📱 [Device & Session Keys](https://github.com/OpenClaiming/Documentation/blob/main/docs/device_keys.md)
- 🛡 [Capability Claims](https://github.com/OpenClaiming/Documentation/blob/main/docs/capabilities.md)

### Distributed Systems

- ☁️ [Intercloud Claims](https://github.com/OpenClaiming/Documentation/blob/main/docs/intercloud.md)
- ⛓ [Blockchain Anchoring](https://github.com/OpenClaiming/Documentation/blob/main/docs/blockchain.md)

### Implementation

- ⚙ [Implementation Guide](https://github.com/OpenClaiming/Documentation/blob/main/docs/implementation.md)
- 🔒 [Security](https://github.com/OpenClaiming/Documentation/blob/main/docs/security.md)
- 📊 [Comparisons](https://github.com/OpenClaiming/Documentation/blob/main/docs/comparisons.md)
- ❓ [FAQ](https://github.com/OpenClaiming/Documentation/blob/main/docs/faq.md)

---

## 🧪 Reference Implementations

| Language | Signing | Verification | EVM Extension |
|----------|---------|-------------|---------------|
| JavaScript (browser + Node) | `Q.Crypto.OpenClaim.sign` | `Q.Crypto.OpenClaim.verify` | `Q.Crypto.OpenClaim.EVM` |
| PHP | `Q_Crypto_OpenClaim::sign` | `Q_Crypto_OpenClaim::verify` | `Q_Crypto_OpenClaim_EVM` |

- **JS crypto:** noble p256, SubtleCrypto, eip712.js — no external dependencies
- **PHP crypto:** mdanter/phpecc — no OpenSSL dependency
- **Canonicalization:** RFC 8785 / JCS via `json-canonicalize`

Official test vectors: **https://github.com/OpenClaiming/test-vectors**

---

## 🧩 Ecosystem Repositories

| Repository | Purpose |
|-----------|---------|
| [OpenClaiming/spec](https://github.com/OpenClaiming/spec) | Normative protocol specification |
| [OpenClaiming/Documentation](https://github.com/OpenClaiming/Documentation) | Human-readable documentation |
| [OpenClaiming/test-vectors](https://github.com/OpenClaiming/test-vectors) | Interoperability test suite |
| [OpenClaiming/examples](https://github.com/OpenClaiming/examples) | Example claims |
| [OpenClaiming/website](https://github.com/OpenClaiming/Website) | openclaiming.org source |
| [openclaiming.org](https://openclaiming.org) | Official website |

---

## 📜 License

MIT License
