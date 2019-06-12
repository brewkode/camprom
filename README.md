# Camprom - Build promo campaign backend in a jiffy

Campaigns involving Promocodes generally involve a few common pieces
- Promo code generation
- Promo code assignment to Entities
- Promo code tracking

## Context
Promocodes are basically a `n-digit sequence of characters` that are unique and trackable. `n` & the `charset` define the strength & space of the promocode. Higher `n` & larger `charset` means more `difficult` to brute-force.

For ex: a 3-digit promo code which allows only the digits 0-9, gives us 10^3 possibilities. i.e., from 000 to 999

## Details

### Promo code generation process
- One-time generation & persistence to DB
- Based on the promocode config(n, charset) - we can setup a code generation process and prepare for persistence

### Promo code table
- Promocode
	- code, in_use, created_at, deleted_at

### Assignment service
- Manages the promocode table
```
def getNextCode(userId)
  slot = hash(userId) % 36^6
  // block that needs race protection
  while slot in bloom_filter (membership true with defined error bound)
    lookup slot from db and check in_use
      if in_use
        slot = slot + attempt * attempt - quad probing
  set slot to in_use
  add slot to bloom_filter
  // block ends
  return slot.code
```

### User / promo_code storage
	- user, promo_code
	- inverted index of promo_code to user (if query needs)

### User signup flow
- User signup happens
- User verification succeeds
- promoCodeService.getNextCode(userId)
- Persist user, promocode


## DB Choice
Cassandra / ScyllaDB / DynamoDB / HBase - choice is based on combination of existing infra tools / maturity / team strength and community support.

## Scaling when user load explodes
- The promocode service - which assigns the promocode to user needs to scale out. Given that the underlying DB choice is horizontally scalable, we can scale out the DB too if we see hotspots in the DB. But, before doing anything, first step would be to measure where the hotspot is.

## Alternate design for Promocode service:
- Sharded service which manages its own state - promo codes & their in_use state
- CHR to decide which shard to forward the request to based on hash(userId)
- Since, data is owned locally by a CHR node shard, contention is reduced by factor of N, where N is number of nodes in CHR
