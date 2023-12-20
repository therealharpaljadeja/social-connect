# SocialConnect

*For Dev Setup see [CONTRIBUTING.MD](.github/CONTRIBUTING.md)*

SocialConnect is an open source protocol that maps off-chain personal **identifiers** (such as phone numbers, twitter handles, etc.) to on-chain account **addresses**. This enables a convenient and interoperable user experience for use cases such as:

- payments - send money directly to your friend's phone number!
- social discovery - find someone's account based on their twitter!
- any other identity applications!

Here is a short demo of a payment from a [Kaala](https://kaala.app/) wallet user to a [Libera](https://medium.com/impactmarket/ready-to-unlock-your-potential-meet-libera-your-new-crypto-wallet-d1053f917b95) wallet user, with only a phone number:

[<img width="800" alt="image" src="https://user-images.githubusercontent.com/46296830/207285114-6ef73be4-10f2-4afc-a066-811e1f3e1042.png">](https://www.loom.com/share/8afddd73ba324ec18aeb63fc96d568f9)

## Introduction to SocialConnect

[![Introduction to SocialConnect](https://img.youtube.com/vi/XO_33vb45V4/0.jpg)](https://www.youtube.com/watch?v=XO_33vb45V4)

## 🛠 How it Works

SocialConnect uses a federated model, meaning that anyone has the power to be an **issuer** of attestation mappings. Issuers have the freedom to decide how to verify that the user actually has ownership of their identifier. After verification, issuers register the mapping as an attestation to the [on-chain smart contract registry](https://github.com/celo-org/celo-monorepo/blob/master/packages/protocol/contracts/identity/FederatedAttestations.sol). Attestations are stored under the issuer that registered them. When looking up attestations, we then have to decide which issuers are trusted.

Here are some active issuers verifying and registering attestations:

| Issuer Name | Address                                                                                                         |
| ----------- | --------------------------------------------------------------------------------------------------------------- |
| Kaala       | `0x6549aF2688e07907C1b821cA44d6d65872737f05` (mainnet)                                                          |
| Libera      | `0x388612590F8cC6577F19c9b61811475Aa432CB44` (mainnet) `0xe3475047EF9F9231CD6fAe02B3cBc5148E8eB2c8` (alfajores) |

Off-chain identifiers, originally in plaintext, are obfuscated before they are used in on-chain attestations to ensure user privacy and security. This is done with the help of the [Oblivious Decentralized Identifier Service (**ODIS**)](https://docs.celo.org/protocol/identity/odis). The details of the obfuscation process and how to interact with ODIS are described in the [docs about privacy](docs/privacy.md).

### Want a more profound understanding?

We've made a mini-series to explain you:

- [Celo Spark: SocialConnect Mini-Series (1/3) — What Is It?](https://www.youtube.com/watch?v=a_756GRPcV4&list=PLsQbsop73cfErtQwacE4WgqQwoVcLvLZS&index=1)
- [Celo Spark: SocialConnect Mini-Series (2/3) — How Does It Works?](https://www.youtube.com/watch?v=bzZbfoPLYM4&list=PLsQbsop73cfErtQwacE4WgqQwoVcLvLZS&index=2)
- [Celo Spark: SocialConnect Mini-Series (3/3) — Coding Session](https://www.youtube.com/watch?v=qrIHC496avs&list=PLsQbsop73cfErtQwacE4WgqQwoVcLvLZS&index=3)

## 🧑‍💻 Quickstart

The following steps use the Celo [ContractKit](https://docs.celo.org/developer/contractkit) to quickly set you up to play around with the protocol. If you would like to use a different library instead, please refer to the [example scripts](examples/).

1. Add the [`@celo/identity`](https://www.npmjs.com/package/@celo/identity) package into your project.

    ```console
    npm install @celo/identity
    ```

2. Set up your issuer (read "Authentication" section in [privacy.md](docs/docs/privacy.md#authentication)), which is the account registering attestations. When a user requests for the issuer to register an attestation, the issuer should [verify](docs/protocol.md#verification) somehow that the user owns their identifier (ex. SMS verification for phone number identifiers).

    ```ts
    import { newKit } from "@celo/contractkit";

    // the issuer is the account that is registering the attestation
    let ISSUER_PRIVATE_KEY;

    // create alfajores contractKit instance with the issuer private key
    const kit = await newKit("https://alfajores-forno.celo-testnet.org");
    kit.addAccount(ISSUER_PRIVATE_KEY);
    const issuerAddress =
        kit.web3.eth.accounts.privateKeyToAccount(ISSUER_PRIVATE_KEY).address;
    kit.defaultAccount = issuerAddress;

    // information provided by user, issuer should confirm they do own the identifier
    const userPlaintextIdentifier = "+12345678910";
    const userAccountAddress = "0x000000000000000000000000000000000000user";

    // time at which issuer verified the user owns their identifier
    const attestationVerifiedTime = Date.now();
    ```

3. Check and top up [quota for querying ODIS](docs/privacy.md#rate-limit) if necessary.

    ```ts
    import { OdisUtils } from "@celo/identity";
    import { AuthSigner } from "@celo/identity/lib/odis/query";

    // authSigner provides information needed to authenticate with ODIS
    const authSigner: AuthSigner = {
        authenticationMethod: OdisUtils.Query.AuthenticationMethod.WALLET_KEY,
        contractKit: kit,
    };
    // serviceContext provides the ODIS endpoint and public key
    const serviceContext = OdisUtils.Query.getServiceContext(
        OdisContextName.ALFAJORES
    );

    // check existing quota on issuer account
    const { remainingQuota } = await OdisUtils.Quota.getPnpQuotaStatus(
        issuerAddress,
        authSigner,
        serviceContext
    );

    // if needed, approve and then send payment to OdisPayments to get quota for ODIS
    if (remainingQuota < 1) {
        const stableTokenContract = await kit.contracts.getStableToken();
        const odisPaymentsContract = await kit.contracts.getOdisPayments();
        const ONE_CENT_CUSD_WEI = 10000000000000000;
        await stableTokenContract
            .increaseAllowance(odisPaymentsContract.address, ONE_CENT_CUSD_WEI)
            .sendAndWaitForReceipt();
        const odisPayment = await odisPaymentsContract
            .payInCUSD(issuerAddress, ONE_CENT_CUSD_WEI)
            .sendAndWaitForReceipt();
    }
    ```

4. Derive the obfuscated identifier from your plaintext identifier. Refer to documentation on the [ODIS SDK](docs/privacy.md#using-the-sdk) for detailed explanations on these parameters and steps.

    ```typescript
    // get obfuscated identifier from plaintext identifier by querying ODIS
    const { obfuscatedIdentifier } =
        await OdisUtils.Identifier.getObfuscatedIdentifier(
            userPlaintextIdentifier,
            OdisUtils.Identifier.IdentifierPrefix.PHONE_NUMBER,
            issuerAddress,
            authSigner,
            serviceContext
        );
    ```

5. Register an attestation mapping between the obfuscated identifier and an account address in the `FederatedAttestations` contract. This attestation is associated under the issuer. See [docs](docs/protocol.md#registration) for more info.

    ```typescript
    const federatedAttestationsContract =
        await kit.contracts.getFederatedAttestations();

    // upload identifier <-> address mapping to onchain registry
    await federatedAttestationsContract
        .registerAttestationAsIssuer(
            obfuscatedIdentifier,
            userAccountAddress,
            attestationVerifiedTime
        )
        .send();
    ```

6. Look up the account addresses owned by an identifier, as attested by the issuers that you trust (in this example only your own issuer), by querying the `FederatedAttestations` contract. See [docs](docs/protocol.md#lookups) for more info.

    ```ts
    const attestations = await federatedAttestationsContract.lookupAttestations(
        obfuscatedIdentifier,
        [issuerAddress]
    );

    console.log(attestations.accounts);
    ```

## 🚀 Examples

|                                             Type                                              |
| :-------------------------------------------------------------------------------------------: |
|                            [ContractKit](examples/contractKit.ts)                             |
|                              [EthersJS (v5)](examples/ethers.ts)                              |
|                                  [web3.js](examples/web3.ts)                                  |
|         [NextJS based web app (Phone Number)](https://github.com/celo-org/emisianto)          |
|         [NextJS based templated](https://github.com/celo-org/socialconnect-template)          |
| [React Native App (Phone Number)](https://github.com/celo-org/SocialConnect-ReactNative-Demo) |
|      [NextJS based web app (Twitter)](https://github.com/celo-org/SocialConnect-Twitter)      |
| [Server side NextJS (Twitter)](https://github.com/celo-org/SocialConnect-Twitter-Server-Side) |

<!-- -   [@celo/contractkit](https://docs.celo.org/developer/contractkit) (see [`examples/contractKit.ts`](examples/contractKit.ts)),
-   [ethers.js](https://ethers.org/) (see [`examples/ethers.ts`](examples/ethers.ts)), and
-   [web3.js](https://web3js.readthedocs.io/en/v1.8.1/) (see [`examples/web3.ts`](examples/web3.ts)). -->

The [Runtime Environments section](docs/privacy.md#runtime-environments) shows instructions for using SocialConnect with:

- [NodeJS](https://nodejs.org) (see [Runtime Environments > Node](docs/privacy.md#node)),
- [React Native](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwiK9paNjYH9AhUIesAKHQZ1CvYQFnoECA0QAQ&url=https%3A%2F%2Freactnative.dev%2F&usg=AOvVaw3N725EvNXK2_crezzoIs9d) (see [Runtime Environments > React Native](docs/privacy.md#react-native)), and
- Web (see [Runtime Environments > Web](privacy.md#web))
<!--
The [emisianto web app](https://emisianto.vercel.app/) is a sample implementation of a phone number issuer. The code is hosted at [celo-org/emisianto](https://github.com/celo-org/emisianto). -->

<img width="500" alt="image" src="https://user-images.githubusercontent.com/46296830/205343775-60e429ea-f5e5-42b2-9474-8ca7dfe842cc.png">

## 📄 Documentation

For a deeper dive under the hood and specific implementation details, check out the documentation of the [protocol](docs/protocol.md) for details on how to interact with the on-chain registry, [privacy](docs/privacy.md) for how identifiers are obfuscated, and [key-setup](docs/key-setup.md) to setup your role keys to interact with the protocol.

## 🤝 Get In Touch

Interested in Integrating SocialConnect, get in touch by filling this [form](https://docs.google.com/forms/d/e/1FAIpQLSeePUyzd2VQfawO8OsXdvmut3OiyICoLPRtfNfPpvtRw3tEfw/viewform).

## 📣 Feedback

**SocialConnect is in beta**! Help us improve by sharing feedback on your experience in the Github Discussion section. You can also open an issue or a PR directly on this repo.

## FAQ

<details>
  <summary>What is a "plainTextIdentifier"?</summary>

`plainTextIdentifier` is any string of text that a user can use to identify other user.

Phone number, Twitter handle, GitHub username anything that makes it easier to represent an evm based address.

For example:- Alice's phone number: `+12345678901`

</details>

<details>
  <summary>What is an "obfuscatedIdentifier"?</summary>

Identifier that is used on-chain, to which the account address is mapped and used by dApps to lookup. It preserve the privacy of the user by not revealing the underlying `plainTextIdentifier`. Used for on-chain attestations, obtained by hashing the plaintext identifier, identifier prefix, and pepper using this schema: `sha3(sha3({prefix}://{plaintextIdentifier})__{pepper})`.

</details>

<details>
  <summary>What is an "identifier prefix"?</summary>

Identifier Prefix is used to differentiate users having same plainTextIdentifier for different purposes and composability.

For example:- Consider Alice having same username on both Twitter and Github, `alicecodes`.

How do we differentiate between Alice verified using Twitter and Github? <br>
This where `prefix` comes into play, the `plainTextIdentifier alicecodes` can be represented as `twitter://alicecodes` and `github://alicecodes` this helps differentiate whether Alice was verified using Twitter or Github.

Moreover, it also helps in composability if dApps follow a standard and use prefix then the corresponding `obsfuscatedIdentifier` will be the same thus making it easier for dApps to lookup identifier verified by other issuers.

You can keep an eye on prefixes suggested by us [here](https://github.com/celo-org/celo-monorepo/blob/8505d060fef3db3b0ce0cadf2bb879512bb20534/packages/sdk/base/src/identifier.ts#L31).

</details>

<details>
  <summary>What is a "pepper"?</summary>

`pepper` is a unique secret, obtained by taking the first 13 characters of the `sha256` hash of the `unblinded signature`

</details>

<details>
  <summary>What is a "unblinded signature"?</summary>

Obtained by unblinding the signature returned by ODIS which is the combined output, comprised of signature by ODIS signers.

</details>

<details>
  <summary>What is an Issuer?</summary>

Issuer is an entity that is willing to take the overhead of verifying a user's ownership of an identifier.

</details>

<details>
  <summary>Does Issuer need to pay for gas?</summary>

For lookup there is no requirement for gas, assuming that the `obfuscatedIdentifier` to be used for lookup is available.

For registering attestations it is optional, once the `obfuscatedIdentifier` is obtained issuer can decide whether to just sign the attestation and provide it back to the user which will then **use its own funds for gas for registering itself** or the `issuer` can perform the transaction which will require the issuer to pay for gas.

</details>

<details>
  <summary>Does Issuer need to have ODIS quota?</summary>

Yes, Issuer needs to have ODIS Quota to register and lookup users.

</details>

<details>
  <summary>What is cost to register a user?</summary>
  With 10 cUSD worth of ODIS quota you can attest 10,000 users!

</details>

<details>
  <summary>Can I just lookup users and not register them?</summary>
  Yes, you can lookup users under other Issuers. By doing this, you are trusting that the Issuer took care about verifying that the identifier does actually belong to the user.

You might want to do this if you don't want to create a registry of your own and use an already existing registry created by other Issuers.

</details>

<details>
  <summary>Can anyone become an Issuer?</summary>
  Yes, SocialConnect is open for all. Anyone can become an Issuer

</details>

<details>
<summary>What are some security & trust assumptions differences between the ASv1 vs. Social Connect?</summary>

| ASv1                                                    | SocialConnect                                                                                                                    |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Phone number verified by 3 randomly selected validators | Phone number verified by issuer (no guarantee about authenticity or quality of verification), app developers choose who to trust |
| Single root of trust = Collective of Validators         | Many roots of trust = Respective attestation issuer that verified phone number                                                   |

</details>

<details>

<summary>What's the best way to map an address returned by lookupAttestations to the issuer? </summary>

```sol
function lookupAttestations(bytes32 identifier, address[] calldata trustedIssuers)
    external
    view
    returns (
      uint256[] memory countsPerIssuer,
      address[] memory accounts,
      address[] memory signers,
      uint64[] memory issuedOns,
      uint64[] memory publishedOns
    )
```

`lookupAttestations` returns 4 arrays, depending on the order `trustedIssuers` was provided respectively the return values are returned.

For example:-

if trustedIssuers = [I1, I2, ...]
then countsPerIssuer = [CI1, CI2, ...] where CIx = number of accounts attested under the Xth issuer

</details>

<details>
<summary>Is there a convention for phone number format?</summary>

Yes, the SDK function `getObfuscatedIdentifier` will only accept E164 formatted phone numbers.

</details>