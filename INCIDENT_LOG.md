# Incident Log

**Group:** Zero One  
**Member:** Shine Khant Moe — 6705142026  
**Company / domain:** novapeak.lab

## Fault 1 — Incorrect DNS Zone File Path

**Symptom (what we observed):**

The BIND forward zone failed to load because its configured file path was incorrect.

```text
zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
zone novapeak.lab/IN: not loaded due to errors.
_default/novapeak.lab/IN: file not found
```

**Hypothesis (what we suspected and why):**

The forward zone could not load because BIND referenced a nonexistent zone file.

**Proof (the command that confirmed the cause):**

```bash
cat /etc/bind/named.conf.local
named-checkconf -z
```

The original configuration contained:

```conf
file "/etc/bind/db.MISSING";
```

**Fix (what we changed):**

Corrected the path to `/etc/bind/db.novapeak.lab`, validated the configuration, and restarted `named`.

**Confirmation (output showing it now works):**

```text
zone novapeak.lab/IN: loaded serial 2024010101
zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
```

The checker confirmed:

```text
[ 2/2 ] C4 named-checkconf -z clean; named-checkzone clean
[ 2/2 ] C1 named is authoritative for novapeak.lab and reachable from the client.
```

**Found/proved by:** Shine Khant Moe

---

## Fault 2 — Incorrect API A Record

**Symptom (what we observed):**

The original API hostname pointed to the gateway address instead of the API server.

**Hypothesis (what we suspected and why):**

The DNS A record had been configured incorrectly, which would direct API traffic to the wrong host.

**Proof (the command that confirmed the cause):**

```bash
cat /etc/bind/db.novapeak.lab
```

Original record:

```dns
api IN A 192.168.19.1
```

**Fix (what we changed):**

Changed the record to `api IN A 192.168.19.40`, validated the zone, and restarted BIND.

**Confirmation (output showing it now works):**

```bash
ip netns exec client dig @192.168.19.10 api.novapeak.lab +short
```

Actual result from the recorded run:

```text
192.168.19.40
```

The checker also confirmed all four A records were correct.

**Found/proved by:** Shine Khant Moe

---

## Fault 3 — Incorrect DHCP DNS Resolver

**Symptom (what we observed):**

DHCP provided the public DNS server `8.8.8.8`, which could not resolve the company's internal domain.

**Hypothesis (what we suspected and why):**

DHCP was assigning clients the wrong resolver instead of the internal DNS server.

**Proof (the command that confirmed the cause):**

```bash
cat /etc/dhcp/dhcpd.conf
```

Original option:

```conf
option domain-name-servers 8.8.8.8;
```

Initial checker:

```text
[ 0/3 ] C9 DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10
```

**Fix (what we changed):**

Changed the resolver to `192.168.19.10`, validated the DHCP configuration, and restarted `isc-dhcp-server`.

**Confirmation (output showing it now works):**

The recorded client lease included:

```text
ip_address=192.168.19.104
routers=192.168.19.1
domain_name_servers=192.168.19.10
domain_name=novapeak.lab
```

Final checker:

```text
[ 3/3 ] C9 DHCP advertises the company DNS (192.168.19.10)
[ 3/3 ] C11 client resolves www via the leased resolver
```

**Found/proved by:** Shine Khant Moe

---

## Fault 4 — Firewall Blocking DNS

**Symptom (what we observed):**

DNS queries timed out even after the DNS service was restarted.

**Hypothesis (what we suspected and why):**

A firewall rule was dropping incoming DNS requests on UDP port 53.

**Proof (the command that confirmed the cause):**

```bash
iptables -S INPUT
```

Original output:

```text
-P INPUT ACCEPT
-A INPUT -p udp -m udp --dport 53 -j DROP
```

**Fix (what we changed):**

Deleted the blocking rule:

```bash
iptables -D INPUT -p udp --dport 53 -j DROP
```

**Confirmation (output showing it now works):**

The firewall output became:

```text
-P INPUT ACCEPT
```

The DNS query then succeeded:

```bash
dig @192.168.19.10 api.novapeak.lab +short
```

```text
192.168.19.40
```

**Found/proved by:** Shine Khant Moe

---

## Overall Verification

The previous recorded run improved from **11.0/25** to **25.0/25**.

All four required A records, reverse DNS, DHCP leasing, gateway, resolver, and end-to-end client resolution passed the live checker. The new KillerCoda run must be verified separately using its final checker output.
