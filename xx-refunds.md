# Refunds without offers

## Context
Disclaimers: Sorry in advance if these ideas were discussed in some other threads and
discarded or something.

payer=alice
payee=bob

Hi all,

in https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-November/002276.html
rusty proposes a way to deal with refunds, but I wonder if refunds in
lightning can be solved in a more general way that doesn't require
implementations to support all the features included in the offers
bolt proposal while that's in progress.

What follows is a simplified analysis and a series of incremental
design proposals to solve the problem.
Some of them satisfying new requirements.
Some of them kind of orthogonal to each other

## Analysis

Let's summarize some requirements that seem to be taken to account in that proposal:

A) We want the payer to prove it was her who paid to the payee.

Whether that is enough for the payee to proceed with a refund or not is out of the scope
of the this proposal, it depends on the business and the situation.

B) We don't want the payer to need to reveal its node's id to proof she paid

That would partially defeat the purpose of using onion routing for
payments in the first place.

C) If a refund occurs, the payee must be able to prove it occurred.

## Design

Let's assume the payee has an http server url to process refunds he
considers to be legitimate beyond the proof of payment.
For the purpose of this document, it is assumed the payee always
accepts refunds as legitimate provided requirement A is met in the
payer's request.

### Design D1: Optionally provide a refund_pk field in the hop_payload to the payee

A naive approach at the http server could get the following arguments:

- refund_invoice (this kind of breaks requirement B, although not necessarily)
- payment_hash (so that the payee can identify the claimed refund)
- payment_preimage
- refund_signature (this signs both payment_hash and the payment_hash
  in refund_invoice)

The public key (refund_pk) for the refund_signature could be simply optionally transmitted in the hop_payload to the payee.
Invoices could use a bit to signal whether they will read this refund_pk field or not.

But by default, a refund_invoice reveals your node id as a payer,
something you didn't need to do before and breaks requirement B.
There's ways around it like rendezvous payments.

### Design D2: No invoice needed for refund

Another option could be to directly replace refund_invoice with a
refund_onion argument plus a refund_payment_hash to be signed with
refund_signature.

The refund_payment_hash argument is needed when using refund_onion
argument instead of refund_invoice, since the payee has no invoice to
read the hash from.

This way, the payer asking for a refund doesn't reveal its own node
id, and the route used to pay could be reused backwards without it being
revealed to the payee (other routes are possible too).

The payee doesn't even need to calculate any route since it is
provided to him without being him able to see anything beyond what he
needs to know to pay to one of his channels by following the onion received,
and what payment_hash he is paying to and needs to store,
to be able to prove he refunded the payer when he needs to
(requirement C).

This should be compatible with trampoline payments provided they
require the payee to be a trampoline-compatible node.

### Design D3: Ephemeral merkle key for the payee's onion

The payer knows the ephemeral key for the payee and all the components
needed to calculate it before sending any onion to any of her channels' node.
But that's only for the payee to decrypt, not for the payer to sign.

In other words, the ephemeral key for the onion destined for the payer and that
payment_hash is calculated by f_ephemeral(payee_pk, more_stuff).

f_ephemeral(payee_pk, more_stuff) could be replaced with:
g_ephemeral(payee_pk, refund_pk, more_stuff)

This can be done in a way in which all the previous properties, that is:

- payer can still privately communicate with payee by encrypting to
  g_ephemeral instead of f_ephemeral.
- payee doesn't know payer's id

But adding the properties:

- payer can prove she was the payer by revealing a merkle tree to the payee with:
  + The result of g_ephemeral as the root of the tree
  + payee_pk in a specific position of the tree
  + refund_pk in a specific position of the tree
  + more_stuff is allowed in the rest of the tree

The payer transmitting refund_pk only when it's needed removes the
need for refund_pk in the hop_payload to the payee.

### Design D4: No need for http server

The http server answering refund requests shouldn't be needed given the
payee already runs a lightning node with id known to the payer and a
working route just used to pay him.

Just like in D1, all the argument required to execute a refund could
be simply transmitted in the hop_payload to the payee.

But refund_onion could get very large, we're talking onion_packet
large, not single hop_payload large.

The refund_onion data could be implicitly contained in the filler of hop_payloads,
instead of in the individual hop_payload directed to the payee.

When calculating the route, the payer could act like it is a payment
to itself composed of 2 payments: the actual payment and the refund payment.

They have different payment hashes, but that's fine as long as they
both fit in the same onion_packet.

As examples with 20 fixed sized legacy `hop_data` payloads:

- the payment could have 10 hops and the refund 10 hops
- the payment could have 15 hops and the refund 5 hops
- the payment could have 5 hops and the refund 15 hops

### Design D5: Back to the past's limits

But that last limitation isn't practical, specially since most
payments won't actually need a refund. So D4 doesn't seem like an
improvement.

Perhaps another way to remove the need for the http server (or some
other server) is having the payer request a refund to the payee by
doing a new payment (any amount) with, again, extra payload with the
refund request data. But the refund_onion may not fit there.

I don't like this option much either, I think my favorite option here is D3.

## Questions

- Does this make any sense to you?
- How could this be improved?
- Do you have better ideas for refunds?
