+++
title = "From zero to an App Store Connect app record"
description = "Instructions for a partner team: which portal steps to complete, what files and access to send so we can archive signed iOS and tvOS IPAs, then how you upload builds with Transporter."
date = 2026-04-17

[taxonomies]
tags = ["howto", "publish", "app-store-connect"]
+++

This page is for **you, the partner**, when our arrangement is: **we build the project and give you signed App Store `.ipa` files** produced with **your** Apple Developer Program membership, and **your** people own App Store Connect (metadata, TestFlight, submission, contracts). Nothing here runs without the items below—treat it as a **single checklist** your IT or account owner can follow.

**Naming in this guide:** treat **`com.yourcompany.app`** as the main application’s **bundle ID** (swap `yourcompany` for your reverse-DNS prefix). **iOS, iPadOS, and tvOS** targets all use **that same** bundle string.

**What we need from you in one sentence:** an **explicit App ID** for `com.yourcompany.app` (same bundle ID for iOS/iPadOS and tvOS), a **second App ID** for the extension whose bundle ID is **`com.yourcompany.app` plus a suffix** (see §2), **distribution signing material** (`distribution.p12` plus the downloaded **`.mobileprovision`** files—**App Store Connect** profiles for iOS, tvOS, and the extension target as applicable), an **App Store Connect app record** for the main bundle ID, and—if you use server push—**APNs** credentials. **Keychain Sharing** (same access group on app and extension) covers shared secrets between targets. Apple’s Help topics linked below include **inline screenshots** wherever the UI matters.

**What you get from us:** signed **`.ipa`** archives ready for upload. **What you do next:** upload with **Transporter** (or Xcode)—see the last section.

## Access we need

Someone with **Account Holder** or **Admin** on the **Apple Developer** account must create identifiers, certificates, and profiles (see Apple’s roles in [Certificates overview](https://developer.apple.com/help/account/certificates/certificates-overview/)). For **App Store Connect**, whoever will create the app record needs **Account Holder**, **Admin**, or **App Manager** ([Add a new app](https://developer.apple.com/help/app-store-connect/create-an-app-record/add-a-new-app/), [role permissions](https://developer.apple.com/help/app-store-connect/reference/account-management/role-permissions)).

Invite the build engineer to the developer team with enough rights to use the signing assets you export, or be ready to **export and transfer** the files listed under “Send us” yourself.

## Where each task lives

| Task | Site | Area |
| --- | --- | --- |
| **App IDs**, certificates, keys, devices, profiles | [Apple Developer Account](https://developer.apple.com/account) | **Certificates, Identifiers & Profiles** |
| **App record** (name, SKU, platforms, bundle pick list) | [App Store Connect](https://appstoreconnect.apple.com) | **My Apps** → **+** |

## Agreements (do this first)

Your **Account Holder** must sign the current agreements in App Store Connect; Apple blocks **New App** until that is done. Follow [Read and agree to agreements](https://developer.apple.com/help/app-store-connect/manage-agreements/read-and-agree-to-agreements/).

## Devices (strongly recommended)

Apple requires **registered devices** for **development** and **Ad Hoc** profiles ([Devices overview](https://developer.apple.com/help/account/devices/devices-overview/), [Register a single device](https://developer.apple.com/help/account/devices/register-a-single-device/)). **App Store Connect** distribution profiles are created from an App ID plus a **distribution** certificate, without picking devices ([Create an App Store Connect provisioning profile](https://developer.apple.com/help/account/provisioning-profiles/create-an-app-store-provisioning-profile/)).

Still register **at least one** physical device: it prevents dead ends if anyone later enables **Automatically manage signing** in Xcode or needs a device build.

## 1) Main app — explicit App ID (iOS, iPadOS, tvOS)

**You register** one **explicit** App ID whose **Bundle ID** is **`com.yourcompany.app`** (or whatever exact string we align in Xcode—this placeholder is the main app only). **iPhone/iPad and Apple TV use the same bundle ID** here; Apple documents one explicit App ID across those platforms (see the note in [Register an App ID](https://developer.apple.com/help/account/identifiers/register-an-app-id/)).

Steps: [Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources) → **Identifiers** → **+** → Apple’s illustrated guide [Register an App ID](https://developer.apple.com/help/account/identifiers/register-an-app-id/).

Enable only capabilities we agreed on. For remote notifications, turn on **Push Notifications** and complete the APNs key step below. For app ↔ extension secrets via Keychain, we use the **same Keychain Sharing group** in **Xcode** on both targets ([Configuring Keychain Sharing](https://developer.apple.com/documentation/xcode/configuring-keychain-sharing)); flip matching switches on the App IDs only if we ask you to ([Enable app capabilities](https://developer.apple.com/help/account/identifiers/enable-app-capabilities/)). **Wildcard** App IDs are not acceptable for this workflow.

## 2) Extension — second explicit App ID

The extension **must not** reuse **`com.yourcompany.app`**. Apple requires a **distinct bundle ID** for the extension target—take the main ID and **append a suffix** (for example **`com.yourcompany.app.notificationservice`**). If the extension App ID equals the host app’s bundle ID, signing and provisioning **will not work**.

**You register** that second **explicit** App ID in the portal the same way as the main app: [Register an App ID](https://developer.apple.com/help/account/identifiers/register-an-app-id/). Capability edits: [Enable app capabilities](https://developer.apple.com/help/account/identifiers/enable-app-capabilities/). Use the **same Keychain Sharing group name** in Xcode on the extension target as on the main app.

## 3) APNs (only if we ship push)

**You create** an **Auth Key** with the Apple Push Notification service scope. Instructions: [Create a private key](https://developer.apple.com/help/account/keys/create-a-private-key/) and [Communicate with APNs using authentication tokens](https://developer.apple.com/help/account/capabilities/communicate-with-apns-using-authentication-tokens/). The **`.p8` downloads once**—store it like a password and send it through a secrets channel together with **Key ID** and **Team ID**.

## 4) Apple Distribution certificate as `.p12`

We need a **team distribution identity** we can import on the build machine.

Apple’s default path uses Keychain Access: [Create a certificate signing request](https://developer.apple.com/help/account/certificates/create-a-certificate-signing-request/). If you generate the CSR on a Mac-less or automated host, OpenSSL is fine (private key never leaves your controlled machine); a long-form OpenSSL-only walkthrough is [this Stack Overflow answer](https://stackoverflow.com/questions/44839232/how-to-create-certificates-keys-pem-and-p12-file-without-using-mac-to-create). Example:

```bash
openssl genrsa -out distribution.key 2048
openssl req -new -key distribution.key -out CertificateSigningRequest.certSigningRequest \
  -subj '/emailAddress=you@example.com, CN=Your Name Or Company'
```

Then in [Certificates](https://developer.apple.com/account/resources/certificates/list) → **+** → **Apple Distribution** ([Certificates overview](https://developer.apple.com/help/account/certificates/certificates-overview/)), upload the CSR, download the `.cer`, and next to `distribution.key` run:

```bash
openssl x509 -in distribution.cer -inform DER -out distribution.pem -outform PEM
openssl pkcs12 -export -inkey distribution.key -in distribution.pem -out distribution.p12
```

The `pkcs12` step asks for an **export password**—choose a strong one and **send it to us separately** from the file. For scripted runs only, you may use `-passout pass:YOUR_PASSWORD` instead of the interactive prompt—never reuse a sample password from documentation.

**Send us:** `distribution.p12` and that export password (two channels if possible).

## 5) Provisioning profiles (App Store Connect)

**You create** and download **App Store Connect** distribution profiles—Apple’s UI walkthrough: [Create an App Store Connect provisioning profile](https://developer.apple.com/help/account/provisioning-profiles/create-an-app-store-provisioning-profile/). In [Profiles](https://developer.apple.com/account/resources/profiles/list) → **+**:

1. **App Store Connect** for the iOS / iPadOS app → select the **main** explicit App ID.
2. **tvOS App Store Connect** for Apple TV → select the **same** App ID (same bundle ID on both platforms).
3. If our pipeline signs the extension as its own product, **App Store Connect** again for the **extension** App ID.

Each profile must reference the **same** **Apple Distribution** certificate you used for `distribution.p12`. Download every `.mobileprovision` and **send us** each file.

**How keychain fits signing (brief):** A provisioning profile carries an **entitlements allowlist**. Apple’s walkthrough shows `keychain-access-groups` there as a **`<TeamID>.*`** wildcard—anything your signed app claims under that team prefix can match—while Xcode still lists the concrete shared group on each target ([TN3125: *The how*](https://developer.apple.com/documentation/technotes/tn3125-inside-code-signing-provisioning-profiles)).

## 6) App Store Connect — app record

**You create** the store listing shell so Apple accepts uploads for this bundle ID: [App Store Connect](https://appstoreconnect.apple.com) → **My Apps** → **+** → **New App**, following [Add a new app](https://developer.apple.com/help/app-store-connect/create-an-app-record/add-a-new-app/). Pick the **explicit** bundle ID you registered. Status semantics: [App and submission statuses](https://developer.apple.com/help/app-store-connect/reference/app-information/app-and-submission-statuses).

## 7) After we return the `.ipa` — upload to App Store Connect

**You** (or release management on your side) upload the binaries. Apple’s overview of tools and roles: [Upload builds](https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds).

**Transporter** is Apple’s **macOS** app to validate and upload `.ipa` files to App Store Connect without opening our project in Xcode.

1. Install from the [Mac App Store](https://apps.apple.com/us/app/transporter/id1450874784?mt=12) (linked from Apple’s [Upload builds](https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds) topic).
2. Use [Transporter Help](https://help.apple.com/itc/transporter/) for the window layout, logs, and delivery history.
3. Sign in with an Apple ID that has your team in App Store Connect. Upload permission is limited to **Account Holder**, **Admin**, **App Manager**, or **Developer** ([Upload builds](https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds), [role permissions](https://developer.apple.com/help/app-store-connect/reference/account-management/role-permissions)).
4. Add each **`.ipa`**, deliver, and wait for Apple’s **processing** email before the build appears for TestFlight or submission.

The **bundle ID**, **marketing version**, and **build number** inside the `.ipa` must match the app record and version you expect—Apple uses those fields to attach the build ([Upload builds](https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds)).

Alternatives Apple documents on the same page include **Xcode** ([Distribute an app through the App Store](https://developer.apple.com/documentation/xcode/distributing-your-app-for-beta-testing-and-releases)), **`xcrun altool`** ([altool help](https://help.apple.com/asc/appsaltool/)), and API-driven flows with the **Transporter CLI** and JWTs. For staff who only move a file from a secure share into Connect, **Transporter is usually the simplest**.

## Send us (secure channel)

Bundle this for the engineer who will run the archive:

- `distribution.p12` and its **export password**
- **`.mobileprovision`** files: iOS / iPadOS App Store Connect, **tvOS** App Store Connect (same App ID, different profile), extension profile if applicable
- **APNs** `.p8`, **Key ID**, **Team ID** if we use push from your servers
- The **Keychain Sharing** group name we coordinate for the app and extension in Xcode (optional)

## Apple Help quick index (figures on these pages)

- [Register an App ID](https://developer.apple.com/help/account/identifiers/register-an-app-id/)
- [Enable app capabilities](https://developer.apple.com/help/account/identifiers/enable-app-capabilities/)
- [Create a certificate signing request](https://developer.apple.com/help/account/certificates/create-a-certificate-signing-request/)
- [Certificates overview](https://developer.apple.com/help/account/certificates/certificates-overview/)
- [Create an App Store Connect provisioning profile](https://developer.apple.com/help/account/provisioning-profiles/create-an-app-store-provisioning-profile/)
- [Add a new app](https://developer.apple.com/help/app-store-connect/create-an-app-record/add-a-new-app/)
- [Read and agree to agreements](https://developer.apple.com/help/app-store-connect/manage-agreements/read-and-agree-to-agreements/)
- [Upload builds](https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds) · [Transporter Help](https://help.apple.com/itc/transporter/)
- [TN3125: Inside Code Signing: Provisioning Profiles](https://developer.apple.com/documentation/technotes/tn3125-inside-code-signing-provisioning-profiles) (profile `Entitlements` allowlist, including `keychain-access-groups`)

If Apple renames UI labels, follow the current Help page titles—the URLs above remain the stable entry points.
