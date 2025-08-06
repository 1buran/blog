---
title: "Add Encryption to Json"
date: 2024-07-25T11:00:18+03:00
draft: false
toc: true
images:
tags:
  - JSON
  - Golang
  - closures
  - Encryption
  - AES
---

# Intro

Sometimes you need to save sensitive information in JSON: tokens, passwords or just
something that you do not want anyone else to see this e.g. your personal financial
informations etc.

The post about it: how do you add encryption of part of JSON.

# Use cipher & aes

There are a lot of articles with ready for copy-paste snippets about using block cyphers,
also take a look at examples in `crypto/cipher` examples.

Let's not retell them all, just give basic code examples.

Here are the basic `encrypt/decrypt` functions you might write:

```go
package main

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"io"
)

func encrypt(key string, b []byte) (encrypted []byte, err error) {

	block, err := aes.NewCipher([]byte(key))
	if err != nil {
		return nil, err
	}

	aesgcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}

	nonce := make([]byte, aesgcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return nil, err
	}

	encrypted = aesgcm.Seal(nonce, nonce, b, nil)
	return
}

func decrypt(key string, b []byte) (decrypted []byte, err error) {

	block, err := aes.NewCipher([]byte(key))
	if err != nil {
		return nil, err
	}

	aesgcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}

	nonceSize := aesgcm.NonceSize()

	nonce, encrypted := b[:nonceSize], b[nonceSize:]
	if decrypted, err = aesgcm.Open(nil, []byte(nonce), []byte(encrypted), nil); err != nil {
		return nil, err
	}

	return
}
```

The first one encrypts passed bytes and the second decrypts them back to plain text. There
is a little nuance: due to block nature of AES you should use 16 bytes length key/password
or 24 or 32 bytes (in this case will be used AES-256):

| Secret Key Length   | AES Variant | Block Size          |
| :------------------ | :---------- | :------------------ |
| 16 bytes (128 bits) | AES-128     | 16 bytes (128 bits) |
| 24 bytes (192 bits) | AES-192     | 16 bytes (128 bits) |
| 32 bytes (25 6bits) | AES-256     | 16 bytes (128 bits) |

Ok, so far so good. Let's talking about JSON. So, as you know Go has marshaling and
unmarshaling of structs and provide your a way to add custom `MarshalJSON/UnmarshalJSON`
for your data stypes. That's look like the way (or place) to implemented encryption for
special data type.

Let's create a new type e.g. `EncryptedString`:

```go
type EncryptedString string
```

derived (or better say - type alias) from simple, basic type string looks suitable:
everywhere we need to store sensitive information we can use that type instead of
`string`, for code it will looks like the same as using `string` but when you marshal this
data to JSON, it should be encrypted automatically. And when you will unmarshaling back
that encrypted data from JSON to golang struct, they should be decrypted automatically. So
that is the goal.

So for example you might write something like this:

```go

type MoneyTransfer struct {
    ID, Amount                                             int64
    CashPointAddress, ManagerBonus, Gateway                string
    Sender, Recipient, Description, Currency, TransferCode EncryptedString
}
```

here is we define something which can hold based information about money transfer: amount,
currency, sender and recipient, code of transfer to pick up the money at a cash point,
etc.

It also may has a lot extra, non sensitive data, which should not be encrypted for
decreasing the CPU consumption and size that is why we not encrypted full JSON.

Another reason is want to has plain text with basic, non confidential data which can be
used in other data pipeline processing. So in that way we can easy separate confidential
data from non-confidential data.

Ok, let's define custom `MarshalJSON/UnmarshalJSON`, it is pretty easy:

```go

func (s EncryptedString) MarshalJSON() ([]byte, error) {
	b, err := Encryptor([]byte(s))
	if err != nil {
		return nil, err
	}
	return json.Marshal(b)
}

func (s *EncryptedString) UnmarshalJSON(b []byte) error {
	var b1 []byte
	if err := json.Unmarshal(b, &b1); err != nil {
		return err
	}

	dec, err := Decryptor(b1)
	if err != nil {
		return err
	}
	*s = EncryptedString(dec)
	return nil
}
```

take a look at this code, it is pretty simple (ignore the `Encryptor/Decryptor` for now,
we will talk about them later): encrypt passed bytes and marshal them, unmarshal passed
bytes and decrypt them back to original strings.

Ok, here is an example how it will look:

```go
	tr1 := MoneyTransfer{
        ID: 123,
        Amount: 15000,
        CashPointAddress: "7001 E Belleview Ave, Denver, CO 80237, USA",
        ManagerBonus: "+$5.0 if processed less than 1 hour",
        Gateway: "MoneyGram",
        Sender: "John Doe",
        Recipient: "Jane Doe",
        Description: "Casio G Shock Retro 1986",
        Currency: "JPY",
        TransferCode: "L1K45M12XCBGAQ"
    }

	b, err := json.Marshal(tr1)
	if err != nil {
		fmt.Println(err)
	}

	fmt.Printf("Plain text: %#v\n\n", tr1)
	fmt.Printf("Marshaled JSON: %s\n\n", b)

	var tr2 MoneyTransfer
	if err := json.Unmarshal(b, &tr2); err != nil {
		fmt.Println(err)
	}

	fmt.Printf("Unmarshaled JSON: %#v\n", tr2)
```

this code produces output:

```
Plain text: main.MoneyTransfer{ID:123, Amount:15000,
CashPointAddress:"7001 E Belleview Ave, Denver, CO 80237, USA",
ManagerBonus:"+$5.0 if processed less than 1 hour",
Gateway:"MoneyGram", Sender:"John Doe", Recipient:"Jane Doe",
Description:"Casio G Shock Retro 1986", Currency:"JPY", TransferCode:"L1K45M12XCBGAQ"}

Marshaled JSON: {"ID":123,"Amount":15000,"CashPointAddress":"7001 E
Belleview Ave, Denver, CO 80237, USA","ManagerBonus":"+$5.0 if
processed less than 1 hour","Gateway":"MoneyGram",
"Sender":"YeVdx4cMLOeGOrR7skkuvalwI5oExg1Yb9S2hdMl8T1LiE1t",
"Recipient":"q/WtmxTyI5x6hvxWDKmYcj04xZ/O7PSdFzQF8OMSqVssfZrX",
"Description":"PMY16+ucUdK81JcLmhO4qDrAqrvIIRYbV5kNcORKDZhYs5Bv9Yzx4AOVKVHlD6zZsC/CcA==",
"Currency":"r5Pm1bNaHxUadIteGmCzXUaxFdAwZrwkewbXX18Cdw==",
"TransferCode":"45bCkb4cUn41/kJZcym19V8K1pie57gICtPQFc8gJA55OinTNy1xBJp7"}

Unmarshaled JSON: main.MoneyTransfer{ID:123, Amount:15000,
CashPointAddress:"7001 E Belleview Ave, Denver, CO 80237, USA",
ManagerBonus:"+$5.0 if processed less than 1 hour",
Gateway:"MoneyGram", Sender:"John Doe", Recipient:"Jane Doe",
Description:"Casio G Shock Retro 1986", Currency:"JPY", TransferCode:"L1K45M12XCBGAQ"}
```

as we see, the JSON string contains unencrypted data (ID, Amount, CashPointAddress,
MangerBonus and Gateway) and encrypted data: Sender, Recipient, Description, Currency and
TransferCode. Ok it looks good: we can safety transmit this json over network or save it
in persistent storage. When we want to decrypt this data back into plain text, it is also
works well - take a look at third line of output: "Unmarshaled JSON".

So this is completely transparent way to encrypt/decrypt the part of JSON based on Go
idiomatic way to marshaling/unmarshaling structs to/from JSON.

The last thing what i should clear is about encryptor/decryptor. Well, this is just a way
save password somewhere is more hidden than simple variable, i thing the closure is more
or less suitable for this (maybe some of you know the better way, without use 3rd party
libs or services like Hashicorp Vault etc, please leave the comment!).

So, technically it needs two global vars like these ones:

```go
var Encryptor, Decryptor func(b []byte) ([]byte, error)
```

we used them inside `func (s *EncryptedString) UnmarshalJSON` and
`func (s EncryptedString) MarshalJSON()`

and two functions which initialize them:

```go

func CreateEncryptor(key string) func(b []byte) ([]byte, error) {
	return func(b []byte) ([]byte, error) {
		return encrypt(key, b)
	}
}

func CreateDecryptor(key string) func(b []byte) ([]byte, error) {
	return func(b []byte) ([]byte, error) {
		return decrypt(key, b)
	}
}
```

These functions create closures with saved passwords inside, in that way we can simple use
them without passed the password every time when we need encrypt or decrypt data.

I hope this was useful, thanks for reading!
