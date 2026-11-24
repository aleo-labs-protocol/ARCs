---
arc: 137
title: Aleo Domain Name Service (ANS) - Specification
authors: snowtigersoft
discussion: https://github.com/AleoHQ/ARCs/discussions/45
topic: Application
status: Draft
created: 2023-11-12
updated: 2025-02-03
---


## Abstract

This ARC proposes the Aleo Name Service (ANS), a protocol and program definition designed to resolve short, human-readable names to service and resource identifiers on the Aleo network. 

ANS aims to simplify the interaction with Aleo's resources by allowing memorable and updatable human-readable identifiers. 

It supports public and private domain names, each serving distinct use cases and privacy needs.

The goal of public domain names is to provide stable, human-readable identifiers that can be used to specify network resources. In this way, users can enter a memorable string, such as ‘snowtiger.ans’ or ‘wallet.leo’, and be directed to the appropriate resource.

For a private domain name, users can transfer Aleo Credits(ACs) to a ANS domain, and the holder can claim the ACs without let others know the real aleo address.

## Specification

### Overview

The ANS protocol comprises three primary components:

- **ANS Registry Program**: Manages the domain name system, mapping names to resolvers and allowing updates to these mappings.
- **Registrars**: Assign domain names within the ANS, ensuring efficient and secure name distribution.
- **Resolvers**: Retrieve resource information associated with a name, such as contract addresses, content hashes, or IP addresses.

### ANS Registry

- **Functionality**: Enables domain owners to set resolver addresses and create subdomains.
- **Administration**: Includes a `register_tld` method to assign TLDs to registrars.

### ANS Resolvers

- **Purpose**: Handle resource lookups for names, returning the requested data.
- **Flexibility**: Different resolver types can be implemented for varied resources.

### ANS Registrars

- **Domain Allocation**: Distribute domain names to users.
- **TLD Management**: Specific TLDs can be managed by designated registrars.

### Name Syntax

ANS names adhere to the following syntax:

```
<domain> ::= <label> | <domain> "." <label>
<label> ::= [0-9a-z-_]{1,64}
```

### Namehash Algorithm

The `NameHash` algorithm is implemented in the Leo programming language as follows:

```leo
struct Name {
    name: [u128; 4],
    parent: field
}

struct data {
    metadata: [field; 4], // the first element is the name_field of the name
}

// The name field is calc with : Poseidon2::hash_to_field(Name)
let name_field: field = Poseidon2::hash_to_field(Name);
let nft_data: data = data {
    metadata: [name_field, 0field, 0field, 0field],
};
let data_hash: field = BHP256::hash_to_field(nft_data);
let name_hash: field = BHP256::commit_to_field(data_hash, 0scalar);

// Example hashes:
// snowtiger.ans -> {"name_hash":"6693959521410375224785930739327684197035498658576576569580044426861265697143field","name_field":"30665291794647428136195279638773259127813058861265393424959682110890737520field"}
// aleo.ans -> {"name_hash":"2339280018181543938491454882369764555700582383836645176101714191730818438169field","name_field":"6273747515721100082959884163283721366188875329060143080399862383799249990077field"}
```

### Registry Specification

The ANS Registry is the central component of the Aleo Name Service (ANS) that maps human-readable names to resource identifiers. It allows for the registration, transfer, and management of domain names.

#### Structs

```leo
// Struct representing a domain name within the ANS.
// The `name` field holds the ASCII bits of a domain name. If the length of the bits is less than 512,
// zeros are appended to the end. The bits are then split into four parts, with `name[0]` holding the
// first 128 bits and `name[3]` holding the last 128 bits.
struct Name {
    name: [u128; 4],
    parent: field // The name_hash of the parent name, for top level domain(tld), this attribute is 0field.
}

// Extends the `Name` struct to include the resolver address.
// The `resolver` field points to the resolver program responsible for handling queries related to the domain.
// A value of 0u128 for the resolver indicates that the current program is used as the resolver.
struct NameStruct {
    name: [u128; 4],
    parent: field,
    resolver: u128 // The resolver program address.
}

struct data {
    metadata: [field; 4], // the first element is the name_hash of the name
}

record NFT {
    owner: address,
    data: data,
    edition: scalar
}

record NFTView {
    owner: address,
    data: data,
    edition: scalar,
    is_view: bool
}
```

#### Mappings
```leo
// Mapping from a name_hash to the owner address.
// This mapping is utilized to track the ownership of NFTs representing public domain names in the ANS ecosystem.
// It serves a dual purpose:
// 1. It records the owner of an NFT(a specific public domain name).
// 2. It facilitates the mapping between ANS public domains and Aleo addresses, meaning the owner of a specific ANS name is also the entity to which this name resolves on the Aleo blockchain. 
// For resolution of addresses on other blockchains, or for resolving other types of data associated with the domain (such as avatar URLs), the resolver functionality is employed. 
mapping nft_owners: field => address;

// Mapping from a name_hash to its corresponding NameStruct.
mapping names: field => NameStruct;

// Mapping from an address to a primary name
mapping primary_names: address => field;
// Mapping from a name_hash to a version number, version plus one when the name is transferred or burned
mapping name_versions: field => u64;
```

#### Functions
Below are detailed descriptions for each function within the registry specification:
```leo
// Function to register a top-level domain (TLD) with the ANS Registry.
// Only administrators can invoke this function.
// @param registrer: The address of the registrar assigned to the TLD.
// @param tld: The TLD to be registered.
async transition register_tld(
    registrar: address,
    tld: [u128; 4]
) -> Future

// Allows the registration of a domain name by verifying the ownership of the parent ANS name.
// The caller must be the owner of the parent domain(an authorized program or address).
// @param name: The domain name to be registered.
// @param parent: The name_hash of the parent domain.
// @param receiver: The address to receive the NFT representing the domain ownership.
// @param resolver: The resolver program address.
// @return NFT: The NFT record representing the new domain registration.
async transition register(
    name: [u128; 4],
    parent: field,
    receiver: address,
    resolver: field
) -> (NFT, Future)

// Registers a subdomain where the parent domain is a private domain name represented by an NFT record.
// @param name: The domain name to be registered.
// @param parent: The NFT representing the parent domain.
// @param receiver: The address to receive the NFT representing the subdomain ownership.
// @param resolver: The resolver program address.
// @return (NFT, NFT): The NFT records representing the parent and new domain registration.
async transition register_private(
    name: [u128; 4],
    parent: NFT,
    receiver: address,
    resolver: field
) -> (NFT, NFT, Future)

// the difference between register_public and register is that register_public does not require the caller to be the owner of the parent name
// instead, the signer must be the owner of the parent name
// that means, the owner of the parent name must be an account not a program
// @param name: The domain name to be registered.
// @param parent: The name_hash of the parent domain.
// @param receiver: The address to receive the NFT representing the domain ownership.
// @param resolver: The resolver program address.
// @return NFT: The NFT record representing the new domain registration.
async transition register_public(
    name: [u128; 4],
    parent: field,
    receiver: address,
    resolver: field
) -> (NFT, Future) 

// Function to transfer the ownership of a private domain name represented by an NFT.
// @param nft: The NFT record representing the domain to be transferred.
// @param receiver: The address to receive the ownership of the domain.
// @return NFT: The updated NFT record with the new owner.
async transition transfer_private(
    nft: NFT,
    private receiver: address
) -> (NFT, Future)

// Function to transfer the ownership of a public domain name.
// @param nft_data: The NFT data representing the domain to be transferred.
// @param nft_edition: always 0scalar
// @param receiver: The address to receive the ownership of the domain.
async transition transfer_public(
    private nft_data: data,
    private nft_edition: scalar,
    public receiver: address
) -> (NFTView, Future)

// Function to convert a private domain name represented by an NFT to public domain name, 
// and transfer the ownership of the public domain name to the receiver, the receiver can be the original owner of the private domain name
// @param nft: The NFT record representing the domain to be converted.
// @param receiver: The address to receive the ownership of the domain.
async transition transfer_private_to_public(
    nft: NFT,
    private receiver: address
) -> (NFTView, Future)

// Function to convert a public domain name to private domain name represented by an NFT,
// and transfer the ownership of the private domain name to the receiver, the receiver can be the original owner of the public domain name.
// @param nft_data: The NFT data representing the domain to be transferred.
// @param nft_edition: always 0scalar
// @param receiver: The address to receive the ownership of the domain.
// @return NFT: The NFT record with the new owner.
async transition transfer_public_to_private(
    private nft_data: data,
    private nft_edition: scalar,
    private receiver: address,
) -> (NFT, Future)

// Sets the primary name associated with the caller's address.
// @param name_hash: The name_hash of the domain to be set as primary.
async transition set_primary_name(
    name_hash: field
) -> Future

// Unsets the primary name associated with the caller's address.
async transition unset_primary_name() -> Future

// Updates the resolver address for a given name_hash in the names mapping.
// @param name_hash: The name_hash of the domain whose resolver is being set.
// @param resolver: The new resolver program address.
async transition set_resolver(name_hash: field, resolver: field)

// check if the ANS is owned by the caller, return a new ANS with same data
transition check(
    nft: NFT
) -> NFT
```

### Registrar Specification

A registrar can have multiple different registration methods to facilitate subdomain distribution, as long as each of them calls the registry’s register method for registration.

```leo
// register a first level domain
// the registrar program must be the holder of parent ANS name
async transition register_fld(
    name: [u128; 4],
    receiver: address
) -> Future {
    // this is "ans" name hash, different TLDs have different name hash, and assigned to different registrars
    let parent: field = 559532657689873513833888656958509165446284001025178663602770230581478239512field;
    aleo_name_service_registry.aleo/register(name, parent, receiver, 0u128);
    return then finalize();
}
```

### Resolver Specification

The Resolver is a critical component of the Aleo Name Service (ANS) that translates human-readable names into machine-readable identifiers. The resolver allows domain owners to set various records, including but not limited to external blockchain addresses and content links, thereby extending the utility and flexibility of domain name usage within the ANS system.

#### Structs

```leo
// Struct to index resolvers by the name_hash and a category for the type of resource they resolve to.
// This allows for flexibility in the type of data associated with a name.
struct ResolverIndex {
    name: field, // The name_hash of the domain.
    category: u128, // The type of the resolver, indicating the resource category.
    version: u64 // The version of the resolver, useful for updates and cache invalidation.
}
```

#### Mappings

```leo
// Mapping from a ResolverIndex to the resolver data. The data typically includes the resource record associated with a name.
mapping resolvers: ResolverIndex => [u128; 4];
```

#### Functions

Below are detailed descriptions for each function within the resolver specification:

```leo
// Adds or updates a resolver record for a given private ANS with a specified category.
// @param name_hash: The name_hash of the domain to update the resolver record for.
// @param category: A u128 bit representation of the category type for the record.
// @param content: The data associated with the resolver record, such as an address or hash.
async transition set_resolver_record(
    nft: aleo_name_service_registry.aleo/NFT,
    category: u128,
    content: [u128; 8]
) -> Future

// Removes a resolver record for a given private ANS and category.
// @param name_hash: The name_hash of the domain to remove the resolver record from.
// @param category: A u128 bit representation of the category type for the record being removed.
async transition unset_resolver_record(
    nft: aleo_name_service_registry_v4.aleo/NFT,
    category: u128
) -> Future

// Adds or updates a resolver record for a given public ANS with a specified category.
// @param name_hash: The name_hash of the domain to update the resolver record for.
// @param category: A u128 bit representation of the category type for the record.
// @param content: The data associated with the resolver record, such as an address or hash.
async transition set_resolver_record_public(
    name_hash: field,
    category: u128,
    content: [u128; 8]
) -> Future

// Removes a resolver record for a given public ANS and category.
// @param name_hash: The name_hash of the domain to remove the resolver record from.
// @param category: A u128 bit representation of the category type for the record being removed.
async transition unset_resolver_record_public(
    name_hash: field,
    category: u128
) -> Future
```

## Privacy Credit Transfer Scheme

The Privacy Credit Transfer Scheme is an innovative feature built upon the Aleo Name Service (ANS) that facilitates the private transfer of credits. This scheme ensures that neither party in the transaction is required to disclose their Aleo address, thereby enhancing privacy while maintaining ease of use.

### Mappings

```leo
// Mapping from a name_hash to the secret associated with the domain.
// This secret is used to verify the claim of the credits transferred to the domain.
mapping domain_credits: field => u64;
```

### Transfer Credits

This function enables the transfer of credits to an ANS domain without revealing the recipient's real Aleo address. It involves a secret that allows the domain holder to claim the transferred credits privately.

```leo
// Function for transferring credits to an ANS domain without revealing the domain holder's real address.
// @param receiver: The name_hash of the recipient ANS domain.
// @param secret: The secret associated with the transaction, used for claim verification.
// @param amount: The amount of credits to be transferred.
// @param pay_record: The record of the payment being made.
transition transfer_credits(receiver: field, secret: [u128; 2], amount: u64, pay_record: credits.leo/credits)
```

### Claim Credits

These functions allow domain holders to claim the transferred credits. Depending on whether the domain is public or private, the appropriate claim function is used.

Claim Credits Private
```leo
// Function for a domain holder to claim credits using a private ANS domain represented by an NFT and a secret.
// This ensures that the claim process remains private and the domain holder's real address is not exposed.
// @param nft: The NFT record representing the private ANS domain.
// @param secret: The secret used to verify the claim.
// @param amount: The amount of credits to be claimed.
transition claim_credits_private(nft: NFT, secret: [u128; 2], amount: u64)
```

Claim Credits Public
```leo
// Similar to the private claim function, this enables the claiming of credits for a public ANS domain.
// The domain holder uses a secret to claim the credits, maintaining privacy.
// @param name_hash: The name_hash of the public ANS domain.
// @param secret: The secret used to verify the claim.
// @param amount: The amount of credits to be claimed.
transition claim_credits_public(name_hash: field, secret: [u128; 2], amount: u64)
```

## Compatibility with ARC-0721

The Aleo Name Service (ANS) aims to be as compatible as possible with the ARC-0721 standard ([AleoHQ/ARCs/discussions/36](https://github.com/AleoHQ/ARCs/discussions/36)), which establishes a framework for non-fungible tokens on the Aleo platform. However, there are some irreconcilable differences between the two, primarily due to the unique requirements of the ANS.

### Divergence in NFT Structure

One of the key differences lies in the structure of the NFT used within ANS. While ARC-0721 defines a standard structure for NFTs, ANS requires a dynamic approach to the `data` field within the NFT record. This is due to the nature of domain name registration, where each NFT must reflect a unique identifier (name_hash) that is only determined at the time of domain registration by the user. Below is the ANS-specific NFT structure:

```leo
// The ANS NFT structure diverges from ARC-0721 in the `data` field.
// Here, `data` is not predefined but is dynamically created based on the domain name registered by the user.
// This `data` serves as the name_hash of the name, uniquely identifying the domain within ANS.
record NFT {
    owner: address,
    data: field,  // The dynamic name_hash of the registered domain name.
    edition: scalar // The edition number, similar to ARC-0721's structure.
}
```

### Embracing Differences for Enhanced Functionality

The modifications to the NFT structure within ANS are necessary to support the protocol's functionality and objectives. While ANS strives to align with existing standards like ARC-0721, it also recognizes the need to innovate and adapt its NFT representation to serve its unique purpose effectively. This approach ensures that ANS can provide a robust and privacy-centric naming service that complements the broader Aleo ecosystem.

It is important for the community and developers to be aware of these differences for a seamless integration and to leverage the strengths of both standards where they apply.

## Reference Implementations

1. Registry: https://github.com/S-T-Soft/ans-programs/blob/main/programs/registry/src/main.leo
2. Registrar: https://github.com/S-T-Soft/ans-programs/blob/main/programs/ansregistrar/src/main.leo
3. Resolver: https://github.com/S-T-Soft/ans-programs/blob/main/programs/ans_resolver/src/main.leo
4. Credit Transfer: https://github.com/S-T-Soft/ans-programs/blob/main/programs/credit_transfer/src/main.leo

## Dependencies

As this is an application ARC, there are no dependencies other than what is currently available in Aleo.

### Backwards Compatibility

Not necessary.


## Security & Compliance

This is an application ARC standard, so this only affects the security of managing assets on-chain.


## References

1. [ERC-137: Ethereum Domain Name Service - Specification](https://eips.ethereum.org/EIPS/eip-137)
2. [ARC-0721: NFTs with owner and data privacy](https://github.com/ProvableHQ/ARCs/discussions/79)
