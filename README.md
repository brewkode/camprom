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
```scala
def getNextCode(userId) = {
  val maxSlots = charsetSize ^ n
  var slot = hash(userId) % maxSlots
  var res = setAndGet(slot, inUse=false) // Should leverage CAS
  while not res.success
  	slot = (slot + attempt * attempt) % maxSlots // quad probing
	res = setAndGet(slot, inUse=false)
  return res.code
}
```
- The above assignment should adapt to a free-list based mechanism when promocodes slots are more than X% full

### User / promo_code storage
- user, promo_code
- inverted index of promo_code to user (if query needs)

### User signup flow
- User signup happens
- User verification succeeds
- promoCodeService.getNextCode(userId)
- Persist user, promocode


## DB Choice
Cassandra / ScyllaDB / DynamoDB / HBase - based on combination of existing infra / maturity / team strength and community support.

## Scaling when user load explodes
- Before doing any improvements, first step would be to measure bottlenecks.
- Potential areas of bottleneck - the service layer, code assignment logic and the DB
- The promocode service - can horizontally scale out, since it doesn't have any local state
- Given that the underlying DB choice is horizontally scalable, we can scale out the DB too if we see hotspots in the DB 
- The contention will go up since there are now more code assignments (potentially) happening. Evolution to free-list based mechanism will mitigate some of this.

## Alternate design for Promocode service:
- Service with handleOrForward semantic - to allow request localization and avoid contention from many service hosts. Shard level flag to signify if a particular shard is fully taken or not.
