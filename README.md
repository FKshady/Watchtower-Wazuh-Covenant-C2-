# 🛡️ Project WatchTower
### Building a Home SOC with Wazuh + Covenant C2

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux-lightgrey.svg)
![Purpose](https://img.shields.io/badge/purpose-Educational-green.svg)

> ⚠️ **Disclaimer:** Everything in this repo was done on my own machines and my own network. Please only do this stuff on systems you own. Don't be silly.

---

## Why I Made This

I wanted to actually understand how attacks get detected — not just read about it. So I built a mini SOC at home using Wazuh as my SIEM and Covenant as a C2 framework to simulate real attacks against a Windows machine I own.

The idea was simple: attack my own laptop, watch what gets captured, and learn how to write detection rules based on what I see. Turns out it's one of the best ways to learn both offensive and defensive security at the same time.

This guide walks through everything I did so you can set it up yourself.

---

## What You'll Need

### Hardware
- A **PC** — this is where Covenant C2 runs
- A **laptop** (victim machine) — Windows 10 or 11
- Both need to be on the same home network

### Software
- [VirtualBox](https://www.virtualbox.org/) — to run the Wazuh server VM
- [Ubuntu 22.04 or 24.04](https://ubuntu.com/download) — for the Wazuh VM
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) — to run Covenant
- [Git](https://git-scm.com/) — to clone Covenant
- [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) — for endpoint telemetry on the victim
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config) — the best community config for Sysmon

---

## How It All Fits Together

```
PC (Attacker)                VM on PC                    Laptop (Victim)
┌─────────────────┐         ┌──────────────────┐         ┌───────────────┐
│  Covenant C2    │         │  Ubuntu VM       │         │  Windows 11   │
│  (Docker)       │────────▶│  Wazuh Server    │◀────────│  Wazuh Agent  │
│                 │         │                  │         │  Sysmon       │
└─────────────────┘         └──────────────────┘         └───────────────┘
                    All three on the same home network
```

The PC runs the C2, the Ubuntu VM runs the Wazuh server, and the laptop is the victim with the Wazuh agent installed. All three sit on the same home network so they can talk to each other.

---

## Setup

### 1. Install Wazuh on the Ubuntu VM

Set up a Ubuntu VM in VirtualBox. Important — set the network adapter to **Bridged Adapter** so the VM gets a real IP on your home network instead of being stuck behind NAT.

Once Ubuntu is running, install Wazuh:

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

After it finishes, grab the VM's IP:

```bash
ip a
```

You'll use this IP for everything else. Access the Wazuh dashboard at `https://<VM_IP>`.

---

### 2. Connect the Victim Laptop to Wazuh

In the Wazuh dashboard go to **Endpoints → Deploy new agent**, select Windows, and enter your Wazuh VM's IP. It'll generate a command for you — run it in admin PowerShell on the laptop:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.2-1.msi -OutFile $env:tmp\wazuh-agent.msi; msiexec /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='<YOUR_WAZUH_IP>' WAZUH_AGENT_NAME='LT-Endpoint'
```

Start the agent:

```powershell
NET START WazuhSvc
```

If everything worked you'll see the laptop show up as **Active** in the Wazuh endpoints page.

---

### 3. Install Sysmon on the Victim Laptop

Without Sysmon you'll miss most of the interesting detections. This is what actually gives you visibility into process creation, network connections, file changes etc.

Run this in admin PowerShell on the laptop:

```powershell
# Download Sysmon
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile $env:tmp\Sysmon.zip
Expand-Archive $env:tmp\Sysmon.zip -DestinationPath $env:tmp\Sysmon

# Download the SwiftOnSecurity config
Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile $env:tmp\sysmonconfig.xml

# Install it
& "$env:tmp\Sysmon\Sysmon64.exe" -accepteula -i "$env:tmp\sysmonconfig.xml"
```

You should see `Sysmon64 started` at the end.

---

### 4. Tell Wazuh to Collect Sysmon Logs

There are two config files to edit on the Wazuh VM.

**First, the manager config:**

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this before the closing `</ossec_config>` tag:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**Then the shared agent config:**

```bash
sudo nano /var/ossec/etc/shared/default/agent.conf
```

Replace everything with:

```xml
<agent_config>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>
```

Verify the config is valid then restart:

```bash
sudo /var/ossec/bin/verify-agent-conf
sudo systemctl restart wazuh-manager
```

Restart the agent on the laptop too:

```powershell
Restart-Service WazuhSvc
```

---

### 5. Set Up Covenant C2 on the Attacker PC

Install Docker Desktop and create a free account. If it asks you to update WSL run:

```powershell
wsl --update
```

Then clone and build Covenant — type this out manually rather than copy pasting, the dashes can cause issues:

```powershell
git clone --recursive https://github.com/cobbr/Covenant
cd Covenant\Covenant
docker build -t covenant .
```

The build takes about 5-10 minutes. Once it's done, check the image ID:

```powershell
docker images
```

Run it using the image ID:

```powershell
docker run -it -p 7443:7443 -p 80:80 -p 443:443 --name covenant <IMAGE_ID>
```

When you see `Covenant has started! Navigate to https://127.0.0.1:7443` open that in your browser, accept the certificate warning, and create your admin account.

---

### 6. Create a Listener

In Covenant go to **Listeners → Create** and fill in:

- **Name:** `HTTPListener`
- **BindAddress:** `0.0.0.0`
- **BindPort:** `80`
- **ConnectPort:** `80`
- **ConnectAddress:** your PC's local IP (run `ipconfig` to find it)
- **ValidateCert:** `False`
- **UseCertPinning:** `False`

Click Create. It should show as **Active**.

---

### 7. Generate and Deploy the Grunt

Go to **Launchers → PowerShell** and set:
- **Listener:** HTTPListener
- **DotNetVersion:** Net40
- **ValidateCert:** False
- **UseCertPinning:** False

Click **Generate**, then copy the **EncodedLauncher**.

Before running it on the laptop, disable Windows Defender:

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

You may also need to turn off Tamper Protection manually in Windows Security settings first.

Paste the EncodedLauncher into admin PowerShell on the laptop and run it. Switch to Covenant and watch the Grunts page — your laptop should check in within a few seconds.

---

## Running Attacks

Once you have an active Grunt, click on it and go to the **Interact** tab. Here are some commands to try:

| Command | What it does | MITRE Technique |
|---|---|---|
| `whoami` | Who are we running as | T1033 |
| `shell ipconfig` | Network recon | T1016 |
| `ps` | List running processes | T1057 |
| `shell net user` | Enumerate local users | T1087 |
| `shell tasklist` | List all processes | T1057 |
| `shell cmd.exe /c echo Machine successfully pwned > C:\Users\<user>\Desktop\hacked.txt && notepad.exe C:\Users\<user>\Desktop\hacked.txt` | File drop + process spawn | T1059.003 |

---

## Seeing It in Wazuh

Go to **Threat Hunting → your agent → Events** and set the time filter to Last 1 hour. After running some Grunt commands you should start seeing alerts like:

- `Suspicious Windows cmd shell execution`
- `Windows command prompt started by an abnormal process`
- `Discovery activity executed`
- `PowerShell process created an executable file in Windows root folder`
- `Executable file dropped in folder commonly used by malware`

It's genuinely satisfying watching your own attacks get caught in real time.

---

## Writing Custom Detection Rules

This is where it gets really interesting. You can write your own rules to detect specific things you've seen in your attacks.

Custom rules go in `/var/ossec/etc/rules/local_rules.xml` on the Wazuh VM:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Here are the two rules I wrote based on what Covenant was doing:

```xml
<group name="covenant,c2,custom">

  <rule id="100001" level="12">
    <if_group>sysmon</if_group>
    <field name="win.eventdata.parentImage" type="pcre2">(?i)powershell</field>
    <field name="win.eventdata.image" type="pcre2">(?i)notepad</field>
    <description>Possible C2 Activity: Notepad launched by PowerShell</description>
    <mitre>
      <id>T1059.001</id>
    </mitre>
  </rule>

  <rule id="100002" level="15">
    <if_group>sysmon</if_group>
    <field name="win.eventdata.image" type="pcre2">(?i)cmd.exe</field>
    <field name="win.eventdata.parentImage" type="pcre2">(?i)powershell</field>
    <description>Critical: CMD spawned by PowerShell - Possible C2</description>
    <mitre>
      <id>T1059.003</id>
    </mitre>
  </rule>

</group>
```

After saving, verify and restart:

```bash
sudo /var/ossec/bin/verify-agent-conf
sudo systemctl restart wazuh-manager
```

Run the notepad command from Covenant again and you should see your own custom rule IDs firing in Wazuh. Pretty cool feeling.

---

## Packing Up After a Session

**Kill the Grunt in Covenant:**
```
kill
```

**Stop the Docker container on your PC:**
```powershell
docker stop covenant
```

**Re-enable Defender on the laptop:**
```powershell
Set-MpPreference -DisableRealtimeMonitoring $false
```
Then turn Tamper Protection back on in Windows Security settings.

**Clean up any files you dropped:**
```powershell
Remove-Item C:\Users\<user>\Desktop\hacked.txt
```

**Next session** just run:
```powershell
docker start covenant
```

The Wazuh VM and laptop agent can stay running between sessions.

---

## What I Learned

Honestly this project taught me more than any course I've done. Actually seeing your attacks get detected — and then writing the rules yourself — makes everything click in a way that just reading about it doesn't.

If you're studying for something like SOC analyst roles, blue team work, or just want to understand how detection engineering works, I'd really recommend building something like this yourself.

---

## MITRE ATT&CK Coverage

| Technique | ID | How it's detected |
|---|---|---|
| PowerShell | T1059.001 | Sysmon Event ID 1 + Custom Rule 100001 |
| Windows Command Shell | T1059.003 | Sysmon Event ID 1 + Custom Rule 100002 |
| System Owner Discovery | T1033 | Wazuh Rule 92031 |
| Process Discovery | T1057 | Wazuh Rule 92031 |
| Network Config Discovery | T1016 | Wazuh Rule 92031 |

---

## Credits

- [Wazuh](https://wazuh.com/) — the SIEM/XDR that makes this all possible
- [Covenant](https://github.com/cobbr/Covenant) — C2 framework by cobbr
- [SwiftOnSecurity](https://github.com/SwiftOnSecurity/sysmon-config) — Sysmon config
- [Sysinternals](https://learn.microsoft.com/en-us/sysinternals/) — Sysmon itself

---

## License

MIT — use it, share it, build on it.
