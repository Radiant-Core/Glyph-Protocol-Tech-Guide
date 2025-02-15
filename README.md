# Glyph-Protocol-Tech-Guide
Guide on the Glyph Protocol 

How the Glyphs Protocol works on Radiant RXD

Introduction: 
This is a guide to the Glyphs Protocol on the Radiant Blockchain.

Glyphs is a token standard utilizing Radiant's induction proof system and advanced scripting capabilities. Each token is identified by a Radiant reference (ref) and uses smart contracts for enforcing token transfer rules. Glyphs is a fully layer one protocol with all rules enforced by miners.

The Glyphs protocol operates efficiently with minimal indexing requirements, not requiring wallets or indexers to track each token transfer back to the mint. This allows wallets to efficiently verify tokens using SPV. This makes Glyphs a more secure, efficient and scalable system than layer two alternatives.

The protocol supports both fungible and non-fungible tokens. Fungible tokens use Radiant photons as the unit of account, with each fungible token backed by one photon. NFTs may be backed by any number of Radiant photons.

Token contracts are simple, concise and extensible, allowing composition with other custom contracts to support many complex use cases.

Glyphs is currently implemented in Photonic Wallet (Web App and Browser Extension) and Glyph Miner.

For support please join the Radiant community Discord server.

Overview
Glyphs are implemented using Radiant's UTXO model and operate in a similar way to native Radiant photons. Radiant photons are "colored" with a Radiant ref which acts like a contract identifier for the token. A token is minted by creating a UTXO containing an amount of photons, a standard token contract and a Radiant ref. The contract along with the ref is carried forward with each spend. The contract and ref remain the same for the lifetime of the token, until it is melted.

With refs enforced by miners we can be certain a token is authentic simply by verifying block headers in the same way as native photons.

Minting
Glyphs are minted using a commit/reveal procedure with token metadata contained in the reveal transaction's unlocking script. The token's ref is created from an input to the reveal transaction. Therefore a glyph's ref will always point to a commit outpoint.

The token payload in the reveal transaction contains data such as name, image, ticker, relationships to other tokens etc.

The protocol does not dictate how the mint transaction should be created. Developers may use any locking script for the commit/reveal procedure, as long as the token payload is correctly encoded in the reveal.

For each ref that is required an input must be prepared. Radiant singleton refs are used for NFTs, so each NFT requires a unique ref.

Verifying a token
When a wallet receives a token, it extracts the ref from the script and looks up the reveal transaction on the blockchain to obtain the token data.

For example, an NFT script:

OP_PUSHINPUTREFSINGLETON a32f3628a967ce64cca282262e63879136f9c3f4f8e1125345a85c71d90b10b401000000 OP_DROP
OP_DUP OP_HASH160 3a23c6b9055b533db3900e4755dd58d1234c8582 OP_EQUALVERIFY OP_CHECKSIG
The ref a32f3628a967ce64cca282262e63879136f9c3f4f8e1125345a85c71d90b10b401000000 refers to an outpoint in a commit transaction.

The reveal will be the transaction that spent this output, with a token payload contained in the unlocking script.

Radiant's implementation of ElectrumX has been improved to simplify the process of finding the reveal transaction.

Transferring a token
Transferring is done in much the same way as native Radiant photons. For NFTs, the UTXO is spent and the script is moved to a new output. Fungible tokens are similar, except since they use normal refs instead of singletons, token UTXOs may be split and merged.

Any number of different tokens may be provided as inputs and outputs of a transaction. Token transfer rules are all implemented in script, with token input and output values summed within script, ensuring token supply is enforced. Token UTXOs may be positioned anywhere in a transaction's inputs and outputs and there is no restriction on the number of token UTXOs. The Radiant opcodes used to sum token values operate over any number of inputs and outputs.

Encoding
Acquiring and holding RXD coins is a fundamental way to support the Radiant ecosystem. Your participation in the coin and token economy helps drive growth and stability. More holders and Radiant ecosystem participants can strengthen the lindy effect. Here's why buying RXD matters:

Payload structure
A glyph's token data is encoded in the reveal unlocking script, identified by the string "gly" followed by a CBOR encoded token payload:

OP_PUSH "gly"
OP_PUSH <CBOR Payload>
Payload structure
The Glyphs protocol defines standard top level fields shown below in CDDL:

payload = {
  p: [* (number / string)]       ; Array of protocol identifiers
  name: tstr,                    ; Name of the token
  ? type: tstr,                  ; Type of the token (e.g. container, user or other custom types)
  ? in: [*bstr],                 ; Array of container token refs the token is part of
  ? by: [*bstr],                 ; Array of author token refs the token is created by
  ? main: {                      ; Main file
    t: tstr,                     ; File mime type
    b: bstr                      ; File bytes
  },
  ? attrs: { * tstr => any }     ; Attributes as key value pairs (e.g. color, rarity, etc)
  ? loc: number / bytes / string ; Location for link tokens
}
Example CBOR payload:

{
    p: [1, 4],
    name: "My token",
    ticker: "XYZ",
    main: {
        t: "image/jpeg",
        b: <bytes>
    },
    in: [
        <containerRefBytes>
    ],
    by: [
        <authorRefBytes>
    ],
}
Protocols
The p property contains an array of protocols used by the token. Current protocol identifiers are as follows:

ID	Protocol
1

Fungible token
2

Non-fungible token
3

Data storage
4

Decentralized mint
5

Mutable
Some protocols may depend on other protocols also being implemented. For example for a mineable token, a p value of [1, 4] must be used, indicating the token implements the FT and dmint contracts.

File encoding
In addition to the main property other files can be added to the payload. These can be either embedded or remote.

Embed:

  embed = {
    t: tstr, ; File mime type
    b: bstr  ; File bytes
  }
Remote:

  remote = {
    t: tstr,   ; File mime type
    u: tstr    ; URL
    ? h: bstr  ; File hash
    ? hs: bstr ; HashStamp image (low resolution copy of image)
  }
Example:

{
    main: {
        t: "image/jpeg",
        b: <bytes> // File bytes embedded in transaction
    },
    alt: {
        t: "image/jpeg",
        u: "https://url/to/image.jpg", // Link to remote file
        h: <hashBytes>,
        hs: <hashStampBytes>
    }
}
If a link to a remote file is used, it is recommended to include a hash so the downloaded file can be verified.

HashStamp
For remote images a HashStamp may be provided in the hs property. This is a low resolution, highly compressed copy of the image in WebP format. This may be desirable for people who want to store data off-chain but still have some representation of the file on-chain. It is recommended to keep HashStamps within 64x64 pixels and quality 20 to keep size to a minimum.

HashStamps can be used by wallets to display a version of the image if downloading the file is deemed insecure, or used as a loading placeholder similar to a BlurHash.

Types
The Glyphs protocol uses types container and user, for allowing the creation of collections and associating tokens to a creator. Wallets should implement these types and render the tokens accordingly.

Custom types are also permitted.

Custom fields
Any custom fields may be added to the payload.

Fungible tokens
Fungible tokens must be created with the protocol p field set to 1. This informs wallets that the token uses a standard fungible token script.

The script is a P2PKH followed by a script to enforce fungible token rules:

// P2PKH
OP_DUP OP_HASH160 <address> OP_EQUALVERIFY OP_CHECKSIG

OP_STATESEPARATOR

// Fungible token ref
OP_PUSHINPUTREF <ref>

// Get count of ref used in outputs
OP_REFOUTPUTCOUNT_OUTPUTS OP_INPUTINDEX

// Get a hash of the script after state separator
OP_CODESCRIPTBYTECODE_UTXO OP_HASH256 OP_DUP

// Sum token value for all outputs
OP_CODESCRIPTHASHVALUESUM_UTXOS

// Sum token value for all inputs
OP_OVER OP_CODESCRIPTHASHVALUESUM_OUTPUTS

// Input sum must be greater than or equal to output sum
OP_GREATERTHANOREQUAL OP_VERIFY

// To prevent supply inflation, ref must be used within this script and ref count must be equal to contract count
OP_CODESCRIPTHASHOUTPUTCOUNT_OUTPUTS OP_NUMEQUALVERIFY
The script after the P2PKH must be unchanged for every transfer.

The protocol permits locking scripts other than P2PKH, such as multisig, time locks and other more complex scripts. The only requirement is that the script after the state separator is unchanged.

Transfer
Tokens are transferred in a similar way to regular RXD photons. The contract and ref are carried forward with each spend and UTXOs can be split and merged. Tokens may be at any position in the inputs or outputs. A transaction may also contain any number of different tokens.

Melt
The contract permits output value sum to be less than input value sum. Melting fungible tokens only requires making the output value less than input value. The RXD photons backing the tokens are then reclaimed.

Non-fungible tokens
Non-fungible tokens must be created with the protocol p field set to 2.

The script is simply a singleton ref followed by a P2PKH or other locking script:

OP_PUSHINPUTREFSINGLETON <ref> OP_DROP
OP_DUP OP_HASH160 <address> OP_EQUALVERIFY OP_CHECKSIG // P2PKH
Radiant rules enforce that there can only ever be one instance of a singleton ref.

Transfer
NFTs are transferred in a similar way to regular RXD photons. The singleton ref is carried forward with each spend and may be at any position in the transaction inputs or outputs.

Similar to fungible tokens, any number of different tokens may be present in a transaction.

Melt
Melting a non-fungible token can be done in two ways:

Omit the ref from the outputs
Spend the NFT to an unspendable op return script containing the ref
Relationships
The Glyph protocol provides in and by arrays in the token payload for defining relationships to other tokens, such as NFT collections. Relationships can also be defined in custom properties.

For a token to be related its ref must be present in the commit transaction. Wallets must verify this by checking each related token's ref is in an output of the commit. This prevents authorized users creating relationships, such as assigning an author or adding NFTs to a collection they don't control.

Delegates
Having a related token in a commit is not always practical if there are many thousands of mint transactions with the same relationship. For example, minting a large NFT collection with the same container and author tokens. To solve this a token delegate may be used.

The delegate is a temporary normal ref that links to one or more token refs such as a user or container token.

A delegate is created by a transaction containing the related tokens, and one or more outputs with the following script:

OP_PUSHINPUTREF <ref> OP_DROP
OP_DUP OP_HASH160 <address> OP_EQUALVERIFY OP_CHECKSIG // P2PKH
Any number of delegates may be created to simplify bulk mints.

Verifying related tokens
When a wallet does not find a related token's ref in the commit transaction, it must look for delegates within the commit script. Delegate refs must be required using a OP_REQUIREINPUTREF opcode in the commit script.

If the wallet finds required refs, it fetches the delegate creation transaction and looks for related tokens in its outputs. If the refs are found in the delegate creation transaction then the relationship is valid.

Custom relationships
Developers are free to define their own relationships separate to the in and by arrays.

Links
Tokens may be linked to any resource, either tokens or off-chain resources. This has many uses such as creating a record of a token mint or even building a decentralized redirect service.

Links are created using the loc field which is used by wallets in a similar way to the HTTP location header.

To create a link the loc field is set to either a number, bytes or string:

 location = number / bytes / string
number - Link to a token with given output index and transaction ID matching the link ref
bytes - Link to a token given as ref bytes
string - Link to any URI
Mutable tokens
Token payloads can be made mutable by using a mutability contract and adding protocol 5 to the p array in the token payload.

The mutability contract must be created in the mint transaction and assigned a singleton ref with an output index one higher than the token ref (token ref + 1).

The contract uses a "pay to token" script which does not contain a P2PKH or any signature checking opcodes. The contract requires the token to be present in the transaction in order to modify the payload.

Modify mod and seal sl operation are implemented. Seal is called to make the token immutable and enforces burning of the mutability contract.

Pay to token
With a "pay to token" script, the mutability contract is always controlled by the token owner. This means every time a token is transferred to a new owner, they automatically get access to updating the token's data. The mutability output never needs to be transferred, only called to update the data.

Fetching mutable data
If a wallet receives a mutable token, the most recent data can be obtained by looking up the most recent location of the mutability ref and fetching the transaction. Data is encoded similar to a token mint, with the Glyph token payload in the unlocking script.

Script
// SHA256 of payload must be provided
<payloadHash> OP_DROP

OP_STATESEPARATOR

// Mutable contract ref
OP_PUSHINPUTREFSINGLETON <mutableRef>

// Build token ref (mutable ref -1)
OP_DUP 20 OP_SPLIT OP_BIN2NUM OP_1SUB OP_4 OP_NUM2BIN OP_CAT

// Check token ref exists in token output at given refdatasummary index
OP_2 OP_PICK OP_REFDATASUMMARY_OUTPUT OP_4 OP_ROLL 24 OP_MUL OP_SPLIT OP_NIP 24 OP_SPLIT OP_DROP OP_EQUALVERIFY

// Compare ref + scriptsig hash in token output to this script's ref + scriptsig hash
OP_SWAP OP_STATESCRIPTBYTECODE_OUTPUT OP_ROT OP_SPLIT OP_NIP 45 OP_SPLIT OP_DROP OP_OVER 20 OP_CAT OP_INPUTINDEX OP_INPUTBYTECODE OP_SHA256 OP_CAT OP_EQUALVERIFY

// Modify operation
OP_2 OP_PICK 6d6f64 OP_EQUAL OP_IF

// Contract script must exist unchanged in output
OP_OVER OP_CODESCRIPTBYTECODE_OUTPUT OP_INPUTINDEX OP_CODESCRIPTBYTECODE_UTXO OP_EQUALVERIFY

// State script must contain payload hash
OP_OVER OP_STATESCRIPTBYTECODE_OUTPUT 20 OP_5 OP_PICK OP_HASH256 OP_CAT 75 OP_CAT OP_EQUALVERIFY OP_ELSE

// Seal operation
OP_2 OP_PICK 736c OP_EQUALVERIFY OP_OVER OP_OUTPUTBYTECODE d8 OP_2 OP_PICK OP_CAT 6a OP_CAT OP_EQUAL OP_OVER OP_REFTYPE_OUTPUT OP_0 OP_NUMEQUAL OP_BOOLOR OP_VERIFY OP_ENDIF

// Glyph header
OP_4 OP_ROLL ${glyphMagicBytesHex} OP_EQUALVERIFY OP_2DROP OP_2DROP OP_1
Unlocking script
In order to use a "pay to token" contract securely, the data must be signed. Without a signature check opcode in the contract, this must be done in a different way.

The contract requires the token owner to provide the token as an input. In the token's output it must include a hash of the mutability contract's unlocking script. This way, the token owner signs the updated data in the token output, the contract verifies it, and the transaction is not vulnerable to malleation. In addition to the unlocking script hash, the output must have an OP_REQUIREINPUTREF requiring the mutability contract as an input. This ref + hash pair is checked by the mutability contract.

To spend the contract the following parameters must be provided in the unlocking script

OP_PUSH gly
OP_PUSH mod // Modify operation
OP_PUSH <cbor payload> // Updated token payload
OP_PUSH <contract output index> // Location of mutability contract UTXO in outputs
OP_PUSH <ref+hash index in token output> // Index of the ref+hash in the token output, allowing multiple authorizations in a single transaction (zero for a single update)
OP_PUSH <ref index in token output data summary> // Position of token ref in OP_REFDATASUMMARY_OUTPUT (the sorted list of refs used in all outputs)
OP_PUSH <token output index> // Location of token UTXO in outputs
Authorizing an update
To authorize this update, the token output must have the hash of the above unlocking script following an OP_REQUIREINPUTREF. For example:

// Require the mutability contract (token ref + 1)
OP_REQUIREINPUTREF a32f3628a967ce64cca282262e63879136f9c3f4f8e1125345a85c71d90b10b402000000
// Hash of mutability contract's unlocking script
OP_PUSH a204a95c54bf73975256898be5f47d7844146845f9b749408b535d12e3ad85a9
// Drop ref and hash
OP_2DROP
// Following is a standard NFT contract
OP_PUSHINPUTREFSINGLETON a32f3628a967ce64cca282262e63879136f9c3f4f8e1125345a85c71d90b10b401000000 OP_DROP
OP_DUP OP_HASH160 3a23c6b9055b533db3900e4755dd58d1234c8582 OP_EQUALVERIFY OP_CHECKSIG
The mutability contract will verify that the hash of the provided unlocking script, along with the necessary OP_REQUIREINPUTREF are contained in the token UTXO.

Restoring token script
An optional final step is for the token owner to restore the token script to a standard NFT contract that can be recognized by wallets. The OP_REQUIREINPUTREF and hash are no longer required.

Decentralized mints
Decentralized mints (dmint) are mineable tokens that can be claimed by anyone using proof-of-work. These tokens are created with protocol [1, 4] and are locked in a layer one smart contract.

PoW algorithm
With a "pay to token" script, the mutability contract is always controlled by the token owner. This means every time a token is transferred to a new owMineable Glyphs use the follow proof-of-work algorithm:

hash = sha256(sha256(
    sha256(currentLocationTxid + contractRef) +
    sha256(anyInputHash + anyOutputHash) +
    nonce
))
Resulting hash must be below the target.

anyInputHash and anyOutputHash must be the hash of any input or output in the transaction. This allows work to be bound to the miner's address and prevents nonces being stolen. This will typically be a pay-to-public-key-hash but any script can be used. The contract will verify these scripts exist.

Further information
For more information including code for the mining smart contract see the Glyph Miner GitHub.

ElectrumX
Radiant's implementation of ElectrumX has modifications to support Glyphs.

Querying
ElectrumX indexes each script with the ref zeroed out. The zeroed ref acts as a wildcard meaning a wallet can fetch all transactions for an address by using a ref with 36 zero bytes in the token script.

For example:

OP_PUSHINPUTREFSINGLETON 000000000000000000000000000000000000000000000000000000000000000000000000 OP_DROP OP_DUP OP_HASH160 <address> OP_EQUALVERIFY OP_CHECKSIG
This script is hashed and can be used with listunspent and script hash subscriptions. This prevents having to perform many queries for each token ref or having to know token refs in advance.

Ref get
A glyph ref will point to an outpoint in the commit transaction, therefore this alone cannot be used to fetch the token payload. To simplify this task, the ref.get method will return the first and last locations of a ref.

The first location will be the reveal transaction. The last location is used to track the current location of a contract.

List unspent
The listunspent method is improved to return an array of all refs contained in the script. Since refs are zeroed when querying for tokens, the wallet does not know the token ref in the returned UTXOs and would need to query each UTXO individually to determine the token.

With refs returned as part of the listunspent response, the wallet can build scripts without the additional queries.
