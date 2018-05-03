### Encoding of a Bitcoin signed message

A Bitcoin signed message is data of the following format :
`[ length(header) ][ header ][ length(text) ][ text ]`

1. A variable length integer encoding the length of the header
2. The header, in text :  
    `Bitcoin signed message:\n` where `\n` means a newline character
3. A variable length integer encoding the length of the text
4. The text itself

Example 1.  Hex encoding of a Bitcoind signed message

```
header  = "Bitcoin signed message:\n"
text = "This is an example of a Bitcoin signed message."

header.hex    = 0x426974636F696E205369676E6564204D6573736167653A0A
# note the last byte, 0x0A, which is the newline
header.length = 0x18

text.hex    = 0x5468697320697320616E206578616D706C65206F66206120426974636F696E207369676E6564206D6573736167652E
text.length = 0x2F
```

The final encoding is :

```
18426974636F696E205369676E6564204D6573736167653A0A2F5468697320697320616E206578616D706C65206F66206120426974636F696E207369676E6564206D6573736167652E
```

### Encoding of a recoverable signature

A signature for a Bitcoin signed message is a *recoverable ECDSA signature* where the value that is signed is the *HASH256* of the encoded message.
Such a signature encodes three values:
1. `recid`, a single byte which points the verifier to a specific public key to be recovered from the signature and message
2. The signature's value `r`
3. The signature's value `s`

`r` and `s` are both encoded as a constant length of 32 bytes each, zero padded on the left, with `s` being **low** (bip62 ref)
The `recid` byte will be decided at signing time, as the signer signs the message. It is determined by multiple factors during signing.
The three factors which determine `recid` encoding are:

1. The Y value of the **point** R being even or odd
2. The X value of the **point** R being less than or more than the curve order
3. The signing user's public key P being used in validation is compressed or uncompressed

The first two items hint at which one of the two or four recovered public keys belong to the signer, and the third item hints at whether the compressed or uncompressed form should be used when hashing the pubkey for comparing against the provided base58 P2PKH address.
The concatenation of `[ recid ][ r ][ s ]` is then encoded in base64.

A `recid` byte encoding may be one of the following:

```
0x1B ->  R_y even | R_x < n | P uncompressed
0x1C ->  R_y odd  | R_x < n | P uncompressed
0x1D ->  R_y even | R_x > n | P uncompressed
0x1E ->  R_y odd  | R_x > n | P uncompressed
0x1F ->  R_y even | R_x < n | P compressed
0x20 ->  R_y odd  | R_x < n | P compressed
0x21 ->  R_y even | R_x > n | P compressed
0x22 ->  R_y odd  | R_x > n | P compressed
```

### Signing a Bitcoin signed message

An application should require the user to enter a base58 P2PKH address, and their text message to be signed. Message encoding and private key selection can then be done internally.

Since Bitcoin uses ECDSA, a secret random value `k` along with the private key `d` is also used at signing time.
To safely choose `k`, it is advised to use RFC6979.  The RFC6979 function takes as input both the **raw** private key and a SHA256 of the encoded message.

Example 2.  Generating `k` safely using RFC6979 (ref rfc6979)

A compressed WIF key:  
`wifkey = KzoXoCkcjfQt9mWBQ8xP5f2LfchMPDTH9NtYmmaNn95P9caVNriw`

Decodes to the raw private key:  
`d = 0x6AE933D922A5FC858EE65341576F2D255B18EB8893871788720FAC40F5E63619`

Taking the message from example 1, a SHA256 hash of the encoded message is:  
`message.sha256 = 0xAA56196ADB5699449B84E300944B0EC6CAD4584BC1C16307428C99433B48DD54`

Using these two values to generate `k`:

```
k = RFC6979( d, message.sha256 )
k : FE70FEAAB64187B22227B662138E603EAA3AF9AE1A29AB5C02E0E92D6580A6E6
```


Example 3.  The hash value to be signed is the HASH256 of the encoded message

```
message.hash256 = 0x9868B373DA46FC29ADFFDAAE7DEAA6E964E5B5EE0945BA75FA91AD7722CADF1B
z : 9868B373DA46FC29ADFFDAAE7DEAA6E964E5B5EE0945BA75FA91AD7722CADF1B
```

Using the four input values:
1. `d` the private key
2. `k` the secret nonce
3. `z` the HASH256 of the message to be signed
4. `compr` a boolean value determining if the signer's public key is compressed (`true`) or uncompressed (`false`)

It is now possible to sign a recoverable signature for a message.

Example 4.  Signing a message in a recoverable manner :

```
# Inputs:
d = 6AE933D922A5FC858EE65341576F2D255B18EB8893871788720FAC40F5E63619
k = FE70FEAAB64187B22227B662138E603EAA3AF9AE1A29AB5C02E0E92D6580A6E6
z = 9868B373DA46FC29ADFFDAAE7DEAA6E964E5B5EE0945BA75FA91AD7722CADF1B
compr = true

# Outputs:
recid , r , s
```

1. Set `recid = 1B`
2. Set `R = k*G`                 , this is an EC multiply operation
3. Set `s = 1/k * (z + r*d)`     , this is the ECDSA signing operation
4. If `s` is high :  
    4.a. Set `s = curve_order - s` , this changes `s` to be low  
    4.b. Set `R = -R`              , meaning, `(R_x, -R_y)`
5. If the point's `R` `y` coordinate is is odd :  
    5.a. Set `recid += 1`
6. If the point's `R` `x` coordinate is larger than `curve_order` :  
    6.a. Set `recid += 2`
7. If `compr` is `true` :  
    7.a. Set `recid += 4`

The resulting signature is then the concatenation of `recid` , `r` and `s` in base64:  
`H0dLiG/FSePsSaIkEk9xrfoejRPH4cEU8fgCTWtqluaWXen/PW/4Sh8DwgJVsl/IY7XBsiRAGkVO3h6WyKY7RM4=`

Encoded in this signature are the values:
```
1. recid = 1F
2. r = 474B886FC549E3EC49A224124F71ADFA1E8D13C7E1C114F1F8024D6B6A96E696
3. s = 5DE9FF3D6FF84A1F03C20255B25FC863B5C1B224401A454EDE1E96C8A63B44CE
```

### Verifying a Bitcoin signed message

An application should require the user to enter:

1. The base58 P2PKH address claimed by the message signer
2. The recoverable signature in base64
3. The text portion of the complete encoded message

First the application will compute the signed value `z` from the provided text by using the same process as in examples 1 and 3.  
Verifying a recoverable signature over a message always results in at least two possible public keys belonging to the signer, and in special cases there may be four possible public keys, requiring up to two tries at verification.  
The use of the `recid` byte assures that the verifier always runs once, and only checks one of the possible public keys.  
First, public key recovery will be described. Next, the verification algorithm will make use of this process and finally a single recovered public key will be matched against the base58 P2PKH address provided to the verifier by the message signer.

The process of pubkey recovery takes as input a signature and message and returns two public keys, with the first key being the one with an even `y` coordinate:

```
# Inputs:
1. r = 474B886FC549E3EC49A224124F71ADFA1E8D13C7E1C114F1F8024D6B6A96E696
2. s = 5DE9FF3D6FF84A1F03C20255B25FC863B5C1B224401A454EDE1E96C8A63B44CE
3. z = AA56196ADB5699449B84E300944B0EC6CAD4584BC1C16307428C99433B48DD54

# Outputs:

Two public keys
```

1. Set `z_neg = curve_order - z`
2. Set `Z = z_neg*G`
3. Find the two possible `y` coordinates `y_1` and `y_2` for the `r` value by solving the curve's equation
4. Set `R_1 = (r, y_1)` and `R_2 = (r, y_2)`
5. Set `M_1 = s*R1 + Z`
6. Set `M_2 = s*R2 + Z`
7. Set `P_a = (1/r) * M_1`  , where the division symbol is the inverse mod operation
8. Set `P_b = (1/r) * M_2`  , where the division symbol is the inverse mod operation
9. If `y_1` is even:  
    9.a. return `(P_a, P_b)`  
   else:  
    9.b. return `(P_b, P_a)`


Example 5.  Verifying a message signed by a recoverable signature

```
# Inputs
1. recid = 1F
2. r = 474B886FC549E3EC49A224124F71ADFA1E8D13C7E1C114F1F8024D6B6A96E696
3. s = 5DE9FF3D6FF84A1F03C20255B25FC863B5C1B224401A454EDE1E96C8A63B44CE
4. z = AA56196ADB5699449B84E300944B0EC6CAD4584BC1C16307428C99433B48DD54
5. signer_addr = 14rVJfMZQGm9XruP2boYKrTZNCBoMp2ekK

# Output:
true OR false
```

1. Set `compr = ( (recid - 1B) & 4 != 0 )`  , where `&` is bitwise AND
2. Set `recid = ( (recid - 1B) & 3 )`       , where `&` is bitwise AND
3. If `recid >= 2` :  
    3.a. Set `r = r + curve_order`
4. Set `(P_1, P_2) = pubkey_recovery(z, r, s)`  
5. If `recid` is even:  
    5.a. Set `P = P_1`  
   else:  
    5.b. Set `P = P_2`
6. If `compr` is `true`:  
    6.a. Set `P` to the compressed form
7. Set `rec_addr` to be the P2PKH address matching the recovered `P`
8. If `signer_addr != rec_addr`:  
    8.a. return `false`  
   else:  
    8.b. return `true`
