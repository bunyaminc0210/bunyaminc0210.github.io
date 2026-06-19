---
toc: true
toc_label: "Table of Contents"
toc_icon: "cog"
toc_sticky: true
title: "Setting Up an Asterisk PBX for the Corleone Family VoIP, SIP, and a Touch of Sicily"
header:
  image: /assets/images/asterisk-corleone/asterisk_header.png
  teaser: /assets/images/asterisk-corleone/asterisk_teaser.png
---

**Introduction**

A couple of weeks ago, a friend hit me up with a weirdly specific request: *"Can you set up a phone system for a small organization? Oh, and everyone uses names from The Godfather."* I laughed. Then I said yes, because honestly, who wouldn't want to build a PBX where you can dial extension `101` and hear Don Vito himself pick up?

So I spun up a Debian VPS, installed Asterisk, and got to work. This post walks you through what I did — not a deep-dive into telephony engineering, but a practical, human-readable guide to getting your own Asterisk server running, wiring it up to a SIP trunk provider like OVH, and having some fun with extensions while you're at it.

---

## Why Asterisk?

Asterisk is the open-source PBX that's been around forever. It's battle-tested, ridiculously flexible, and free. You can route calls, set up voicemail, build IVR menus ("Press 1 for the Consigliere..."), and connect to real phone numbers through a SIP trunk. For a small business or a tight-knit family operation — fictional or otherwise — it's perfect.

I went with Asterisk 20 LTS on Debian 12. Nothing fancy, just stable.

---

## Step 1: Installing Asterisk

I'm not going to copy-paste the entire installation manual here, but the gist is:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install asterisk -y
sudo systemctl enable asterisk
sudo systemctl start asterisk
```

Once it's running, you hop into the CLI to make sure everything is alive:

```bash
sudo asterisk -rvvv
```

If you see the `CLI>` prompt, you're in. Type `core show version` to double-check.

Quick tip: the Debian package ships with a sensible default config, but you'll want to back it up before messing with anything:

```bash
sudo cp -r /etc/asterisk /etc/asterisk.bak
```

---

## Step 2: The SIP Trunk — Connecting to the Real World

Asterisk handles internal calls out of the box, but if you want to dial actual phone numbers, you need a SIP trunk. I used **OVH Telecom** because their SIP trunk offering is straightforward and the pricing is decent — you pay per channel, no hidden nonsense.

After creating a SIP trunk in the OVH manager, you get:
- A **SIP username** (usually a phone number or alphanumeric string)
- A **SIP password**
- The **SIP server address** (something like `sip.ovh.fr`)

Open up `/etc/asterisk/pjsip.conf` and add your trunk:

```ini
[transport-udp]
type = transport
protocol = udp
bind = 0.0.0.0

[ovh-trunk]
type = registration
outbound_auth = ovh-auth
server_uri = sip:sip.ovh.fr
client_uri = sip:YOUR_SIP_USERNAME@sip.ovh.fr

[ovh-auth]
type = auth
auth_type = userpass
username = YOUR_SIP_USERNAME
password = YOUR_SIP_PASSWORD

[ovh-trunk-endpoint]
type = endpoint
context = from-external
disallow = all
allow = alaw,ulaw
from_domain = sip.ovh.fr
outbound_auth = ovh-auth
aors = ovh-trunk

[ovh-trunk-aor]
type = aor
contact = sip:sip.ovh.fr
```

Then in `/etc/asterisk/extensions.conf`, you define how incoming and outgoing calls are handled. Incoming calls from the trunk land in the `from-external` context:

```ini
[from-external]
exten => _X.,1,NoOp(Incoming call from ${CALLERID(num)})
 same => n,Goto(from-internal,100,1)
```

This just dumps every incoming call to extension 100 — Don Vito's desk. We'll fix that later with a proper IVR.

For outgoing calls, add a route that prefixes `9` (so dialing `9` + number sends it through OVH):

```ini
[from-internal]
exten => _9.,1,NoOp(Outgoing call to ${EXTEN:1})
 same => n,Dial(PJSIP/${EXTEN:1}@ovh-trunk-endpoint)
 same => n,Hangup()
```

Reload the config:

```bash
sudo asterisk -rx "module reload"
```

If `pjsip show registrations` shows `Registered`, you're golden.

---

## Step 3: Extensions for the Family

Now the fun part. The Corleone family needs their own extensions. I kept it simple: each person gets a three-digit extension and a softphone (or a physical SIP phone if they have one) registered to Asterisk.

Here's the roster I set up:

| Extension | Name | Role | Why This Extension? |
|-----------|------|------|---------------------|
| **100** | Don Vito Corleone | The Boss | He's the man. Everything flows through him. |
| **101** | Michael Corleone | Underboss / Successor | Quiet, calculating, always picks up on the second ring. |
| **102** | Sonny Corleone | Capo / Enforcer | Hot-headed, talks loud, probably yells into the phone. |
| **103** | Fredo Corleone | Family Associate | Nice guy, means well, but don't route mission-critical calls to him. |
| **104** | Tom Hagen | Consigliere | The lawyer. The calm voice of reason. Handles the sensitive stuff. |
| **105** | Luca Brasi | Muscle / Loyalist | Speaks in monosyllables. Nobody wants a call from Luca. |

In `/etc/asterisk/pjsip.conf`, each extension gets its own endpoint. Here's Don Vito's:

```ini
[100]
type = endpoint
context = from-internal
disallow = all
allow = alaw,ulaw
auth = auth100
aors = 100

[auth100]
type = auth
auth_type = userpass
username = 100
password = SicilianCannoli2026!

[100-aor]
type = aor
max_contacts = 1
```

Repeat the same block for 101–105, changing the extension number and password. Yes, it's repetitive — I wrote a small shell loop for it, but you get the idea.

For the dialplan in `extensions.conf`, I made sure each extension can call the others:

```ini
[from-internal]
; Internal extension-to-extension dialing
exten => _10[0-5],1,NoOp(Internal call to ${EXTEN})
 same => n,Dial(PJSIP/${EXTEN},30)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

; Voicemail access
exten => *97,1,VoiceMailMain(${CALLERID(num)}@default)
```

The `_10[0-5]` pattern matches 100 through 105. Calls ring for 30 seconds, then fall through to voicemail.

---

## Step 4: The IVR — "Press 1 for the Don"

What's a Corleone phone system without a classy auto-attendant? I recorded a simple greeting (well, a deep voice saying *"You've reached the Corleone family. State your business."*) and set up an IVR menu in `/etc/asterisk/extensions.conf`:

```ini
[from-internal]
exten => 200,1,Answer()
 same => n,Wait(1)
 same => n,Background(corleone-welcome)   ; plays "corleone-welcome.wav"
 same => n,WaitExten(10)

exten => 1,1,Dial(PJSIP/100,30)            ; Don Vito
 same => n,VoiceMail(100@default,u)
 same => n,Hangup()

exten => 2,1,Dial(PJSIP/101,30)            ; Michael
 same => n,VoiceMail(101@default,u)
 same => n,Hangup()

exten => 3,1,Dial(PJSIP/104,30)            ; Tom Hagen (legal matters)
 same => n,VoiceMail(104@default,u)
 same => n,Hangup()

exten => 4,1,Dial(PJSIP/102,30)            ; Sonny (urgent matters)
 same => n,VoiceMail(102@default,u)
 same => n,Hangup()

exten => 0,1,Dial(PJSIP/100,30)            ; Operator → Vito
 same => n,VoiceMail(100@default,u)
 same => n,Hangup()
```

So when someone calls the main number, they land at extension 200, hear the greeting, and press:
- **1** → Don Vito
- **2** → Michael
- **3** → Tom (legal / consigliere)
- **4** → Sonny (urgent / heated situations)
- **0** → Operator (also Vito — he likes to know everything)

I then updated the incoming route so external calls hit the IVR instead of Vito's direct line:

```ini
[from-external]
exten => _X.,1,Goto(from-internal,200,1)
```

Much cleaner.

---

## Step 5: Voicemail — The Classic "Leave a Message After the Beep"

Voicemail config lives in `/etc/asterisk/voicemail.conf`. Here's my setup:

```ini
[default]
100 => 1111,Don Vito,vito@corleone.local,,attach=yes|tz=Europe/Paris
101 => 2222,Michael Corleone,michael@corleone.local,,attach=yes|tz=Europe/Paris
102 => 3333,Sonny Corleone,sonny@corleone.local,,attach=yes|tz=Europe/Paris
103 => 4444,Fredo Corleone,fredo@corleone.local,,attach=yes|tz=Europe/Paris
104 => 5555,Tom Hagen,tom@corleone.local,,attach=yes|tz=Europe/Paris
105 => 6666,Luca Brasi,luca@corleone.local,,attach=yes|tz=Europe/Paris
```

The format is: `extension => voicemail-pin,Name,email,options`. I set `attach=yes` so voicemails get emailed as `.wav` attachments. When Tom's out of the office, the message still reaches him.

---

## Step 6: Firewall — Keep the Bad Guys Out

SIP attacks are constant. If you expose UDP 5060 to the internet, you **will** get scanned. I locked down the VPS with `iptables`:

```bash
# Allow SIP and RTP from OVH's IP range only
sudo iptables -A INPUT -p udp --dport 5060 -s sip.ovh.fr -j ACCEPT
sudo iptables -A INPUT -p udp --dport 5060 -j DROP

# Allow RTP ports for the softphones
sudo iptables -A INPUT -p udp --dport 10000:20000 -j ACCEPT
```

If family members need to connect from dynamic IPs (home offices, etc.), I'd recommend setting up a VPN or at least using `fail2ban` with Asterisk's built-in security log:

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
```

Asterisk's `fail2ban` config usually ships with the package, but double-check `/etc/fail2ban/jail.d/asterisk.conf` is active.

---

## Step 7: Testing Everything

Once the configs are in place, I tested the whole setup:

1. **Internal calls**: Dialed 101 from 100 — Michael picks up. Works.
2. **IVR**: Called 200 from a softphone — greeting plays, pressed 3, Tom Hagen rings. Works.
3. **Outgoing via OVH**: Dialed `9` + a French mobile number — call goes through, audio is clear.
4. **Incoming from outside**: Called the OVH number from my cell — IVR triggers, everything routes correctly.
5. **Voicemail**: Let a call ring out on 105 (Luca Brasi) — voicemail kicks in, email arrives with the `.wav` attachment.

All green.

---

## What I'd Do Differently Next Time

- **Use Ansible**: Configuring extensions manually in `pjsip.conf` gets old fast. Ansible or a small Python script would make this repeatable.
- **WebRTC support**: Would let family members take calls right from a browser. Asterisk supports it, but I didn't have time to mess with `http.conf` and TLS certs this round.
- **Encrypted SIP (TLS/SRTP)**: Right now calls are unencrypted over the trunk. OVH supports TLS — I should flip that on.
- **Better IVR recordings**: I used a TTS engine. It works, but a real voice actor reading the Corleone prompt would be *chef's kiss*.

---

## Wrapping Up

This took me a couple of evenings to get right. Asterisk has a steep-ish learning curve — the config syntax is old-school, and `pjsip.conf` is verbose — but once it clicks, it's incredibly satisfying. You're essentially running your own miniature telecom operator.

If you're building something similar, my advice: **start simple**. Get one extension working. Get one trunk working. Then layer on the IVR, voicemail, and firewall. Don't try to configure everything at once, or you'll spend hours chasing syntax errors in `extensions.conf`.

Now, if you'll excuse me, extension 102 is ringing — Sonny wants to know why Fredo dropped a call again.

---

*Got questions or want to share your own PBX setup? Hit me up on [Twitter/X](https://twitter.com/bunyaminc0210) or drop a comment below. I'm always curious how others are running their home-lab phone systems.*
