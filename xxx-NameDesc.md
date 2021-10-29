# NameDesc

`author: Hampus Sjoberg`

---

Pionereed by [Blixt Wallet](https://blixtwallet.com/) and implemented by [@lntxbot](https://t.me/lntxbot), **NameDesc** is a standard for conveying "receiver name" in BOLT11 invoices.

It consists in prepending the actual invoice description with a name plus two spaces. For example, Alfred's wallet can produce an invoice with the following description:

```
Alfred:  payment for the ice cream you owed me
```

When Barbara's wallet receives that, it displays something like:

```
Paying to:   Alfred
Description: payment for the ice cream you owed me
```

The good thing about such a simple standard is that if a wallet doesn't support parsing the "Name" part it will still not be bad, as it will show:

```
Description: Alfred:  payment for the ice cream you owed me
```

And if the payee's wallet doesn't support including a "Name", still Alfred can manually type his name in the description field. Not the best experience in the world, but doable.
