---
CIP: 88
Title: Token Policy Registration
Category: Tokens
Status: Proposed
Authors:
    - Adam Dean <adam@crypto2099.io>
Implementors: N/A
Discussions:
    - https://github.com/cardano-foundation/cips/pulls/467
Created: 2023-02-27
License: CC-BY-4.0
---

## Abstract

Currently, token projects (NFT or FT) have no mechanism to register the intent, details, or feature set of their minting
policy on-chain. This CIP will aim to create a method that is backwards compatible and enable projects to declare, and
update over time, the supported feature set and metadata details pertaining to their tokens.

This CIP will aim to make use of a hybrid (on and off-chain) information schema to enable maximum flexibility and 
adaptability as new and novel use cases for native assets expand and grow over time.

## Motivation

This CIP was borne out of a distaste for the lack of on-chain token policy intent registration that has been cited as
both a centralization and security concern at various points over the preceding two years of native asset history on
Cardano.

### Example 1: The Cardano Token Registry

Many Fungible Token (FT) projects require special treatment of their NAs such as decimal places for proper display and
formatting, project information, and token logo. As it stands, these projects must currently register via a GitHub
repository ([Cardano Token Registry](https://github.com/cardano-foundation/cardano-token-registry)) in order to have
their token properly appear in wallets. This GitHub repository is currently managed by the Cardano Foundation (CF).
While there is no reason to believe that the CF would take any malicious or nefarious action, forcing token projects to
register in this fashion introduces a point of centralization, _potential_ gate keeping, and ultimately reliance on the
interaction of a 3rd party (the human at CF responsible for merging pull requests) in order for projects to register, or
update, their information.

***Note:*** The original intent of the *CIP-26 Cardano Token Registry* was never to have a sole provider or
controller of the repository. However, time has shown that there is little interest from the community or various
service providers to contribute or participate in this or alternative solutions.

This CIP attempts to provide a decentralized solution to this problem.

### Example 2: Token Metadata Insecurity

One of the stated rationales for [CIP-68](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0068) was that the
"original" Cardano NFT Metadata Standard ([CIP-25](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0025)) was
insecure in some example use cases. This is due to the link between the transactional metadata and the minting of native
assets. For example, a smart contract that cannot/does not validate against transaction metadata (i.e. Liquidity Pool
tokens) could have a malicious user inject CIP-25 metadata into the transaction, potentially inserting illicit, illegal,
or otherwise nefarious metadata information tied to the tokens which may be picked up and displayed to end users 
unwittingly by explorers and wallets.

### Example 3: De-duplication of Data

Similar to [Example 1](#example-1--the-cardano-token-registry) above, when it comes to Non-Fungible Token (NFT) projects
currently operating on the chain there is usually a desire to provide some level of information that pertains to all
tokens under the given policy. Examples include project/collection names, social media handles, and miscellaneous
project registration information. At current, this is generally solved by adding "static" fields in the metadata of
every token. By moving this project-specific information a layer higher, not only do we achieve a path to dynamism
but also reduce ledger bloat size by de-duplicating data. 

Similarly, there are currently multiple marketplaces and decentralized exchanges (DEXes) in operation in the Cardano
ecosystem. At current, most DEXes pull information from the _Cardano Token Registry_ but there is no similar function
for NFT projects. As such, much information must be manually provided to individual marketplaces by the token projects
creating an undue burden on the project creators to provide a largely static amount of information via different web
forms and authentication schemas rather than simply publishing this information to the blockchain directly.

## Specification

### Registration Metadata Format

A Native Script registration is a regular Cardano transaction with specific metadata attached (similar to CIP-36
Catalyst Voter Registration). At the root level, the registration consists of two pieces of data:

1. A Token Registration Payload Object [(TRPO)](#token-registration-payload-object)
2. A Token Registration Witness array [(TRWA)](#token-registration-witness-array)

### Token Registration Payload Object

The Token Registration Payload Object (TRPO) consists of 4 required fields and then optional, additional fields to
provide additional context and information as part of the registration payload. The top-level metadata label of **867**
has been chosen for the purposes of this standard.

### Required Fields

The following fields are required in all token registration submissions.

#### 1. Script Hash (Policy ID) \<String\>

This should be the hex-encoded policy ID (**Script Hash**) that is being registered. This information is readily 
available and accessible via Cardano DB-Sync and other chain indexers and is an easy point of reference. The script must
include at least one key-based signature script requirement in order to witness and sign the policy registration.

If a script requires multiple signatures (multi-sig) then the same validation logic should be applied when verifying
the authenticity of the registration.

**Example:**

`'00000002df633853f6a47465c9496721d2d5b1291b8398016c0e87ae'`

#### 2. Feature Set \<Array:Integer\>

The Feature Set should consist of an array of CIPs that should be applied to the tokens under the given policy. For
example, a project may specify that their tokens utilize CIP-25 (Cardano NFT Metadata Standard) and CIP-27 (Cardano NFT
Royalty Standard) or CIP-68, or other, to be developed and determined CIPs.

A registration with an empty feature set, or lack of a specific CIP, should be considered an explicit omission on behalf
of the project and therefore any potential metadata associated with said CIPs ignored from the time of the registration
forward.

**Example:**

`[25, 27]`

#### 3. Witness Method \<Array:String\>

The witness method portion of the payload should describe the method used to generate the witness signature (CIP-8, etc)
to aid in validation by third parties.

**Example:**

`['CIP-8','Ed25519']`

#### 4. Nonce \<Integer\>

As with CIP-36 Voter Registration method, we must include a piece of data that changes to avoid a replay attack. As with
CIP-36, and stake pool KES counter rotation, the nonce is expected to be an integer value that is incremented as new or
updated registrations occur. The suggestion to use the current slot tip of the network at the time of submission from 
CIP-36 is also recommended here.

Validators and indexers observing token registrations should always honor the verified registration with the highest
nonce value.

**Example:**

`12345`

### Optional Fields

The following are optional fields that may be included in the token registration to provide additional information or
context to the registration.

#### 5. Data Oracle URI \<UriArray:String\>

To be utilized and expanded upon in a separate CIP, this should be a valid URI pointing to a source of additional,
potentially dynamic information relating to the project and/or the tokens minted under it.

**Example:**

`'https://oracle.mytokenproject.io/'`

#### 6. CIP-Specific Information

This entry, if present, should be a CIP ID indexed object containing additional information pertaining to that CIP.
When and where possible the CIP-Specific registration should follow the CBOR-like declaration syntax to ensure that
the content is well-formed and easily parseable.

These additional CIPs/Schemas to be determined by the community could include:

- CIP-25/68 NFT Project top-level project information
- CIP-26 FT Project monetary policy information
- CIP-27 NFT Project Royalty information

##### CIP-25 & CIP-68: Non-Fungible Tokens / Policy Information

See [CIP25](CIP-25) for a description of a non-fungible token project policy registration. 
([CIP25.json](CIP-25/CIP25.json) as an example, [CIP25.schema.json](CIP-25/CIP25.schema.json) for schema documentation).

Because the information registered here is mostly helpful to aggregator services and marketplaces, it applies equally to
both CIP-25 and CIP-68 metadata standards. A project utilizing one or the other should reference this documentation and
include the relevant information under index #6, prefixed by the number of the CIP (25 or 68) depending upon the metadata
format.

##### CIP-26: Fungible Tokens / Monetary Policy

See [CIP26](CIP-26) for a description of a fungible token specific registration ([CIP26.json](CIP-26/CIP26.json) as an 
example, [CIP26.schema.json](CIP-26/CIP26.schema.json) for schema documentation). This information can replace the information
currently housed in the [Cardano Token Registry](https://github.com/cardano-foundation/cardano-token-registry) and is 
based on the format currently used in those registrations along with a few additional fields.

##### CIP-27: Cardano NFT Royalty Information

##### Inline Datum / Reference UTXO Registration

Depending on the needs of a specific project or CIP, the CIP-Specific registration should include a reference pointer to 
an inline datum UTXO that can be consumed and used by smart contract integrations.

**Example:**

```json 
{
  "25": {
    "1": {
      "0": "Cool NFT Project",
      "1": [
        "This is a description of my project",
        "longer than 64 characters so broken up into a string array"
      ],
      "2": [
        "https://",
        "static.coolnftproject.io",
        "/images/icon.png"
      ],
      "3": [
        "https://",
        "static.coolnftproject.io",
        "/images/banner1.jpg"
      ],
      "4": 0,
      "5": {
        "twitter": [
          "https://",
          "twitter.com/spacebudzNFT"
        ],
        "discord": [
          "https://",
          "discord.gg/spacebudz"
        ]
      },
      "6": "Virtua Metaverse"
    }
  },
  "27": {
    "0": "0.05",
    "1": [
      "addr_test1qqp7uedmne8vjzue66hknx87jspg56qhkm4gp6ahyw7kaahevmtcux",
      "lpy25nqhaljc70094vfu8q4knqyv6668cvwhsq64gt89"
    ]
  }
}
```

### Token Registration Witness Array

The Witness Array included in the on-chain metadata should consist of an array of arrays with two elements, the public
key hash of the signing key (as included in the native script) and the signed key witness. If a script requires multiple
signatures, enough signatures to meet the criteria of the script should be included and required for proper validation
of an updated token registration.

**Example**

```json 
[
  [
    "e97316c52c85eab276fd40feacf78bc5eff74e225e744567140070c3j",
    [
      "witness may be broken up into a sub-array of 64-character strings",
      "assuming that the overall length of the witness is longer than the",
      "64-character on-chain metadata limit for strings."
    ]
  ],
  [
    "26bacc7b88e2b40701387c521cd0c50d5c0cfa4c6c6d7f0901395757",
    [
      "this is the second witness for a 2 out of 3 multi-sig script.",
      "it's also broken up into an array.",
      "strings in the witness array should be concatenated as-is",
      "withouts spaces or any formatting applied"
    ]
  ]
]
```

### Example NFT Token Registration Metadata

Below is a complete example of the hypothetical metadata payload for an NFT project registering their policy on-chain.

```json
{
  "867": {
    "1": {
      "1": "00000002df633853f6a47465c9496721d2d5b1291b8398016c0e87ae",
      "2": [
        25,
        27
      ],
      "3": [
        "Ed25519"
      ],
      "4": 12345,
      "5": "https://oracle.mycoolnftproject.io/",
      "6": {
        "25": {
          "1": {
            "0": "Cool NFT Project",
            "1": [
              "This is a description of my project",
              "longer than 64 characters so broken up into a string array"
            ],
            "2": [
              "https://",
              "static.coolnftproject.io",
              "/images/icon.png"
            ],
            "3": [
              "https://",
              "static.coolnftproject.io",
              "/images/banner1.jpg"
            ],
            "4": 0,
            "5": {
              "twitter": [
                "https://",
                "twitter.com/spacebudzNFT"
              ],
              "discord": [
                "https://",
                "discord.gg/spacebudz"
              ]
            },
            "6": "Virtua Metaverse"
          }
        },
        "27": {
          "0": "0.05",
          "1": [
            "addr_test1qqp7uedmne8vjzue66hknx87jspg56qhkm4gp6ahyw7kaahevmtcux",
            "lpy25nqhaljc70094vfu8q4knqyv6668cvwhsq64gt89"
          ]
        }
      }
    },
    "2": [
      [
        "e97316c52c85eab276fd40feacf78bc5eff74e225e744567140070c3j",
        [
          "witness may be broken up into a sub-array of 64-character strings",
          "assuming that the overall length of the witness is longer than the",
          "64-character on-chain metadata limit for strings."
        ]
      ],
      [
        "26bacc7b88e2b40701387c521cd0c50d5c0cfa4c6c6d7f0901395757",
        [
          "this is the second witness for a 2 out of 3 multi-sig script.",
          "it's also broken up into an array.",
          "strings in the witness array should be concatenated as-is",
          "withouts spaces or any formatting applied"
        ]
      ]
    ]
  }
}
```

### Example FT Token Registration Metadata

```json
{
  "867": {
    "1": {
      "1": "00000002df633853f6a47465c9496721d2d5b1291b8398016c0e87ae",
      "2": [
        26
      ],
      "3": [
        "Ed25519"
      ],
      "4": 12345,
      "5": "https://oracle.tokenproject.io/",
      "6": {
        "26": {
          "1": [
            {
              "0": [
                "d894897411707efa755a76deb66d26dfd50593f2e70863e1661e98a0",
                "7370616365636f696e73"
              ],
              "1": "spacecoins",
              "2": "SPACE",
              "3": [
                "the OG Cardano community token",
                "-",
                "whatever you do, your did it!",
                "",
                "Learn more at https://spacecoins.io!"
              ],
              "4": 0,
              "5": [
                "https://",
                "spacecoins.io"
              ],
              "6": [
                "ipfs://",
                "bafkreib3e5u4am2btduu5s76rdznmqgmmrd4l6xf2vpi4vzldxe25fqapy"
              ],
              "7": [
                [
                  "ipfs://",
                  "bafkreibva6x6dwxqksmnozg44vpixja6jlhm2w7ueydkyk4lpxbowdbqly"
                ],
                "3507afe1daf05498d764dce55e8ba41e4acecd5bf42606ac2b8b7dc2eb0c305e"
              ],
              "8": [
                "37da68d092f4e61feb237de5e86b404171b59d3880d340023a16a6983491736d",
                "0"
              ]
            }
          ]
        }
      }
    },
    "2": [
      [
        "e97316c52c85eab276fd40feacf78bc5eff74e225e744567140070c3j",
        [
          "witness may be broken up into a sub-array of 64-character strings",
          "assuming that the overall length of the witness is longer than the",
          "64-character on-chain metadata limit for strings."
        ]
      ]
    ]
  }
}
```

## Rationale

For this specification, I have drawn inspiration from 
[CIP-36: Catalyst/Voltaire Registration Metadata Format](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0036)
which succinctly and canonically publishes data to the main chain (L1) via a metadata transaction and without any
required modification or customization to the underlying ledger with regard to Native Assets.

By leveraging the existing signing keys present in native asset scripts from the beginning of the Mary Era on Cardano we
can enable all projects to update and provide additional, verified information about their project in a canonical, 
verifiable, and on-chain way while also providing for additional off-chain information.

This makes this CIP backwards compatible with all existing standards (CIP-25, 26, 27, 68, etc) while also providing the
flexibility for future-proofing and adding additional context and information in the future as additional use cases,
utility, and standards evolve.

## Path to Active

### Acceptance Criteria

- [ ] This CIP should receive feedback, criticism, and refinement from: CIP Editors and the community of people involved
  with token projects (both NFT and FT) to review any weaknesses or areas of improvement.
- [ ] Guidelines and examples of publication of data as well as discovery and validation should be included as part of
  of criteria for acceptance.
- [ ] Specifications should be updated to be written in both JSON Schema and CBOR CDDL format for maximum compatibility.
- [ ] Implementation and use demonstrated by the community: Token Projects, Blockchain Explorers, Wallets, Marketplaces/DEXes.

#### TO-DO ACCEPTANCE ACTIONS ####

- [ ] Convert schema specifications to CDDL format
- [ ] Publish instructions and tooling for publication and verification

### Implementation Plan

1. Publish open source tooling and instructions related to the publication and verification of data utilizing this standard.
2. Achieve "buy in" from existing community actors and implementors such as: blockchain explorers, token marketplaces,
   decentralized exchanges, wallets.

## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).