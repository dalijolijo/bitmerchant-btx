[![Build Status](https://travis-ci.org/sbuss/bitmerchant.png?branch=master)](https://travis-ci.org/sbuss/bitmerchant) [![Coverage Status](https://coveralls.io/repos/sbuss/bitmerchant/badge.png)](https://coveralls.io/r/sbuss/bitmerchant)

# WORK IN PROGRESS

The README below is a goal I am working toward. This library is not ready
for public use.

# Bitmerchant
Bitmerchant is a work-in-progress python library for common bitcoin/altcoin
merchant uses.

The first goal is an easy to use [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#)
wallet for linking user payments with their accounts.

Up next is a system that monitors the blockchain and sends out a signal
when a payment is received at an address you're tracking.

---

If you find this library useful, please consider a small donation to

|BTC|Doge|
|:-:|:--:|
|19jSqVd8bpevi3qacBedkAdDqEXtGAn5t7|DQ4b7RJfoniVwFsnrMJr6vi6n6UFeubdiv|
|![Donate BTC](https://raw.github.com/sbuss/bitmerchant/master/media/donation_btc_qr_code.gif)|![Donate DOGE](https://raw.github.com/sbuss/bitmerchant/master/media/donation_doge_qr_code.gif)|

# BIP32 wallets

[BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#)
wallets are hierarchical deterministic wallets. They allow you to generate
bitcoin/altcoin addresses without exposing your private key to a potentially
insecure server.

To link a user with a new bitcoin address, you just need to provide the user's
ID to the `create_new_address_for_user` method:

## TL;DR

```python
## DO THIS ON AN OFFLINE MACHINE, NOT YOUR WEBSERVER
from bitmerchant.wallet import Wallet

# Create a wallet, and a primary child wallet for your app
my_wallet = Wallet.new_random_wallet()
print(my_wallet.serialize_b58(private=True))  # Write this down or print it out and keep in a secure location
project_0_wallet = my_wallet.get_child(0, is_prime=True)
project_0_public = project_0_wallet.public_copy()
print(project_0_public.serialize_b58(private=False))  # Put this in your app's settings file


## THINGS BELOW ARE PUBLIC FOR YOUR WEBSERVER

# In your app's settings file, declare your public wallet:
WALLET_PUBKEY = "<public output from above>"

# Create a payment address for a user as needed:
from bitmerchant.wallet import Wallet
from myapp.settings import WALLET_PUBKEY

def get_payment_address_for_user(user):
    user_id = user.id
    assert isinstance(user_id, (int, long))
    wallet = Wallet.deserialize(WALLET_PUBKEY)
    wallet_for_user = wallet.create_new_address_for_user(user.id)
    return wallet_for_user.to_address()
```

## Create a new wallet

If you haven't created a wallet yet, do so like this:

**IMPORTANT** You must back up your wallet's private key, otherwise you won't
be able to retrieve the coins sent to your public addresses.

```python
from bitmerchant.wallet import Wallet

my_wallet = Wallet.new_random_wallet()

# Then back up your private key

private_key = my_wallet.serialize()
print(private_key)
# Make sure that you can load your wallet successfully from this key
wallet_test = Wallet.deserialize(private_key)
assert my_wallet == wallet_test
# If that assertion fails then open a ticket!
# NOW WRITE DOWN THE PRIVATE KEY AND STORE IT IN A SECURE LOCATION
```

BIP32 wallets (or hierarchical deterministic wallets) allow you to create
child wallets with limited permissions. You can, for example, create a new
child wallet for every website you run. It's a good idea to create at least
*one* child wallet for use on your website. The thinking being that if your
website's wallet gets compromised somehow, you haven't completely lost control
because your master wallet is secured on an offline machine. You can use your
master wallet to move any funds in compromised child wallets to new child
wallets and you'll be ok. We'll talk more about a breach and how to handle it
below.

But first, let's generate a new child wallet for your first website!

```python
# Lets assume you're loading a wallet from your safe private key backup
my_wallet = Wallet.deserialize(private_key)

# Create a new, public-only child wallet. Since you have the master private
# key, you can recreate this child at any time in the future and don't need
# to securely store its private key.
child = my_wallet.get_child(0, as_private=False)

# And lets export this child key
public_key = my_wallet.serialize(private=False)
print(public_key)
```

You can safely store your public key in your app's source code. There's
no need to be paranoid\* about anyone getting it. All they can do is generate
payment addresses that YOU control.

\*Ok.. you should be a *little* paranoid. If someone gets a hold of your public
key then they can generate all of your subsequent child addresses, which means
they'll know exactly how many coins you have. The attacker cannot spend any
coins, though.

## Generating new public addresses

BIP32 wallets allow you to generate public addresses without revealing your
private key. Just pass in the user ID that needs a wallet:

```python
from bitmerchant.wallet import Wallet
from myapp.settings import WALLET_PUBKEY  # Created above

master_wallet = Wallet.deserialize(WALLET_PUBKEY)
user_wallet = master_wallet.create_new_address_for_user(user_id)
payment_address = user_wallet.to_address()
```

This assumes that `user_id` is a unique positive integer and does not change
for the life of the user (and is less than 2,147,483,648). Now any payments
received at `payment_address` should be credited to the user identified by
`user_id`.

# Staying secure

## Public Keys

Public keys are safe to keep on a public webserver. However, even though a
public key does not allow an attacker to spend any of your coinds, you should
still try to protect the public key from hackers or curious eyes. Knowing the
public key allows an attacker to generate all possible child wallets and know
exactly how many coins you have. This isn't terrible, but nobody likes having
their books opened up like this.

Your master public key can be used to generate a virtually unlimited number of
child public keys. Your users won't pay to your master public key, but instead
you'll use your master public key to generate a new wallet for each user.

## Private Keys

You must have the private key to spend any of your coins. If your private key
is stolen then the hacker also has control of all of your coins. With a BIP32
Wallet, generating a new master wallet is one of the only times that you need
to be paranoid (and you're not being paranoid if they really *are* out to get
you). Paranoia here is good because if anyone gets control of your master
wallet they can spend all funds in all child wallets.

You should create your wallet on a computer that is not connected to the
internet. Ideally, this computer will *never* be connected to the internet
after you generate your private key. The safest way to do this is to run
Ubuntu on a livecd, install python and bitmerchant, and generate a new wallet.

Once you generate a new wallet you should write down the private key on a
piece of paper (or print it out ...but can you *really* trust your printer?)
and store it in a secure location.

```sh
sudo apt-get install python
sudo apt-get install pip

pip install bitmerchant
pip install ipython

# Then launch the ipython shell
ipython
```

Once inside your ipython shell, generate a new wallet:

```python
from bitmerchant.wallet import Wallet

my_wallet = Wallet.new_random_wallet()

# Then back up your private key

private_key = my_wallet.serialize()
print(private_key)
# Write down this private key.
# Double check it.
# Then shut down the computer without connecting to the internet.
```

## Master private key

Your master private key allows you to spend coins sent to any of your public
addresses. Guard this with your life, and never put it on a computer that's
connected to the internet.

Master private keys must NEVER be put on the internet. They must NEVER be
located on a computer that is even *connected* to the internet. The only key
that should be online is your PUBLIC key. Your private key should be written
down (yes, on paper) and stored in a safe location, or on a computer that is
never connected to the internet.

Security wise, this is the most important part of generating secure public
payment addresses. A master private key is the only way to retrieve the funds
paid to a public address. You can use your master private key to generate the
private keys of any child wallets, and then transfer those to a networked
computer as necessary, if you want slightly smaller surface area for attacks.

Forthcoming versions of bitmerchant will allow you to generate transactions
offline that you can safely transfer to a networked computer, allowing you to
spend your child funds without ever putting a private key on a networked
machine.
