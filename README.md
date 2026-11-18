# Wedgie BTC

A self-custodial Bitcoin spending wallet for the Wedgie pocket computer, built around the Raspberry Pi Pico (RP2040), Waveshare Pico-LCD-1.3 display, and Infineon OPTIGA Trust M secure element.

**Current status:** specification and architecture planning. No wallet implementation is complete.

## Project goals

- Start with Bitcoin Signet for development and interoperability testing.
- Enable Bitcoin mainnet shortly after the Signet acceptance criteria pass.
- Provide single-signature, native SegWit (BIP84) on-chain wallet functionality.
- Display receiving-address QR codes directly on the Wedgie LCD.
- Prepare transactions on a connected computer, but verify and sign them on the Wedgie.
- Use a shared BIP39 recovery phrase with a dedicated hardened HD account for Wedgie.
- Add Lightning send and receive through a configurable backend, with a public swap provider as the no-node-required default.
- Allow advanced users to select a custom swap service or connect their own Lightning infrastructure.

## Planned releases

| Release | Scope |
|---|---|
| v0.1 — Signet | Provisioning, BIP39/BIP32/BIP84, receive QR, watch-only host, PSBT signing, broadcast, restore tests |
| v0.2 — Mainnet | Mainnet support shortly after Signet acceptance tests pass; separate mainnet account and explicit network safeguards |
| v0.3 — Lightning | Lightning send/receive via submarine and reverse submarine swaps; default public provider; configurable alternatives |
| v0.4 — Advanced backends | Personal Lightning-node integration and self-hosted swap-service support |

Lightning backend interfaces should be considered in the design from the beginning, but Lightning payments are not part of the initial Signet implementation unless separately scoped.

## Hardware and security

- Target: original Raspberry Pi Pico RP2040, not Pico W.
- Secure element: Infineon OPTIGA Trust M SLS32AIA over I²C.
- Preserve existing Wedgie key slots and applications. Do not overwrite KEY1/KEY2/KEY3.
- KEY4 is only a candidate for hardware-assisted protection. Its metadata, usage permissions, lifecycle, and derivation design must be inspected and tested before provisioning or permanently locking it.
- OPTIGA Trust M does not natively support secp256k1. Bitcoin signing therefore runs in software on the RP2040.
- Treat this as a hot wallet, not a Ledger-equivalent secure signer. A compromised or replaced firmware can undermine transaction authorization.
- Never send the BIP39 phrase, master seed, or private key to the host or a Lightning provider.

See [SPEC.md](SPEC.md) for the detailed requirements, architecture, security assumptions, and acceptance criteria.

## Important design constraints

1. **Recoverability:** The master BIP39 phrase is the recovery root. The Wedgie retains only its dedicated account material, encrypted at rest if the proposed KEY4 construction is validated.
2. **HD account isolation:** Use hardened account boundaries so compromise of the Wedgie account does not expose sibling accounts, assuming the master seed and higher-level private keys remain uncompromised.
3. **Host is untrusted:** The host may construct and broadcast transactions, but the device independently parses and validates every input, output, fee, and change output before signing.
4. **QR receiving:** The Wedgie generates standards-compliant QR codes for its current on-chain receiving address. Old addresses remain monitored.
5. **Lightning is a separate payment rail:** A swap-based Lightning interface does not create a native Lightning balance or channel wallet on the Pico. Swap fees, on-chain fees, timeouts, claim/refund handling, and provider availability must be visible and handled safely.
6. **Mainnet follows Signet quickly, but not blindly:** Mainnet is enabled after the explicit Signet acceptance tests pass, not after a lengthy unrelated feature roadmap.

## Repository status

This repository begins with the specification. Implementation should proceed in small, testable milestones, starting with reference-vector tests and a host-side cryptographic prototype before deployment to the Pico.
