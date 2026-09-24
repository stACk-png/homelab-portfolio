# Incident: SSH Agent Lockout via Stale Cached Passphrase

## Summary

SSH access to a Docker VM began hanging indefinitely immediately after key exchange, with no error message the connection would negotiate, appear to accept the key, and then sit silently until timeout. This happened while accessing the homelab remotely, over Tailscale, from a laptop tethered to a mobile hotspot several environmental factors that all looked like plausible root causes and had to be individually ruled out.

## Symptoms

- `ssh <host>` would hang after the client log showed `Server accepts key`
- No error, no prompt, no connection just silence until the client eventually gave up
- Occurred both over LAN-style addressing and over Tailscale's assigned IP
- `ssh-agent` initially reported "could not connect to agent" before later showing keys loaded

## Investigation

Worked through the connection path layer by layer rather than guessing, ruling out each layer with direct evidence before moving to the next:

1. **DNS / reverse-lookup delay** — checked `sshd -T | grep usedns`; came back `no`. Ruled out.
2. **Multiple SSH identities being offered** — the client's `~/.ssh/` directory held four separate keypairs. Checked `~/.ssh/config`; it already scoped the correct key per host alias using `IdentitiesOnly yes`. This was good practice already in place, but the *initial* diagnostic command had been run against a raw IP rather than the configured alias, which meant the config's `Host` block never actually applied to that specific test — a good lesson in verifying which exact invocation is being tested.
3. **Network path over Tailscale / mobile hotspot (MTU, NAT, DERP relay)**  used `tailscale ping <manager-ip>` to check whether the connection was direct peer-to-peer or relayed through a DERP server (relayed connections are more prone to fragmentation issues on cellular networks). Result: direct P2P, real IP:port, ~72ms. Ruled out.
4. **Server-side auth handling** tailed the server's own auth log (`journalctl -u ssh -f`) during a live connection attempt. Every attempt logged `Connection closed by authenticating user ... [preauth]` meaning the *server* never saw authentication complete, despite the client-side log appearing to show the key being accepted. This was the key discrepancy: the client's "accepted key" message only confirms the server would accept that key if correctly signed the actual signing step, delegated to the SSH agent, was where the hang was actually happening.
5. **Bypassing the agent entirely** (`ssh -o IdentityAgent=none`) to isolate whether the agent itself was the bottleneck, separate from the network or server.

## Root Cause

The desktop's keyring/credential manager (GNOME Keyring) held a **stale, incorrect cached passphrase** for the SSH key, and was auto-supplying it to unlock the key on every connection attempt. The wrong passphrase silently failed to unlock the key, so the agent never produced a valid signature the client sat waiting on a signing operation that would never succeed, with no error surfaced to the user.

## Fix

1. Cleared the stale cached secret from the desktop keyring manager
2. Re-ran `ssh-add <key>` manually, entering the correct passphrase directly, which repopulated the keyring with a valid entry
3. Verified with `ssh-add -l` that the key was actually loaded before retesting the connection

A secondary issue surfaced during recovery: the private key file itself was accidentally deleted while clearing the keyring entry through a GUI tool that grouped the cached secret and the key under the same visual entry. Recovered by generating a new keypair, deriving its public key with `ssh-keygen -y`, and appending it to the server's `authorized_keys` verified via SHA256 fingerprint comparison (`ssh-keygen -lf`) between the client key and each server-side entry, rather than trusting a visual string comparison, to guarantee the right key was actually being trusted before attempting a real connection.

## Prevention / Lessons

- SSH's "server accepts key" client-side log line indicates the server's *willingness* to accept that key, not that authentication has completed — the actual signature step is a separate point of failure worth checking independently (`journalctl -u ssh -f` on the server, watching for `[preauth]` closures).
- Desktop keyring auto-fill for SSH passphrases can fail *silently* a wrong cached value doesn't throw a visible error, it just prevents the agent from producing a signature, which looks identical to a network problem.
- When a GUI credential manager groups a cached secret and its underlying key file under one visual entry, deleting the secret can unintentionally delete the key too. Verify what's about to be deleted before confirming.
- Always verify key identity by fingerprint (`ssh-keygen -lf`) when troubleshooting multi-key setups comparing long public key strings by eye is unreliable.
