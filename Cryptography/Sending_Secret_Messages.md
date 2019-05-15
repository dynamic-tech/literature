# Overview

This article starts off by clearly showing the steps you need to send and receive secret messages. These steps involve the use of `openssl`, so there's no need to rely on (or learn) S/MIME (encrypted emails).

# Table of Contents

- [Generating your key-pair](#generating-your-key-pair)
  - [Why Public-key Cryptography?](#why-public-key-cryptography)
- [OpenSSL Reference](#openssl-reference)
  - [Which OpenSSL Implementation?](#which-openssl-implementation)
- [Sending Encrypted Messages](#sending-encrypted-messages)
  - [PEM Key Format](#pem-key-format)
  - [Generate an Encryption Key](#generate-an-encryption-key)
  - [Encrypt the Encryption Key](#encrypt-the-encryption-key)
  - [Encrypt the Message](#encrypt-the-message)
- [Receiving Encrypted Messages](#receiving-encrypted-messages)
  - [Decrypt the Encryption Key](#decrypt-the-encryption-key)
  - [Decrypt the Message](#decrypt-the-message)

(TOC manually generated courtesy of [this generator](https://imthenachoman.github.io/nGitHubTOC/).)

# Generating Your Key-pair

To send and receive secret messages, you will need to create a *cipher key*. Since we'll be using **public-key cryptography** (aka **asymmetric cryptography**, explained soon in next subsection), the relevenat term here is *key-pair*. If you're confused, just hold on to the term **key**, for now.

The mathematics behind **public-key cryptography** is harder to explain as it involves *number theory* and [*modular arithmetic*](https://en.wikipedia.org/wiki/Modular_arithmetic) (which takes a good part of a weekend to run through). It's much easier to consider how **symmetric keys** are risky to communicate to your collaborators *directly*, as we'll illustrate in the following.

## Why Public-key Cryptography?

We will be using [public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography), aka **asymmetric cryptography**, so that you won't have to be concerned with "*dessiminating your secret symmetric key*" to everyone you want to talk secretly with. That is, you won't ever have to "*meet your collaborators in dark alleys*" to agree on a **symmetric key**.

You can imagine such a **symmetric key** as having a one-to-one **substitution** dictionary. It would translate [ciphertext](https://en.wikipedia.org/wiki/Ciphertext) into [plaintext](https://en.wikipedia.org/wiki/Plaintext), like "*eagle*" into "*target*" and "*ruffled*" into "*arrived*", so your secret agents will know when "*the eagle is ruffled*". In its simplest form, it would possibly translate the alphabet '*A*' into '*B*', '*B*' into '*C*', and so on.

Historically, the transport/delivery of a **symmetric key** is the weakest link in cryptography. The person(s) transporting said key(s) can easily be intercepted. Phone lines can be tapped. Tom Cruise (à la *Mision Impossible*) could wear a mask and appear exactly like your friendly university professor. The list of possible *key-loss scenarios* goes on.

## OpenSSL and LibreSSL

We use OpenSSL, the man(ual) pages of which are consolidated [on OpenSSL's official site](https://www.openssl.org/docs/manmaster/man1/).

The actual implementation of OpenSSL we will use is [LibreSSL](https://www.libressl.org). Current version of this writing is `2.9.1`.

    openssl version
    # Should show 'LibreSSL 2.9.1'.

## Generate Key-Pair in PEM Format

    openssl genpkey \
      -algorithm RSA \
      -out key.pem \
      -aes-256-cbc \
      -pass stdin

Command `openssl genpkey` generates a private key.

Option `-algorithm RSA` chooses the RSA algorithm for key-pair generation; there's no need to understand why `RSA` is used here.

Option `-out key.pem` outputs the generated key-pair to a file named `key.pem`. PEM is the default output format, and is the format we want. You might want to use your key to send encrypted emails later, which is why it's best to stick to the PEM format now.

Option `-aes-256-cbc` encrypts the output key-pair using cipher *AES* with key length of *256 bits* and mode of operation *Cipher Block Chaining* (CBC). There's no real need to understand any of that, except that `aes-256-cbc` is the easiest and safest cipher to use. If you're interested to know what ciphers (aka *cryptography algorithms*) are supported, feel free to query such:

    openssl list-cipher-algorithms

Option `-pass stdin` tells `openssl` to perform the encryption of the key-pair using the passphrase you will enter via `stdin` (keyboard).

TODO: Should we explain that passing in the password on the command-line (eg `-pass pass:my_password`) will leak the password to a `ps` snoop? Or just leave it as an instruction without further messy explanations?

# Sending Encrypted Messages

Create a sample message:

    mkdir ~/temp
    cd ~/temp
    echo "Hello Yellow!" > data

## PEM Key Format

Use PEM key format. The format is Base64 encoded, for easy transport over text-based channels. In any case, you would eventually install your public-private key into your email system, which requires this key format.

**PEM** stands for **Privacy Enhanced Mail**.

Export your public and private key:

    openssl rsa -in id_rsa -outform pem > id_rsa.pem

    openssl rsa -in id_rsa -pubout -outform pem > id_rsa.pub.pem

## Generate an Encryption Key

Do not use your recipient's *public key* to directly encrypt the message you send.

**Reduce Key-Use Profile.** Encrypting a lot of data (say 1 year's worth of voluminous messages) with said *public key* will present a bigger profile for hackers to crack that *public key*. Granted, you will normally salt all encryptions, but it's still best to provide as little material for dictionary attacks (and similar) as possible.

**Generate a random encryption key** for each message you send. Generate one now.

    openssl rand -base64 32 > data.key

## Encrypt the Encryption Key

Encrypt the *encryption key* with your *public key*. This simulates you encrypting a secret (the *encryption key*, in this case) *with your recipient's public key* to send to your recipient. You would normally encrypt with your recipient's *public key*.

    openssl rsautl -encrypt \
      -inkey id_rsa.pub.pem -pubin \
      -in data.key -out data.key.enc

**Default Input Key Type.** Of special note is the `-inkey <public-key.pem> -pubin` phrase. It indicates that the input key file is a public key. By default, [the command](https://www.openssl.org/docs/manmaster/man1/rsautl.html) `openssl rsautl` expects the input key file to be a private key.

TODO: Use `pkeyutl` instead.

## Encrypt the Message

Now, we encrypt the secret message with our *encryption key*.

    openssl enc -aes-256-cbc -salt \
      -in data -out data.enc \
      -pass file:./data.key

We will now refer to [the man page](https://www.openssl.org/docs/manmaster/man1/enc.html) for `openssl enc`.

**Cipher to Use.** As per the man page, the easiest and safest cipher to use is the [AES block cipher](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) in [CBC mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_Block_Chaining_(CBC)) using the longest key length of 256 bits.

**Salting** is required to prevent dictionary attacks.

**Password** is not needed; our argument to `-pass` is a file. We use the *encryption key* as "*password*", which is a lot more secure than a password that is humanly possible to type.

# Receiving Encrypted Messages

As per PKI, the recipient will use her private key for decryption.

## Decrypt the Encryption Key

Use the private key to decrypt the *encryption key*. Note that the *encryption key* is a *symmetric key*, so it acts as both an *encryption key* as well as a *decryption key*.

    openssl rsautl -decrypt \
      -inkey id_rsa.pem \
      -in data.key.enc -out received.key

TODO: Use `pkeyutl` instead.

## Decrypt the Message

Use the *encryption key* to decrypt the encrypted message.

    openssl enc -d -aes-256-cbc \
      -in data.enc -out received.data \
      -pass file:./received.key

---

# Never Encrypt Voluminous Data with Public Key

Even though the *deterministic* RSA algorithm has been made *probabilistic* with pseudorandom padding (**PKCS 1.5**), we should still not assume that attacks will not be invented in the future.

(*Deterministic* in this case simply means `some-same-gibberish` always decrypts into `some-same-plaintext`.)

Hence, we should never encrypt actual data with our public key, especially when such data has **predictable patterns**. Natural language has lots of predictable patterns, and so does numerous file or data formats.

---

TODO: Use "*known-plaintext attack*" to explain why we should not use the public key to encrypt secret messages.

TODO: Mention the Claude Shannon's "*confusion and diffusion*"? Will that require too much logic skills from readers?

TODO: Mention deterministc RSA ("*Textbook RSA*") and padding? Theoretically, even the the OAEP-padded *PKCS encryption/decryption scheme* (**PKCS 2.0**) is susceptible to attack, let alone the default padding (**PKCS 1.5**) used by `openssl`. Mention that we can "*never be too sure that PKCS 1.5 or 2.0 won't ever be attacked*" by *Chosen Ciphertext Attack*, or worse, by *Chose/Known Plaintext Attack*?

[Manger, 2001](https://link.springer.com/content/pdf/10.1007/3-540-44647-8_14.pdf). *Chosen Ciphertext Attack* on PKCS 2.0.

[Bleichenbacher, 1998](https://link.springer.com/content/pdf/10.1007%2FBFb0055716.pdf). *Chosen Ciphertext Attack* on PKCS 1.5.

TODO: Mention that **PKCS 1.5** padding cannot be attacked without *oracles*? Explain *oracles*? Seems too deep into the rabbit hole. Need to find an easier way to explain why "*PKCS 1.5 is secure for offline encryption/decryption*".

TODO: I think it's best we don't go anywhere near **semantic security** (aka **IND-CPA**). Maybe we can explain more cybersecurity concepts *in future installments* if interested readers ask for more.
