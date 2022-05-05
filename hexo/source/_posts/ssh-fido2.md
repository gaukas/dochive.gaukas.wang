---
title: Secure your SSH key with a FIDO2 Authenticator
categories:
- tutorial
- devops
tags: 
- fido2
- ssh
- ed25519
- ecdsa
- yubikey 
- linux
date: 05/04/2022 21:08
---
## Requirements
- __OpenSSH 8.2p1__ or later
- A FIDO2 Authenticator
- For Windows operating system user, always run your terminal in Admin Mode*

\* For unknown reason, [Win32-OpenSSH](https://github.com/PowerShell/Win32-OpenSSH/releases) up to **V8.9.1.0p1-Beta** won't function correctly without running in Admin Mode. 

In this tutorial, I am using a YubiKey 5C NFC as my FIDO2 Authenticator

## Discoverable Key

### Generate Key
You may create your key pair with the steps below:

1. Connect your FIDO2 Authenticator
2. Run command in Terminal to generate a **discoverable key**
```
ssh-keygen -t [key-type] -O resident -O application=ssh:YourFancyKeyNameHere -O verify-required
```
`[key-type]` could be either `ecdsa-sk` or `ed25519-sk` <br>
`-O application=ssh:YourFancyKeyNameHere` names the resident key (optional) <br>
`-O verify-required` will require the PIN for the FIDO2 Authenticator when authenticating with the SSH key generated (optional) <br>
Add `-O no-touch-required` then you don't need to touch your Authenticator (therefore skipping the user presence check)

3. SSH will prompt for the PIN and verify the user presence correspondingly if needed
4. Your public and private key will be then generated. Rename the private key to `id_ecdsa_sk` or `id_ed25519_sk` to make sure your system picks it up automatically.

### Use your key with FIDO2 Authenticator on a new local machine
You may safely carry your key pairs in your FIDO2 Authenticator to other devices.

On a new local machine, use steps below to setup key pairs:

1. Export key pair from FIDO2 Authenticator
```
ssh-keygen -K
```
2. SSH will prompt for PIN and/or user presence. Pay attention to your authenticator as on some systems there might be no literal indication when user presence is required.
3. Rename the private key to `id_ecdsa_sk` or `id_ed25519_sk` to make sure your system picks it up automatically.

### Install your public key to a remote server
There's no difference from how you do it on other key types. Just append your public key to the end of `~/.ssh/authorized_keys`, start on a new line.

### Advantages
With this you can securely install your private key on a untrusted system. And the exportable key feature makes it easier to carry your keys. 

## Yubikey: Manage your resident keys

Download and install `ykman`.

```
./ykman fido credentials list # list all resident keys
./ykman fido credentials delete ABCDEFGHIJKLMN # delete key identified by ABCDEFGHIJKLMN 
```