# Online Platform ID Certificate Signing

## Players

M: Manufacturing, played by Phil

N: Narrator, played by Alex

O: Operations, played by Ian

## Context

N: Before we ship a big board (Gimlet, Sidecar, or PSC), we assign it
a unique _platform identity_, which is represented by a signed X.509
certificate (see RFDs 273, 303). The private key used to sign those
certificates is quite sensitive: if someone were to steal it, they
could potentially mint fraudulent certificates for non-Oxide hardware.

N: Right now, the key used to sign platform identity certificates lives
in a YubiHSM (a fancy, auditable, more expensive YubiKey) that an Oxide
employee has to physically carry to the manufacturing station when we're
ready to issue certs, unlock it with a password, keep tabs on it at all
times, and take it home and secure it when they're done signing. This
process worked well enough to issue certs for our first few racks, but
is inconvenient and risky. It's also basically impossible to hand off to
our manufacturing partner without introducing even more inconvenience and
risk.

N: Our solution, which we're going to demo today, is to use the Online Signing
Service, aka "Permission Slip" (RFD 343) to manage and sign with the
platform ID certificate signing key. This is an intermediate key, not a
root, but _its_ certificate will need to be signed by an offline root;
we're going to be doing that in a ceremony next week. Once that initial
signing is done, though, the manufacturing stations will be able to use the
OSS to safely sign platform id certs without physical access to an HSM.
And that means that we can hand off that signing process to our manufacturing
partener _without_ entrusting them with the key itself, saving both of us
a lot of headaches. That's the theory, anyway.

## Preparation

M: To set things up for the demo today, and for the real signing at
Benchmark in the future, we need to do some prep work. First, we need
to create keys in the Online Signing Service to do the signing.
Then we need to have certificates for those keys signed by some
root keys. Those certificates then get imported back into the OSS.

M: For today's demo we're using a sandbox deployment of OSS, which runs
a real instance of Permission Slip with real keys stored in KMS, but
obviously *not* the production keys. So everything here is real, just
the key and signature _bits_ will be different in production. The root
key here is on a YubiHSM just like the real root key, but it's not locked
in a bank, it's just on Phil's desk.

O: Today we'll run through a "mock ceremony" where we signed certificates
for the keys in the sandbox OSS with that development HSM. Next week we'll
do approximately the same thing, just with higher stakes; those keys have
*meaning* by virtue of being locked away, only used in carefully scripted
ceremonies, and so on.

## Authentication

N: So, the first step in getting a signature on anything from Permission Slip
is authentication, where we prove that we represent a certain identity,
human or otherwise. Previously permslip supported only the same kind of
OAuth device authentication flow that we use for the Oxide CLI; you'd do a
little dance where you log in to an IdP like Google or GitHub, verify a
little code, and then have an access token that's good for an hour or so.

N: But for the manufacturing station, that won't work. For one thing, we don't
have a web browser there, and aren't going to. For another, even if we did,
who would a Benchmark employee authenticate as? We'd have to somehow
synchronize with _their_ IdP or authentication server, which we don't know
what it is, or how to talk to it.

N: So after much batting around of ideas, we settled on a good, old idea: SSH
public key authentication. Some of you may remember using such a thing at
JoyEnt. Josh (or maybe someone else?) wrote a little piece of code that
could talk to the SSH agent, and ask it to sign authentication tokens.
He's re-written that in Rust, and uses it in Facade (the main manufacturing
station software), and now Permission Slip uses it, too.

N: The idea is that instead of authenticating as a person, you
authenticate "as" a certain public key by proving that you have access
to the corresponding private key. For the manufacturing stations,
those keys can be in a YubiKey that stays with the station. Loss of
one of those keys is not the end of the world, since they're authentication
keys, not signing keys. We'd just revoke the corresponding public key,
buy a new one, import its public key, and get back to our day.

N: Since Phil's going to be pretending to be driving a manufacturing station,
I'll ask him for the SSH key he'd like to authenticate as. Phil, could you
please run `ssh-add -L` and paste your public key to the chat?

M: Sure, it's: `ecdsa-sha2-nistp256 AAAA...`

N: Thanks! So, to import that, I'll run:

```
echo 'ecdsa-sha2-nistp256 AAAA...' | permslip import-ssh-key -
```

N: Ok, that's an error, but it tells us what to do. What this means is that
the request to import this public key hasn't been approved, which is
permslip's way of authorizing requests. We'll get to the signing
authorization in a second, but this shows that there's protections at
this level, too.

N: So what we do is we re-run the command, prefixed with `approve --`, just
like it said:

```
echo 'ecdsa-sha2-nistp256 AAAA...' | permslip approve -- import-ssh-key -
```

N: And now we can re-run the import itself, and it goes through.
Having registered that key, Phil can now use `permslip --sshauth`,
which will use the corresponding key to sign authentication tokens.
The manufacturing station software will only use this kind of
authentication.

## Authorization

N: Ok, now that we know how we'll be authenticating the manufacturing
station, we can talk about how we'll be authorizing requests for signatures
on platform ID certs. The idea is to mitigate against both accidental
and malicious misuse of the authentication credentials by constraining the
valid signing requests as much as possible. The way we do this is to
require ahead-of-time approval of a batch of related signing requests,
all within a certain time interval:

```
permslip approve-batch \
  --single-use \
  --start='2024-03-29T18:00:00Z' \
  --end='2024-03-29T19:00:00Z' \
  --constraints='C=US,O=Oxide Computer Company,CN=PDV2:PPP-PPPPPPP:RRR:SSSSSSSSSSS' \
  --constraints='C=US,O=Oxide Computer Company,CN=PDV2:PPP-PPPPPPP:RRR:TTTTTTTTTTT' \
  -- sign $key
```

N: A couple of things to notice here. First, these approvals are
"single-use", which means once they're used to authorize a particular
signing request, they can't be used again; this prevents us from
accidentally (or someone else from maliciously) granting multiple
certificates to the same device. Second, they're bounded in time.
And finally, they're "constrained" to a certain set of distinguished
names, which include serial numbers of the devices to be certified.

N: And with that, we'll hand things off to our manufacturing partner.

## Signing

M: Thanks. We use a set of tools called `dice-utils` to program the
platform ID stuff during manufacturing. Moving to online signing just
means that instead of shelling out to `openssl` to sign the certs,
it shells out to `permslip` instead.

M: ...

```
dice-mfg sign-cert \
    --cert-out=platform-id-request.crt.pem \
    platform-id-request.csr.pem \
    permslip 'Platform Identity Signer ...'
```

M: Now, we verify ....

## Failing to sign

M: If we try to run `dice-mfg sign-cert ...` again, it fails because
the approval was already used. If we were to leave off the `--sshauth`,
it would fail because the authentication method would be wrong.
If we hadn't used the approval now but had waited an hour, it would have
failed because the approval would be expired; and so on. The point is
that the ways and chances of getting a signature are highly constrained.

## Conclusion

N: So, that's our demo. We've shown that we can have the online signing
service manage intermediate platform ID certificate signing keys, approve
bounded batches of specific serial numbers for platforms to certify, and
use those approvals to sign certs during the manufacturing flow.
