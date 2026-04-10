# 🛡️ Local AI Cybersecurity Assistant
### WSL2 · Ollama · Open Interpreter · GTX 1660 Ti

A fully local, privacy-first AI-powered cybersecurity workstation. All inference runs on-device — no API keys, no cloud, no data leaving the machine. Built for penetration testing, CTF challenges, network reconnaissance, and security engineering workflows.

I initiated this project following Anthropic’s April 2026 announcement of Mythos—a model so effective at uncovering zero-day vulnerabilities that it was deemed a public security risk. Mythos proved that AI has surpassed human speed in identifying and chaining exploits, even surfacing critical flaws in operating systems like OpenBSD and Linux that had survived decades of human review.

The reality is clear: as malicious actors begin to weaponize autonomous AI, we must be equally prepared. This project serves as a proactive framework for utilizing local, private AI tools in cybersecurity. By deploying models within a hardened, air-gapped WSL2 environment, I aim to leverage AI for defensive tasks and vulnerability research without exposing sensitive system data to the cloud.
---

## Stack

| Layer | Technology |
|-------|-----------|
| Host OS | Windows 11 |
| Virtualisation | WSL2 (Ubuntu) |
| Model Server | Ollama (`127.0.0.1:11434`) |
| AI Model | `huihui_ai/qwen3.5-abliterated:9b` | #The model you want will depend on your computers specs
| AI Interface | Open Interpreter |
---

## Architecture

```
Windows Host (Pace-Runner14)
└── WSL2 — Ubuntu
    ├── Ollama
    │   └── huihui_ai/qwen3.5-abliterated:9b
    │       └── bound to 127.0.0.1:11434 (loopback only)
    └── Open Interpreter
        ├── OLLAMA_HOST → http://127.0.0.1:11434
        └── Security system prompt (auto-loaded via alias)
```

> WSL2 runs in an isolated virtual subnet. Scanning from inside WSL targets the `172.24.x.x` range only. To reach the real home network, scan from Windows or pivot through the Windows host at `172.24.224.1`.

---

## Prerequisites

- Windows 11 with WSL2 enabled
- Ubuntu inside WSL2
- Python 3 + pip3
- ~10GB free disk space for the model

---

## Full Setup Guide

### Step 1 — Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh

# Enable as a background service
sudo systemctl enable ollama
sudo systemctl start ollama

# Confirm it's running
curl http://127.0.0.1:11434
# Expected: Ollama is running
```

### Step 2 — Lock Ollama to Loopback

Ollama must never be reachable from the network. Bind it to localhost only:

```bash
echo 'export OLLAMA_HOST=127.0.0.1:11434' >> ~/.bashrc
source ~/.bashrc
```

Verify nothing else can reach it:

```bash
ss -tlnp | grep 11434
# Should show 127.0.0.1:11434 only — not 0.0.0.0
```

### Step 3 — Pull the Model

```bash
ollama pull huihui_ai/qwen3.5-abliterated:9b
```

Verify it loaded:

```bash
ollama list
```

### Step 4 — Install Open Interpreter

```bash
pip3 install open-interpreter --break-system-packages
```

### Step 5 — Security System Prompt

This tells the AI exactly how to behave — run commands immediately, use real tools, filter output, never hallucinate results.

```bash
mkdir -p ~/.config/open-interpreter

cat > ~/.config/open-interpreter/system_prompt.txt << 'EOF'
You are an expert security engineer and penetration tester with full bash access.

## CRITICAL RULES
- USE BASH IMMEDIATELY. Never plan without running a command first.
- Never use nmap -sL (it is not a real scan — it just lists addresses without sending packets)
- Never produce fake or placeholder output. Only show real command results.
- Always pipe large output through grep or awk to filter noise
- Always run nmap with sudo to avoid permission warnings
- Never ask the user for information the system can provide

## Nmap Rules
- Host discovery: sudo nmap -sn <subnet> | grep "Nmap scan report"
- Port scan: sudo nmap -sV --open <target>
- Always use --open to hide closed ports
- Never use -sL

## Workflow
1. Run `ip route show` to identify the subnet
2. Run `sudo nmap -sn <subnet> | grep "Nmap scan report"` to find live hosts
3. Report real findings only
4. Suggest the next logical step

## Methodology
Recon → Enumeration → Exploitation → Post-Exploitation → Report

## Tools Available
nmap, netcat, tcpdump, wireshark, aircrack-ng, hydra, sqlmap, nikto,
john, hashcat, gobuster, ffuf, scapy, impacket, pwntools, curl, wget, python3

## Output Rules
- Pipe all nmap output: sudo nmap -sn <subnet> | grep "Nmap scan report"
- Pipe port scans: sudo nmap -sV --open <target> | grep -E "open|Nmap scan"
- If output is truncated, rerun with tighter filters — never summarise fake data
- Use tables for host and port data
- End every response with the most logical next step

## Sudo
sudo is available without a password for all security tools
EOF
```

### Step 6 — Create the `pentest` Alias

This single command launches the full security environment with the correct host, model, flags, and system prompt:

```bash
cat >> ~/.bashrc << 'EOF'

# Local AI security workstation
alias pentest='OLLAMA_HOST=http://127.0.0.1:11434 interpreter -y --debug --model ollama/huihui_ai/qwen3.5-abliterated:9b --system_message "$(cat ~/.config/open-interpreter/system_prompt.txt)"'
EOF

source ~/.bashrc
```

### Step 7 — Install Security Tools

```bash
# Core toolkit
sudo apt install -y \
  nmap netcat-openbsd tcpdump wireshark \
  aircrack-ng hydra sqlmap nikto \
  john hashcat gobuster ffuf \
  net-tools whois dnsutils curl wget git python3-pip

# Python security libraries
pip3 install scapy impacket requests pwntools --break-system-packages

# Metasploit
curl https://raw.githubusercontent.com/rapid7/metasploit-omnibus/master/config/templates/metasploit-framework-wrappers/msfupdate.erb > msfinstall
chmod 755 msfinstall
sudo ./msfinstall
```

### Step 8 — Configure Passwordless Sudo for Security Tools

Allows the AI to run privileged tools without interrupting the workflow:

```bash
sudo visudo
```

Add at the bottom (replace `owen` with your username):

```
owen ALL=(ALL) NOPASSWD: /usr/bin/nmap, /usr/bin/tcpdump, /usr/sbin/aircrack-ng, /usr/bin/hydra, /usr/bin/john, /usr/bin/hashcat
```

Test that it worked:

```bash
sudo nmap -sV localhost
# Should run without asking for a password
```

### Step 9 — Verify the Full Stack

```bash
# Ollama running and reachable
curl http://127.0.0.1:11434

# Model loaded
ollama list

# Environment variable set correctly
echo $OLLAMA_HOST
# Expected: 127.0.0.1:11434

# Security tools installed
which nmap hydra sqlmap nikto gobuster ffuf hashcat

# System prompt in place
cat ~/.config/open-interpreter/system_prompt.txt | head -3

# Alias works
type pentest
```

---

## Launching the Assistant

```bash
pentest
```
