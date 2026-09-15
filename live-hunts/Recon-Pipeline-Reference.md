# Recon Pipeline Reference

Standard sequence run against any new in-scope target before manual testing begins. Replace `[target]`with the actual domain, or set ``TARGET="[target]"``once and reuse across steps.

### Steps

1. Subdomain discovery (passive)

Expands one domain into its full subdomain surface. Doesn't touch the target directly — queries external sources (cert transparency logs, DNS records).

```bash
subfinder -d $TARGET -silent -o subs_$TARGET.txt
```

1. Alive check + tech fingerprint

Confirms which discovered subdomains actually respond, and identifies server/framework in one pass.

```bash
cat subs_$TARGET.txt | httpx -title -tech-detect -status-code -location -web-server -o alive_$TARGET.txt
```

1. Hidden path discovery

Brute-forces common directory/file names against the target using a wordlist.

```bash
ffuf -u https://$TARGET/FUZZ -w ~/SecLists/Discovery/Web-Content/raft-medium-directories.txt -mc 200,301,302,403 -t 50
```

1. Header inspection

Checks for proxy/backend mismatch signals (smuggling), server info, redirect targets. Always use ``-L`` to see the full redirect chain, not just the first hop.

```bash
curl -sIL https://$TARGET
```

1. Auth endpoint check

Confirms presence and behavior of a login/auth flow.

```bash
curl -sIL https://$TARGET/login
```

After step 3 — manual follow-up (not optional)

Any hit from ffuf that looks distinct from the rest (different size/words, not part of an obvious wildcard-redirect pattern) gets an individual ``curl -sIL``or content pull to see the actual response. Don't treat a fuzz hit as a finding until you've manually verified what it actually returns.

```bash
curl -sIL https://$TARGET/[path]
```

```bash
curl -s https://$TARGET/[path]      # full body, not just headers, when you need to see content
```

## Known patterns to recognize; Signal Reference

### Dead Ends — Stop / Pivot

| Signal | Meaning | Action |
| --- | --- | --- |
| `Server: BigIP` + redirect to `/my.policy` | F5 APM SSO gateway | Fuzzing dead end — pivot to manual auth-flow testing |
| Identical status/size/words across many fuzzed paths | Wildcard catch-all redirect | Not real distinct endpoints, stop fuzzing this host |
| `403 Forbidden` with generic Apache boilerplate body | Correctly restricted | Dead end, don't over-invest |

### Worth Following Up

| Signal | Meaning | Action |
| --- | --- | --- |
| Different sizes/status per fuzzed path | Likely real, distinct endpoints | Worth individual manual follow-up |
| `<title>Index of /...</title>` | Directory listing enabled | Check contents for actual sensitivity before assuming severity — public marketing content ≠ finding |
| Envoy / unusual proxy header in tech-detect | Microservices/API gateway present | Relevant to smuggling — proxy chains are where front-end/back-end parsing mismatches live |

## Tips

**Triage fast**

- Title tags are your fastest signal. Scan `<title>` before reading anything else — dead-end vs real-content in under a second.
- Always compare against a deliberately fake path (`/thisdoesnotexist12345`) before trusting any result. Identical behavior to garbage = dead end, regardless of status code.
- Status code alone means nothing. A 302 with a distinct error can be more interesting than a boring 200 — judge by deviation from baseline, not the code itself.

**Know what recon can and can't do**

- Directory fuzzing won't find JWT/OAuth/SSRF/smuggling bugs directly — it finds *where* to point manual protocol-level testing (Burp) next. Recon narrows the target list; it doesn't replace manual testing.
- Most recon cycles end in a dead end — that's expected, not a failure. The job is to filter fast (30 min–few hours) before committing real hunting time, not to guarantee a lead every time.

---

## Notes

|  |  |
| --- | --- |
| **Wordlist** | SecLists — `~/SecLists/Discovery/Web-Content/raft-medium-directories.txt`, cloned via `git clone --depth 1` to avoid large-repo connection drops |
| **Script** | `~/scripts/recon.sh` — usage: `recon.sh [target]` |
