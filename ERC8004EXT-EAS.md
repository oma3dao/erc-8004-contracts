# ERC-8004 Extension: Ethereum Attestation Service Integration

**Extension for attestation-based trust using EAS**

## Abstract

This extension integrates Ethereum Attestation Service (EAS) into ERC-8004 to provide a standardized, composable trust layer for agents. It defines:

1. **DID → DID Address mapping** - Deterministic address derivation for EAS recipient indexing
2. **Standard attestation schemas** - User reviews, security assessments, certifications, and endorsements
3. **Trust model integration** - How attestations complement the base ERC-8004 Reputation Registry

This extension enables permissionless, multi-party attestations while maintaining compatibility with existing EAS infrastructure and tooling.

## Motivation

The base ERC-8004 Reputation Registry provides on-chain feedback storage but has limitations:

- **Discoverability**: Even if the Client knows tokenUri and the address and chain ID of the Identity Registry, it still doesn't know the contact address and chain ID of the corresponding reputation registries. 
- **Limited expressiveness**: Fixed Reputation Registry schema may not accommodate all attestation types and doesn't provide granular semantic information. 
- **Semantic meaning**: A single Reputation Registry doesn't provide granular semantic information on an attestation. 
- **Ecosystem fragmentation**: Separate system from existing attestation infrastructure widely used in the Ethereum ecosystem. 

EAS integration addresses these by:

- **Leveraging existing contracts**: EAS is widely deployed across multiple chains, and its contract addresses are widely known. 
- **Multiple schemas**: EAS supports multiple schemas, which increases expressiveness and introduces semantic meaning. 
- **Schema platform**: EAS is a permissionless system that allows anyone to define their own schemas, enriching the ERC-8004 trust layer.
- **Adoption**: EAS is already adopted by the Ethereum ecosystem. 

## Definitions

The following terms are used throughout this specification:

| Term                     | Definition                                                                                                                          |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| **Extension**            | This specification document.                                                                                                        |
| **Base Specification**   | The [base ERC-8004 specification document](https://github.com/erc-8004/erc-8004-contracts/blob/master/ERC8004SPEC.md).              |
| **EAS**                  | [Ethereum Attestation Service](https://attest.sh).                                                                                  |
| **Identity Registry**    | The ERC-721 contract that stores agent registrations as defined in the Base Specification.                                          |
| **Reputation Registry**  | The ERC-721 contract that stores agent feedback as defined in the Base Specification.                                               |
| **Agent**                | A software service registered in the Identity Registry.                                                                             |
| **Client**               | Software that queries the Extension to obtain information about an Agent. Examples include wallets, marketplaces, and other agents. |
| **DID**                  | Decentralized Identifier as defined by the [W3C specification](https://www.w3.org/TR/did-core/#terminology).                        |
| **canonicalDID**         | The normalized form of a DID string (see [ERC-8004 Security Extension](./ERC8004EXT-SECURITY.md#didhash)).                          |
| **didHash**              | Hash of the canonical DID (see [ERC-8004 Security Extension](./ERC8004EXT-SECURITY.md#didhash)).                                    |

Terms defined in the Base Specification are incorporated by reference.

## Background

### What is a DID?

### What is Ethereum Attestation Service?

Ethereum Attestation Service (EAS) is an open-source, permissionless attestation infrastructure that enables anyone to make attestations on-chain or off-chain about anything. It was developed with support from the Ethereum Foundation and is maintained by a growing ecosystem of projects and developers.

**Key Resources:**

- **Website**: [https://attest.org](https://attest.org)
- **Documentation**: [https://docs.attest.org](https://docs.attest.org)
- **GitHub**: [https://github.com/ethereum-attestation-service](https://github.com/ethereum-attestation-service)
- **Explorer**: [https://easscan.org](https://easscan.org)
- **GraphQL Indexer**: [https://easscan.org/graphql](https://easscan.org/graphql)

**Deployment Status:**

The EAS team deployed EAS on multiple EVM-compatible chains including:

- Ethereum Mainnet
- Optimism
- Base
- Arbitrum
- Polygon
- Linea
- Scroll

EAS repositories are open-source, so many more unofficial deployments also exist. 

**Semantics of `recipient` in EAS**

In EAS, `recipient` represents the entity being attested to.  It is the same as "subject" in the DID specification.  EAS uses the Solidity type `address` for `recipient` as it is the only native identifier type in Solidity.  EAS uses the Solidity type address for recipient purely as a compact 20-byte identifier. The EAS protocol never interprets this as an executable account or owner. It is a data key, not a security principal. In practice, this means that the `recipient` value:

- MAY correspond to an externally owned account (EOA) or smart contract
- but also MAY represent a logical identity, such as a DID, an ERC-721 token, or another off-chain entity deterministically encoded into 20 bytes

This background is important when reading this specification. 

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### DID Address

In order to integrate EAS into ERC-8004, this Extension defines the formal use of the EAS `recipient` field as the generic identifier of a subject. This generic subject identifier uses a new logical identifier type: **DID Address**, a 20-byte value derived from a `didHash`, which is derived from a `did`.  

A DID Address MUST be computed by taking the least significant 160 bits of a didHash (keccak256 hash of the canonical DID). The conversion SHALL truncate the 256-bit hash to 160 bits by discarding the most significant 96 bits, with the resulting 20-byte value interpreted as a Solidity address type.  The specification for computing didHash from a DID can be found in the [ERC-8004 Security Extension](./ERC8004EXT-SECURITY.md#didhash).

DID Address:
- Preserves determinism (same DID always produces the same DID Address)
- Ensures collision resistance (≈ 1 / 2^160 probability)
- Fits naturally into EAS's existing `recipient` field size without requiring protocol changes

Implementations MAY implement the DID to DID Address conversion using the following example code:

```solidity
function didToAddress(string memory did) internal pure returns (address didAddress) {
    return address(uint160(uint256(computeDidHash(did))));
}

// didToAddress() implementation can be found in the ERC-8004 Security Extension
didAddress = didToAddress(did);
```

**URI to DID**

To convert a URL/URI to a `did`, implementations MAY use the following exampole code:

```javascript
// Convert URL to did:web format
function urlToDid(url) {
    const urlObj = new URL(url);
    const host = urlObj.hostname;
    const path = urlObj.pathname.replace(/^\//, '').replace(/\/$/, '');
    
    if (path) {
        return `did:web:${host}:${path.replace(/\//g, ':')}`;
    }
    return `did:web:${host}`;
}

// Examples:
// https://agent.example.com → did:web:agent.example.com
// https://example.com/agents/myagent → did:web:example.com:agents:myagent
// https://example.com/.well-known/agent.json → did:web:example.com:.well-known:agent.json
```

**Critical: DID Address is NOT a Wallet**

The DID Address derived from a DID:
- MUST NOT be interpreted as a signer or owner
- MUST NOT receive asset transfers (ETH, tokens, NFTs)
- MUST NOT be used for access control or permissions
- SHOULD only be used as an EAS `recipient` for indexing and querying

**Rationale**

This usage of the EAS `recipient` enables efficient DID discovery in EAS, and in doing so enables attestations on agents **even if they are not registered in an Identity Registry**. As long as an agent has a URL (e.g.- tokenUri), anyone can:

1. Convert the URL to a `did:web` DID
2. Compute the `didHash`
3. Derive the DID Address
4. Create or search for attestations using that address as the EAS `recipient`

**Use Cases Without Registry Registration:**

1. **Early-stage agents**: Accumulate reviews before formal registration
2. **Cross-registry attestations**: Same attestations work across multiple registries
3. **Off-chain agents**: Attest to agents that never register on-chain
4. **Migration**: Attestations survive registry changes or migrations
5. **Permissionless trust**: Anyone can start building reputation for any URL

## Schema Design

When creating attestation schemas for DIDs, implementations MUST include a `subject` field in the schema definition. The `subject` field:

- SHOULD use type `string` to store the full DID (e.g., `"did:web:example.com"`)
- MUST be the DID that was used to derive the `recipient` address

Why Both `recipient` and `subject`?

- **`recipient` (address)**: EAS indexing key - enables efficient queries by DID Address
- **`subject` (string)**: Verification field - proves attestation is about the correct DID

This dual-field approach prevents spoofing attacks where an attacker could create attestations with a valid `recipient` but different `subject` content.

Example EAS schemas:

```
"uint8 rating,bytes32 contentHash,bytes5 locale,uint16 version,string subject"
```

## Standard Attestation Schemas

This extension defines standard schemas for agent trust attestations. All schemas include the required `subject` field for DID verification.  These will be listed here in a future version of the Extension.

## Attestation Querying

Clients retrieve attestations about a DID using this flow:

```javascript
// 1. Compute DID Address
const didAddress = didToAddress(did);

// 2. Query EAS for attestations
const attestations = await eas.getAttestations({
    recipient: didAddress,
    schemaUID: SCHEMA_UID_USER_REVIEW
});

// 3. Verify each attestation
for (const attestation of attestations) {
    const payload = decodeAttestationData(attestation.data);
    
    // MUST verify subject matches
    if (payload.subject !== did) {
        throw new Error("DID hash mismatch");
    }
    
    // Process valid attestation
    processAttestation(payload);
}
```

Clients MAY index EAS attestations by DID Address using subgraphs or EAS GraphQL endpoints.

Example query filters: recipient=<didAddress>, schema=<schemaUID>.”

**Verification Requirements:**

Clients MUST verify:
1. `recipient` equals `didAddress(did)`
2. `subject` in payload equals the `did` used to derive recipient
3. Attestation is not revoked
4. Attestation is not expired (if `expirationTime` is set)
5. Attester is trusted (per client's trust policy)


## Future Direction: EAS v2 Migration Path

The current v1 EAS Extension uses a DID address as the value of the subject field in order to remain compatible with the canonical EAS contract interface, which strictly defines subject as an address.

This approach allows ERC-8004 identity registries and DIDs to participate in the EAS ecosystem today, but it is a transitional mechanism, not a permanent design choice. The DID address format is only a proxy representation of a DID or registry identifier—it was never intended to be a spendable or externally owned
 account, and implementations MUST treat it as a non-transferable identifier rather than a wallet address.

In v2, the EAS extension will migrate away from this workaround toward a more flexible subject model that allows:

- Native support for non-address subject types such as bytes32 (DID hashes), string (canonicalized DIDs), or structured identifiers (e.g., CAIP-19 asset references).
- A unified attestation schema capable of expressing cross-chain and cross-namespace subjects.
- Optional backward compatibility for v1 DID address subjects through SDK-level resolution.

This evolution is not a competing alternative to v1, but a planned migration path. The v1 approach ensures interoperability with existing EAS deployments. The v2 framework defines the long-term direction for EAS-compatible attestations across heterogeneous identifiers, registries, and chains.

Implementations integrating this specification should plan to:

- Continue supporting DID address subjects for legacy attestations.
- Add support for resolving v2 subject types as they become available.
- Transition indexers and SDKs to a unified query layer that transparently handles both v1 and v2 attestations.

## Copyright

Copyright and related rights waived via CC0.
