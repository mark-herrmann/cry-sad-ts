# cry-sad-ts

Cryptomator's crypto architecture, simplified and decoupled, written in TypeScript.

Makes data thieves sad and makes them cry because they can't do anything with your data anymore, which is good for you.

## Introduction

### Cryptomator

With [Cryptomator](https://cryptomator.org/), the key to your data is in your hands. Cryptomator encrypts your data quickly and easily. Afterwards you upload them protected to your favorite cloud service. \
It creates on-the-fly encrypted filesystems, physically stored in a directory, called vault. \
[Cryptomator Website](https://cryptomator.org/) 

### This package

cry-sad-ts is an simplfied adaption. \
It only consists of the crypto-architecture components from cryptomator without filesystem features or any gui.
Encryption and decryption is done on buffers.
A further simplification is a simple deterministic string encryption instead of cryptomator's directory id handling and filename encryption.

## Features

* Creation of encrypted vault
* Password changes
* Buffer encryption and decryption
* Partial buffer decryption
* Deterministic string encryption and decryption

## Installation
```bash
npm i sad-cry-ts
```

## Usage

!TBD

## Crypto Architecture

### Vault Configuration

Every vault has a vault configuration. It is a JWT containing basic information about the vault and specification what key to use. The JWT is signed using the 512 bit raw masterkey (encryptionMasterKey and macMasterKey concatenated). When you create a new vault, this JWT is created for you.

This is an example of an encoded vault configuration JWT:
```
eyJraWQiOiJtYXN0ZXJrZXlmaWxlOm1hc3RlcmtleS5jcnlwdG9tYXRvciIsInR5cCI6IkpXVCIsImFsZyI6IkhTMjU2In0.eyJmb3JtYXQiOjgsInNob3J0ZW5pbmdUaHJlc2hvbGQiOjIyMCwianRpIjoiY2U5NzZmN2EtN2I5Mi00Y2MwLWI0YzEtYzc0YTZhYTE3Y2Y1IiwiY2lwaGVyQ29tYm8iOiJTSVZfQ1RSTUFDIn0.IJlu4dHb3fqB2fAk9lf8G8zyEXc7OLB-5m9aNxOEXIQ
```

The decoded header:
```json
{
  "kid": "type:object;name:masterKeyObject",
  "typ": "JWT",
  "alg": "HS256"
}
```
`"kid": "type:object;name:masterKeyObject"` means, that masterKey info is retrieved from function parameter called masterKeyObject as object. \
Current implementation only supports HS256 as signature algorithm.

The decoded payload:
```json
{
  "format": 8,
  "jti": "ce976f7a-7b92-4cc0-b4c1-c74a6aa17cf5",
  "cipherCombo": "SIV_CTRMAC"
}
```
`"format": 8`: vault format for checking software compatibility \
`"jti": "ce976f7a-7b92-4cc0-b4c1-c74a6aa17cf5"`: random UUID to uniquely identify the vault \
Currently SIV_CTRMAC is the only supported cipherCombo.

When opening a vault, the following steps are done:
* Decode the JWT without verification
* Read kid header and, depending on its value, retrieve the masterKeyObject from the specified function parameter
* Verify the JWT signature using the masterkey, retrieved from masterKeyObject and unwrapped,
  using the password (See: [Masterkey Derivation](#masterkey-derivation))
* Make sure format and cipherCombo are supported

### Masterkey Derivation

Each vault has its own 256 bit encryption as well as MAC masterkey used for encryption of buffer specific keys and buffer authentication, respectively.

These keys are random sequences generated by a CSPRNG. We use random from node-forge.

Both keys are encrypted using RFC 3394 key wrapping (also known as AES-KeyWrap) with a KEK (key-encryption-key) derived from the password using scrypt.
```
encryptionMasterKey := createRandomBytes(32)
macMasterKey := createRandomBytes(32)
kek := scrypt(password, scryptSalt, scryptCostParam, scryptBlockSize)
wrappedEncryptionMasterKey := aesKeyWrap(encryptionMasterKey, kek)
wrappedMacMasterKey := aesKeyWrap(macMasterKey, kek)
```

The wrapped keys and the parameters needed to derive the KEK are then stored as integers or Base64-encoded strings in the masterKeyObject.
```json
{
    "scryptSalt": "QGk...jY=",
    "scryptCostParam": 16384,
    "scryptBlockSize": 8,
    "primaryMasterKey": "QDi...Q==",
    "hmacMasterKey": "L83...Q==",
}
```
`"primaryMasterKey": "QDi...Q=="`: wrappedEncryptionMasterKey (shortened in this example) \
`"hmacMasterKey": "L83...Q=="`: wrappedMacMasterKey (shortened in this example)

When unlocking a vault the KEK is used to unwrap (i.e. decrypt) the stored masterkeys.
```
kek := scrypt(password, scryptSalt, scryptCostParam, scryptBlockSize)
encryptionMasterKey := aesKeyUnwrap(wrappedEncryptionMasterKey, kek)
macMasterKey := aesKeyUnwrap(wrappedMacMasterKey, kek)
```

### Buffer Encryption

Buffers are encrypted as follows.

#### Buffer Header Encryption

The buffer header stores certain metadata, which is needed for buffer content encryption. It consists of 88 bytes.
* 16 bytes nonce (random per buffer encryption) used during header payload encryption
* 40 bytes AES-CTR encrypted payload consisting of:
  * 8 bytes filled with ones for future use
  * 32 bytes buffer content key
* 32 bytes header MAC of the previous 56 bytes

```
headerNonce := createRandomBytes(16)
contentKey := createRandomBytes(32)
cleartextPayload := 0xFFFFFFFFFFFFFFFF . contentKey
ciphertextPayload := aesCtr(cleartextPayload, encryptionMasterKey, headerNonce)
mac := hmacSha256(headerNonce . ciphertextPayload . associatedData, macMasterKey)
```

#### Buffer Content Encryption

This is where your actual buffer contents get encrypted.

The cleartext is broken down into multiple chunks, each up to 32 KiB + 48 bytes consisting of:
* 16 bytes chunk nonce (random per buffer encryption)
* up to 32 KiB encrypted payload using AES-CTR with the buffer content key
* 32 bytes MAC of:
  * buffer header nonce (to bind this chunk to the buffer header)
  * chunk number as 8 byte big endian integer (to prevent undetected reordering)
  * chunk nonce
  * encrypted payload

Afterwards, the encrypted chunks are joined preserving the order of the cleartext chunks. The payload of the last chunk may be smaller than 32 KiB.

```
cleartextChunks[] := split(cleartext, 32KiB)
for (int i = 0; i < length(cleartextChunks); i++) {
    chunkNonce := createRandomBytes(16)
    ciphertextPayload := aesCtr(cleartextChunks[i], contentKey, chunkNonce)
    mac := hmacSha256(headerNonce . bigEndian(i) . chunkNonce . ciphertextPayload, macMasterKey)
    ciphertextChunks[i] := chunkNonce . ciphertextPayload . mac
}
ciphertextBufferContent := join(ciphertextChunks[])
```

### Deterministic string encryption

String are encrypted with the encryptionMasterKey and macMasterKey, using AES-SIV, and then base64-encoded, using base64_url_variant.
```
encryptedString := base64Url(aesSiv(string, encryptionMasterKey, macMasterKey))
```

Deterministic in this context means, that the same string always encrypts to same result, using the same keys. \
Normally, the often randomly generated IV results in the same plaintext being encrypted to a different ciphertext each time. With SIV-Mode (SIV = synthetic IV), however, the IV is derived from the plaintext, so the result is the same every time. \
This can be useful for encryption of filenames, paths and the like.

## Used libraries and self-implementations

!TBD

## Differences to cryptomator

* MasterKey and key derivation information and JWT is not saved in files, but returned and passed as object and string
* Encyption works on buffers instead of files
* Directory management and filename encryption is replaced with simplified deterministic string encryption

## Known security issues
* An attacker could swap the contents of two encrypted buffers without any detection. \
  Example: Assuming two encrypted buffers where saved into two files called file1 and file2.
  An attacker could swap the filenames, so file1 contains the second buffer content and file2 contains first buffer content. \
  This can be exploited if following conditions are met:
  * the attacker has write access to the storage
  * the user stores data, that is meant to be published or shared, in the same vault alongside secret data
  * the user publishes content of decrypted directories without checking (e.g. huge vaults, missing clarity)
  
  A fix (passing buffer-associated data on buffer header mac creation) could lead to performance issues. \
  Example: Assuming the example above, if the user renames a file, the header mac must be rewritten.
  If the file is saved online this can require complete file download and upload (download, change header mac, upload). \
  The same [issue exists in Cryptomator](https://github.com/cryptomator/cryptomator/issues/119). \
  We and the Cryptomator team decided (independently) to define this risk as accepted, because an exploit is hard due to the conditions, and the fix would cause unacceptable I/O performance issues.

## License

This project is licensed under the MIT License. \
[See License](LICENSE.txt)
