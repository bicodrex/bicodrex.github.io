+++
date = '2025-08-12T09:04:17+02:00'
draft = false
title = 'Encrypting Wallets, Unlocking Knowledge: My Coinswap Summer'
ShowToc = true
TocOpen = true
+++

## Warming up

I feel a bit weird writing this blog post. Usually, writing is something I've been forced to do in school, but this time it's different: I started this blog as a showcase, or rather, a small window into my work.

Why now? The best moment to start would have been months or even years ago, but the second best moment is now!

Without further ado, let me guide you through what I learned this summer by contributing to Coinswap as part of the Summer of Bitcoin program.

### TL;DR

Participating in Summer of Bitcoin was amazing, more than I expected:

- Technical growth:
  - I became proficient with Git rebase and rewriting git history.
  - I improved significantly in Rust programming.
  - I even set up a NitroKey for GPG signing (a story for another day!).
- Teamwork:
  - I met an incredible team and learned design patterns and best practices for Rust.
  - Collaborating with them taught me how to work effectively in a team.

I also wanted to thank Summer of Bitcoin for making this internship possible. They also sent a stipend and some goodies, which made me feel that my work was truly valued and recognized.

A huge thanks to the CoinSwap team as well: they guided me, shared their expertise, and made collaborating on the project an incredible learning experience.

See [The PRs I Opened](#the-prs-i-opened)!

## What is coinswap?

In my mind, coinswap is an incredible project, that is very unique, but let me use coinswap official README to explain it:

> Coinswap is a decentralized [atomic swap](https://bitcoinops.org/en/topics/coinswap/) protocol that enables trustless swaps of Bitcoin UTXOs through a decentralized, Sybil-resistant marketplace.
>
> Existing atomic swap solutions are centralized, rely on large swap servers, and have service providers as single points of failure for censorship and privacy attacks. This project implements atomic swaps via a decentralized market-based protocol.
>
> The project builds on Chris Belcher's [teleport-transactions](https://github.com/bitcoin-teleport/teleport-transactions) and has significantly matured with complete protocol handling, functional testing, Sybil resistance, and command-line applications.
>
> Anyone can become a swap service provider (**Maker**) by running `makerd` to earn fees. **Takers** use the `taker` app to swap with multiple makers, routing through various makers for privacy. The system uses a _smart-client-dumb-server_ philosophy with minimal server requirements, allowing any home node operator to run a maker.
>
> The protocol employs [fidelity bonds](https://github.com/JoinMarket-Org/joinmarket-clientserver/blob/master/docs/fidelity-bonds.md) for Sybil and DoS resistance. Takers coordinate swaps and handle recovery; makers respond to queries. All communication occurs over Tor.
>
> For technical details, see the [Coinswap Protocol Specification](https://github.com/citadel-tech/Coinswap-Protocol-Specification).

## What I needed to work on

My main task during Summer of Bitcoin was to implement wallet encryption and a backup mechanism for Coinswap.

Coinswap wallets are stored in the `wallets/` folder as binary files. This worked functionally, but there was a big issue: the files were completely unencrypted. That meant if malware or any malicious actor gained access to your computer, they could simply decode the file and extract your private keys.

The goal was clear:

- Encrypt the wallet file using a password chosen by the user. This password would be set during wallet creation, and then required again every time the wallet is loaded.

- Add a reliable backup system. Wallets are critical: if a user loses their wallet data, they lose their funds. I needed to design a compact backup format that contains the minimum but sufficient information to completely restore the wallet in case of data loss.

This might sound simple at first (“just encrypt it, right?”), but in practice, it was tricky. I had to think about:

- How to securely derive encryption keys from a password.
- How to make the process user-friendly without compromising on safety.
- How to balance compactness and completeness in backups, so that restoring works perfectly even in edge cases.

And, of course, making sure everything integrates cleanly with the existing Rust codebase, without breaking other parts of Coinswap.

In short, my job was to make sure that wallets in Coinswap are both safe against attackers and resilient against accidental loss.

## What was it like working with the Coinswap team?

Honestly, it was an excellent experience. The team taught me a lot: their reviews were super useful, and they were always kind and supportive. Every time I needed help, they had me covered.

I learned many design patterns and discovered that working alone often hides bugs you wouldn't notice. Multiple sets of eyes reviewing your code is invaluable.

## How is my encryption mechanism working?

### Background

Basically, the representation of the Wallet that is stored in the disk is called `WalletStore`, and reside inside `coinswap/src/wallet/storage.rs`.

Here it is, as a reference:

```rust
/// Represents the internal data store for a Bitcoin wallet.
#[derive(Debug, PartialEq, Serialize, Deserialize)]
pub(crate) struct WalletStore {
    /// The file name associated with the wallet store.
    pub(crate) file_name: String,
    /// Network the wallet operates on.
    pub(crate) network: Network,
    /// The master key for the wallet.
    pub(super) master_key: Xpriv,
    /// The external index for the wallet.
    pub(super) external_index: u32,
    /// The maximum size for an offer in the wallet.
    pub(crate) offer_maxsize: u64,
    /// Map of multisig redeemscript to incoming swapcoins.
    pub(super) incoming_swapcoins: HashMap<ScriptBuf, IncomingSwapCoin>,
    /// Map of multisig redeemscript to outgoing swapcoins.
    pub(super) outgoing_swapcoins: HashMap<ScriptBuf, OutgoingSwapCoin>,
    /// Map of prevout to contract redeemscript.
    pub(super) prevout_to_contract_map: HashMap<OutPoint, ScriptBuf>,
    /// Map of swept incoming swap coins to prevent mixing with regular UTXOs
    /// Key: ScriptPubKey of swept UTXO, Value: Original multisig redeemscript
    pub(super) swept_incoming_swapcoins: HashMap<ScriptBuf, ScriptBuf>,
    /// Map for all the fidelity bond information.
    pub(crate) fidelity_bond: HashMap<u32, FidelityBond>,
    pub(super) last_synced_height: Option<u64>,

    pub(super) wallet_birthday: Option<u64>,

    /// Maps transaction outpoints to their associated UTXO and spend information.
    #[serde(default)] // Ensures deserialization works if `utxo_cache` is missing
    pub(super) utxo_cache: HashMap<OutPoint, (ListUnspentResultEntry, UTXOSpendInfo)>,
}
```

Before my work, `WalletStore`, was just serialized with [CBOR](https://cbor.io/) format and then written to disk unencrypted.

### My solution

My major work is visible inside [`coinswap/src/security.rs`](https://github.com/citadel-tech/coinswap/blob/master/src/security.rs).

It is based on top of two awesome crates, both coming from [RustCrypto project](https://github.com/rustcrypto):

- [aes-gcm](https://github.com/RustCrypto/AEADs/tree/master/aes-gcm): for the encryption
- [pbkdf2](https://github.com/RustCrypto/password-hashes/tree/master/pbkdf2): for the key derivation from the user passphrase

#### How the whole encryption/decryption process flows

##### Encryption

Let’s imagine a user is running the `taker` binary for the first time.  
Here’s what happens behind the scenes when the wallet gets encrypted:

1. The user is asked to set a wallet encryption passphrase
2. A random `pbkdf2_salt` is generated
3. Passphrase + salt are fed into PBKDF2 to derive an AES-GCM encryption key
4. A random nonce (IV) is generated for the AES-GCM operation
5. A brand new wallet is generated in memory
6. The `WalletStore` is serialized into bytes using CBOR
7. Those bytes are encrypted with AES-GCM, using the derived key + nonce
8. Everything (ciphertext + nonce + salt) is bundled into an `EncryptedData` struct
9. That struct is serialized:
   - **CBOR** for the main wallet file (compact & binary)
   - **JSON** for the backup file (human-readable & printable)
10. Finally, it’s written to disk

At this point the wallet only exists on disk in encrypted form.  
Without the passphrase, it’s just noise.

##### Decryption

Now, when the user comes back and wants to load their wallet, the process just runs in reverse:

1. Read the encrypted wallet file from disk
2. Deserialize it into an `EncryptedData` struct
3. Ask the user for their passphrase
4. Re-run PBKDF2 with the stored salt + user’s passphrase to regenerate the AES key
5. Use that key + the saved nonce to decrypt the ciphertext
6. Deserialize the plaintext bytes back into a `WalletStore`
7. And magically, the wallet is unlocked in memory and ready to be used

#### Key data structure

I designed a generic data structure for the encryption, that can be used on any rust struct, that I named `EncryptedData`, that right now we are using both for the Wallet and the WalletBackup.

```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct EncryptedData {
    /// Nonce used for AES-GCM encryption (must match during decryption).
    nonce: EncryptionNonce,
    /// AES-GCM-encrypted CBOR-serialized plaintext struct data.
    encrypted_payload: Vec<u8>,
    /// Salt for the PBKDF2 key generation
    pbkdf2_salt: PBKDF2Salt,
}
```

#### User interface

For the user experience, I took inspiration from how SSH handles key encryption. When creating a new wallet, the user is prompted to set an encryption passphrase. If they prefer not to encrypt their wallet, they can simply leave the passphrase blank: in that case, the wallet will remain unencrypted.

When loading a wallet, Coinswap automatically checks whether the wallet file is encrypted. If it is, the user is asked to provide their passphrase to decrypt it. If not, the wallet loads directly.

To maintain backward compatibility, unencrypted wallets are still supported. This means existing users don’t lose access to their old wallets. At the same time, we made it possible to upgrade an existing unencrypted wallet: the user can back it up and then restore it with a chosen passphrase, effectively encrypting it going forward.

## What about the wallet backup and restore?

### Background

Encrypting the wallet is great, but it’s not enough: if the data on disk gets lost or corrupted, the user would still lose access to their funds.  
Implementing the backup and restore mechanism turned out to be a bit trickier than expected, but in the end, it fit nicely with the rest of the system.

The main idea was to design a **compact backup format** that contains only the minimal information required to fully restore a wallet: no extra noise, just what’s strictly necessary.

### My solution

#### How the whole backup/restore process flows

##### Backup

Here’s what happens behind the scenes during a wallet backup:

1. If the wallet is encrypted, the user is prompted for the existing wallet passphrase.

2. A `WalletBackup` struct is created from the current wallet (`WalletStore`). Only the **essential fields** are included:

   - `network`
   - `master_key`
   - `wallet_birthday`
   - `file_name`

3. If the user chose to encrypt the backup, a **new passphrase** is prompted interactively, and the backup is encrypted using the same AES-GCM + PBKDF2 process as the main wallet.

4. The final backup file `{wallet_name}-backup.json` is written to the current working directory.

##### Restore

Restoring a wallet from a backup is straightforward and safe. Here’s what happens behind the scenes:

1. The user provides the backup file path. Coinswap automatically detects whether the backup is **encrypted**.

   - If encrypted, the user is prompted for the **backup passphrase**.

2. The backup file is loaded from disk and deserialized into a `WalletBackup` struct.

3. A new `WalletStore` is initialized from the backup data, including:

   - `network`
   - `master_key`
   - `wallet_birthday`
   - `file_name` (or a new custom wallet name if specified)

4. The wallet is synced with the blockchain via RPC to ensure all balances, UTXOs, and fidelity bonds are up-to-date.

5. The restored wallet is saved to disk.
   - If the user chose to store the wallet encrypted during the restore procedure, it will be encrypted using the same AES-GCM + PBKDF2 mechanism as normal.

#### Key data structure

Most of the backup-related logic resides in its dedicated file `coinswap/src/wallet/backup.rs`
We chose JSON as the storage format for backups to make backups human-readable and easily portable.

The first thing to keep in mind when designing the backup is that the `WalletStore` contains **dynamic data**, like incoming/outgoing swapcoins, cached UTXOs, and other runtime state.  
A backup is something the user does once and keeps safely: we don’t need to include everything that can later be reconstructed from the blockchain.

To achieve this, we created the following struct for wallet backups:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WalletBackup {
    /// Network the wallet operates on.
    pub(crate) network: Network, //Can be asked to the user, but is nice to save
    /// The master key for the wallet.
    pub(super) master_key: Xpriv,

    pub(super) wallet_birthday: Option<u64>, //Avoid scanning from genesis block
    /// The file name associated with the wallet store.
    pub file_name: String, //Can be asked to user, or stored for convenience
}
```

One thing to note is that this backup does not include in-progress swaps that might be interrupted in the middle of execution.  
If a swap is interrupted, the information about it isn’t stored in the backup.

This is a conscious trade-off: including every dynamic piece of runtime data would make the backup more complex and harder to manage, while most of the wallet’s essential data (keys, network, wallet birthday, file name) is fully preserved.
In practice, this means that restored wallets are fully functional, with the only missing edge case being interrupted swaps: something we deemed acceptable for a simple, compact, and reliable backup structure.

#### Comparing two wallets

While developing tests for the backup and restore procedure, it became necessary to having a way to check if the restored wallet was equal to the backed up one, so I also added a custom `PartialEQ` for the `Wallet` struct, to check if two wallets are cryptographic equivalent:

```rust
impl PartialEq for Wallet {
    fn eq(&self, other: &Self) -> bool {
        //self.store == other.store
        //avoided filename
        self.store.network == other.store.network &&
        self.store.master_key == other.store.master_key &&
        self.store.external_index == other.store.external_index &&
        self.store.offer_maxsize == other.store.offer_maxsize &&
        //avoided incoming_swapcoins
        //avoided outgoing_swapcoins
        //avoided prevout_to_contract_map
        self.store.fidelity_bond == other.store.fidelity_bond &&
        //avoided last_synced_height
        self.store.wallet_birthday == other.store.wallet_birthday &&
        self.store.utxo_cache == other.store.utxo_cache
    }
}
```

#### User interface

The backup and restore procedures are fully accessible to users via the CLI. The commands are simple and intuitive, here are some examples using `taker` binary

- **Backup the current wallet (plain)**:

```bash
./taker wallet-backup
```

- **Backup the current wallet with encryption**:

```bash
./taker wallet-backup --encrypt
```

- **Restore from a backup file using the default wallet name**:

```bash
./taker wallet-restore --backup-file taker-wallet-backup.json
```

- **Restore from a backup file to a custom wallet name**:

```bash
./taker -w restored-wallet wallet-restore --backup-file my-backup.json
```

## The PRs I Opened

Finally, here is the list of PRs that I opened during my journey!

- [#507 - Wallet encryption mechanism using PBKDF2 + AES-GCM](https://github.com/citadel-tech/coinswap/pull/507)  
  This was the first step in my journey: securing the wallet with a user-chosen password. It marked the first half of the project's functionality. In hindsight, the code could have been optimized, but it served its purpose effectively.

- [#559 - Replace implicit wallet saving from `wallet.sync()` with explicit `sync_and_save` wrapper](https://github.com/citadel-tech/coinswap/pull/559)  
  While working on the backup and restore functionality, I discovered that wallet saving was happening implicitly in some parts of the codebase. I had to dig into the code to determine exactly which operations triggered writes, and introduced an explicit `sync_and_save` wrapper to give developers a clearer understanding and full control over when the wallet is saved.

- [#560 - Prevent unencrypted wallet store write on initialization](https://github.com/citadel-tech/coinswap/pull/560)  
  In the same vein as the previous PR, I identified a transient state issue that I missed during the development of the encryption mechanism. This fix ensures that the wallet store isn't written in an unencrypted state upon initialization.

- [#570 - Add Wallet Backup and Restore Functionality (Encrypted and Unencrypted)](https://github.com/citadel-tech/coinswap/pull/570)  
  This PR represents the core of the second milestone. Implementing backup and restore functionality required modularizing encryption logic, which led to PRs #577 and #578.

- [#573 - Better panic message for subprocess errors in `test_maker`](https://github.com/citadel-tech/coinswap/pull/573)  
  A valuable enhancement that improved error handling in the `test_maker` module by providing more informative panic messages, aiding in quicker diagnosis and resolution of subprocess-related issues.

- [#577 - Add Rustdoc comments for public Wallet API functions](https://github.com/citadel-tech/coinswap/pull/577)  
  During development, I noticed the absence of Rustdoc comments in the public Wallet API functions. We decided to add comprehensive documentation to facilitate better understanding and usage of the API.

- [#578 - Extract wallet encryption logic into dedicated security module](https://github.com/citadel-tech/coinswap/pull/578)  
  This major refactor modularized the wallet encryption logic, making the codebase cleaner and more maintainable. It also allowed my work to fit into its own crate, which I'm particularly proud of.
