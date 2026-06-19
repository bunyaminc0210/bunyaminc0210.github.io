---
toc: true
toc_label: "Table of Contents"
toc_icon: "terminal"
toc_sticky: true
title: "Claude Code : From API Setup to Cisco Network Automation"
header:
  image: /assets/images/claude-network/claude_network_header.png
  teaser: /assets/images/claude-network/claude_network_teaser.png
---

**Introduction**

Claude Code is Anthropic's CLI agent that turns natural language into working code, shell commands, and infrastructure changes — directly from your terminal. But what many people don't realize is how flexible and powerful Claude Code has become when you chain it with the right environment.

In this article, I'll walk you through three things:

1. **Configuring Claude Code** with different LLM backends — Anthropic, OpenAI, and DeepSeek
2. **Building a virtual network lab** with a Cisco switch using GNS3
3. **Using Claude Code to automate network configuration** — SSH into a Cisco switch, create VLANs, set up inter-VLAN routing, OSPF, and more

By the end, you'll see that Claude Code can now do things that used to require a network engineer with years of CLI muscle memory. Let's go.

---

## Part 1: Configuring Claude Code with Different API Providers

Claude Code is an agentic CLI tool. By default, it uses the Anthropic API with Claude Opus 4.8. But with the right configuration, you can point it at other providers like OpenAI or DeepSeek.

### 1.1 Installing Claude Code

```bash
# macOS / Linux
npm install -g @anthropic-ai/claude-code

# Or via Homebrew
brew install claude-code

# Verify the install
claude --version
```

### 1.2 Default Configuration — Anthropic API (Native)

The native setup is straightforward. Claude Code resolves credentials in this order:

1. `ANTHROPIC_API_KEY` environment variable
2. `ANTHROPIC_AUTH_TOKEN` (OAuth token from `ant auth login`)
3. Active profile from `ant auth login`

```bash
# Option A: API Key
export ANTHROPIC_API_KEY="sk-ant-api03-..."

# Option B: OAuth (no static key to manage)
ant auth login   # Opens browser, exchanges for token
```

**Model selection** — Claude Code uses `claude-opus-4-8` by default with adaptive thinking and `effort: "xhigh"`. You can configure the model in `~/.claude/settings.json`:

```json
{
  "model": "claude-opus-4-8"
}
```

Available Anthropic models for Claude Code:

| Model | Best For | Context | Input $/1M | Output $/1M |
|-------|----------|---------|-----------|-------------|
| `claude-opus-4-8` | Most capable, best for complex coding | 1M | $5.00 | $25.00 |
| `claude-sonnet-4-6` | Speed/intelligence balance | 1M | $3.00 | $15.00 |
| `claude-haiku-4-5` | Fast, cost-effective | 200K | $1.00 | $5.00 |

### 1.3 Using OpenAI-Compatible APIs (DeepSeek, OpenAI, etc.)

Claude Code can be configured to work with any OpenAI-compatible API endpoint. This is useful when:

- You want to use DeepSeek V3/R1 for cost savings
- You need OpenAI's GPT-5 for specific tasks
- You're running a local model via Ollama or vLLM

**Configuration via `settings.json`:**

```json
{
  "apiKeyHelper": "echo $OPENAI_API_KEY",
  "baseUrl": "https://api.openai.com/v1",
  "model": "gpt-5"
}
```

For **DeepSeek** specifically:

```json
{
  "apiKeyHelper": "echo $DEEPSEEK_API_KEY",
  "baseUrl": "https://api.deepseek.com/v1",
  "model": "deepseek-chat"
}
```

For **local models via Ollama**:

```json
{
  "apiKeyHelper": "echo ollama",
  "baseUrl": "http://localhost:11434/v1",
  "model": "llama3.1-70b"
}
```

**Setting environment variables (recommended for CI/CD):**

```bash
# For OpenAI
export OPENAI_API_KEY="sk-..."

# For DeepSeek
export DEEPSEEK_API_KEY="sk-..."

# For Anthropic (native)
export ANTHROPIC_API_KEY="sk-ant-api03-..."
```

### 1.4 Claude Code `settings.json` — Key Options

The configuration file lives at `~/.claude/settings.json` (global) or `.claude/settings.json` (per-project). Here are the most useful options:

```json
{
  "model": "claude-opus-4-8",
  "baseUrl": "https://api.anthropic.com/v1",
  "apiKeyHelper": "echo $ANTHROPIC_API_KEY",
  "permissions": {
    "allow": [
      "Bash(curl:*)",
      "Bash(npm:*)",
      "Bash(git:*)",
      "Bash(ssh:*)"
    ],
    "deny": []
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "command": "echo 'Command executed'"
      }
    ]
  }
}
```

**Key settings explained:**

| Setting | Purpose |
|---------|---------|
| `model` | The model ID Claude Code uses |
| `baseUrl` | API endpoint (change for third-party providers) |
| `apiKeyHelper` | Shell command that prints the API key to stdout |
| `permissions.allow` | Pre-approved tool patterns (no permission prompts) |
| `permissions.deny` | Blocked tool patterns |
| `hooks` | Commands that run before/after tool calls or events |

### 1.5 Testing Your Configuration

```bash
# Quick test — ask Claude Code a simple question
claude -p "What model are you running on? Reply with just the model name."

# If you see "claude-opus-4-8" — Anthropic native is working
# If you see "deepseek-chat" — your DeepSeek config is active
# If you get an auth error — check your API key and base URL
```

---

## Part 2: Building a Virtual Network Lab with Cisco Switch

Now that Claude Code is configured, let's build a virtual environment where it can actually do network automation. We'll use **GNS3** — the most popular open-source network emulator.

### 2.1 Installing GNS3

```bash
# Ubuntu / Debian
sudo add-apt-repository ppa:gns3/ppa
sudo apt update
sudo apt install gns3-gui gns3-server

# macOS
brew install gns3

# Docker-based (headless server — best for automation)
docker run -d --name gns3-server \
  -p 3080:3080 \
  -v /opt/gns3:/data \
  gns3/gns3-server
```

### 2.2 Obtaining Cisco Images

You need Cisco IOS images to emulate switches and routers. **Legal options:**

- **Cisco CML (Cisco Modeling Labs)** — Free tier available via Cisco DevNet. Comes with IOSv, IOSvL2, IOS-XE images.
- **Cisco DevNet Sandbox** — Free 4-hour reservations with pre-built topologies.

Once you have the images, import them into GNS3:

```bash
# Place images in the GNS3 images directory
mkdir -p ~/GNS3/images/IOS
cp ~/Downloads/i86bi-linux-l2-adventerprisek9-15.2d.bin ~/GNS3/images/IOS/
cp ~/Downloads/i86bi-linux-l3-adventerprisek9-15.4.2T.bin ~/GNS3/images/IOS/
```

Then in GNS3 GUI: **Edit → Preferences → Dynamips → IOS Routers → New** and follow the wizard.

### 2.3 Our Lab Topology

We'll build this topology:

```
┌─────────────────────────────────────────────────────┐
│                   GNS3 Lab                           │
│                                                      │
│   ┌──────────┐                    ┌──────────┐       │
│   │  SW-01   │                    │  SW-02   │       │
│   │  Cisco   │◄─── Trunk ────────►│  Cisco   │       │
│   │  IOSvL2  │                    │  IOSvL2  │       │
│   └────┬─────┘                    └────┬─────┘       │
│        │                               │             │
│        │ Access                        │ Access      │
│        │                               │             │
│   ┌────▼─────┐                    ┌────▼─────┐       │
│   │  RTR-01  │                    │  RTR-02  │       │
│   │  Cisco   │◄── OSPF Area 0 ───►│  Cisco   │       │
│   │  IOSv    │                    │  IOSv    │       │
│   └────┬─────┘                    └────┬─────┘       │
│        │                               │             │
│        └───────────┬───────────────────┘             │
│                    │                                 │
│             ┌──────▼──────┐                          │
│             │  Cloud/NAT  │                          │
│             │  (Management│                          │
│             │   Network)  │                          │
│             └─────────────┘                          │
│                    │                                 │
│              ┌─────▼─────┐                           │
│              │  Your PC  │                           │
│              │ (SSH from │                           │
│              │ Claude CD)│                           │
│              └───────────┘                           │
└─────────────────────────────────────────────────────┘
```

### 2.4 Configuring the Devices for SSH Access

First, let's configure each device with a management IP and enable SSH. On **RTR-01** (the main router):

```
! Enter configuration mode
enable
configure terminal

! Set hostname
hostname RTR-01

! Enable SSH
ip domain-name lab.local
crypto key generate rsa modulus 2048
ip ssh version 2

! Create a user for Claude Code to SSH with
username claude privilege 15 secret ClaudeLab123!

! Enable AAA
aaa new-model
aaa authentication login default local
aaa authorization exec default local

! Configure management interface
interface GigabitEthernet0/0
 description MANAGEMENT
 ip address 192.168.100.10 255.255.255.0
 no shutdown

! Enable SSH on VTY lines
line vty 0 15
 transport input ssh
 login local
! Save config
end
write memory
```

Repeat for **RTR-02** with IP `192.168.100.20`:

```
hostname RTR-02
interface GigabitEthernet0/0
 ip address 192.168.100.20 255.255.255.0
 no shutdown
!
ip domain-name lab.local
crypto key generate rsa modulus 2048
username claude privilege 15 secret ClaudeLab123!
aaa new-model
aaa authentication login default local
line vty 0 15
 transport input ssh
 login local
end
write memory
```

And for **SW-01** with IP `192.168.100.30` and **SW-02** with IP `192.168.100.40`, using the same credentials.

### 2.5 Testing SSH Connectivity

```bash
# From your host machine, test SSH to each device
ssh claude@192.168.100.10   # RTR-01
ssh claude@192.168.100.20   # RTR-02
ssh claude@192.168.100.30   # SW-01
ssh claude@192.168.100.40   # SW-02

# You should get a Cisco CLI prompt:
# RTR-01#
```

If SSH works manually, Claude Code can automate it. Let's move to the fun part.

---

## Part 3: Claude Code Network Automation — The Real Magic

This is where everything comes together. We're going to write prompts that make Claude Code:

1. SSH into Cisco devices
2. Configure VLANs across two switches
3. Set up inter-VLAN routing on routers
4. Configure OSPF between routers
5. Verify the entire configuration

### 3.1 Giving Claude Code SSH Access

First, make sure `ssh` is allowed in Claude Code's permissions. Add this to `.claude/settings.json` in your project directory:

```json
{
  "permissions": {
    "allow": [
      "Bash(ssh:*)",
      "Bash(sshpass:*)"
    ]
  }
}
```

Install `sshpass` for non-interactive SSH (Claude Code can use it to pass passwords):

```bash
# Ubuntu/Debian
sudo apt install sshpass

# macOS
brew install hudochenkov/sshpass/sshpass
```

### 3.2 Prompt 1 — VLAN Configuration Across Switches

Here's the prompt you give Claude Code:

```
I have two Cisco IOSvL2 switches at these IPs:
- SW-01: 192.168.100.30
- SW-02: 192.168.100.40

Credentials: username "claude", password "ClaudeLab123!"

Using SSH with sshpass, configure the following VLANs on BOTH switches:

VLAN 10 — Name: ENGINEERING — Subnet: 10.10.10.0/24
VLAN 20 — Name: MARKETING — Subnet: 10.10.20.0/24
VLAN 30 — Name: FINANCE — Subnet: 10.10.30.0/24
VLAN 99 — Name: MANAGEMENT — Subnet: 192.168.100.0/24

Also:
- Configure the link between the two switches as an 802.1Q trunk
- Set the native VLAN to 99 on the trunk
- Configure the access ports facing the routers for each VLAN
- Verify VTP is in transparent mode
- Show the VLAN database when done

Use sshpass for password authentication. Do one switch at a time and show the output.
```

**What Claude Code does:**

Claude Code will read the prompt, understand the network requirements, construct the exact Cisco IOS commands, SSH into each switch one at a time, execute the configuration, and show you the output. It will look something like:

```
sshpass -p 'ClaudeLab123!' ssh -o StrictHostKeyChecking=no claude@192.168.100.30 << 'EOF'
enable
configure terminal
vtp mode transparent
vlan 10
 name ENGINEERING
vlan 20
 name MARKETING
vlan 30
 name FINANCE
vlan 99
 name MANAGEMENT
interface GigabitEthernet0/1
 description TRUNK-TO-SW-02
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
interface GigabitEthernet0/2
 description ACCESS-TO-RTR-01-VLAN10
 switchport mode access
 switchport access vlan 10
...
end
show vlan brief
EOF
```

### 3.3 Prompt 2 — Inter-VLAN Routing on Routers

Once VLANs are in place, the routers need to route between them. This is **Router-on-a-Stick** or direct routing depending on how you cabled things.

```
Now configure inter-VLAN routing on RTR-01 (192.168.100.10) and RTR-02 (192.168.100.20).

RTR-01 should:
- Have sub-interfaces on GigabitEthernet0/1 for VLANs 10, 20, and 30
- Use 802.1Q encapsulation
- Act as the default gateway for:
  - VLAN 10: 10.10.10.1/24
  - VLAN 20: 10.10.20.1/24
  - VLAN 30: 10.10.30.1/24
- Enable IP routing

RTR-02 should:
- Have the same sub-interface setup on its GigabitEthernet0/1
- Act as the secondary gateway for each VLAN with .2 addresses:
  - VLAN 10: 10.10.10.2/24
  - VLAN 20: 10.10.20.2/24
  - VLAN 30: 10.10.30.2/24

Use sshpass, one device at a time, and show the routing table when done.
```

**What Claude Code does:**

Claude Code generates the sub-interface configuration, applies 802.1Q encapsulation per VLAN, assigns IP addresses, and verifies with `show ip route`. This is production-quality router configuration — done by an LLM from a plain English prompt.

```
sshpass -p 'ClaudeLab123!' ssh -o StrictHostKeyChecking=no claude@192.168.100.10 << 'EOF'
enable
configure terminal
ip routing
interface GigabitEthernet0/1
 no shutdown
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 10.10.10.1 255.255.255.0
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 10.10.20.1 255.255.255.0
interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 10.10.30.1 255.255.255.0
end
show ip route
show ip interface brief
write memory
EOF
```

### 3.4 Prompt 3 — OSPF Routing Between Routers

Now let's make the two routers talk to each other with OSPF:

```
Configure OSPF on both RTR-01 and RTR-02.

Requirements:
- OSPF Process ID: 1
- Area: 0 (backbone)
- Advertise all connected VLAN subnets
- The link between RTR-01 and RTR-02 is on network 10.0.0.0/30
  - RTR-01: 10.0.0.1/30 on GigabitEthernet0/2
  - RTR-02: 10.0.0.2/30 on GigabitEthernet0/2
- Configure the point-to-point link first, then OSPF
- Use "network" statements with wildcard masks
- Set router-id manually:
  - RTR-01: 1.1.1.1
  - RTR-02: 2.2.2.2
- Verify with "show ip ospf neighbor" — both should see each other as FULL
```

**What Claude Code does:**

Claude Code configures the point-to-point link, sets up OSPF with proper network statements and wildcard masks, assigns router IDs, and verifies the OSPF adjacency comes up.

Example of what it runs on RTR-01:

```
interface GigabitEthernet0/2
 ip address 10.0.0.1 255.255.255.252
 no shutdown
!
router ospf 1
 router-id 1.1.1.1
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.30.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
```

### 3.5 Prompt 4 — Full Verification

Once everything is configured, ask Claude Code to verify the entire setup:

```
Run a full verification of the network. SSH into each device and run:

On BOTH switches:
1. show vlan brief
2. show interfaces trunk
3. show spanning-tree summary
4. show vtp status

On BOTH routers:
1. show ip route
2. show ip ospf neighbor
3. show ip ospf interface brief
4. show ip interface brief
5. ping the other router's loopback (if configured) or its inter-VLAN IPs

Also, run a traceroute from RTR-01's VLAN 10 interface to RTR-02's VLAN 30 interface to prove inter-VLAN routing across OSPF works.

Organize all output clearly. Flag anything that looks wrong.
```

### 3.6 Full Automation Script (Prompt 5 — All-in-One)

For the ultimate demo, here's a single prompt that does everything end-to-end:

```
You are a senior network automation engineer. I have a freshly-built GNS3 lab with these devices:

- RTR-01 (Cisco IOSv) — 192.168.100.10 — acting as core router
- RTR-02 (Cisco IOSv) — 192.168.100.20 — acting as secondary router
- SW-01 (Cisco IOSvL2) — 192.168.100.30 — access switch
- SW-02 (Cisco IOSvL2) — 192.168.100.40 — access switch

Credentials on all devices: claude / ClaudeLab123!
All devices have SSH enabled on their management interfaces.

Your task is to build a production-grade small enterprise network. Execute in this order:

PHASE 1 — Layer 2 (Switches)
- Create VLANs 10 (ENGINEERING), 20 (MARKETING), 30 (FINANCE), 99 (MGMT) on both switches
- Configure trunk between SW-01 Gi0/1 and SW-02 Gi0/1 with native VLAN 99
- Assign access ports: Gi0/2-5 → VLAN 10, Gi0/6-10 → VLAN 20, Gi0/11-15 → VLAN 30
- Set VTP to transparent mode
- Enable Rapid PVST+

PHASE 2 — Layer 3 (Routers)
- Configure inter-VLAN routing on RTR-01 with sub-interfaces (Router-on-a-Stick)
- RTR-01 is the primary gateway (.1) for each VLAN
- RTR-02 is the secondary gateway (.254) for each VLAN
- Configure the point-to-point link between routers: 10.0.0.0/30

PHASE 3 — Dynamic Routing
- Configure OSPF Area 0 between the routers
- Advertise all VLAN subnets
- Configure passive-interface on the VLAN-facing sub-interfaces
- Set manual router-ids

PHASE 4 — Security Hardening
- Configure an ACL that blocks traffic from VLAN 30 (FINANCE) to VLAN 20 (MARKETING)
- Apply the ACL on RTR-01's VLAN 30 sub-interface inbound
- Enable port-security on all access ports (max 2 MACs, sticky, shutdown on violation)

PHASE 5 — Verification
- Show VLAN database on both switches
- Show OSPF neighbors and routes on both routers
- Test connectivity: ping from VLAN 10 to VLAN 20, VLAN 10 to VLAN 30
- Verify ACL: ping from VLAN 30 to VLAN 20 should fail
- Show port-security status on switches

Execute each phase sequentially. For each device, open one SSH session and apply all needed commands at once (no interactive mode). Use "sshpass -p 'ClaudeLab123!' ssh -o StrictHostKeyChecking=no claude@<IP>" to connect.

After each phase, summarize what was done and any issues found. If a command fails, explain why and suggest a fix.
```

**Claude Code will:**

1. Parse the 5-phase plan
2. SSH into each device in sequence
3. Apply real Cisco IOS commands
4. Handle errors (wrong interface names, missing features)
5. Show you the complete output
6. Summarize what was accomplished

This is what used to take a CCNP-certified engineer 2–3 hours. Claude Code does it in minutes.

---

## Part 4: Lessons Learned & Best Practices

### 4.1 Prompt Engineering for Network Automation

After running through this many times, here are the patterns that work best:

**DO:**
- Give Claude Code the **exact IP addresses, usernames, and passwords** upfront
- Specify the **execution order** — switches first, then routers, then verification
- Ask for **one device at a time** to avoid confusion
- Request **verification after each phase** — `show` commands are your friend
- Be specific about **VLAN IDs, subnet masks, and interface names**

**DON'T:**
- Don't ask Claude Code to "figure out" the topology — lay it out explicitly
- Don't use interactive SSH — always use `sshpass` or expect scripts with heredocs
- Don't skip the `-o StrictHostKeyChecking=no` flag in lab environments

### 4.2 Claude Code's Advantages Over Traditional Automation

| Traditional (Ansible/Nornir) | Claude Code |
|------------------------------|-------------|
| Write YAML playbooks | Natural language prompts |
| Debug YAML indentation | Claude debugs itself |
| Need to know Cisco syntax | Claude knows every Cisco command |
| Template-based | Adaptive — handles edge cases |
| Static playbook execution | Dynamic reasoning about state |
| 2 hours to write a playbook | 2 minutes to write a prompt |

### 4.3 Claude Code's Limitations

- **No persistent SSH session** — Each `ssh` command opens a new connection. For stateful configuration (like Cisco's `configure terminal` mode), use heredocs to send all commands at once.
- **No interactive `enable` password** — Use `sshpass` or pre-configure privilege level 15 on the user.
- **Rate limits** — If using DeepSeek or OpenAI, you may hit rate limits during long configuration sessions.
- **GNS3 API would be better** — Ideally, use the GNS3 REST API to programmatically create topologies, then let Claude Code SSH into them. That's the next level.

### 4.4 Going Further — The Full Vision

Here's what the ultimate Claude Code + Network pipeline looks like:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  1. Prompt   │────►│ 2. GNS3 API  │────►│ 3. Topology  │
│ Claude Code  │     │ Create nodes │     │    Built     │
│ describes    │     │ & links      │     │              │
│ network      │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
┌──────────────┐     ┌──────────────┐            │
│  6. Report   │◄────│ 5. Verify    │◄───────────┘
│ + Diagram    │     │ ping/trace/  │
│              │     │ show commands│
└──────────────┘     └──────────────┘
       │                    │
       ▼                    ▼
  "Network is up.     ┌──────────────┐
   10 VLANs, OSPF,    │ 4. Configure │
   ACLs in place."    │ SSH to each  │
                      │ device,      │
                      │ apply config │
                      └──────────────┘
```

---

## Conclusion

What I've shown you in this article is more than just a tutorial — it's a glimpse of where infrastructure engineering is heading.

**Claude Code** with the right LLM backend can:
- Understand complex multi-device network architectures from plain English
- Generate syntactically correct Cisco IOS configuration
- Execute configuration safely via SSH
- Verify the results and flag anomalies
- Do it all in a fraction of the time manual configuration takes

Whether you're running the native Anthropic API with `claude-opus-4-8`, saving costs with DeepSeek, or going fully self-hosted — the capability is here. Network automation has never been this accessible.

**The prompt is the new CLI.**

---

## Quick Reference Card

### Claude Code + SSH One-Liner Template

```bash
claude -p "SSH to 192.168.100.10 as claude/CiscoLab123 using sshpass. \
Configure VLAN 50 named IOT with subnet 10.10.50.0/24. \
Assign it to interfaces Gi0/16 through Gi0/20 as access ports. \
Set the default gateway to 10.10.50.1. Show vlan brief when done."
```

### GNS3 Quick Start

```bash
# Install
sudo apt install gns3-gui gns3-server

# Headless server
docker run -d --name gns3 -p 3080:3080 gns3/gns3-server

# Import Cisco image
# GUI: Edit → Preferences → IOS Routers → New → Select .bin file
```

### API Provider Configuration

```bash
# Anthropic (native)
export ANTHROPIC_API_KEY="sk-ant-api03-..."

# OpenAI
export OPENAI_API_KEY="sk-..."
# settings.json: {"baseUrl": "https://api.openai.com/v1", "model": "gpt-5"}

# DeepSeek
export DEEPSEEK_API_KEY="sk-..."
# settings.json: {"baseUrl": "https://api.deepseek.com/v1", "model": "deepseek-chat"}
```

---

*Last updated: June 8, 2026 — The prompt is the new CLI. 🚀*
