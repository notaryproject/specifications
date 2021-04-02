# Key Revocation

One of the goals of Notary v2 is to build in solutions for key revocation that are easy to use and ensure that users will always use non-compromised keys. This document discusses some potential mechanisms for key revocation.

In existing systems, there are three main approaches to key revocation: automatic revocation through key expiration, key revocation lists, and providing a list of trusted keys. This document discusses some of the benefits and pitfalls of each of these techniques, and how some of these techniques are combined to provide a holistic approach to key revocation in TUF.


## Key Expiration

Adding an expiration time to every key allows keys to automatically be revoked after a certain period of time. The expiration time is usually included with the key so that it is easy for users to find. This technique does not require any action from the key holder, and ensures that users will have to refresh their trusted keys before those keys expire.

However, if a key is compromised before it expires, key expiration alone does not protect users. An attacker with the compromised key could sign arbitrary images or metadata until the key's expiration time.

Pros:
* No action required from key holder
* Simple: only requires a timestamp

Cons:
* Keys can't be revoked before expiration
* Artifacts must be re-signed after expiration


## Key revocation lists (Deny lists)

Distributing a list of revoked keys allows key holders to notify users quickly if a key is compromised or lost. Users can check against the key revocation list before using a key to verify signatures. This technique is more flexible than key expiration times, and allows for more current information about key security.

However, the user must be able to ensure that the key revocation list is accurate and up to date. If an attacker is able to replay an old revocation list, the user may continue to trust compromised keys. Therefore the distribution of the key revocation list must allow the user to verify authenticity and timeliness.

Also, for security reasons, keys cannot be removed from a key revocation list, so the list will grow larger and larger over time and may eventually have a noticeable bandwidth impact, although this can be mitigated by combining key revocation lists with keys that expire.

Pros:
* Allows key revocation at any time
* Anyone can add a key to the revocation list

Cons:
* Distribution of the key revocation list needs to be secured and verified for timeliness to prevent replays
* Additional maintenance overhead
* What to do when the revocation list query fails?


## List of trusted keys (Allow lists)

Instead of distributing untrusted keys, this method distributes a list of currently trusted keys. If a key needs to be revoked, it is removed from the list of trusted keys. This technique has the added benefit of ensuring that users have access to the new trusted key as soon as they learn of a revocation.

Similar to revocation lists, the list of trusted keys must be securely distributed and include mechanisms for the user to verify timeliness.

Pros:
* Allows key revocation at any time
* Can be combined with key distribution
* Does not require an additional query

Cons:
* Distribution of trusted key list needs to be secured and verified for timeliness to prevent replays
* Additional maintenance overhead


## Combining explicit and implicit revocation

TUF, and the [Notary TUF prototype](https://github.com/notaryproject/nv2/pull/38), use a combination of the first and third techniques to achieve both implicit and explicit key revocation. All keys have an expiration time, but may be revoked before that expiration time by replacing the key listed in the delegation for a given role. This must be done using a notion of timeliness for delegations, such as the snapshot and timestamp roles in TUF. All delegations are protected by signatures of more trusted entities, tracing back to the root keys.

This model for key revocation combines key distribution with revocation through the use of delegations. The user verifies delegations using key listed in the root metadata or a delegating targets metadata file on every update, and so receives the most up-to-date list of trusted keys, as well as which keys should be used to sign each piece of metadata.

Root keys in this system can be rotated in-band when they expire without a compromise. However, if a root key is compromised, an out-of-band process using one of the above revocation techniques may be necessary.

Pros:
* Allows key revocation at any time
* Keys will expire and so will not be used forever
* Can be combined with key distribution
* Does not require an additional query
* Also supports revocation of metadata

Cons:
* Distribution of trusted key list needs to be secured and verified for timeliness to prevent replays
