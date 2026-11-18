# Wedgie BTC — Technical Specification

**Version:** 0.1.0-draft  
**Status:** Architecture and implementation specification  
**Initial network:** Bitcoin Signet  
**Hardware:** Original Raspberry Pi Pico (RP2040) + Infineon OPTIGA Trust M SLS32AIA  
**Application:** `wedgie-btc`

---

## 1. Product definition

Wedgie BTC is a small-balance, self-custodial Bitcoin hot wallet for the Wedgie pocket computer. It provides on-chain receiving and spending first, followed by Lightning interoperability through a configurable backend.

The computer or phone-side interface is responsible for data entry, transaction construction, blockchain queries, and broadcasting. The Wedgie is responsible for holding the device-specific signing account, independently validating transaction data, displaying what will happen, and signing only after physical confirmation.

The host is untrusted. A malicious host must not be able to substitute a destination, amount, fee, input, or change output without the Wedgie detecting and displaying the actual transaction it is about to authorize.

This is an experimental hot wallet, not an audited hardware wallet and not a replacement for cold storage.

## 2. Goals and non-goals

### Goals

- Begin on Signet with realistic transaction and signature interoperability.
- Enable mainnet shortly after Signet acceptance tests pass.
- Generate and display an optional-to-shared, standard 24-word BIP39 recovery phrase during first provisioning; never persist or transmit the phrase.
- Derive a dedicated hardened HD account for Wedgie.
- Use BIP84 native SegWit P2WPKH addresses.
- Rotate receiving addresses and use independently derived change addresses.
- Display receiving QR codes on the Wedgie LCD.
- Prepare and broadcast transactions through a PC-connected host.
- Parse, validate, display, and sign PSBTs on the Wedgie.
- Add Lightning send and receive through a selectable service backend. Default path should not require each user to operate a Lightning node.
- Preserve the possibility of user-selected public providers, custom swap services, or personal Lightning-node infrastructure.

### Non-goals for initial on-chain releases

- Full Bitcoin node on the RP2040.
- Native Lightning node or channel management on the Pico.
- Large-value cold storage.
- Multi-signature in the first release.
- iPhone direct USB signing support in the first release.
- Any claim of Ledger-equivalent secure boot or tamper resistance.

## 3. Hardware and firmware environment

| Component | Target |
|---|---|
| MCU | Raspberry Pi Pico RP2040 |
| CPU | Dual-core ARM Cortex-M0+ |
| Clock | 133 MHz nominal |
| SRAM | 264 KB |
| Display | Waveshare Pico-LCD-1.3 |
| Input | Existing Wedgie buttons |
| Secure element | Infineon OPTIGA Trust M SLS32AIA |
| Secure-element bus | I²C, GP4 SDA / GP5 SCL, address 0x30 |
| Host link | USB serial / existing Wedgie protocol |
| Wireless | None on original Pico |

Memory is constrained. Large transaction payloads and cryptographic operations must use bounded buffers and avoid heap fragmentation.

## 4. Secure-element allocation and constraints

Preserve existing objects:

| OPTIGA object | OID | Existing role | Rule |
|---|---:|---|---|
| FACTORY_KEY | E0F0 | Infineon factory identity | Never modify |
| KEY2 | E0F1 | Wedgie Safe Signer | Preserve |
| KEY3 | E0F2 | Wedgie Secret | Preserve |
| KEY4 | E0F3 | Candidate for Wedgie BTC | Inspect before use |

The exact state of KEY2–KEY4 on each device must be read from metadata. Never infer that a slot is empty based on application state alone.

Before any provisioning:
- Read and record metadata for KEY4.
- Confirm algorithm and usage bits.
- Confirm lifecycle and change/use rules.
- Confirm the exact chip revision (the documented Adafruit Trust M hardware has been tested as V1).
- Test the proposed construction on a disposable test chip or non-production object.
- Do not call lifecycle or metadata writes that make an object operational until all tests and recovery checks pass.
- Never use `genkey` on an occupied slot: it replaces the existing key irreversibly.

### Cryptographic limitation

The OPTIGA Trust M V1 supports P-256 and P-384 among its ECC curves, but does not support Bitcoin's secp256k1. Its private key cannot be exported. It therefore cannot directly hold and sign with a standard Bitcoin secp256k1 private key.

The RP2040 must perform secp256k1 operations in software.

KEY4-based ECDH/key derivation may be investigated as an at-rest protection mechanism, but no design should be considered approved until the derivation is deterministic across power cycles, uses an approved KDF and authenticated-encryption scheme, and is recoverable from the BIP39 backup when the original chip is lost.

A seed-encryption design must not make the device's only usable copy dependent on an unrecoverable hardware secret. The BIP39 phrase must restore the account on replacement hardware.

## 5. Recovery phrase and hierarchical wallet

### 5.1 BIP39 provisioning

- 256-bit entropy from the OPTIGA hardware random generator.
- 24 English BIP39 words with standard checksum.
- Display words only during initial provisioning.
- Require deliberate user acknowledgement and backup verification before proceeding.
- Do not log, save, transmit, or include the phrase in diagnostics.
- Do not offer a later "show seed" function.
- Do not claim that one-time display makes the phrase unrecoverable: the user's offline backup is deliberately capable of restoring the wallet.
- Do not silently create a new wallet if stored account data is missing or corrupt.

An optional BIP39 passphrase is deferred from v0.1 unless its backup UX and recovery semantics are fully specified.

### 5.2 Derivation paths

Use BIP32 hierarchical deterministic derivation and BIP84.

| Use | Derivation path |
|---|---|
| Signet account | `m/84'/1'/0'` |
| Signet external/receiving branch | `m/84'/1'/0'/0/i` |
| Signet internal/change branch | `m/84'/1'/0'/1/i` |
| Mainnet account | `m/84'/0'/0'` |
| Mainnet external/receiving branch | `m/84'/0'/0'/0/i` |
| Mainnet internal/change branch | `m/84'/0'/0'/1/i` |

The Wedgie account is isolated at a hardened account boundary. Other apps or wallets may use different hardened accounts from the same master recovery phrase.

Do not store the master seed on the Pico. Store only the dedicated account material needed by the wallet, encrypted at rest if the reviewed protection design is implemented.

An account-level xprv can derive all receiving and change keys in that account. Its compromise therefore compromises the entire Wedgie account, not merely one address. Do not export it to the host.

### 5.3 Address rotation and discovery

- Issue a fresh receiving address for each new payment request.
- Track all previously issued addresses and continue monitoring them.
- Do not assume an address is unused because its current balance is zero.
- Derive change addresses exclusively from the internal branch.
- The host may hold account public derivation material for watch-only discovery.
- Implement gap-limit scanning and recovery of address indices after reinstall or restore.
- Clearly distinguish Signet and mainnet addresses.

## 6. Wallet/network separation

### v0.1 — Signet

Use Signet for development and end-to-end testing. Testnet coins have no monetary value. Implement the same transaction and signature standards intended for mainnet.

### v0.2 — Mainnet

Enable mainnet shortly after Signet acceptance tests pass. Do not wait for Lightning implementation.

Mainnet requirements:
- Separate mainnet BIP84 account/path using coin type 0.
- Strong network identity in UI and protocol.
- Reject Signet addresses on mainnet and mainnet addresses on Signet.
- Never broadcast a Signet transaction to mainnet or vice versa.
- Require a deliberate first-time mainnet enablement acknowledgement.
- Start with very small-value transactions after test acceptance.
- Keep Signet support available for ongoing regression tests.

## 7. On-chain receiving and QR codes

The Wedgie must generate and display a standards-compliant QR code for its current receiving address directly on the LCD.

- QR encodes the correct Bitcoin URI/address for the active network.
- Display the full address in readable pages as an alternative verification method.
- Host interface may also display the QR code at a larger size.
- Test scanning from different phone models, distances, and brightness settings.
- Select QR error correction and module size appropriate for the 240×240 display.
- Never use illustrative or placeholder QR patterns in production.

The host and device must agree on the address and derivation index. The device remains the authority.

Lightning invoices may be much longer than on-chain addresses. The small display's QR capacity must be benchmarked; host-side large QR presentation or another encoding may be required for long invoices.

## 8. On-chain sending

### User flow

1. Connect Wedgie to a compatible computer.
2. Open the host-side Wedgie BTC interface.
3. Paste destination address and enter amount.
4. Select fee policy/rate.
5. Host queries blockchain backend and constructs a PSBT.
6. Host transfers PSBT over USB in bounded chunks.
7. Wedgie parses PSBT and independently verifies every spend.
8. Wedgie displays destination, amount, fee, and change details.
9. User physically confirms on the device.
10. RP2040 signs the supported inputs.
11. Wedgie returns signed PSBT.
12. Host finalizes and broadcasts.
13. Wallet monitors confirmation status.

### Mandatory PSBT verification

- Validate PSBT structure and all required maps.
- Verify ownership and derivation paths for every input.
- Verify input amounts and scriptPubKeys using trusted wallet data.
- Parse all outputs and display every non-change destination.
- Independently identify and verify change outputs.
- Calculate fee from total inputs minus total outputs.
- Reject excessive or negative fees.
- Reject unsupported scripts, sighash modes, transaction versions, and unknown critical fields.
- Refuse any transaction whose full meaning cannot be shown.
- Never trust host-supplied labels, token names, balances, or fee summaries.
- Display full destination addresses across pages and verify the user sees all pages before approval.

PSBT v0 is sufficient for the first implementation. PSBT v2 can be added when needed.

## 9. Cryptographic implementation

Use a reviewed, established secp256k1 implementation in native C, subject to licensing, memory, and build review. Do not implement curve arithmetic from scratch.

Required functions:
- BIP39 PBKDF2-HMAC-SHA512.
- BIP32 private/public derivation.
- SHA-256, double-SHA-256, HASH160.
- HMAC-SHA512.
- secp256k1 public-key operations.
- Deterministic ECDSA signing and low-S normalization.
- BIP143 SegWit v0 signature hashes.
- Bech32 address encode/decode.
- PSBT parsing, validation, and signing.
- Secure random-number interface.
- Best-effort explicit zeroization of mutable native buffers.

The implementation must be benchmarked on real RP2040 hardware alongside display, USB, and Wedgie firmware memory usage.

Python immutable objects and integer values cannot be reliably wiped. Avoid holding long-lived sensitive values in MicroPython. Native code should own temporary secret buffers and minimize copies.

## 10. Host-side interface and USB protocol

The host manages:
- Blockchain queries and transaction broadcast.
- Watch-only account data.
- Address discovery and UTXO selection.
- Fee estimation.
- PSBT construction/finalization.
- Lightning backend API communications.
- Transaction history and status.

The host never receives the recovery phrase, master seed, account xprv, or private keys.

Proposed USB messages (adapt to existing Wedgie protocol):
- `btc_info`
- `btc_account` (public/watch-only data only)
- `btc_address`
- `btc_psbt_begin`
- `btc_psbt_chunk`
- `btc_psbt_sign`
- `btc_psbt_result`
- `btc_cancel`

Use bounded chunk sizes and strict maximum transaction complexity. Reject truncated, duplicated, reordered, or oversized messages.

Original Pico has no Wi-Fi or Bluetooth. Desktop USB is the first signing interface. iPhone direct connection is out of scope for v0.1; a future mobile watch-only interface may use exported public account data.

## 11. Lightning architecture

Lightning is a separate payment rail and must be integrated through a clear backend abstraction.

### User experience

The wallet exposes:
- Receive: Bitcoin on-chain or Lightning.
- Send: Bitcoin on-chain or Lightning.
- Separate fee estimates and clear payment status.
- Separate on-chain and Lightning state where applicable.
- No required account registration for the default non-custodial swap path, subject to provider requirements.

### Default: public swap provider

Use a supported public swap service so new users do not need to run a Lightning node.

Investigate Boltz as an initial candidate, but do not commit until current API compatibility, supported networks, fees, limits, provider policy, and refund/claim semantics have been verified.

### Swap operations

- **Submarine swap:** on-chain BTC is exchanged for a Lightning payment; used to pay Lightning invoices from on-chain funds.
- **Reverse submarine swap:** Lightning payment is exchanged for on-chain BTC; used to receive Lightning payments that settle to the Wedgie on-chain wallet.

The first design is swap-based Lightning interoperability, not a native Lightning balance held by the Pico.

Required protections:
- Non-custodial atomic-swap protocol where supported.
- Correct locktime/CLTV and script validation.
- Claim/refund keys remain under user control.
- Refund transactions and claim transactions must be constructed and persisted safely.
- Persist swap state and recovery material across power loss.
- Monitor swap status independently; do not rely exclusively on host claims.
- Do not expose the wallet xprv or seed to provider.
- Display provider fee, miner fee, limits, expiry, and timeout/refund conditions before approval.
- Reject expired invoices and unsupported invoices.
- Do not mark a payment complete until verified settlement.
- Test timeout, provider outage, host disconnection, reboot, and refund paths on Signet.

### Configurable backends

1. Public swap provider — default, no user-run node.
2. Custom self-hosted swap service — advanced user option.
3. Personal Lightning node — optional separate integration, using scoped authorization.

A user's own Lightning node is not automatically a swap server. Node integration and swap-service integration must be separate adapters.

Do not put unrestricted Lightning node administrator credentials on the Pico or expose them to untrusted browser code.

### Custody and trust statement

A public swap provider is still an external service dependency. Atomic swaps reduce custody risk but do not eliminate implementation, availability, liquidity, fee, or protocol risks. Be explicit about these limitations in the UI and documentation.

A swap-based wallet does not promise instant, low-cost Lightning receipt in every case: reverse swaps require on-chain settlement and may incur mining fees and confirmation delays.

## 12. UI screens

### Home
- Network badge: SIGNET or MAINNET.
- On-chain confirmed balance.
- Unconfirmed balance.
- Receive.
- Send.
- Transaction history.
- Lightning settings/status.

### Receive
- Choose On-chain or Lightning.
- On-chain: fresh BIP84 address, QR code, full address, index.
- Lightning: request invoice from selected backend, display invoice/QR, amount and expiry; show swap provider/fee implications if reverse-swap based.
- Previous receiving addresses remain tracked.

### Send
- Choose On-chain or Lightning.
- On-chain: show destination, amount, fee, change, PSBT details.
- Lightning: decode invoice, show payee/requested amount/description/expiry, quote service fees and swap limits, then request confirmation.
- User can reject or cancel.
- Never sign/pay if the device cannot display the relevant payment details.

### Settings
- Active network.
- Lightning backend/provider.
- Custom backend endpoint.
- Fee policy.
- Wallet public information and recovery guidance.
- Do not display seed after provisioning.

## 13. Testing and acceptance

### Cryptographic vectors
- Official BIP39 vectors.
- BIP32 vectors.
- BIP84 vectors.
- Known secp256k1 signing/verification vectors.
- Bech32 vectors.
- PSBT interoperability fixtures.

### Signet end-to-end
- Provision and restore wallet on separate test instance.
- Receive to multiple fresh addresses.
- Confirm address QR scans correctly.
- Construct PSBT on host.
- Sign on Pico.
- Verify signature independently.
- Broadcast and confirm transaction.
- Verify change output and UTXO state.
- Test multiple inputs and outputs.
- Test interrupted USB, power loss, invalid PSBT, invalid fee, malicious output substitution, and memory exhaustion.

### Recovery
- Restore from the original 24-word phrase on a clean compatible wallet.
- Verify all previously used receive/change addresses and balances are discoverable.
- Test recovery if Pico flash is erased.
- Test recovery if OPTIGA chip is lost/replaced.
- Do not require the original KEY4 to restore.

### Lightning (later release)
- Successful submarine payment.
- Successful reverse submarine receipt.
- Invoice expiry.
- Wrong network.
- Insufficient liquidity.
- Excessive fees.
- Provider unavailable.
- Host disconnected mid-swap.
- Device reboot mid-swap.
- Claim success.
- Refund success after timeout.
- Independent verification of settlement.

### Mainnet gate
Enable mainnet shortly after Signet acceptance tests pass, once:
- All critical Signet tests pass.
- Recovery has been independently demonstrated.
- Transaction display/PSBT verification is reviewed.
- Fee calculation is independently checked.
- No known critical or high-severity signing/recovery defect remains.
- Mainnet and Signet are separated by explicit network safeguards.

Do not wait for Lightning to complete before releasing mainnet on-chain support.

## 14. Development milestones

### Phase A — Host-side crypto prototype
- Select and pin crypto library.
- Validate BIP39/BIP32/BIP84 vectors.
- Generate deterministic Signet addresses.
- Create and verify signatures.
- Measure memory and performance.

### Phase B — RP2040 integration
- Native C module.
- USB integration.
- Memory and fragmentation tests.
- LCD QR implementation.
- Real-hardware benchmarks.

### Phase C — Provisioning and recovery
- One-time 24-word display and backup confirmation.
- Dedicated hardened account.
- KEY4 investigation and optional protection implementation.
- Restore onto replacement device.

### Phase D — Signet wallet
- Watch-only host.
- Address and UTXO discovery.
- PSBT construction and parsing.
- Independent verification and signing.
- Broadcast and confirmation tracking.

### Phase E — Mainnet
- Enable shortly after Signet gate passes.
- Separate network account.
- Small-value operational validation.
- Publish clear hot-wallet risk warning.

### Phase F — Lightning
- Backend abstraction.
- Integrate selected public swap provider.
- Configurable provider/self-hosted endpoint.
- Claim/refund persistence and safety.
- Signet swap testing before mainnet Lightning.

## 15. Release acceptance criteria

### v0.1 Signet
A user can:
1. Provision a recoverable wallet.
2. Record and verify a 24-word backup.
3. Receive Signet BTC to multiple fresh addresses.
4. Scan a valid on-device QR code.
5. See accurate balances and transaction state.
6. Construct a PSBT on a PC.
7. Verify and sign it on Wedgie after physical approval.
8. Broadcast and confirm the transaction.
9. Restore the same account on another device from the BIP39 backup.

### v0.2 Mainnet
All v0.1 functionality works with mainnet derivation, address and transaction protections. Mainnet is enabled shortly after Signet acceptance tests pass.

### v0.3 Lightning
Users without their own node can send and receive Lightning through the default public swap provider, with explicit fees, limits, timeout and refund information. Advanced users can configure compatible backends.

---

## 16. Security posture

Wedgie BTC is a self-custodial hot wallet for limited spending balances. The Pico is physically reflashable and does not have a hardware-enforced secure-boot root of trust equivalent to a dedicated hardware wallet. The OPTIGA protects its own non-exportable keys, but it cannot natively sign secp256k1 transactions. Software signing material exists on the RP2040 during use.

Do not market the device as tamper-proof, immune to malicious firmware, or equivalent to Ledger. Recommend users keep only a small spending balance on it and maintain an independently verified recovery backup.

**Release rule:** No mainnet funds until recovery, PSBT verification, signing, address/change handling, and network separation have passed the acceptance tests.
