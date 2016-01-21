# xing-auth-protocol

***Note: This is not where to find information about the current Xing authorization system. See the [Xing book](https://xingframework.gitbooks.io/the-xing-framework/content/) for information about how authorization works in Xing currently.***

Proposal for a future authorization system.

To be clear: this a proposed idea, for discussion and to get the ideas out of my head so I can work on other things without distraction. I have been thinking about this for quite a while, and it's pretty well considered.

What I'm suggesting is a Macaroons based authorization system, using an SRP based local discharge to handle authentication.

For reference:
* http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/41892.pdf
* https://www.ietf.org/rfc/rfc2945.txt
* http://srp.stanford.edu/ndss.html
* https://github.com/ecordell/macaroon-session-example

As an aside, there are several libraries that already implement the components of this proposal. While the synthesis of these components seems to be novel, and I would want to have them reviewed more publicly, this isn't a roll-your-own solution.

## Parts of the system

The base frontend app - a normal (Xing) JS SPA.

A standalone JS file, accessed by a normally invisible IFrame, which handles the client side of authorization processes.

A Rack middleware, to interface with Warden/Devise, and to handle macaroon verification.

Rails resources to handle backend exchanges as indicated.

## Operations

### General authz flow

This is explained from base principles, with notes regarding the involved cryptographic components.

The base app makes a normal RESTful request for a token to the backend. The request includes a user identifier (e.g. email), as well as a transient public key.

[SRP]
To get the client's transient public key, the base app makes a request to the auth iframe. The iframe chooses a random number `a`, and computes `g^a % N` - where `g` is a public base, and `N` is a public prime number. The result is the client's public key, and the auth iframe reports the value to the base application. The base app uses this value along with the user's email to request a token from the backend.

[Macaroons]
The backend app generates a random nonce value uses a secret server key to "seal" it, using a keyed HMAC. It now has a token that (effectively) looks like "random value" + the HMAC of the random value. The token is an example of a macaroon; the smallest possible example.

Other values will be added to this macaroon by stripping off the HMAC, adding the new value, and then appending an HMAC keyed with the *stripped HMAC* - any entity can safely perform this process. However, it cannot be "unrolled": you can't remove a value from the token, and you need the original secret key to verify the chain. The process of adding extra values to the macaroon is called "attenuating" with new "caveats."

The server can immediately attenuate the macaroon with a maximum session length, as well as authorization information like "member of groups x,y,q"

[SRP]
The base app looks up this user, finds an (opaque) verifier token and the salt value used to generate this verifier for the user. It uses the verifier `v` and a random number `b` to compute `(v + g^b) % N` (`g` and `N` are as above), which is the server's public key.

[Macaroons]
The server attenuates the macaroon with a "third-party caveat" - something that the client must arrange to prove before the macaroon can be considered valid. Such a caveat has a value that identifies what must be proved, as well as some opaque data that a prover can use to determine how to do the proof. In this case, the caveat is "prove that you are <user>" and the data is the salt and public key for the SRP protocol.

The server returns this macaroon to the client. Without the proof of authentication, it's useless - the macaroon could in principle be sent over an unsecured channel.

The base FE app registers the macaroon with the auth iframe JS.

To make an authorized request, the base FE app makes a request of the auth iframe. The iframe may indicate that it needs to collect a password from the user (on first request, or after a configurable expiration), in which case the base FE app needs arrange for the IFrame to be made visible, so that the user can enter their password. The auth iframe will signal when the password has been collected and the FE app can hide it again.

[SRP]
The auth iframe extracts the salt and server's public key from the caveat in the macaroon. It uses the salt and the password provided by the user to do: `x = hash(salt | hash(user_id | ":" | password))`. From there, it can use the server's public key `B` to compute `u` as a the first 32 bits of `hash(B)`, and then `(B - g^x) ^ (a + u * x) % N`. This result is the shared key that the client and server will use to complete the exchange. The server computes it as `(A * v^u) ^ b % N`. 

(I know: the crypto math here is clear as mud, but it's _essentially_ a Diffie-Hellman exchange, which some extras thrown in so that the client side needs to know a password.)

[Macaroons]
The auth iframe uses the SRP shared key as it's own secret key to produce a new macaroon: random nonce value + keyed HMAC. Then it attentuates this macaroon with a very short expiration (about 30 seconds or so? The balance is between client side performance versus security in the face of an exfiltrated key) and performs a step called "binding" where it attaches this macaroon to the macaroon sent by the server in order to "discharge" the caveat. By performing the client part of the SRP exchange, we've proved that we're the user in question, and the client side can limit how long that proof lasts.

The auth iframe returns the bound discharged macaroon to the base FE app, along with a report of how long it will last. The base FE app can use the reported duration to schedule refreshes of the discharge to maintain a smooth authorized data flow. 

The base FE uses the bound macaroon in an Authorization: Bearer token in requests.

The backend server creates a "verifier", something akin to how the macaroon is built. A verifier has a list of facts that it confirms. For instance, the expiration times of any macaroons must be in the future. One useful but counterintuitive feature of macaroons is the relationship between verifiers and macaroons: all the clauses of a macaroon have to be verified for the macaroon to be accepted, but verifier facts can be missing from the macaroon.

So, the verifier starts from basic ideas like expiration and the correct discharge of the user caveat, but also includes the controller and method of the request, as well as special facts constructed from e.g. user roles and the resource itself.

[SRP]
As a particular note, in order to verify the discharge of the authentication proof, the server needs to look up the user again and recall it's part of the session key computation. This is the key reason that the server need to set an expiry on the macaroons: if the database is exposed, any "live" macaroons are in danger; setting an expiration (which the auth iframe could refresh without intervention in a running app) limits this window. Possibly, this might be mitigated by keeping track of the session keys in another data store - conceivably in server memory for a single server, or Redis.

### Create a password for an account

Either during initial account creation or when changing password.

The base application signals the auth iframe, and updates styling to display the iframe, which contains password change interface. User completes a password change UI, and the IFrame computes an [SRP] authorization verification value, which it makes available to the base application. Base application submits a request to the backend, which authorizes the request and records the identifier.

## Assumptions and threat models [WIP]

* Random numbers from Webcrypto
* Sychronizing clocks
* HTTPS required
* User not responsible
* Webcrypto vs. libnacl

https://tonyarcieri.com/whats-wrong-with-webcrypto

I'd argue that we're essentially generalizing on Arcieri's "protecting the server's resources" use case here.

## Discussion

From the SRP RFC:
```
   SRP has been designed not only to counter the threat of casual
   password-sniffing, but also to prevent a determined attacker equipped
   with a dictionary of passwords from guessing at passwords using
   captured network traffic.  The SRP protocol itself also resists
   active network attacks, and implementations can use the securely
   exchanged keys to protect the session against hijacking and provide
   confidentiality.

   SRP also has the added advantage of permitting the host to store
   passwords in a form that is not directly useful to an attacker.  Even
   if the host's password database were publicly revealed, the attacker
   would still need an expensive dictionary search to obtain any
   passwords.  The exponential computation required to validate a guess
   in this case is much more time-consuming than the hash currently used
   by most UNIX systems.
```

In other words: using SRP reduces the vulnerability introduced by a lost database from "until everyone changes their passwords" to e.g. 20 minutes of the freshest macaroon expiring.

The use of macaroons means the the SRP protocol can be carried in the tokens themselves, as opposed to a separate networked protocol exchange. Macaroons also mean the the authorization process can be implemented as a completely separate concern (although initially I imagine them being managed within the controllers or via role objects). Testing authorization would become a separate testing process, caching could be inserted into the Rack middleware chain below authorization - even implemented in a dedicated proxy.

Even longer term, macaroons make a number of possibilities available. For instance, the base FE app could add its own caveats in order to produce "sharing links", or the "local discharge" iframe could be replaced with an Oauth provider.

### Comparison to other schemes
Compared to the existing token auth scheme, I'd expect a more performant server side implementation. (devise-token-auth was consuming 15% of some of our most complex response generation, by using an (imo) inappropriate primative) The maintenance of session seems like it would be smoother. Integration with Xing's REST assumptions should be easier.

Compared to JSON Web Tokens (jwt): jwt might provide better performance: as I understand them, you can completely store the user's information cryptographically in the token, such that you can skip a database retreival to verify the token. Note that something similar could be accomplished by using Macaroons without the SRP discharge component. The criticism I've seen of JWT is that they're very hard to do correctly, and (all? really?) of the existing implementations have problems in the implementation. At the end of the day, transitioning from our existing token scheme to JWT might be easier.

Compared to simple macaroons (i.e. without the SRP piece): before we contemplate using JWT, I'd suggest using simple macaroons: request a token, supply a password, get a macaroon that describes you and your privileges on the system. The behavior would be very similar to JWT, including being able to verify without a db lookup (unless, e.g. we're checking ownership of resources, but that'd be true of JWT.) The chief advantage is that Macaroons are a better crytographic design, with future flexibility being an auxiliary advantage.

In each case, obviously, we trade a reduced browser-side component for the (existing) database vulnerability. On the one hand, being able to reassure a client "no, it doesn't matter, they can't use those passwords to do anything on the site" would be nice - on the other hand, if an application database were stolen, e.g. PII would be much more important than password access.



