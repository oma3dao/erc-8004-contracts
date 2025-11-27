ERC-8004 Trustless Agents
Discover agents and establish trust through reputation and validation

Abstract
This protocol proposes to use blockchains to discover, choose, and interact with agents across organizational boundaries without pre-existing trust, thus enabling open-ended agent economies.

Trust models are pluggable and tiered, with security proportional to value at risk, from low-stake tasks like ordering pizza to high-stake tasks like medical diagnosis. Developers can choose from three trust models: reputation systems using client feedback, validation via stake-secured re-execution, zkML proofs, or TEE oracles.
Motivation
MCP allows servers to list and offer their capabilities (prompts, resources, tools, and completions), while A2A handles agent authentication, skills advertisement via AgentCards, direct messaging, and complete task-lifecycle orchestration. However, these agent communication protocols don't inherently cover agent discovery and trust.

To foster an open, cross-organizational agent economy, we need mechanisms for discovering and trusting agents in untrusted settings. This ERC addresses this need through three lightweight registries, which can be deployed on any L2 or on Mainnet as per-chain singletons:

Identity Registry - A minimal on-chain handle based on ERC-721 with URIStorage extension that resolves to an agent's registration file, providing every agent with a portable, censorship-resistant identifier.

Reputation Registry - A standard interface for posting and fetching feedback signals. Scoring and aggregation occur both on-chain (for composability) and off-chain (for sophisticated algorithms), enabling an ecosystem of specialized services for agent scoring, auditor networks, and insurance pools.

Validation Registry - Generic hooks for requesting and recording independent validators checks (e.g. stakers re-running the job, zkML verifiers, TEE oracles, trusted judges).

Payments are orthogonal to this protocol and not covered here. However, examples are provided showing how x402 payment proofs can enrich feedback signals.
Specification
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

Implementations of ERC-8004 MUST deploy the Identity Registry for agent registration and discovery. The Reputation and Validation Registries are extensions that MAY be deployed as additional trust layers that can be deployed independently or in combination.  Deploying more than one Reputation Registry or more than one Validation Registry per Identity Registry is NOT RECOMMENDED.

Identity Registry
The Identity Registry MUST use ERC-721 with the OPTIONAL URIStorage extension for agent registration, making all agents immediately browsable and transferable with NFTs-compliant apps. Each agent is uniquely identified globally by:

namespace: eip155 for EVM chains
chainId: The blockchain network identifier
identityRegistry: The address where the ERC-721 registry contract is deployed
agentId: The ERC-721 tokenId assigned incrementally by the registry

Throughout this document, ERC-721's tokenId is referred to as agentId. The owner of the ERC-721 token can transfer ownership or delegate management (e.g., updating the registration file) of the NFT to operators, as supported by standard ERC-721 functions (`transferFrom`, `approve`, `setApprovalForAll`).  

Token URI and Agent Registration File
The `tokenURI` function MUST resolve to the agent registration file. It MAY use any URI scheme such as ipfs:// (e.g., ipfs://cid) or https:// (e.g., https://domain.com/agent3.json).  Implementations MAY allow the URI returned by `tokenURI` to be updated by the NFT owner or an authorized operator.

The registration file MUST be a valid JSON object conforming to the following requirements:

**Registration File Fields**

| Field              | Type            | Required | Description                                                                                       |
| ------------------ | --------------- | -------- | ------------------------------------------------------------------------------------------------- |
| `type`             | string          | MUST     | Schema identifier.                                                                                |
| `name`             | string          | MUST     | Agent name.                                                                                       |
| `description`      | string          | MUST     | Natural language description of the agent.                                                        |
| `image`            | string (URI)    | SHOULD   | URI to agent image. SHOULD be present for ERC-721 compatibility.                                  |
| `endpoints`        | array           | MUST     | Array of at least one endpoint object (see Endpoint Object Fields table below).                   |
| `registrations`    | array           | SHOULD   | Array of registration objects. Agents MUST have at least one registration.                        |
| `supportedTrust`   | array (string)  | MAY      | Array of supported trust models (e.g., `"reputation"`, `"crypto-economic"`, `"tee-attestation"`). |

**Endpoint Object Fields**

| Field            | Type    | Required | Description                                                                                            |
| ---------------- | ------- | -------- | ------------------------------------------------------------------------------------------------------ |
| `name`           | string  | MUST     | Identifier for the endpoint type (see Endpoint Name Registry below)                                    |
| `endpoint`       | string  | MUST     | The actual endpoint URI, address, or identifier                                                        |
| `version`        | string  | SHOULD   | Version of the protocol or standard being used                                                         |
| `capabilities`   | object  | MAY      | Protocol-specific capabilities (e.g., for MCP spec)                                                    |

**Endpoint Name Registry**

The `name` field MUST use one of the following standardized values:

| Name         | Description                                      | Endpoint Format                    |
| ------------ | ------------------------------------------------ | ---------------------------------- |
| `A2A`        | Agent-to-Agent protocol                          | URL to agent card                  |
| `MCP`        | Model Context Protocol                           | URL to MCP server                  |
| `OPENAPI`    | OpenAPI (REST) specification                     | URL to API base or OpenAPI spec    |
| `GRAPHQL`    | GraphQL API                                      | URL to GraphQL endpoint            |
| `JSONRPC`    | JSON-RPC API                                     | URL to JSON-RPC endpoint           |
| `OASF`       | Open Agent Skill Format                          | URL to OASF skill definition       |
| `DID`        | Decentralized Identifier                         | DID string (e.g., did:web:...)     |

Future versions of this specification will define required fields for each endpoint type.

**Registration Object Fields**

| Field             | Type    | Required | Description                                                       |
| ----------------- | ------- | -------- | ----------------------------------------------------------------- |
| `agentId`         | number  | MUST     | The agent's token ID in the registry.                             |
| `agentRegistry`   | string  | MUST     | CAIP-10 format identifier (e.g., `"eip155:1:{identityRegistry}"`) |

When using an agentId, Clients MUST validate that JSON returned by the tokenURI is tied to the NFT that points to tokenURI.  One path is to check that the tokenURI associated with the agentId contains a registration object with the matching agentId and agentRegistry.  Other verification mechanisms will be introduced in the future.  This filters out NFTs that masquerade as the registration file's owner.

**Example Registration File**

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "myAgentName",
  "description": "A natural language description of the Agent, which MAY include what it does, how it works, pricing, and interaction methods",
  "image": "https://example.com/agentimage.png",
  "endpoints": [
    {
      "name": "A2A",
      "endpoint": "https://agent.example/.well-known/agent-card.json",
      "version": "0.3.0"
    },
    {
      "name": "MCP",
      "endpoint": "https://mcp.agent.eth/",
      "capabilities": {},
      "version": "2025-06-18"
    },
    {
      "name": "OPENAPI",
      "endpoint": "https://api.example.com/v1",
      "version": "3.0.0"
    },
    {
      "name": "OASF",
      "endpoint": "ipfs://{cid}",
      "version": "0.7"
    },
    {
      "name": "DID",
      "endpoint": "did:method:foobar",
      "version": "v1"
    }
  ],
  "registrations": [
    {
      "agentId": 22,
      "agentRegistry": "eip155:1:{identityRegistry}"
    }
  ],
  "supportedTrust": [
    "reputation",
    "crypto-economic",
    "tee-attestation"
  ]
}
```

Onchain metadata
ERC-8004 Identity Registries MAY enable support for metadata by implementing the `getMetadata(uint256 agentId, string key)` and `setMetadata(uint256 agentId, string key, bytes value)` functions.
Examples of keys are “agentWallet” or “agentName”.

When metadata is set, the following event MUST be emitted:

`event MetadataSet(uint256 indexed agentId, string indexed indexedKey, string key, bytes value)`

Registration
ERC-8004 Identity Registries MUST enable the minting of new agents by supporting the following functions:

```
function register(string tokenURI) returns (uint256 agentId)

struct MetadataEntry {
string key;
bytes value;
}

function register(string tokenURI, MetadataEntry[] calldata metadata) returns (uint256 agentId)

```

The `agentId` returned by `register()` MUST equal the tokenId assigned to the minted NFT.

Each register function MUST emit the standard ERC-721 Transfer event, one MetadataSet event for each metadata entry (if any), and a Registered event:

`event Registered(uint256 indexed agentId, string tokenURI, address indexed owner)`

---

### Solidity Interface

An Identity Registry MUST support the following Solidity interface:

```solidity
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";

/**
 * @title IERC8004
 * @dev Base interface for ERC-8004 Identity Registry
 * @notice Extends ERC-721 with agent registration
 * @notice Implementations MAY support optional metadata via the register() overload
 *         and MetadataSet event. See specification for metadata support details.
 */
interface IERC8004 is IERC721 {
    
    /**
     * @dev Metadata entry
     */
    struct MetadataEntry {
        string key;
        bytes value;
    }
    
    /**
     * @dev Emitted when a new agent is registered
     * @param agentId The token ID assigned to the agent
     * @param tokenURI The URI pointing to the agent's registration file
     * @param owner The address that owns the newly minted agent NFT
     */
    event Registered(
        uint256 indexed agentId,
        string tokenURI,
        address indexed owner
    );
    
    /**
     * @dev Emitted when metadata is set (optional feature)
     * @param agentId The token ID
     * @param key The metadata key (indexed for filtering)
     * @param value The metadata value
     */
    event MetadataSet(
        uint256 indexed agentId,
        string indexed key,
        bytes value
    );
    
    /**
     * @notice Register a new agent
     * @param tokenURI URI pointing to the registration file
     * @return agentId The token ID of the newly registered agent (equals the NFT tokenId)
     */
    function register(string memory tokenURI) external returns (uint256 agentId);
    
    /**
     * @notice Register a new agent with metadata
     * @param tokenURI URI pointing to the registration file
     * @param metadata Array of metadata entries to set
     * @return agentId The token ID of the newly registered agent (equals the NFT tokenId)
     */
    function register(
        string memory tokenURI,
        MetadataEntry[] memory metadata
    ) external returns (uint256 agentId);
    
    // Note: getMetadata() and setMetadata() functions are intentionally omitted from this
    // base interface as they are OPTIONAL.
}
```

**Interface Notes:**

- Extends `IERC721` to inherit standard NFT functions (`ownerOf`, `transferFrom`, `balanceOf`, etc.)
- The `register(string tokenURI)` function is REQUIRED for all implementations
- The `register()` overload with metadata is OPTIONAL
- The `Registered` event MUST be emitted when an agent is registered
- The `MetadataSet` event MUST be emitted when metadata is set (if supported)
- The `agentId` returned by `register()` MUST equal the ERC-721 `tokenId`
- Metadata functions (`getMetadata`, `setMetadata`) are intentionally omitted - use the Security Extension for metadata support
- Extensions (Security, EAS, Web) define additional interfaces that can be composed with this base

---

### Reputation Registry
When the Reputation Registry is deployed, the identityRegistry address is passed to the constructor and publicly visible by calling:

function getIdentityRegistry() external view returns (address identityRegistry)

As an agent accepts a task, it's expected to sign a feedbackAuth to authorize the clientAddress (human or agent) to give feedback. The feedback consists of a score (0-100), tag1 and tag2 (left to developers' discretion to provide maximum on-chain composability and filtering), a file uri pointing to an off-chain JSON containing additional information, and its KECCAK-256 file hash to guarantee integrity. We suggest using IPFS or equivalent services to make feedback easily indexed by subgraphs or similar technologies. For IPFS uris, the hash is not required.
All fields except the score are OPTIONAL, so the off-chain file is not required and can be omitted.
Giving Feedback
New feedback can be added by any clientAddress calling:

function giveFeedback(uint256 agentId, uint8 score, bytes32 tag1, bytes32 tag2, string calldata fileuri, bytes32 calldata filehash, bytes memory feedbackAuth) external

The agentId must be a validly registered agent. The score MUST be between 0 and 100. tag1, tag2, and uri are OPTIONAL.

feedbackAuth is a tuple with the structure (agentId, clientAddress, indexLimit, expiry, chainId, identityRegistry, signerAddress) signed using EIP-191 or ERC-1271 (if clientAddress is a smart contract). The signerAddress field identifies the agent owner or operator who signed.
Verification succeeds only if: agentId, clientAddress, chainId and identityRegistry are correct, blocktime < expiry and indexLimit is greater than the last index of feedback received by that client for that agentId. While in most cases indexLimit is simply lastIndex + 1, it can be much higher. This allows agentId to pre-authorize multiple feedback submissions, useful for agent watch tower use cases.

If the procedure succeeds, an event is emitted:

event NewFeedback(uint256 indexed agentId, address indexed clientAddress, uint8 score, bytes32 indexed tag1, bytes32 tag2, string fileuri, bytes32 filehash)

The feedback fields, except fileuri and filehash, are stored in the contract storage along with the feedbackIndex (the number of feedback submissions that clientAddress has given to agentId). This exposes reputation signals to any smart contract, enabling on-chain composability.
Revoking Feedback
clientAddress can revoke feedback by calling:

function revokeFeedback(uint256 agentId, uint64 feedbackIndex) external

This emits:

event FeedbackRevoked(uint256 indexed agentId, address indexed clientAddress, uint64 indexed feedbackIndex)

Appending Responses

Anyone (e.g., the agentId showing a refund, any off-chain data intelligence aggregator tagging feedback as spam) can call:

function appendResponse(uint256 agentId, address clientAddress, uint64 feedbackIndex, string calldata responseUri, bytes32 calldata responseHash) external

Where responseHash is the KECCAK-256 file hash of the responseUri file content to guarantee integrity. This field is OPTIONAL for IPFS URIs.

This emits:

event ResponseAppended(uint256 indexed agentId, address indexed clientAddress, uint64 feedbackIndex, address indexed responder, string responseUri)
Read Functions

function getSummary(uint256 agentId, address[] calldata clientAddresses, bytes32 tag1, bytes32 tag2) external view returns (uint64 count, uint8 averageScore)
//agentId is the only mandatory parameter; others are optional filters.
//Without filtering by clientAddresses, results are subject to Sybil/spam attacks. See Security Considerations for details

function readFeedback(uint256 agentId, address clientAddress, uint64 index) external view returns (uint8 score, bytes32 tag1, bytes32 tag2, bool isRevoked)

function readAllFeedback(uint256 agentId, address[] calldata clientAddresses, bytes32 tag1, bytes32 tag2, bool includeRevoked) external view returns (address[] memory clientAddresses, uint8[] memory scores, bytes32[] memory tag1s, bytes32[] memory tag2s, bool[] memory revokedStatuses)
//agentId is the only mandatory parameter; others are optional filters. Revoked feedback are omitted.

function getResponseCount(uint256 agentId, address clientAddress, uint64 feedbackIndex, address[] responders) external view returns (uint64)
//agentId is the only mandatory parameter; others are optional filters.

function getClients(uint256 agentId) external view returns (address[] memory)

function getLastIndex(uint256 agentId, address clientAddress) external view returns (uint64)

We expect reputation systems around reviewers/clientAddresses to emerge. While simple filtering by reviewer (useful to mitigate spam) and by tag are enabled on-chain, more complex reputation aggregation will happen off-chain.
Off-Chain Feedback File Structure
The OPTIONAL file at the URI could look like:


{
  //MUST FIELDS
  "agentRegistry": "eip155:1:{identityRegistry}",
  "agentId": 22,
  "clientAddress": "eip155:1:{clientAddress}",
  "createdAt": "2025-09-23T12:00:00Z",
  "feedbackAuth": "...",
  "score": 100,

  //MAY FIELDS
  "tag1": "foo",
  "tag2": "bar",
  "skill": "as-defined-by-A2A",
  "context": "as-defined-by-A2A",
  "task": "as-defined-by-A2A",
  "capability": "tools", // As per MCP: "prompts", "resources", "tools" or "completions"
  "name": "Put the name of the MCP tool you liked!", // As per MCP: the name of the prompt, resource or tool
  "proof_of_payment": {
	"fromAddress": "0x00...",
	"toAddress": "0x00...",
	"chainId": "1",
	"txHash": "0x00..." 
   }, // this can be used for x402 proof of payment
 
 // Other fields
  " ... ": { ... } // MAY
}
Validation Registry
This registry enables agents to request verification of their work and allows validator smart contracts to provide responses that can be tracked on-chain. Validator smart contracts could use, for example, stake-secured inference re-execution, zkML verifiers or TEE oracles to validate or reject requests.

When the Validation Registry is deployed, the identityRegistry address is passed to the constructor and is visible by calling getIdentityRegistry(), as described above.
Validation Request
Agents request validation by calling:

function validationRequest(address validatorAddress, uint256 agentId, string requestUri, bytes32 requestHash) external

This function MUST be called by the owner or operator of agentId. The requestUri points to off-chain data containing all information needed for the validator to validate, including inputs and outputs needed for the verification. The requestHash is a commitment to this data, which is OPTIONAL if requestUri is a content addressable storage uri (e.g. IPFS). All other fields are mandatory.

A ValidationRequest event is emitted:

event ValidationRequest(address indexed validatorAddress, uint256 indexed agentId, string requestUri, bytes32 indexed requestHash)
Validation Response
Validators respond by calling:

function validationResponse(bytes32 requestHash, uint8 response, string responseUri, bytes32 responseHash, bytes32 tag) external

Only requestHash and response are mandatory; responseUri, responseHash and tag are optional. This function MUST be called by the validatorAddress specified in the original request. The response is a value between 0 and 100, which can be used as binary (0 for failed, 100 for passed) or with intermediate values for validations with a spectrum of outcomes. The optional responseUri points to off-chain evidence or audit of the validation, responseHash is its commitment (in case the resource is not on content addressing storages such as IPFS), while tag allows for custom categorization or additional data.

validationResponse() can be called multiple times for the same requestHash, enabling use cases like progressive validation states (e.g., “soft finality” and “hard finality” using tag) or updates to validation status.

Upon successful execution, a ValidationResponse event is emitted with all function parameters:

event ValidationResponse(address indexed validatorAddress, uint256 indexed agentId, bytes32 indexed requestHash, uint8 response, string responseUri, bytes32 tag)

The contract stores requestHash, validatorAddress, agentId, response, lastUpdate, and tag in its memory for on-chain querying and composability.
Read Functions

function getValidationStatus(bytes32 requestHash) external view returns (address validatorAddress, uint256 agentId, uint8 response, bytes32 tag, uint256 lastUpdate)

function getSummary(uint256 agentId, address[] calldata validatorAddresses, bytes32 tag) external view returns (uint64 count, uint8 avgResponse)
//Returns aggregated validation statistics for an agent. agentId is the only mandatory parameter; validatorAddresses and tag are optional filters

function getAgentValidations(uint256 agentId) external view returns (bytes32[] memory requestHashes)

function getValidatorRequests(address validatorAddress) external view returns (bytes32[] memory requestHashes)

Incentives and slashing related to validation are managed by the specific validation protocol and are outside the scope of this registry.
Rationale
Agent communication protocols: MCP and A2A are popular, and other protocols could emerge. For this reason, this protocol links from the blockchain to a flexible registration file including a list where endpoints can be added at will, combining AI primitives (MCP, A2A) and Web3 primitives such as wallet addresses, DIDs, and ENS names.
Feedback: The protocol combines the leverage of nomenclature already established by A2A (such as tasks and skills) and MCP (such as tools and prompts) with complete flexibility in the feedback signal structure.
Gas Sponsorship: Since clients don't need to be registered anymore, any application can implement frictionless feedback leveraging EIP-7702.
Indexing: Since feedback data is saved on-chain and we suggest using IPFS for full data, it's easy to leverage subgraphs to create indexers and improve UX.
Deployment: We expect ERC-8004 to be deployed with singletons per chain. Note that an agent registered and receiving feedback on chain A can still operate and transact on other chains. Agents can also be registered on multiple chains if desired.
Test Cases
This protocol enables:
Crawling all agents starting from a logically centralized endpoint and discover agent information (name, image, services), capabilities, communication endpoints (MCP, A2A, others), ENS names, wallet addresses and which trust models they support (reputation, validation, TEE attestation)
Building agent explorers and marketplaces using any ERC-721 compatible application to browse, transfer, and manage agents
Building reputation systems with on-chain aggregation (average scores for smart contract composability) or sophisticated off-chain analysis. All reputation signals are public good.
Discovering which agents run in a TEE, and which public key to use to trust that instance
Discovering which agents support stake-secured or zkML validation and how to request it through a standardized interface
 Security Considerations
Pre-authorization for feedback only partially mitigates spam, as Sybil attacks are still possible, inflating the reputation of fake agents. The protocol's contribution is to make signals public and use the same schema. We expect many players to build reputation systems, for example, trusting or giving reputation to reviewers (and therefore filtering by reviewer, as the protocol already enables).
On-chain pointers and hashes cannot be deleted, ensuring audit trail integrity
Validator incentives and slashing are managed by specific validation protocols
While ERC-8004 cryptographically ensures the registration file corresponds to the on-chain agent, it cannot cryptographically guarantee that advertised capabilities are functional and non-malicious. The three trust models (reputation, validation, and TEE attestation) are designed to support this verification need
Copyright
Copyright and related rights waived via CC0.

