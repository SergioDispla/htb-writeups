# <Machine Name>

| | |
|---|---|
| **OS** | Linux / Windows |
| **Difficulty** | Easy / Medium / Hard / Insane |
| **Topics** | Topic 1, Topic 2, Topic 3 |

---

## 📁 Folder Structure

- [`content/`](content/) — screenshots and inline references used in the write-up
- [`nmap/`](nmap/) — raw nmap output files
- [`exploit/`](exploit/) — exploit scripts, binaries, payloads, and other tools used

---

# Enumeration

## Nmap

```bash
sudo nmap -sS -p- --open --min-rate 5000 -Pn -n -vvv <target-ip> -oN nmap/AllPorts
```

```bash
nmap -p <ports> -sCV -Pn -n <target-ip> -oN nmap/FullScan
```

> 💡 Tip: save scan output directly into the `nmap/` folder with `-oN nmap/<name>` so it's committed alongside the write-up.

![Description](content/screenshot-1.png)

## <Service Name>

<!-- Add enumeration steps for each service found -->

# Initial Access

<!-- Describe the steps to gain initial foothold -->

> 💡 Tip: drop the exploit script or any custom payload into `exploit/` and link to it from here, e.g. [`exploit/cve-XXXX-XXXX.py`](exploit/cve-XXXX-XXXX.py).

# Privilege Escalation

<!-- Describe the privilege escalation path -->

---

## References

- <https://example.com/exploit-reference>
- <https://example.com/cve-info>
