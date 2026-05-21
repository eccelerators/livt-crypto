# Test Vector Verification

The authoritative test vectors live in the `tests/` components and are executed
by `livt test`. This document keeps optional, human-facing notes for checking
selected vectors with external tools such as OpenSSL.

These checks are useful when updating or reviewing tests, but they should not be
required for normal package builds.

## AES-256 ECB Vectors

NIST FIPS 197 Appendix C.3:

```bash
echo "00112233445566778899aabbccddeeff" | xxd -r -p \
  | openssl enc -aes-256-ecb \
      -K 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f \
      -nopad 2>/dev/null | xxd -p
# Expected: 8ea2b7ca516745bfeafc49904b496089
```

NIST SP 800-38A Appendix F.1.5, block 1:

```bash
echo "6bc1bee22e409f96e93d7e117393172a" | xxd -r -p \
  | openssl enc -aes-256-ecb \
      -K 603deb1015ca71be2b73aef0857d77811f352c073b6108d72d9810a30914dff4 \
      -nopad 2>/dev/null | xxd -p
# Expected: f3eed1bdb5d2a03c064b5a7e3db181f8
```

NIST SP 800-38A Appendix F.1.5, block 2:

```bash
echo "ae2d8a571e03ac9c9eb76fac45af8e51" | xxd -r -p \
  | openssl enc -aes-256-ecb \
      -K 603deb1015ca71be2b73aef0857d77811f352c073b6108d72d9810a30914dff4 \
      -nopad 2>/dev/null | xxd -p
# Expected: 591ccb10d410ed26dc5ba74a31362870
```

NIST SP 800-38A Appendix F.1.5, block 3:

```bash
echo "30c81c46a35ce411e5fbc1191a0a52ef" | xxd -r -p \
  | openssl enc -aes-256-ecb \
      -K 603deb1015ca71be2b73aef0857d77811f352c073b6108d72d9810a30914dff4 \
      -nopad 2>/dev/null | xxd -p
# Expected: b6ed21b99ca6f4f9f153e7b1beafed1d
```

NIST SP 800-38A Appendix F.1.5, block 4:

```bash
echo "f69f2445df4f9b17ad2b417be66c3710" | xxd -r -p \
  | openssl enc -aes-256-ecb \
      -K 603deb1015ca71be2b73aef0857d77811f352c073b6108d72d9810a30914dff4 \
      -nopad 2>/dev/null | xxd -p
# Expected: 23304b7a39f9f3ff067d8d8f9e24ecc7
```

All-zero AES-256 key and plaintext:

```bash
echo "00000000000000000000000000000000" | xxd -r -p \
  | openssl enc -aes-256-ecb \
      -K 0000000000000000000000000000000000000000000000000000000000000000 \
      -nopad 2>/dev/null | xxd -p
# Expected: dc95c078a2408989ad48a21492842087
```

## S-box Values

The S-box boundary values in `AesSBoxTest` come directly from NIST FIPS 197
Table 4. The inverse S-box boundary values in `AesSBoxInverseTest` come directly
from NIST FIPS 197 Table 6.

These table boundary checks are not independently verifiable through OpenSSL's
command-line interface.
