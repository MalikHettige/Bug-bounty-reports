# Recon Cheat Sheet

Substitute `$TARGET` per program before running.

### 1. Subdomain discovery (passive)

```bash
subfinder -d $TARGET -silent -o subs_$TARGET.txt
```

### 2. Alive check + tech fingerprint

```bash
cat subs_$TARGET.txt | httpx -title -tech-detect -status-code -location -web-server -o alive_$TARGET.txt
```

### 3. Hidden path discovery

```bash
ffuf -u https://$TARGET/FUZZ -w ~/SecLists/Discovery/Web-Content/raft-medium-directories.txt -mc 200,301,302,403 -t 50
```

### 4. Header inspection

```bash
curl -sI https://$TARGET
curl -sIL https://$TARGET   # follow full redirect chain
```

### 5. Auth endpoint check

```bash
curl -sI https://$TARGET/login
```

### Known patterns

- Server: BigIP + redirect to /my.policy = F5 APM SSO gateway, fuzzing dead end, pivot to manual auth testing
- Identical status/size/words across many fuzzed paths = wildcard catch-all redirect, not real endpoints

### After step 3 (ffuf) — manual follow-up

Any 200/301/302/403 hit that looks distinct (not part of a wildcard-redirect pattern)
gets an individual curl -s (or `curl -s` ?full, or browser check) to see actual response content.

## Recon Pipeline (parameterized) the "what and why" version

Replace [target] with the actual domain before running.

TARGET="[target]"

subfinder -d $TARGET -silent -o subs_$TARGET.txt
cat subs_$TARGET.txt | httpx -title -tech-detect -status-code -location -web-server -o alive_$TARGET.txt
ffuf -u https://$TARGET/FUZZ -w ~/SecLists/Discovery/Web-Content/raft-medium-directories.txt -mc 200,301,302,403 -t 50
curl -sI https://$TARGET
curl -sI https://$TARGET/login
