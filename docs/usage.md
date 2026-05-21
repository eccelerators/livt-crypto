# Usage Examples

`Livt.Crypto` exposes fixed-size primitives. These examples show call order and
state handling for common one-block workflows. They are intentionally small:
production designs should wrap these primitives with the padding, framing,
nonce management, and streaming policy that fits the larger system.

The snippets are shaped like code inside a Livt component or test component.
Declare the crypto component as a field, instantiate it in `new()`, then call
the public methods in the order shown.

## AES-CTR: Encrypt and Decrypt One Block

CTR mode uses the same operation for encryption and decryption. Reuse the same
key and starting counter to recover the plaintext.

```livt
namespace Example
using Livt.Crypto.Aes

component AesCtrExample
{
    ctr: Ctr128

    new()
    {
        this.ctr = new Ctr128()
    }

    public fn Run()
    {
        var key: byte[16] = [
            0x2b, 0x7e, 0x15, 0x16, 0x28, 0xae, 0xd2, 0xa6,
            0xab, 0xf7, 0x15, 0x88, 0x09, 0xcf, 0x4f, 0x3c]
        var counter: byte[16] = [
            0xf0, 0xf1, 0xf2, 0xf3, 0xf4, 0xf5, 0xf6, 0xf7,
            0xf8, 0xf9, 0xfa, 0xfb, 0xfc, 0xfd, 0xfe, 0xff]
        var plaintext: byte[16] = [
            0x6b, 0xc1, 0xbe, 0xe2, 0x2e, 0x40, 0x9f, 0x96,
            0xe9, 0x3d, 0x7e, 0x11, 0x73, 0x93, 0x17, 0x2a]

        this.ctr.SetKey(key)
        this.ctr.SetCounter(counter)
        this.ctr.LoadBlock(plaintext)
        this.ctr.CryptBlock()
        var ciphertext: byte[16] = this.ctr.GetBlock()

        this.ctr.SetCounter(counter)
        this.ctr.LoadBlock(ciphertext)
        this.ctr.CryptBlock()
        var recovered: byte[16] = this.ctr.GetBlock()
    }
}
```

## AES-GCM: Encrypt, Tag, Verify, Decrypt

GCM authenticates ciphertext. After decryption, do not trust the plaintext until
`VerifyTag` returns `1`.

```livt
namespace Example
using Livt.Crypto.Aead

component AesGcmExample
{
    gcm: Aes128Gcm

    new()
    {
        this.gcm = new Aes128Gcm()
    }

    public fn Run()
    {
        var key: byte[16] = [
            0xfe, 0xff, 0xe9, 0x92, 0x86, 0x65, 0x73, 0x1c,
            0x6d, 0x6a, 0x8f, 0x94, 0x67, 0x30, 0x83, 0x08]
        var nonce: byte[12] = [
            0xca, 0xfe, 0xba, 0xbe, 0xfa, 0xce, 0xdb, 0xad,
            0xde, 0xca, 0xf8, 0x88]
        var plaintext: byte[16] = [
            0xd9, 0x31, 0x32, 0x25, 0xf8, 0x84, 0x06, 0xe5,
            0xa5, 0x59, 0x09, 0xc5, 0xaf, 0xf5, 0x26, 0x9a]

        this.gcm.SetKey(key)
        this.gcm.SetNonce(nonce)
        this.gcm.LoadBlock(plaintext)
        this.gcm.EncryptBlock()
        this.gcm.FinalizeTag(0, 16)
        var ciphertext: byte[16] = this.gcm.GetBlock()
        var tag: byte[16] = this.gcm.GetTag()

        this.gcm.SetNonce(nonce)
        this.gcm.LoadBlock(ciphertext)
        this.gcm.DecryptBlock()
        this.gcm.FinalizeTag(0, 16)
        var ok: int = this.gcm.VerifyTag(tag)
        var recovered: byte[16] = this.gcm.GetBlock()
    }
}
```

For AAD, call `FeedAAD(aadBlock)` after `SetNonce` and before data blocks, then
pass the real AAD byte length to `FinalizeTag(aadLenBytes, dataLenBytes)`.

## SHA-256: Hash One Padded Block

`Sha256` processes complete 64-byte blocks. The caller supplies SHA-256 padding.

```livt
namespace Example
using Livt.Crypto.Hash

component Sha256Example
{
    sha: Sha256

    new()
    {
        this.sha = new Sha256()
    }

    public fn Run()
    {
        // SHA-256("abc") padded to one 64-byte block.
        var block: byte[64] = [
            0x61, 0x62, 0x63, 0x80, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x18]

        this.sha.Reset()
        this.sha.ProcessBlock(block)
        var digest: byte[32] = this.sha.GetDigest()
    }
}
```

## HMAC-SHA-256: One Message Block

`HmacSha256` hashes one outer block internally after each `ProcessBlock`.
Provide the second inner block after `Init()`.

```livt
namespace Example
using Livt.Crypto.Mac

component HmacSha256Example
{
    hmac: HmacSha256

    new()
    {
        this.hmac = new HmacSha256()
    }

    public fn Run()
    {
        // RFC 4231 test case 1: key = 20 x 0x0b, data = "Hi There".
        var key: byte[32] = [
            0x0b, 0x0b, 0x0b, 0x0b, 0x0b, 0x0b, 0x0b, 0x0b,
            0x0b, 0x0b, 0x0b, 0x0b, 0x0b, 0x0b, 0x0b, 0x0b,
            0x0b, 0x0b, 0x0b, 0x0b, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]
        var msg: byte[64] = [
            0x48, 0x69, 0x20, 0x54, 0x68, 0x65, 0x72, 0x65,
            0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02, 0x40]

        this.hmac.SetKey(key)
        this.hmac.Init()
        this.hmac.ProcessBlock(msg)
        var tag: byte[32] = this.hmac.GetHmac()
    }
}
```

## ChaCha20-Poly1305: Encrypt and Verify One Block

`ChaCha20Poly1305` processes a full 64-byte data block. Partial final data
blocks need a higher-level adapter.

```livt
namespace Example
using Livt.Crypto.Aead

component ChaCha20Poly1305Example
{
    aead: ChaCha20Poly1305

    new()
    {
        this.aead = new ChaCha20Poly1305()
    }

    public fn Run()
    {
        var key: byte[32] = [
            0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
            0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f,
            0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17,
            0x18, 0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x1f]
        var nonce: byte[12] = [
            0x00, 0x00, 0x00, 0x09, 0x00, 0x00,
            0x00, 0x4a, 0x00, 0x00, 0x00, 0x00]
        var plaintext: byte[64] = [
            0x4c, 0x61, 0x64, 0x69, 0x65, 0x73, 0x20, 0x61,
            0x6e, 0x64, 0x20, 0x47, 0x65, 0x6e, 0x74, 0x6c,
            0x65, 0x6d, 0x65, 0x6e, 0x20, 0x6f, 0x66, 0x20,
            0x74, 0x68, 0x65, 0x20, 0x63, 0x6c, 0x61, 0x73,
            0x73, 0x20, 0x6f, 0x66, 0x20, 0x27, 0x39, 0x39,
            0x3a, 0x20, 0x44, 0x69, 0x64, 0x20, 0x79, 0x6f,
            0x75, 0x20, 0x6b, 0x6e, 0x6f, 0x77, 0x20, 0x74,
            0x68, 0x61, 0x74, 0x20, 0x79, 0x6f, 0x75, 0x20]

        this.aead.SetKey(key)
        this.aead.SetNonce(nonce)
        this.aead.LoadBlock(plaintext)
        this.aead.EncryptBlock()
        this.aead.FinalizeTag(0, 64)
        var ciphertext: byte[64] = this.aead.GetBlock()
        var tag: byte[16] = this.aead.GetTag()

        this.aead.SetNonce(nonce)
        this.aead.LoadBlock(ciphertext)
        this.aead.DecryptBlock()
        this.aead.FinalizeTag(0, 64)
        var ok: int = this.aead.VerifyTag(tag)
        var recovered: byte[64] = this.aead.GetBlock()
    }
}
```

## Where to Look Next

The executable test components in `tests/` contain the authoritative vectors and
expected outputs. Use this guide for call order, then consult the matching test
file when you need exact bytes for a standard vector.
