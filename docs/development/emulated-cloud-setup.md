# Idea: emulating the cloud during initial pairing, to remove the last internet dependency

**Status: this is an unclaimed idea, and not something thats being worked on.** This is a write-up for anyone who wants to pick it up, not a roadmap item. This is something I'm not willing to troubleshoot and test.

## The gap this would close

Today, `homeconnect_local_hass` is internet-free for everything *except* one step: getting the appliance's profile file. This currently involves either the [Home Connect Profile Downloader](https://github.com/bruestel/homeconnect-profile-downloader) or this integration's own OAuth setup path (`hc_cloud_api.py`), both of which require logging into an account on the Home Connect cloud to pull that data. Once you have it, the appliance is controlled entirely over its local WebSocket, the cloud is never involved again, and you can disable the appliance's "Allow Cloud Connection" switch.

So the *only* remaining internet dependency is a one-time step during setup.

## The idea

A brand-new (or factory/network-reset) BSH appliance's very first WiFi/cloud account pairing is itself a network handshake with BSH's cloud, the appliance registers itself to a Home Connect account, and as part of that, its local encryption keys get established/reported.

If that handshake were pointed at a **local, fake cloud server** instead of BSH's real one, the same way `rethink` emulates LG's AWS backend for LG ThinQ appliances that have no local API at all, the appliance would complete pairing believing it registered to a real account, and the local keys could be captured directly from that handshake. No request would ever reach BSH's actual servers.

Combined with the fact that ongoing operation is already 100% local, this would mean **the entire appliance lifecycle, setup included, never touches BSH's cloud.** That's the appeal over the current OAuth-cloud-API approach: it makes the last "requires internet" optional instead of mandatory.

Since the web socket is the exact same regardless of whether it was setup with the fake or real cloud, this means that this integration can still cover both setup methods. It also benefits other Home Connect Local integrations like the ones for [Homey](https://homey.app/en-us/app/codes.lucasvdh.homeconnect/Home-Connect-(Local)/test/) or [OpenHab](https://www.openhab.org/addons/bindings/homeconnectdirect/).

## Why "Allow Cloud Connection" may still matter

>[!important]
> Whether this behavior will actually happen is unknown until its tested. Treat this like a theory.

Under this scheme, the appliance's own firmware believes it's legitimately registered to a Home Connect account (it doesn't know the account was fake). If its real cloud-connection toggle is left on afterward, it will periodically try to check in with BSH's *real* servers, which have no record of that registration. That mismatch is the whole scheme's exposure, BSH's servers seeing keys/credentials that don't match any real account is the one way this could get noticed or acted on. Turning "Allow Cloud Connection" off before the fake cloud is shutdown means the appliance never talks to the real cloud again, so there's nothing to notice. This may be a hard requirement of the design, not a nice-to-have.

## What's actually unknown (this is the real scope of the project)

None of the following has been reverse-engineered yet. This is not a "wire up rethink for BSH" task, it's a from-scratch protocol investigation, and the answer to the first point below could make the whole idea infeasible:

1. **TLS certificate pinning.** If the appliance validates BSH's real certificate chain during pairing and refuses to proceed against anything else, a fake local server can't complete the TLS handshake at all without getting the appliance to trust a custom root CA, which would require firmware-level access and defeats the point. This needs to be tested empirically (MITM proxy, e.g. `mitmproxy`, in front of a factory/network-reset appliance's first-time setup) before anything else here is worth building. If BSH pins certs strictly, this whole approach is likely dead on arrival.
2. **What hostname(s)/endpoints the appliance is hardcoded to contact** during initial pairing, and how it resolves them — needed to know what a local fake server needs to answer to, and how to redirect the appliance's traffic to it (local DNS interception on the network the appliance provisions over, same mechanism rethink uses for LG).
3. **The actual registration/pairing wire protocol**: request/response shapes, what triggers key generation vs. key upload, timing/retry behavior. This only comes from a live packet capture of a real pairing session.
4. **Failure modes.** If a fake response doesn't match what the appliance's firmware expects at any step, the appliance could error out, retry indefinitely, or get stuck in a half-provisioned state. Worth assuming a bricked pairing (recoverable by factory reset, but still) is a real possibility until proven otherwise, and testing on a spare/non-critical appliance if at all possible rather than a daily-driver one.

## Before making the software

Before writing any server code: capture a full packet trace (via a controlled AP running `mitmproxy` or even just `tcpdump`) of one already-owned appliance's factory/network-reset re-pairing process. That alone answers the certificate-pinning question, which determines whether the rest of this is worth attempting.

Also determine the WI-FI password, here's what I've confirmed so far
SSID: HomeConnect
Password: HomeConnect
I haven't confirmed if the password is this, but the SSID is confirmed. Seeing what the android app sends as the password to the appliance may be able to confirm it.

## Important appliances quirks.

A Home Connect Appliance is setup in a weird way. For some appliances like the Thermador T36IF905SP, pairing to WI-FI and connecting it to the app are two separate processes. But for most dishwashers like the Bosch SHE43DM5N it's one process. Some appliances like the Thermador PRG486WDH have support for WPS which may just connect it to wifi but not to the app. It's unknown how or if the appliance contacts the cloud in this phase, but what's known is that it doesn't open its web socket.

Future BSH Appliances will come with Matter which involves both wifi and bluetooth. The app may have to account for a bluetooth pairing process in the future.

For some reason a Home Connect Appliance can be registered to multiple accounts. Not sure how this would handle it.

## Potential network commands

>[!important]
> Only confirmed on one appliance so far. Treat everything here as a lead, not as documented behavior.

[@moerk-o](https://github.com/moerk-o) found that an already-paired appliance can be moved to a different WiFi network over its local WebSocket, with no hotspot, no network reset and no Home Connect app ([discussion #105](https://github.com/vemboy200/homeconnect_local_hass/discussions/105), [script](https://gist.github.com/moerk-o/5f3835fffee3ef1ad1e113676ef9bf73)). Tested on a Siemens HB876G8B6/75 oven, firmware 2.11.6.13174, AES connection (`ws://`, port 80), services `ro` 1, `ei` 2, `ci` 2, `ni` 1.

All of these go over the existing, authenticated WebSocket, after the handshake:

| Resource | Action | Result |
| --- | --- | --- |
| `/ni/config` | GET with payload `[{"interfaceID": 0}]` | Returns `interfaceID`, `ssid`, `automaticIPv4`, `automaticIPv6`. Never returns the password. A GET without a payload returns 400. |
| `/ni/config` | POST | Changes the WiFi network (see below). |
| `/ni/info` | GET | Shows the network the appliance is currently on. |
| `/ci/wifiNetworks` | GET | Returns a WiFi scan (SSID and RSSI per network). |
| `/ci/wifiSetting`, `/ci/wifiSetting2`, `/ci/networkDetails`, `/ci/networkDetails2` | GET | 404 on this appliance. The protocol notes describe them for `ci` version 1, and this oven speaks version 2. |

The write that moves the appliance:

```json
// POST /ni/config
[
  {
    "interfaceID": 0,
    "ssid": "NewNetwork",
    "passphrase": "…",
    "automaticIPv4": true,
    "automaticIPv6": true
  }
]
```

The password field is `passphrase` (`psk` is rejected with 400). The appliance answers 200, closes the WebSocket, leaves its current network and joins the new one. About 30 seconds later it was reachable on the new network with a new IP, and this integration found it again through mDNS and updated the host in the config entry by itself. Writing the current network back (same SSID and password) is a safe way to test whether an appliance accepts the write at all: it drops briefly and rejoins the same network.

So unlike first-time setup, the appliance never goes into a pairing mode. It stays on its current network the whole time, receives the new network's details over the local connection, and then switches on its own.

### What this means for this idea

- **It doesn't replace the hotspot step.** The write needs an already-authenticated WebSocket, so it needs the appliance's encryption key. A factory-reset appliance on its "HomeConnect" hotspot hasn't been paired to an account yet. Either the app hands over the WiFi details some other way during that step, or the appliance already has a key at that point. If it's the second, that changes a lot here: the key would come from the appliance, not from the cloud handshake. Capturing what the app sends over the hotspot answers this.
- **It's probably what the app uses for its own network settings.** The app's "Extended network settings" refuse to work unless the phone is on the same network as the appliance ("Network change unavailable"), which fits a local-only command like this. Not confirmed, since the app's traffic hasn't been captured.

### Open questions

- **What happens with a wrong password?** Unknown. The appliance might fall back to the last network that worked, or it might be left with no network until it's fixed on the appliance or in the app. Not tested yet. The official app has the same risk during first-time setup: in [this video](https://youtu.be/hpe7zislhqQ?t=1305) (21:45-23:08) a wrong WiFi password leaves setup stuck, "you do not get to retry the password", and setup has to be redone. That shows neither the app nor the appliance checks the password before trying to join, but not whether an appliance that's already on a working network falls back to it.
- **Do other appliances behave the same?** One appliance, one firmware, `ci` version 2 without an `iz` service. Appliances with `ci` 3 and `iz` (e.g. some dishwashers) may expose this differently or not at all. The script's read-only mode is the safe way to check, since it changes nothing on the appliance.

### Not an integration feature

Changing the WiFi network falls in the same category as the commands this integration deliberately leaves out (factory reset, network reset, WiFi deactivation): a mistake can leave the appliance unreachable. It's documented here as protocol knowledge for this project, not as something to expose in Home Assistant. It's also a reminder of why the Full profile export (which contains the encryption key) stays gated: anyone with that key can move the appliance off your network over the local connection.

## How the software would work

During the pairing process the app would be both the cloud and phone in parallel.

- Phone = App emulating the phone during the process
- Cloud = App emulating the cloud during the process

```
Phone --(shares home WiFi creds via appliance's own hotspot)--> Appliance
                                                                    |
                                                                    v
                                                        Appliance joins home WAP
                                                                    |
                                                                    v
                                                    Appliance <==TLS==> Fake Cloud
                                                     (registers, reports/negotiates
                                                            local keys)
                                                                    |
                                                                    v
                                                  Fake Cloud has everything needed
                                                    for the appliance's profile
                                                                    |
                                                                    v
                                                  Profile file lands on the user's
                                                          computer
                                                                    |
                                                                    v
                                             Imported into homeconnect_local_hass
                                                                    |
                                          +-------------------------+-------------------------+
                                          |                                                   |
                                          v                                                   v
                                4.1 Phone/HA side                                    4.2 Cloud side
                        Phone/HA -> appliance's local WS                      Fake Cloud (still connected)
                         using keys from the profile                           sends disable-cloud itself
                    (what the integration can do right now)                    (harder: payload unknown)
                                          |                                                   |
                                          +-------------------------+-------------------------+
                                                                    |
                                                                    v
                                                    Appliance's cloud connection off
                                                                    |
                                                                    v
                                                       100% local from here on
```

### Some stuff to point out

- There has to be a fake account.
- It's unknown if the appliance sends the full profile file to cloud or data that the cloud then refactors into a profile file.
- The good news is this setup **does not** override the appliance's firmware, so once the software is ready for the public it's not super risky.

### Another idea

After the software is developed and thoroughly tested, to simplify setup, the app could also go in a pairing mode. During this pairing mode Home Assistant can connect to the computer which has the profile file, and take it and then connect to the appliance.

## Prior work

- [rethink](https://github.com/anszom/rethink): the LG ThinQ equivalent. Their situation is actually harder than BSH's: LG appliances have no local API at all, so rethink has to keep a fake cloud running permanently to handle ongoing control. Here, the fake cloud would only ever need to run for the few minutes of initial pairing, everything after that is already native and local.
