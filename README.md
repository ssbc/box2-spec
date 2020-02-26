# envelope spec

:warning: **DRAFT STATUS** :warning:

This is a spec for encrypting messages to groups of people.
Initially it will support communication for large groups which share a public key
(secret key cryptography / symmetric keys), but it has also been designed to support
forward-secure secret-key cryptography (a little like Signal's double-ratchet).

_envelope_ assumes each message is part of append-only chain (with a unique `feed_id`), 
made up of backlinked messages such that each message has a unique previous message with 
a unique id (`prev_msg_id`)

## Anatomy

After boxing, a complete envelope message looks like this:

```
 +---------------------------------------+
 | ╔═══════════════════════════════════╗ |
 | ║            header_box             ║ |
 | ╚═══════════════════════════════════╝ |
 | ┌───────────────────────────────────┐ |
 | │            key_slot_1             │ |
 | ├───────────────────────────────────┤ |
 | │            key_slot_2             │ |
 | ├───────────────────────────────────┤ |
 | │               ...                 │ |
 | ├───────────────────────────────────┤ |
 | │            key_slot_n             │ |
 | └───────────────────────────────────┘ |
 | ╔═══════════════════════════════════╗ |
 | ║           extensions              ║ |
 | ║                                   ║ |
 | ╚═══════════════════════════════════╝ |
 | ╔═══════════════════════════════════╗ |
 | ║             body_box              ║ |
 | ║                                   ║ |
 | ║                                   ║ |
 | ║                                   ║ |
 | ║                                   ║ |
 | ║                         ╔═════════╝ |
 | ╚═════════════════════════╝           |
 +---------------------------------------+
```

### header_box 

A secretbox (refering to libsodium `crypto_secretbox_easy`), which describes the layout 
and configuration of the message to follow.
Being able to decrypt this is required for being able to unbox the rest of the message.

```
 ╔═════════════════════════════════╗
 ║            header_box           ║
 ╚═════════════════════════════════╝ 
 |                32               |
 |                                 |

 ┌─────────────────┬───────────────┐
 │       HMAC      │    header*    │
 └─────────────────┴───────────────┘
         16       /       16        \
                 /                   \
                /                     \
               /                       \
              /                         \
             /                           \
   
    +----------------+-------+-------------------- ---+
    | offset         | flags | header_extensions      |
    +----------------+-------+-------------------- ---+
             2           1              13 
```

- `HMAC` - 16 bytes which allows authentication of the integrity of `header*`
- `header*` - the **header** encrypted with `header_key` + zerod nonce
- `offset` - 2 bytes which desribe the offset of the start of [body_box][bb] in bytes
- `flags` - 1 byte where each bit describes which [extensions][e] are active (if any)
- `header_extensions` - 13 bytes for configuration of [extensions][e]
   
### key_slot_n

Each of these slots is like a 'safety deposit box' which contains a copy of the top-level
`msg_key` which allows decryption of everything in the message.

The slots contents are defined by
```
slot_content = xor(
  msg_key,
  Derive(recipient_key, ["slot_key", key_mgmt_scheme], 32)
)
```

Where 
- `Derive` is the same derivation function defined [here](./derive_secret/README.md)
- `recipient_key` is one of the shared keys you're encrypting to 
could be:
  - a private key for a group (symmetric key)
  - a double-ratchet derived key for an individual (this option requires more info in the `header_extensions` + [extensions][e])
- `key_mgmt_scheme` is the type of `recipient_key`, specifically what sort of key management it's involved in, e.g. :
  - "envelope-large-symmetric-group"
  - "envelope-id-based-dm-converted-ed25519"
  - "envelope-signed-dh-key-curve25519"

Note these slots have no HMAC. This is because if you successfully extract `msg_key` from one of
these slots you can immediately confirm if you can decrypt the [header_box][hb], which has an HMAC,
which will confirm whether you have the correct key

### extensions

...WIP

This is where things like keys for double-ratchet-like communication will go.
This section might also contain padding.

### body_box

The section which contains the plaintext which we've boxed.

```
 ╔═════════════════════════════════╗
 ║             body_box            ║
 ║                                 ║
 ║                                 ║
 ║                                 ║
 ║                                 ║
 ║                         ╔═══════╝
 ╚═════════════════════════╝
 |              >=16               |
 |                                 |

 ┌─────────────────┬───────────────┐
 │       HMAC      │               │
 ├─────────────────┘               │
 │                                 │
 │             body*               │
 │                                 │
 │                         ┌───────┘
 └─────────────────────────┘
```
   
- `HMAC` - 16 bytes which allows authentication of the integrity of `body*`
- `body*` - the **body** encrypted with `body_key` and a zerod nonce

## Unboxing algorithm

When you receive a envelope message, the only things you know are:
- the length of the whole box (doesn't tell you much, as there may be padding)
- where the key-slots start (because the [header_box][hb] is exactly 32 bytes)
- where this message was posted (we call this it's "context", and the boxing is bound to this)
  - which `feed_id` posted it
  - what the `prev_msg_id` was (i.e. what was the message before it in this `feed_id`s chain?)

So starting after the [header_box][hb] (32 bytes in), we lift out successive chunks of 32 bytes
(the size of a [key_slot][ks]) chunks and try and decrypt them.

> The way we know if a [key_slot][ks] has yielded us a valid key for the message is by trying to see
> if the "key" we've derived from a slot helps us decrypt the [header_box][hb]. This works because
> the [header_box][hb] has an HMAC, which is an authentication code which allows us know know if our 
> decryption is valid.

> If the first slot doesn't yield a valid key, we move to the second slot (starting 32 + 32 bytes
> into the box), and check the next slot. We either try incrementing through the whole box till 
> we succeed (or reach the end), OR we set a "max depth" we want to try (e.g. if we think there
> will not be more than 10 slots, we can quit after (32 + 10 * 32 bytes).

Once we have the `msg_key`, we can decrypt the [header_box][hb]. This reveals `offset` - the position
of the start of the [body_box][bb] in bytes. This allows us to proceed to decrypt the body of the original message.

Futher detail:
- different keys are used to decrypt [header_box][hb], [extensions][e], [body_box][bb], 
but they are all [derived deterministically from `msg_key`](#key-derivation)
- they are all encrypted with "zerod nonces", as the keys used for each are absolutely specific
to the context (context = `feed_id`, `prev_msg_id` and `msg_key`)

## Design

[Original notes](./original_notes.md) from a week long design session Dominic + Keks did.
(scuttlebutt: `%39f9I0e4bEln+yy6850joHRTqmEQfUyxssv54UANNuk=.sha256`)

## Key derivation

Keys are derived from `msg_key` as follows

```
msg_key
  │
  ├──> read_key = Derive(msg_key, "read_key", 32)
  │      │
  │      ├──> header_key = Derive(read_key, "header_key", 32)
  │      │
  │      └──> body_key = Derive(read_key, "body_key", 32)
  │
  └──> extensions = Derive(msg_key, "extentions", 32)
         │
         └──> TODO
```

Where [the Derive function is defined here](./derive_secret/README.md)

`msg_key` is the symmetric key that is encrypted to each recipient or group.
When entrusting the message, instead of sharing the `msg_key` instead the message `read_key` is shared.
this gives access to header metadata and body but not ephemeral keys.

## Implementations

- Go: https://github.com/cryptoscope/ssb/tree/private-groups/private/envelope
- Node.js: https://github.com/ssbc/envelope-js



[hb]: #header_box
[ks]: #key_slot_n
[e]: #extensions
[bb]: #body_box
[kd]: #key-derivation

