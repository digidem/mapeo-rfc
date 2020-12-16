# Notes for Implementation of P2P Mobile Upgrades

## Problems
1. New versions of Mapeo can take a very long time to reach communities' phones
2. Deploying a new version of Mapeo to a community is unreliable: can result in some devices being unable to sync with the rest

## Outcome
A software mechanism that allows an Android phone with Mapeo to consentually upgrade another Android phone with any older version of Mapeo to the newer version.

## Criteria for Success
### Discovery Protocol
1. Work reliably on a local network
2. Leave room for internet-based discovery in the future
   - the binary downloads are resumable
   - able to verify that an internet peer is authentication + authorized
3. Method cannot ever have a breaking change
4. Method impl should be totally separate from mapeo sync
### Upgrade Protocol
1. Locked protocol that won't ever need a breaking change
2. Binaries must be crytographically signed
3. Binaries must have integrity checking (checksum)
4. Binaries sent + received in a streaming fashion (don't want to keep whole binary in memory)
6. Able to sideload the APK on a usb key, update that phone, then use that phone to update other phones
   - i.e. not requiring APKs acquired WITHOUT the update mechanism to be able to send that update to others
   - does this rule out hypercore? I think so
7. Should leave room for expanding to transferring upgrade binaries for Desktop or other phone platform

## Technical Feasibility Analysis
### Backend
1. Can we create an mDNS setup that deduplicates peers? (able to embed a PeerID in info)
   - I was able to add a TXT entry that includes a 16-byte ID that the querier can read
   - able to resolve from android (using npm + multicast-dns util)
   - checked: Android's browsers (Chrome, IceCat) I tried don't support .local mDNS addresses out of the box
2. Do we need to include a hash of the file if all binaries are already signed?
   - "the Ed25519ph signature system, that pre-hashes the message. In other words, what gets signed is not the message itself, but its image through a hash function"
   - this compared to RSA, where a signature is proportional to the length of the input message (GPG)
   - might be safest, for now, to just include a hash to be safe (lower cost vs deeper investigation on if we need it)
3. What checksum/hash function to use? Doesn't need to be cryptographic.
   - I think blake2b is state-of-the-art? hypercore and cabal use it
   - secure-scuttlebutt uses sha256 and it seems ok
   - sha256 should be fine for our needs
4. Could we change this protocol to be intenet-based in the future if we wanted to without making a breaking change?
   + HTTP vs HTTPS
     - we don't need https (b/c crypto provided by signature+hash)
   + Differences between Local & Internet?
     + latency
       - doesn't really matter (very few round-trips)
     + bandwidth
       - doesn't really matter (just slower binary downloads)
     + lost connections
       - we can add byte-range support to our http server in the future /wo it being breaking change
     + attackers
       - not much they can do!
       - they can give us a bogus APK, but it can't replace the existing app /wo having the right signature
       - could an attacker send or MitM a bogus APK and cause Mapeo to be replaced (data lost, sure, but able to trick users?)?
         - G: no, not possible (diff sig -> diff app)
5. Could we use hypercore-protocol for the upgrade mechanism?
  + provides tamper-proofing + integrity checking
    - we're currently signing all apps (except linux) right now
  - means updates can ONLY be acquired through hypercore
    - android lets you access the local APK which we could send directly
    - so, using hypercore we'd need to store an extra copy IN hypercore as well as the space already used by the app
    - if a device just wants to send their local APK, it wouldn't be signed by us, so we couldn't benefit from the tamper-proofness
      - only a problem if they got the update from non-hypercore
  - conclusion: infeasible for our needs
    - a simple http protocol should meet our needs & be simpler to implement

### Frontend
1. Are we sure Android will correctly verify an APK it's downloaded from over the network?
  - G: it should, but hasn't been explicitly tested
    - G: let's proceed assuming this works (once we implement it)
- Are we sure Android can read its current APK file data?
  - G: Android process definitely can. Node should be able to, but we could stream it over IPC if we really had to.
2. Are we sure Android can determine the version # of the APK it's running?
  - G: yes 100%
3. Are we sure Android can install an APK programmatically from somewhere on local storage?
  - G: yes 100%
4. Are we sure Android can correctly REPLACE an existing Mapeo install via a programmatic APK installation ok + preserve data?
  - G: yes 100%
5. Does Android require this be stored somewhere on disk first before it can be installed from?
  - If so, can we delete the APK after the upgrade is complete?
  - yes 100%
6. Can an attacker send a bogus APK that makes Mapeo get overwritten?
  - G: no. a new signature would result in a new app that would be installed alongside Mapeo, but not over it
7. What happens to partial downloads that failed?
   - make sure these get deleted so they don't accumulate & waste space. maybe in the future maybe we can resume partial downloads

## Implementation Roadmap
1. Problem definition & feasibility analysis (**done**)
2. Clear implementation & deployment roadmap
3. Development
  1. Backend module of upgrade mechanism
  2. Integrate into Mapeo Mobile
  3. Iteratively test that the upgrade process works reliably
  4. Explore adding anonymized metrics on upgrades (#, successes, failures, errors)
4. Broader testing & QA
5. Deploy new version of MM as a non-breaking release
6. Deploy a test upgrade (minimal changes) to test effectiveness of p2p upgrades
7. Collect & analyze deployment metrics results
8. Share results & conclusions

## Phone Upgrade Process
I also have some ideas about how to structure the logical flow of the
upgrade process, but haven't drawn it up yet. The rough flow is:

1. Enter the Upgrade view
  - Starting now your HTTP webserver is live & your node replies to
    upgrade-related mDNS requsets
2. Search for peers for up to 10s on the local network
  - HTTP `GET /upgrades` against each one to see what upgrades they have
3. Pick a peer who has the newest version of Mapeo that also newer than your version
4. Download the binary + save it locally from that peer
5. Verify the binary
6. "Drain Mode": stop allowing new HTTP requests & stop responding to
mDNS upgrade queries. Wait here until all downloads being made against
you are complete (or user manually chooses to cancel them).
7. Install new APK + restart
8. Delete old APK downloaded in Step #4
