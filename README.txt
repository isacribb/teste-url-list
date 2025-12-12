# teste-url-list

Automation to generate IP lists from domains and wildcards using GitHub Actions. The goal is to feed firewalls (e.g. Fortigate) with a single consolidated list of trusted IPs.

## Overview

The repository maintains three main lists:

- `allow`  
  - Manual input.  
  - Each line is a domain or wildcard:  
    - Regular domains: `dns.google`  
    - Wildcards: `*.gov.br`, `*.office.com`

- `allow-ip-domains.txt`  
  - Output of the **Resolve Domains to IPs** workflow.  
  - Contains unique IPs resolved only from “simple” domains (lines in `allow` that **do not** start with `*.`).

- `allow-ip-wildcards.txt`  
  - Output of the **Resolve wildcards** workflow.  
  - Contains unique IPs discovered from `prefix + wildcard` combinations, using `wordlist.txt`.

- `allow-ip.txt`  
  - Final list consumed by the firewall.  
  - It is the deduplicated union of `allow-ip-domains.txt` + `allow-ip-wildcards.txt`.

## Workflows

Workflow files live in `.github/workflows/`.

### 1. resolve-domains.yml

Resolves “simple” domains to IPs:

- Triggers:
  - `push` that changes the `allow` file
  - Daily schedule: `0 6 * * *` (03:00 BRT)
- Main steps:
  - Check out the repository.
  - Install `dnspython`.
  - Read `allow` and ignore lines starting with `*.`.
  - Resolve each domain to A records and write unique IPs to `allow-ip-domains.txt`.
  - Commit `allow-ip-domains.txt` if it changed.
  - Merge with `allow-ip-wildcards.txt` to generate `allow-ip.txt` (domain + wildcard IPs) and commit if it changed.

### 2. resolve-wildcards-domains.yml

Generates IPs from wildcards and the wordlist:

- Triggers:
  - `push` that changes `allow` or `wordlist.txt`
  - Weekly schedule: `0 9 * * 0` (Sunday 06:00 BRT)
- Main steps:
  - Wait 60 seconds to avoid conflicts with other jobs.
  - Check out the repository.
  - Install `dnspython`.
  - Read `allow` and extract only lines starting with `*.` (removing the `*.`).
  - Read `wordlist.txt` (list of possible prefixes).
  - For each `prefix + base` combination (e.g. `api.gov.br`), try to resolve an A record.
  - Write unique, sorted IPs to `allow-ip-wildcards.txt`.
  - Commit `allow-ip-wildcards.txt` if it changed.
  - Merge with `allow-ip-domains.txt` to generate/update `allow-ip.txt` and commit if it changed.

## Important files

- `allow`  
  - Where the relevant domains and wildcards are registered.

- `wordlist.txt`  
  - Prefixes used to “bruteforce” subdomains on top of wildcards (e.g. `www`, `api`, `portal`, etc.).
  - Can be tuned according to the environment.

- `allow-ip.txt`  
  - File that should be published/consumed by the firewall (for example, via GitHub raw URL).

## Typical firewall usage

1. Configure the firewall (e.g. Fortigate) to consume an External IP List / Threat Feed pointing to the raw URL of `allow-ip.txt`.  
2. Keep the `allow` file up to date with new domains and wildcards.  
3. Adjust `wordlist.txt` according to the environment (corporate, government, etc.).  
4. The workflows will:
   - Resolve domains.  
   - Discover subdomains from wildcards.  
   - Consolidate everything into `allow-ip.txt` automatically.
