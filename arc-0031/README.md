---
arc: 31
title: Exchange Addresses
authors: Chris Beck <chris@aleolabs.io>, Stark Ridiesel <stark@aleolabs.io>, Alex Kim <alex@aleolabs.io>, John Reynolds <john@aleolabs.io>
discussion: https://github.com/aleo-labs-protocol/ARCs/discussions/145
topic: Application
status: Draft
created: 2026-10-19
---

## Abstract

This ARC defines an exchange address: a v1 address with the letter `x` inserted after `aleo`, giving a 64-character string that starts with `aleox1`. The marker tells a paying wallet that the recipient is an exchange and accepts public transfers only.

An exchange address contains the v1 address unchanged, including its checksum. You produce one by inserting a character, and you read one by removing that character. The marker is advisory: the paying wallet reads it and removes it before the address reaches the virtual machine. This ARC requires no change to the protocol, snarkOS, snarkVM consensus, or Leo, and it can't fork the network. v1 addresses are unaffected.

## Motivation

An Aleo address carries a public key and nothing else. It can't say how its owner wants to be paid.

That matters most for exchanges. An exchange credits a deposit by reading a mapping, so a public transfer is straightforward to attribute to a customer. A private transfer arrives as a record that the exchange must find by view-key scanning and can't readily attribute. Many exchanges therefore accept public transfers only, and they have no way to say so in the one string they hand out.

This proposal addresses several key objectives:

- **Attributable Deposits** - An exchange can state, in the address it already publishes, that it accepts public transfers only, so conforming wallets send deposits the exchange can credit.
- **No Protocol Change** - The marker is read and removed by wallets. No consensus rule, circuit, transition, record format, or serialization changes, so there is no fork risk.
- **Fail-Closed Adoption** - Every deployed snarkVM decoder already rejects an exchange address, so a wallet that doesn't understand the marker can't silently ignore it.
- **Human Verifiability** - The exchange form and the v1 form of an account differ by exactly one character, so a person can see that both name the same account.

## Specification

### Terminology

This ARC uses the requirement keywords MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY as defined in RFC 2119.

- **Payer** - The party who sends funds.
- **Recipient** - The party who publishes the address and receives funds.
- **v1 address** - The 63-character `aleo1` address format that Aleo already uses.
- **Exchange address** - The 64-character `aleox1` format that this ARC defines.
- **Marker** - The letter `x` that distinguishes an exchange address from a v1 address.

### Address Format

A v1 address is bech32m with the human-readable part `aleo` and a 52-character payload. The payload encodes a point on the Edwards BLS12 curve, serialized as its 32-byte little-endian affine x-coordinate.

An exchange address is a v1 address with the marker `x` inserted between `aleo` and the `1` separator. Everything after the separator, including the 6-character checksum, is copied from the v1 address without change:

```
aleox1ml2xr6fawppd6uaf8gn95uy2fpqqg8gk74k0lu8na7uvayk64v8qu8hw5u
^^^^^^                                                    ^^^^^^
aleo+x+1                                    52-char payload   v1 checksum
```

The following grammar uses ABNF, including the case-sensitive `%s` literal from RFC 7405:

```abnf
address           = v1-address / exchange-address
v1-address        = %s"aleo1"  58bech32-char
exchange-address  = %s"aleox1" 58bech32-char

; The bech32 character set: digits and lowercase letters, excluding
; "1", "b", "i", and "o".
bech32-char       = %x30 / %x32-39 / %x61 / %x63-68 / %x6A-6E / %x70-7A
```

The checksum in an exchange address is the v1 checksum, which bech32m computes over the human-readable part `aleo`. An implementation MUST NOT recompute the checksum over `aleox`. A string checksummed that way is invalid under this ARC.

Keeping the checksum is deliberate. People compare addresses by their leading and trailing characters, so `aleox1ml2xr6f…u8hw5u` and `aleo1ml2xr6f…u8hw5u` are recognizably the same account. A recomputed checksum would replace all 6 trailing characters, and the two forms of one account would look like two accounts. It would also make every implementation depend on a bech32 encoder rather than a string insertion. The cost is that the checksum doesn't cover the marker (see Security & Compliance).

Both forms are lowercase only. BIP-173 and BIP-350 also permit an all-uppercase string, which encodes more compactly in a QR code's alphanumeric mode. This ARC doesn't, because snarkVM already rejects an uppercase v1 address. QR codes must therefore use byte mode.

This ARC defines one marker. A later ARC can define another marker by the same mechanism.

### Encoding

1. Encode the account's v1 address as usual.
2. Insert the character `x` at index 4, between `aleo` and `1`.
3. Don't modify any other character.

### Decoding

- For a 63-character string, decode it as a v1 address. The recipient has stated no preference.
- For a 64-character string that starts with `aleox1`, remove the `x` at index 4 and decode the remaining 63 characters as a v1 address. The recipient is an exchange that accepts public transfers only.

A decoded exchange address and its v1 form denote the same account.

### Validation

A conforming wallet MUST reject an address when any of the following is true:

1. The length is neither 63 nor 64 characters.
2. A 63-character string doesn't start with `aleo1`.
3. A 64-character string doesn't start with `aleox1`.
4. The string contains an uppercase character.
5. The string contains `_`.
6. bech32m validation of the 63-character v1 form fails.
7. The recovered point doesn't lie in the prime-order subgroup.

Rule 5 departs from snarkVM's program parser, which accepts `_` separators inside an address literal. If a wallet accepted them, two different strings would denote the same payee.

A wallet MUST report a rejection to the payer.

### Paying an Exchange Address

After a wallet decodes an exchange address, it MUST either send the funds through a function that it knows lands them in the recipient's public balance, or refuse. It MUST NOT remove the marker to proceed.

For `credits.aleo`, the payer's own balance determines the function:

| Payer Holds     | Function                     |
|-----------------|------------------------------|
| Public balance  | `transfer_public`            |
| Private records | `transfer_private_to_public` |

`transfer_private` and `transfer_public_to_private` produce a record for the recipient, so a wallet MUST NOT use them. Source visibility is the payer's own choice, and the marker doesn't constrain it.

A wallet can't tell from an arbitrary function's signature who ends up owning a record. For a token program or any other program, a wallet MUST refuse unless it knows the function's behavior.

BIP-21 takes the same position with its `req-` convention: a payment that ignores a required preference isn't the payment the recipient asked for. A wallet that can't send a public transfer SHOULD say so rather than report a generic failure.

### Test Cases

In the following tables, `ADDR` is the valid v1 address `aleo1ml2xr6fawppd6uaf8gn95uy2fpqqg8gk74k0lu8na7uvayk64v8qu8hw5u`. Every vector belongs to that same account.

Accepted:

| Input                                                              | Meaning                          |
|--------------------------------------------------------------------|----------------------------------|
| `ADDR`                                                             | No stated preference.            |
| `aleox1ml2xr6fawppd6uaf8gn95uy2fpqqg8gk74k0lu8na7uvayk64v8qu8hw5u` | Exchange. Public transfers only. |

Removing the marker from the second produces the first, character for character.

Rejected:

| Input                                                              | Reason                                                     |
|--------------------------------------------------------------------|------------------------------------------------------------|
| `aleox1ml2xr6fawppd6uaf8gn95uy2fpqqg8gk74k0lu8na7uvayk64v8q7y7vj3` | The checksum was recomputed over `aleox`.                  |
| `aleox1ml2xr6fawppd6uaf8gn95uy2fpqqg8gk74k0lu8na7uvayk64v8qu8hw5q` | The checksum fails.                                        |
| `aleox1ml2xr6fawppd6uaf8gn95uy2fpqqg8gk74k0lu8na7uvayk64v8qu8hw5`  | The address is 63 characters and doesn't start with `aleo1`. |
| `ALEOX1ML2XR6FAWPPD6UAF8GN95UY2FPQQG8GK74K0LU8NA7UVAYK64V8QU8HW5U` | The address is uppercase.                                  |
| `aleo1ml2xr6fawppd6uaf8gn95uy2fpqqg8gk74k0lu8na7uvayk64v8qu8hw_5`  | The address contains `_`.                                  |

The first rejected vector is the one implementers get wrong. It's a well-formed bech32m string with the human-readable part `aleox`, and this ARC rejects it.

## FAQs

The following are commonly asked questions that are intended to provide clarity on the design choices in this ARC.

### Why not recompute the checksum over an `aleox` prefix?

Treating `aleox` as a genuine human-readable part would make the checksum cover the marker, so deleting the `x` would produce an invalid string. This ARC keeps the v1 checksum instead, so the two forms of an account stay visibly identical and implementations need only a string insertion (see Address Format).

### Why a single marker instead of a general-purpose flag field?

A character after the separator could encode several preference bits. That costs a larger specification, validity rules for combinations, and agreement on preferences nobody has asked for. A later ARC can add markers if the need arises.

### Why not enforce the preference on-chain?

The virtual machine could reject a private transfer to an exchange account by reading a recipient-controlled mapping, which a payer couldn't defeat by stripping the marker. `transfer_private` has no finalize block, though, so the recipient exists only inside the circuit. Rejecting it would need an in-circuit membership proof against registry state. A later ARC can revisit enforcement without changing this encoding.

## Reference Implementations

An exploratory snarkVM branch validates the encoding: https://github.com/cbeck88/snarkVM/tree/explore/colored-addresses

It adds `AddressMarker` and `Address::{to_marked_string, from_marked_str}` in `console/types/address`, and its tests use the vectors in Test Cases verbatim. It leaves `FromStr`, the program parser, `Display`, and serde unchanged, and tests that they still reject an exchange address.

The branch surfaced two constraints:

- **Existing parsers must not accept the marker** - Serde's human-readable path calls `FromStr`, so a `FromStr` that accepted `aleox1` would make node and REST JSON accept it too. Decoding a marked address needs a separate entry point.
- **snarkVM can't enforce the pay rule** - The rule depends on what each function does, so it belongs in wallet code.

Wallet integration, which selects the transfer function and applies the send-public-or-refuse rule, isn't implemented yet.

## Dependencies

The address encoding lives in snarkVM (`console/types/address`), and the transfer functions a payer selects between are in `credits.aleo`. This ARC affects wallets, SDKs, block explorers, and any application that displays or accepts an address.

### Backwards Compatibility

v1 addresses don't change. No consensus rule, circuit, transition, record format, or serialization changes, so there's no fork risk and no network upgrade is required.

Every deployed snarkVM decoder rejects an exchange address. `Address::from_str` rejects it on length:

```
Invalid account address length: found 64, expected 63
```

The program parser, which handles program literals and CLI inputs, rejects it because the string doesn't start with `aleo1`. Tooling that matches addresses with a pattern such as `aleo1[a-z0-9]{58}` doesn't match one either.

This fail-closed behavior is intended. A wallet that accepted the address and ignored the marker would send the exchange a private transfer that it asked not to receive, and report no error.

The cost is that adoption depends on wallets. An exchange that publishes an exchange address before its depositors' wallets support it gets deposits refused. Exchanges SHOULD NOT publish exchange addresses until the wallets they expect to be paid from support them.

## Security & Compliance

### The Marker Isn't Enforced

A payer that doesn't conform can disregard the marker, and the exchange still receives the transfer. An exchange MUST NOT rely on the marker for correctness, accounting, or compliance, and MUST still handle an incoming private receipt.

The chain stores a record's owner as an `Address<N>`, which is a curve point. The marker never reaches the virtual machine, so no circuit, transition, or validator observes it.

### The Checksum Doesn't Cover the Marker

Deleting the `x` produces a valid v1 address for the same account, and nothing in the encoding detects that. Any system that trims, extracts, or normalizes an address can drop the marker silently. Stripping it never sends funds to a different account; it only loses the preference.

Exchanges that need a preference no intermediate system can remove should treat this format as a convenience rather than a guarantee.

### The Marker Identifies the Recipient as an Exchange

An exchange address is a public label. It appears on deposit pages, in QR codes, and on explorer pages, and it tells anyone who sees it that the account belongs to an exchange. An operator who doesn't want that should publish a v1 address.

### One Account, Two Address Strings

An exchange address and its v1 counterpart denote the same account, but the two strings aren't equal. A system that keys on the address string, such as an address book, an allowlist, or an explorer index, treats one account as two unless it removes the marker first. Store and compare the v1 form. A block explorer SHOULD accept an exchange address and display the v1 form.

This is the most likely integration defect, because it fails silently and produces a missed credit rather than a rejected payment.

## References

- BIP-21 (Bitcoin payment URIs, origin of the `req-` convention): https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki
- BIP-173 (bech32): https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
- BIP-350 (bech32m): https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki
- RFC 2119 (requirement keywords): https://www.rfc-editor.org/rfc/rfc2119
- RFC 5234 and RFC 7405 (ABNF and the case-sensitive `%s` literal): https://www.rfc-editor.org/rfc/rfc5234, https://www.rfc-editor.org/rfc/rfc7405
