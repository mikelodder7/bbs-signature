%%%
title = "The BBS Signature Scheme"
abbrev = "The BBS Signature Scheme"
ipr= "none"
area = "Internet"
workgroup = "none"
submissiontype = "IETF"
keyword = [""]

[seriesInfo]
name = "Individual-Draft"
value = "draft-bbs-signatures-latest"
status = "informational"

[[author]]
initials = "M."
surname = "Lodder"
fullname = "Mike Lodder"
#role = "editor"
organization = "CryptID"
  [author.address]
  email = "redmike7gmail.com"

[[author]]
initials = "T."
surname = "Looker"
fullname = "Tobias Looker"
#role = "editor"
organization = "Mattr"
  [author.address]
  email = "tobias.looker@mattr.global"

[[author]]
initials = "A."
surname = "Whitehead"
fullname = "Andrew Whitehead"
#role = "editor"
organization = ""
  [author.address]
  email = "cywolf@gmail.com"
%%%

.# Abstract

BBS is a form of short group digital signature scheme that supports multi-message signing that produces a single output digital signature. The scheme allows a possessor of a signature to derive proofs that selectively reveal from the originally signed set of messages, whilst preserving verifiable authenticity and integrity of the messages. Derived proofs are said to be zero-knowledge in nature as they do not reveal the underlying signature, instead proof of knowledge of the signature.

{mainmatter}

# Introduction

A digital signature scheme is a fundamental cryptographic primitive that is used to provide data integrity and verifiable authenticity in various protocols. The core premise of digital signature technology is built upon asymmetric cryptography where-by the possessor of a private key is able to sign a payload (often revered to as the signer), where anyone in possession of the corresponding public key matching that of the private key is able to verify the signature.

However traditional digital signatures are limited to fixed mode of signing and verifying, that is the entire payload that was signed by a signer must be known by the verifier in-order to validate the digital signature.

This document describes the BBS signature scheme. (FIXME: THE BBS signature scheme? Or A BBS signature schemes? Also: BBS citation needed/informative reference).
The scheme feature important properties that allow the scheme to be used in applications where privacy and data minimization techniques are desired and/or required:

1. Signatures can be created blinded or un-blinded.

2. Traditional signature schemes require the entire signature and message to be disclosed during verification. BBS allows a fast and small zero-knowledge signature proof of knowledge to be created from the signature and the public key. This allows the signature holder to selectively reveal any number of signed messages to another entity (none, all, or any number in between).

A recent emerging use case applies signature schemes in [verifiable credentials](https://www.w3.org/TR/vc-data-model/). One problem with
using simple signature schemes like ECDSA or ED25519 is that a holder must disclose the entire signed message and signature for verification. Circuit based logic can be applied to verify these in zero-knowledge like SNARKS or Bulletproofs with R1CS but tend to be complicated. BBS on the other hand adds, to verifiable credentials or any other application, the ability to do very efficient zero-knowledge proofs. A holder gains the ability to choose which claims to reveal to a relying party without the need for any additional complicated logic. (FIXME: Informative references missing)

## Notational Conventions

The keywords **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**,
**SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL**, when they appear in this
document, are to be interpreted as described in [@!RFC2119].

## Terminology

The following terminology is used throughout this document:

SK
: The secret key for the signature scheme.

PK
: The public key for the signature scheme.

U
: The set of messages that are blinded from the signer during a blind signing.

K
: The set of messages that are known to the signer during a blind signing.

L
: The total number of messages that the signature scheme can sign.

R
: The set of message indices that are retained or hidden in a signature proof of knowledge.

D
: The set of message indices that are disclosed in a signature proof of knowledge.

msg
: The input to be signed by the signature scheme.

h\[i\]
: The generator corresponding to a given msg.

h0
: A generator for the blinding value in the signature.

signature
: The digital signature output.

s'
: The signature blinding factor held by the signature recipient.

blind\_signature
: The blind digital signature output.

commitment
: A pedersen commitment composed of 1 or more messages.

nonce
: A cryptographic nonce

presentation_message (pm)
: A message generated and bound to the context of a specific spk.

spk
: Zero-Knowledge Signature Proof of Knowledge.

nizk
: A non-interactive zero-knowledge proof from fiat-shamir heuristic.

dst
: The domain separation tag.

I2OSP
: As defined by Section 4 of [@!RFC8017]

OS2IP
: As defined by Section 4 of [@!RFC8017].

a || b
: Denotes the concatenation of octet strings a and b.

Terms specific to pairing-friendly elliptic curves that are relevant to this document are restated below, originally defined in [@!I-D.irtf-cfrg-pairing-friendly-curves]

E1, E2
: elliptic curve groups defined over finite fields. This document assumes that E1 has a more compact representation than E2, i.e., because E1 is defined over a smaller field than E2.

G1, G2
: subgroups of E1 and E2 (respectively) having prime order r.

GT
: a subgroup, of prime order r, of the multiplicative group of a field extension.

e
: G1 x G2 -> GT: a non-degenerate bilinear map.

P1, P2
: points on G1 and G2 respectively. For a pairing-friendly curve, this document denotes operations in E1 and E2 in additive notation, i.e., P + Q denotes point addition and x \* P denotes scalar multiplication. Operations in GT are written in multiplicative notation, i.e., a \* b is field multiplication.

hash\_to\_curve\_g1(ostr) -> P
: The cryptographic hash function that takes as an arbitrary octet string input and returns a point in G1 as defined in [@!I-D.irtf-cfrg-hash-to-curve]. The algorithm first requires selection of the pairing friendly curve and digest
algorithm, once selected apply the isogeny simplified SWU map to compute a point in G1 using the random oracle method. The domain separation tag value is dst.

hash\_to\_curve\_g2(ostr) -> P
: The cryptographic hash function that takes as an arbitrary octet string input and returns a point in G2 as defined in [@!I-D.irtf-cfrg-hash-to-curve]. The algorithm first requires selection of the pairing friendly curve and digest algorithm, once selected the isogeny simplified SWU map to compute a point in G2 using the random oracle method. The domain separation tag value is dst.

point\_to\_octets\_min(P) -> ostr
: returns the canonical representation of the point P as an octet string in compressed form. This operation is also known as serialization. (FIXME: This either requires a normative wire format or a reference to the normative wire format)

point\_to\_octets\_norm(P) -> ostr
: returns the canonical representation of the point P as an octet string in uncompressed form. This operation is also known as serialization. (FIXME: This either requires a normative wire format or a reference to the normative wire format)


octets\_to\_point(ostr) -> P
: returns the point P corresponding to the canonical representation ostr, or INVALID if ostr is not a valid output of point\_to\_octets.  This operation is also known as deserialization. Consumes either compressed or uncompressed ostr.

subgroup\_check(P) -> VALID or INVALID
: returns VALID when the point P is an element of the subgroup of order r, and INVALID otherwise. This function can always be implemented by checking that r \* P is equal to the identity element.  In some cases, faster checks may also exist, e.g., [Bowe19].

# Overview

//TODO

## Organization of this document

This document is organized as follows:

* The remainder of this section defines terminology and the high-level API.

* Section 2 defines primitive operations used in the BLS signature scheme. These operations MUST NOT be used alone.

* Section 3 defines security considerations.

* Section 4 defines the references.

* Section 5 defines test vectors.

# Core operations

This section defines core operations used by the schemes defined in Section 3. These operations MUST NOT be used except as described in that section.

## Parameters

The core operations in this section depend on several parameters:

* A pairing-friendly elliptic curve, plus associated functionality given in Section 1.4.

* H, a hash function that MUST be a secure cryptographic hash function. For security, H MUST output at least ceil(log2(r)) bits, where r is the order of the subgroups G1 and G2 defined by the pairing-friendly elliptic curve.

* PRF(n): a pseudo-random function similar to [@!RFC4868]. Returns n pseudo randomly generated bytes.

## KeyGen

The KeyGen algorithm generates a secret key SK deterministically from a secret octet string IKM.

KeyGen uses a HKDF [@!RFC5869] instantiated with the hash function H.

For security, IKM MUST be infeasible to guess, e.g., generated by a trusted source of randomness.

IKM MUST be at least 32 bytes long, but it MAY be longer.

Because KeyGen is deterministic, implementations MAY choose either to store the resulting SK or to store IKM and call KeyGen to derive SK when necessary.

KeyGen takes an optional parameter, key\_info. This parameter MAY be used to derive multiple independent keys from the same IKM.  By default, key\_info is the empty string.

```
SK = KeyGen(IKM)
```

Inputs:

- IKM, a secret octet string. See requirements above.

Outputs:

- SK, a uniformly random integer such that 0 < SK < r.

Parameters:

- key\_info, an optional octect string. if this is not supplied, it MUST default to an empty string.

Definitions:

- HKDF-Extract is as defined in [@!RFC5869], instantiated with hash H.
- HKDF-Expand is as defined in [@!RFC5869], instantiated with hash H.
- I2OSP and OS2IP are as defined in [@!RFC8017], Section 4.
- L is the integer given by ceil((3 \* ceil(log2(r))) / 16).
- "BBS-SIG-KEYGEN-SALT-" is an ASCII string comprising 20 octets.

Procedure:

1. PRK = HKDF-Extract("BBS-SIG-KEYGEN-SALT-", IKM || I2OSP(0, 1))

2. OKM = HKDF-Expand(PRK, key\_info || I2OSP(L, 2), L)

3. SK = OS2IP(OKM) mod r

4. return SK

## SkToPk

SkToPk algorithm takes a secret key SK and outputs a corresponding public key.

```
PK = SkToPk(SK)
```

Inputs:

- SK, a secret integer such that 0 <= SK <= r

Outputs:

- PK, a public key encoded as an octet string

Procedure:

1. w = SK \* P2

2. PK = w

3. return point\_to\_octets\_min(PK)

## KeyValidate

KeyValidate checks if the public key is valid.

As an optimization, implementations MAY cache the result of KeyValidate in order to avoid unnecessarily repeating validation for known keys.

```
result = KeyValidate(PK)
```

Inputs:

- PK, a public key in the format output by SkToPk.

Outputs:

- result, either VALID or INVALID

Procedure:

1. (w, h0, h) = octets\_to\_point(PK)

2. result = subgroup\_check(w) && subgroup\_check(h0)

3. for i in 0 to len(h): result &= subgroup\_check(h\[i\])

4. return result

## Sign

Sign computes a signature from SK, PK, over a vector of messages.

```
signature = Sign((msg[i],...,msg[L]), SK, PK)
```

Inputs:

- msg\[i\],...,msg\[L\], octet strings
- SK, a secret key output from KeyGen
- PK, a public key output from SkToPk

Outputs:

- signature, an octet string

Procedure:

1. (w, h0, h) = octets\_to\_point(PK)

2. e = H(PRF(8*ceil(log2(r)))) mod r

3. s = H(PRF(8*ceil(log2(r)))) mod r

4. b = P1 + h0 \* s + h\[i\] \* msg\[i\] + ... + h\[n\] \* msg\[L\]

5. A = b \* (1 / (SK + e))

6. signature = (point\_to\_octets\_min(A), e, s)

7. return signature

## Verify

Verify checks that a signature is valid for the octet string messages under the public key.

```
result = Verify((msg[i],...,msg[L]), signature, PK)
```

Inputs:

- msg\[i\],...,msg\[L\], octet strings.
- signature, octet string.
- PK, a public key in the format output by SkToPk.

Outputs:

- result, either VALID or INVALID.

Procedure:

1. (A, e, s) = octets\_to\_point(signature.A), e, s

2. pub\_key = octets\_to\_point(PK)

3. if subgroup\_check(A) is INVALID

4. if KeyValidate(pub\_key) is INVALID

5. b = P1 + h0 \* s + h\[i\] \* msg\[i\] + ... + h\[n\] \* msg\[L\]

6. C1 = e(A, w + P2 \* e)

7. C2 = e(b, P2)

8. return C1 == C2

## PreBlindSign

The PreBlindSign algorithm allows a holder of a signature to blind messages that when signed, are unknown to the signer.

The algorithm returns a generated blinding factor that is used to un-blind the signature from the signer, and a pedersen commitment from a vector of messages and the domain parameters h and h0.

```
(s', commitment) = PreBlindSign((msg[1],...,msg[U]), CGIdxs)
```

Inputs:

- msg\[1\],...,msg\[U\], octet strings of the messages to be blinded.
- CGIdxs, vector of unsigned integers. Indices of the generators from the domain parameter h, used for the messages to be blinded.

Outputs:

- s', octet string.
- commitment, octet string

Procedure:

1. (i1,...,iU) = CGIdxs

2. s' = H(PRF(8 \* ceil(log2(r)))) mod r

3. if subgroup\_check(h0) is INVALID abort

4. if (subgroup\_check(h\[i1\]) && ... && subgroup\_check(h\[iU\])) is INVALID abort

5. commitment = h0 \* s' + h\[i1\] \* msg\[1\] + ... + Ch\[iU\] \* msg\[U\]

6. return s', commitment

## BlindSign

BlindSign generates a blind signature from a commitment received from a holder, known messages, a secret key, the domain parameter h0 and generators from the domain parameter h. The signer also validates the commitment using the proof of knowledge of committed messages received from the holder and checks that the generators used in the commitment are not also used for the known messages.

```
blind_signature = BlindSign(commitment, (msg[1],...msg[K]), SK, GIdxs, CGIdxs, nizk, nonce)
```

Inputs:

- commitment, octet string receive from the holder in output form PreBlindSign.
- nizk, octet string received from the holder in output from BlindMessagesProofGen.
- msg\[1\],...,msg\[K\], octet strings.
- SK, a secret key output from KeyGen.
- GIdxs, vector of unsigned integers. Indices of the generators from the domain parameter h, used for the known messages.
- CGIdxs, vector of unsigned integers. Indices of the generators from the domain parameter h, used for the commited messages.
- nonce, octet string, suplied to the holder by the signer to be used with BlindMessagesProofGen.

Outputs:

- blind\_signature, octet string

Procedure:

1. (j1, ..., jK) = GIdxs

2. e = H(PRF(8 \* ceil(log2(r)))) mod r

3. s'' = H(PRF(8 \* ceil(log2(r)))) mod r

4. if BlindMessagesProofVerify(commitment, nizk, CGIdxs, nonce) is INVALID abort

5. if GIdxs intersection with CGIdxs is not empty abort

6. b = commitment + h0 \* s'' + h\[j1\] \* msg\[1\] + ... + h\[jK\] \* msg\[K\]

7. A = b \* (1 / (SK + e))

8. blind\_signature = (A, e, s'')

9. return blind\_signature

## UnblindSign

UnblindSign computes the unblinded signature given a blind signature and the holder's blinding factor. It is advised to verify the signature after un-blinding.

```
signature = UnblindSign(blind_signature, s')
```

Inputs:

- s', octet string in output form from PreBlindSign
- blind\_signature, octet string in output form from BlindSign

Outputs:

- signature, octet string

Procedure:

1. (A, e, s'') = blind\_signature

2. if subgroup\_check(A) is INVALID abort

3. if (subgroup\_check(blind\_signature)) is INVALID abort

4. s = s' + s''

5. signature = (A, e, s)

6. return signature

## BlindMessagesProofGen

BlindMessagesProofGen creates a proof of committed messages zero-knowledge proof. The proof should be verified before a signer computes a blind signature. The proof is created from a nonce given to the holder from the signer, a vector of messages, a blinding factor output from PreBlindSign, the domain parameter h0 and generators from the domain parameter h.

```
nizk = BlindMessagesProofGen(commitment, s', (msg[1],...,msg[U]), CGIdxs, nonce)
```

Inputs:

- commitment, octet string as output from PreBlindSign
- s', octet string as output from PreBlindSign
- msg\[1\],...,msg\[U\], octet strings of the blinded messages.
- CGIdxs, vector of unsigned integers. Indices of the generators from the domain parameter h, used for the commited messages.
- nonce, octet string.

Outputs:

- nizk, octet string

Procedure:

1. (i1,...,iU) = CGIdxs

2. r\~ = \[U\]

3. s\~ = H(PRF(8 \* ceil(log2(r)))) mod r

4. for i in 1 to U: r\~\[i\] = H(PRF(8 \* ceil(log2(r)))) mod r

5. U~ = h0 \* s\~ + h\[i1\] \* r\~\[1\] + ... + h\[iU\] \* r\~\[U\]

6. c = H(commitment || U\~ || nonce)

7. s^ = s\~ + c \* s'

8. for i in 1 to U: r^\[i\] = r\~\[i\] + c \* msg\[i\]

9. nizk = ( c, s^, r^)

## BlindMessagesProofVerify

BlindMessagesProofVerify checks whether a proof of committed messages zero-knowledge proof is valid.

```
  result = BlindMessagesProofVerify(commitment, nizk, CGIdxs, nonce)
```

Inputs:

- commitment, octet string in output form from PreBlindSign
- nizk, octet string in output form from BlindMessagesProofGen
- CGIdxs, vector of unsigned integers. Indices of the generators from the domain parameter h, used for the commited messages.
- nonce, octet string

Outputs:

- result, either VALID or INVALID.

Procedure:

1. (i1,...,iU) = CGIdxs

2. ( c, s^, r^ ) = nizk

3. U^ = commitment \* -c + h0 \* s^ + h\[i1\] \* r^\[1\] + ... + h\[iU\] \* r^\[U\]

4. c\_v = H(U || U^ || nonce)

5. return c == c\_v

## SpkGen

A signature proof of knowledge generating algorithm that creates a zero-knowledge proof of knowledge of a signature while selectively disclosing messages from a signature given a vector of messages, a vector of indices of the revealed messages, the signer's public key, and a presentation message.

```
spk = SpkGen(PK, (msg[1],...,msg[L]), RIdxs, signature, pm)
```

Inputs:

- PK, octet string in output form from SkToPk
- (msg\[1\],...,msg\[L\]), octet strings (messages in input to Sign).
- RIdxs, vector of unsigned integers (indices of revealed messages).
- signature, octet string in output form from Sign
- pm, octet string

Outputs:

- spk, octet string

Procedure:

1. (A, e, s) = signature

2. (w, h0, h\[1\],...,h\[L\]) = PK

3. (i1, i2,..., iR) = RIdxs

4. if subgroup\_check(A) is INVALID abort

5. if KeyValidate(PK) is INVALID abort

6. b = commitment + h0 \* s + h\[1\] \* msg\[1\] + ... + h\[L\] \* msg\[L\]

7. r1 = H(PRF(8\*ceil(log2(r)))) mod r

8. r2 = H(PRF(8\*ceil(log2(r)))) mod r

9. e\~ = H(PRF(8\*ceil(log2(r)))) mod r

10. r2\~ = H(PRF(8\*ceil(log2(r)))) mod r

11. r3\~ = H(PRF(8\*ceil(log2(r)))) mod r

12. s\~ = H(PRF(8\*ceil(log2(r)))) mod r

13. r3 = r1 ^ -1 mod r

14. for i in RIdxs m\~\[i\] = H(PRF(8\*ceil(log2(r)))) mod r

15. A' = A \* r1

16. Abar = A' \* -e + b \* r1

17. d = b \* r1 + h0 \* -r2

18. s' = s - r2 \* r3

19. C1 = A' \* e\~ + h0 \* r2\~

20. C2 = d \* r3\~ + h0 \* s\~ + h\[i1\] \* m\~\[i1\] + ... + h\[iR\] \* m\~\[iR\]

21. c = H(Abar || A' || h0 || C1 || d || h0 || h\[i1\] || ... || h\[iR\] || C2 || pm)

22. e^ = e\~ + c \* e

23. r2^ = r2\~ - c \* r2

24. r3^ = r3\~ + c \* r3

25. s^ = s\~ - c \* s'

26. for i in RIdxs: m^\[i\] = m\~\[i\] - c \* msg\[i\]

27. spk = ( A', Abar, d, C1, e^, r2^, C2, r3^, s^, (m^\[i1\], ..., m^\[iR\]) )

28. return spk

How a signature is to be encoded is not covered by this document. (TODO perhaps add some additional information in the appendix)
(FIXME: Encoding out of scope OK, but wire format is required for test vectors. Unless this document is not normative.)

## SpkVerify

SpkVerify checks if a signature proof of knowledge is VALID given the proof, the signer's public key, a vector of revealed messages, a vector with the indices of these revealed messages, and the presentation message used in SpkGen.

```
result = SpkVerify(spk, PK, (Rmsg[1],..., Rmsg[R]), RIdxs, pm)
```

Inputs:

- spk, octet string.
- PK, octet string in output form from SkToPk.
- (Rmsg\[1\], ..., Rmsg\[R\]), octet strings (revealed messages).
- RIdxs, vector of unsigned integers (indices of revealed messages).
- pm, octet string

Outputs:

- result, either VALID or INVALID.

Procedure:

1. if KeyValidate(PK) is INVALID

2. (i1, i2, ..., iR) = RIndxs

3. (A', Abar, d, C1, e^, r2^, C2, r3^, s^, (m^\[i1\],...,m^\[iR\])) = spk

4. (w, h0, h\[1\],...,h\[L\]) = PK

5. if A' == 1 return INVALID

6. X1 = e(A', w)

7. X2 = e(Abar, P2)

8. if X1 != X2 return INVALID

9. c = H(Abar || A' || h0 || C1 || d || h0 || h\[i1\] || ... || h\[iR\] || C2 || pm)

10. T1 = Abar - d

11. T2 = P1 + h\[i1\] \* Rmsg\[1\] + ... + h\[iR\] \* Rmsg\[R\]

12. Y1 = A' \* e^ + h0 \* r2^ + T1 \* c

13. Y2 = d \* r3^ + h0 \* s^ + h\[i1\] \* m^\[i1\] + ... + h\[iR\] \* m^\[iR\] - T2 \* c

14. if C1 != Y1 return INVALID

15. if C2 != Y2 return INVALID

16. return VALID

# Security Considerations

## Validating public keys

All algorithms in Section 2 that operate on points in public keys require first validating those keys.  For the sign, verify and proof schemes, the use of KeyValidate is REQUIRED.

## Skipping membership checks

Some existing implementations skip the subgroup\_check invocation in Verify (Section 2.8), whose purpose is ensuring that the signature is an element of a prime-order subgroup.  This check is REQUIRED of conforming implementations, for two reasons.

1.  For most pairing-friendly elliptic curves used in practice, the pairing operation e (Section 1.3) is undefined when its input points are not in the prime-order subgroups of E1 and E2. The resulting behavior is unpredictable, and may enable forgeries.

2.  Even if the pairing operation behaves properly on inputs that are outside the correct subgroups, skipping the subgroup check breaks the strong unforgeability property [ADR02].

## Side channel attacks

Implementations of the signing algorithm SHOULD protect the secret key from side-channel attacks.  One method for protecting against certain side-channel attacks is ensuring that the implementation executes exactly the same sequence of instructions and performs exactly the same memory accesses, for any value of the secret key. In other words, implementations on the underlying pairing-friendly elliptic curve SHOULD run in constant time.

## Randomness considerations

The IKM input to KeyGen MUST be infeasible to guess and MUST be kept secret. One possibility is to generate IKM from a trusted source of randomness.  Guidelines on constructing such a source are outside the scope of this document.

Secret keys MAY be generated using other methods; in this case they MUST be infeasible to guess and MUST be indistinguishable from uniformly random modulo r.

BBS signatures are nondeterministic, meaning care must be taken against attacks arising from signing with bad randomness, for example, the nonce reuse attack on ECDSA [HDWH12]. It is recommended that the nonces used in signature proof generation are from a trusted source of randomness.

BlindSign as discussed in 2.10 uses randomness from two parties so care MUST be taken that both sources of randomness are trusted. If one party uses weak randomness, it could compromise the signature.

When a trusted source of randomness is used, signatures and proofs are much harder to forge or break due to the use of multiple nonces.

## Implementing hash\_to\_point\_g1 and hash\_to\_point\_g2

The security analysis models hash\_to\_point and hash\_pubkey\_to\_point as random oracles.  It is crucial that these functions are implemented using a cryptographically secure hash function.  For this purpose, implementations MUST meet the requirements of [@!I-D.irtf-cfrg-hash-to-curve].

In addition, ciphersuites MUST specify unique domain separation tags for hash\_to\_point.  The domain separation tag used in Section 1.4 is the RECOMMENDED one.

## Use of Contexts

Contexts can be used to separate uses of the protocol between different protocols (which is very hard to reliably do otherwise) and between different uses within the same protocol. However, the following SHOULD be kept in mind:

The context SHOULD be a constant string specified by the protocol using it. It SHOULD NOT incorporate variable elements from the message itself.

Contexts SHOULD NOT be used opportunistically, as that kind of use is very error prone. If contexts are used, one SHOULD require all signature schemes available for use in that purpose support contexts.

Contexts are an extra input, which percolate out of APIs; as such, even if the signature scheme supports contexts, those may not be available for use. This problem is compounded by the fact that many times the application is not invoking the signing, verification, and proof functions directly but via some other protocol.

The ZKP protocols use nonces which MUST be different in each context.

## Choice of Signature Primitive

BBS signatures can be implemented on any pairing-friendly curve. Using BLS12-381 the signature achieves close to 128-bit security, and is the recommended curve at this time.

# IANA Considerations

This document does not make any requests of IANA.

# Appendix

## BLS12-381

BLS12-381

The cipher-suites in Section 4 are based upon the BLS12-381 pairing-friendly elliptic curve.  The following defines the correspondence between the primitives in Section 1.3 and the parameters given in Section 4.2.2 of [@!I-D.irtf-cfrg-pairing-friendly-curves].

* E1, G1: the curve E and its order-r subgroup.

* E2, G2: the curve E' and its order-r subgroup.

* GT: the subgroup G\_T.

* P1: the point BP.

* P2: the point BP'.

* e: the optimal Ate pairing defined in Appendix A of [@!I-D.irtf-cfrg-pairing-friendly-curves].

* point\_to\_octets and octets\_to\_point use the compressed serialization formats for E1 and E2 defined by [ZCash].

* subgroup_check MAY use either the naive check described in Section 1.3 or the optimized check given by [Bowe19].

## Test Vectors

//TODO


## Blind Sign Flow Example

The example below illustrates the creation of a blind signature. Let the Signer have a public key PK = (w, h0, h[1],...,h[L]) where (h[1],...,h[L]) generators. The end result will be a signature to the messages (m[1],...,m[K]) (K less than L). The messages (m[1],...,m[U]) (U less than K), will be committed by the Client using the first U generators from the Signers PK (i.e., h[1],,,,h[U]). The messages (m[U+1],...,m[K]) will be known to the Signer and will be signed using the generators (h[U+1],...,h[K]) from their PK.

~~~ ascii-art
+--------+                               +--------+
|        | <-(1)------- nonce ---------- |        |
|        |                               |        |
| Client | --(2)- Commitment, nizk, U -> | Signer |
|        |                               |        |
|        | <-(3)--- Blind Signature ---- |        |
+--------+                               +--------+
~~~

1. The Signer and the Client will agree on a nonce to be used by the BlindMessagesProofGen and BlindMessagesProofVerify functions.

2. The Client will use the PreBlindSign function to calculate a Pedersen commitment for the messages (m[1],...,m[U]), using the generators (h[1],...,h[U]). Then they will create a proof of knowledge (nizk) for that commitment using BlindMessagesProofGen. The Signer will receive the commitment, the proof of knowledge (nizk) and the number of committed messages (U).

3. Before sending the blinded signature to the Client the Signer must execute the following steps,
    - Validate the proof of knowledge of the commitment using BlindMessagesProofVerify, on input the commitment, nizk, the nonce (from step 1) and the U first generators from their PK. Then check that the intersection between the generators used by the Client for the commitment, and the generators (h[U+1],...,h[K]), used by the Signer for the known messages, is indeed empty.
    - Create the blind signature using the BlindSign function. Note that the blinded signature is not a valid BBS signature.

    After the Client receives the blind signature they will use the UnblindSign function to unblinded it, getting a valid BBS signature on the messages (m[1],...,m[K]).


{backmatter}
