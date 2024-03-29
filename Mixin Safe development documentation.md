# Mixin Safe

Mixin Safe uses the native Bitcoin multisig and timelock script as a state-of-the-art solution to secure BTC. The 2/3 multisig is composed of three keys, holder, signer and observer. With a timelock of 1 year, if and only if holder and signer both sign the transaction, the BTC can be spent. In any case of holder or signer key lost, the observer can be the rescuer after 1 year.

Specifically, the signer key is MPC generated by Mixin Safe nodes, so it's decentralized controlled. Whenever a deposit detected to a safe account, Mixin Safe will issue the same amount of safeBTC to the account owner. To make the signer key sign a transaction with the holder, the account owner needs to send safeBTC to the safe network, and sign the raw transaction with the holder key.

## Prepare the Holder Key

There is not much Bitcoin wallets can do custom script signing, not even the bitcoin-core wallet. So for now it's recommended to use https://github.com/btcsuite/btcd.

With btcd you can generate a public and private key pair as below:

```golang
priv, pub := btcec.PrivKeyFromBytes(seed)
fmt.Printf("public: %x\nprivate: %x\n", pub.SerializeCompressed(), priv.Serialize())

`
public: 032e423d3bf1d054f0c33ff3c2d8bc1f341b51ad28cb2575314dee57808ff77681
private: 27e81d37bb56bdb8d22363a735bb4391caa2768def768fc1b50893e38c7ca3c2
`
```

Then generate a random UUID as the session id:

```
de77d0a9-8d96-476b-987a-0f9dc64c9dc3
```

## Send Account Proposal

All messages to the safe network should be encoded as the following operation struct

```golang
type Operation struct {
	Id     string
	Type   uint8
	Curve  uint8
	Public string
	Extra  []byte
}

func (o *Operation) Encode() []byte {
	pub, err := hex.DecodeString(o.Public)
	if err != nil {
		panic(o.Public)
	}
	enc := common.NewEncoder()
	writeUUID(enc, o.Id)
	writeByte(enc, o.Type)
	writeByte(enc, o.Curve)
	writeBytes(enc, pub)
	writeBytes(enc, o.Extra)
	return enc.Bytes()
}
```

To send the account proposal, with the holder prepared from last step, the operation value should be like

```golang
op := &Operation {
  Id: "de77d0a9-8d96-476b-987a-0f9dc64c9dc3",
  Type: 110,
  Curve: 1,
  Public: "032e423d3bf1d054f0c33ff3c2d8bc1f341b51ad28cb2575314dee57808ff77681",
}
```

All these four fields above are mandatory for all safe network messages, now we need to make the extra.

```golang
threshold := byte(1)
total := byte(1)
owners := []string{"fcb87491-4fa0-4c2f-b387-262b63cbc112"}
extra := []byte{threshold, total}
uid := uuid.FromStringOrNil(owners[0])
op.Extra = append(extra, uid.Bytes()...)
```

So the safe account proposal operation extra is encoded with threshold, owners count, and all owner UUIDs.

Then we can encode the operation and use it as a memo to send the transaction to safe network MTG.

```golang
memo := base64.RawURLEncoding.EncodeToString(op.Encode())
input := mixin.TransferInput{
  AssetID: "c6d0c728-2624-429b-8e0d-d9d19b6592fa",
  Amount:  decimal.NewFromFloat(0.0001),
  TraceID: op.Id,
  Memo:    memo,
}
input.OpponentMultisig.Receivers = []{
  "71b72e67-3636-473a-9ee4-db7ba3094057",
  "148e696f-f1db-4472-a907-ceea50c5cfde",
  "c9a9a719-4679-4057-bcf0-98945ed95a81",
  "b45dcee0-23d7-4ad1-b51e-c681a257c13e",
  "fcb87491-4fa0-4c2f-b387-262b63cbc112",
}
input.OpponentMultisig.Threshold = 4
```


## Approve Safe Account

After the proposal transaction sent to the safe network MTG, you can monitor the Mixin Network transactions to decode your account details, but to make it easy, it's possible to just fetch it from the Safe HTTP API.

```
curl https://safe.mixin.dev/proposals/de77d0a9-8d96-476b-987a-0f9dc64c9dc3
{"address":"bc1qj0rlafrur5qwnhqvu4e9h066kvayhwy72p7ax62exlxu3rlc9xvs6l6yd8"}
```

You should have noticed that the request was made with the same session UUID we prepared at the first step. That address returned is our safe account address to receive BTC, but to start using it, we must approve it with our holder key.


```golang
hash := sha256.Sum256([]byte(address))
b, _ := hex.DecodeString(priv)
private, _ := btcec.PrivKeyFromBytes(b)
sig := ecdsa.Sign(private, hash[:])
fmt.Println(base64.RawURLEncoding.EncodeToString(sig.Serialize()))

`
MEQCIENpk735gVk-7OWeieOuuEyFThmKpdRFZnn9wZbR-1T8AiBHiFOLVqKxTaZRXm-e4dUoomVujNmE8xCpsQ_-lpbRBA
`
```

With the signature we send the request to safe network to prove that we own the holder key exactly.

```
curl https://safe.mixin.dev/proposals/de77d0a9-8d96-476b-987a-0f9dc64c9dc3 -H 'Content-Type:application/json' \
  -d '{"address":"bc1qj0rlafrur5qwnhqvu4e9h066kvayhwy72p7ax62exlxu3rlc9xvs6l6yd8","signature":"MEQCIENpk735gVk-7OWeieOuuEyFThmKpdRFZnn9wZbR-1T8AiBHiFOLVqKxTaZRXm-e4dUoomVujNmE8xCpsQ_-lpbRBA"}'
{"address":"bc1qj0rlafrur5qwnhqvu4e9h066kvayhwy72p7ax62exlxu3rlc9xvs6l6yd8"}
```

Now we can deposit BTC to the address above, and you will receive safeBTC to the owner wallet.
