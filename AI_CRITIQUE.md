# AI Usage and Critique

**Group:** lab01  
**Company / domain:** novapeak / novapeak.lab  
**Members:** Refer to the group repository README for the members' names and GitHub usernames.

## Part 3a: Declaration

| AI tool | What we used it for |
| --- | --- |
| ChatGPT | Step-by-step KillerCoda guidance, interpretation of BIND9, DHCP and firewall test results, suggested repair commands, and help organizing the incident log and this critique. ChatGPT was used during troubleshooting, not only after it. |
| Google Gemini | A second opinion on the four diagnosed faults, their causes, diagnostic commands, repairs and potential limitations, after the lab results had been collected. |

## Part 3b: Confirm you diagnosed first

The formal comparison with **Gemini** was performed after the live troubleshooting and incident evidence were collected. However, **ChatGPT was used during the repair process**. Therefore, it would not be accurate to claim that the entire group diagnosis was completed independently before *any* AI assistance. This distinction is declared rather than presenting AI-assisted work as unaided discovery.

The recorded KillerCoda session confirms the following four faults and repairs:

1. `named.conf.local` referenced `/etc/bind/db.MISSING`. `named-checkconf -z` reported that the zone file could not be found. The path was corrected to `/etc/bind/db.novapeak.lab`.
2. The `api` A record originally pointed to `192.168.19.1`; the expected address was `192.168.19.40`. The forward-zone record was corrected.
3. DHCP advertised `8.8.8.8`, rather than the internal resolver `192.168.19.10`. The DHCP option was corrected.
4. `iptables -S INPUT` showed an incoming UDP port 53 DROP rule. It was deleted, after which direct DNS queries succeeded.

The initial **build** checker scored **11.0/25**. The final checker scored **25.0/25**. A real client obtained `192.168.19.104` by DHCP and successfully resolved the required company names. The final result establishes that the repaired system worked, although a final checker score alone does not identify which individual intervention removed each symptom.

## Part 3c: Putting the AIs to the test

**Information supplied:** ChatGPT was given the lab files and progressive KillerCoda screenshots/outputs while providing repair guidance. Gemini was later given the novapeak network parameters, the four observed faults, and the original/final checker context; it was asked to explain causes, proof commands, repairs and weaknesses. Because Gemini received the identified faults, this tests the quality of its *analysis and remedies*, not whether it could discover faults unaided.

| Fault (in our words) | We found it? | AI #1 (ChatGPT) | AI #2 (Gemini) | Was each AI actually right? |
| --- | --- | --- | --- | --- |
| Missing BIND forward-zone path | Confirmed in live file and `named-checkconf -z` output, with ChatGPT guidance | Identified `db.MISSING` and recommended correcting `named.conf.local`, then running `named-checkconf -z` and restarting `named`. | Correctly described the missing file and correct path, but suggested plain `named-checkconf` as syntax proof and used the `bind9` service name. | **ChatGPT: correct for this lab. Gemini: mostly correct, but `named-checkconf -z` was the stronger test because it loads zone files.** Actual output: `loading from master file /etc/bind/db.MISSING failed: file not found`; after repair, the zone loaded. |
| Incorrect `api` A record | Confirmed in the original forward zone, with ChatGPT guidance | Recommended changing `api` from `192.168.19.1` to `192.168.19.40`, validating the zone and querying the server. | Recommended the same IP correction, SOA serial increment, and reloading BIND. | **Both correct on the record.** After repair, `dig @192.168.19.10 api.novapeak.lab +short` returned `192.168.19.40`. Gemini's advice that failing to increase the serial *necessarily* leaves caching resolvers serving the old record was too broad; serials are especially important to secondary-zone transfers, while caching behaviour also depends on TTL. |
| DHCP advertises public resolver | Confirmed in original `dhcpd.conf` and initial C9 result, with ChatGPT guidance | Recommended changing `option domain-name-servers` from `8.8.8.8` to `192.168.19.10`, testing syntax and restarting ISC DHCP. | Correctly identified the resolver error but proposed `grep option-domain-name-servers` (wrong literal spelling) and `nmcli`/`dhclient lan0` client checks. | **ChatGPT: correct. Gemini: correct diagnosis and fix, but some checks do not fit this lab.** The DHCP line uses spaces: `option domain-name-servers`. `lan0` is the server-side interface; the client uses `cli0` in namespace `client`. Actual lease dump showed `domain_name_servers=192.168.19.10`, and C9/C11 passed. |
| UDP port 53 blocked by firewall | Confirmed by `iptables -S INPUT` and query timeouts, with ChatGPT guidance | Recommended inspecting `iptables -S INPUT` and deleting the specific DROP rule. | Correctly described UDP 53 blocking, but recommended appending an ACCEPT rule with `iptables -A INPUT ... -j ACCEPT`. | **ChatGPT: correct. Gemini: incomplete/ineffective as written.** The existing `DROP` rule precedes an appended `ACCEPT`, so the later rule would not be reached for matching DNS packets. The recorded fix deleted the DROP rule; afterward `iptables -S INPUT` showed only `-P INPUT ACCEPT`, and the API query returned `192.168.19.40`. |

### Incorrect or misleading suggestions and how they were checked

- **Gemini's firewall fix:** Appending an ACCEPT rule at the end of the INPUT chain would not override an earlier matching DROP rule. The recorded output showed `-A INPUT -p udp -m udp --dport 53 -j DROP`; after deleting it, `iptables -S INPUT` showed `-P INPUT ACCEPT`, and DNS resolved. Its suggestion to save rules with `netfilter-persistent` was not required by this temporary lab and depends on that package being installed.
- **Gemini's DHCP inspection command:** `grep option-domain-name-servers /etc/dhcp/dhcpd.conf` would not match the actual `option domain-name-servers 8.8.8.8;` line. The recorded `cat /etc/dhcp/dhcpd.conf` output revealed the problem, and the client's `dhcpcd -U cli0` output proved the corrected resolver.
- **Gemini's client interface advice:** The generated lab network uses server interface `lan0`, but the test client is on `cli0` inside namespace `client`. The recorded `ip netns exec client ...` commands were appropriate; `dhclient lan0` on the server was not a suitable client check.
- **Gemini's BIND check:** Plain `named-checkconf` checks configuration syntax but does not necessarily reveal a missing master-zone file. `named-checkconf -z` in the session explicitly reported the missing file. Its suggested `bind9` service name may be an alias on some installations, but the working session used `systemctl restart named` and `systemctl is-active named`.
- **Gemini's failure-status claim:** A missing zone does not always result specifically in `SERVFAIL` or `NXDOMAIN`; the response depends on the active configuration and resolver path. The observed initial client symptom was a **timeout**, so the diagnostic evidence should be reported as such rather than inferring a response code not seen.
- **Limitation applying to both tools:** Neither AI could itself execute a command against this KillerCoda instance. Correctness was decided by the real outputs and the checker, not by how confident an answer sounded.

## Part 3d: Verdict

**Where our human diagnosis beat the AIs:** Direct investigation established the actual packet filtering rule and its position in the firewall chain. This mattered because Gemini's proposed ACCEPT-at-the-end change would not fix an earlier DROP. The real network also made the distinction between `lan0` on the server and `cli0` on the test client clear. Reading the live configuration and the client lease avoided relying on commands that were plausible in a generic Ubuntu setup but inappropriate here.

**Where an AI genuinely helped:** ChatGPT provided a structured sequence of commands and explained the relationship between the zone-file path, DNS A records, DHCP-advertised resolver and firewall reachability. Gemini independently provided useful conceptual explanations and correctly identified the target addresses and the need to validate changes. Its weaker advice was still useful as a subject for critical comparison.

**What we learned about trusting AI to diagnose a live system it cannot see:** AI outputs should be treated as proposed explanations, not as evidence. A running `named` process alone did not prove that clients could reach DNS, and a syntactically valid DHCP file did not prove that it advertised the correct resolver. Before the repairs the live checker scored **11.0/25**; afterward it scored **25.0/25**, with the client successfully receiving a lease and resolving company hosts. This outcome, together with the before-and-after command outputs, provided the strongest basis for accepting or rejecting AI recommendations.
