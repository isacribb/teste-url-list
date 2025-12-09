# teste-url-list

Automação para geração de listas de IP a partir de domínios e wildcards, usando GitHub Actions. O objetivo é alimentar firewalls (ex.: Fortigate) com uma única lista consolidada de IPs confiáveis.

## Visão geral

O repositório mantém três listas principais:

- `allow`  
  - Entrada manual.  
  - Cada linha é um domínio ou wildcard:  
    - Domínios comuns: `dns.google`  
    - Wildcards: `*.gov.br`, `*.office.com`

- `allow-ip-domains.txt`  
  - Saída do workflow **Resolve Domains to IPs**.  
  - Contém IPs únicos resolvidos apenas dos domínios “simples” (linhas de `allow` que **não** começam com `*.`).

- `allow-ip-wildcards.txt`  
  - Saída do workflow **Resolve wildcards**.  
  - Contém IPs únicos descobertos a partir de combinações `prefixo + wildcard`, usando a `wordlist.txt`.

- `allow-ip.txt`  
  - Lista final consumida pelo firewall.  
  - É a união deduplicada de `allow-ip-domains.txt` + `allow-ip-wildcards.txt`.

## Workflows

Os arquivos de workflow ficam em `.github/workflows/`.

### 1. resolve-domains.yml

Resolve domínios “simples” para IP:

- Disparo:
  - `push` que altere o arquivo `allow`
  - Agendamento diário: `0 6 * * *` (03:00 BRT)
- Passos principais:
  - Faz checkout do repositório.
  - Instala `dnspython`.
  - Lê `allow` e ignora linhas que começam com `*.`.
  - Resolve cada domínio em registro A e grava IPs únicos em `allow-ip-domains.txt`.
  - Commita `allow-ip-domains.txt` se houve alteração.
  - Faz o merge com `allow-ip-wildcards.txt` para gerar `allow-ip.txt` (IPs de domínios + wildcards) e commita se mudou.

### 2. resolve-wildcards-domains.yml

Gera IPs a partir de wildcards e wordlist:

- Disparo:
  - `push` que altere `allow` ou `wordlist.txt`
  - Agendamento semanal: `0 9 * * 0` (domingo 06:00 BRT)
- Passos principais:
  - Aguarda 60 segundos para evitar conflito com outros jobs.
  - Faz checkout do repositório.
  - Instala `dnspython`.
  - Lê `allow` e extrai apenas linhas que começam com `*.` (removendo o `*.`).
  - Lê `wordlist.txt` (lista de possíveis prefixos).
  - Para cada combinação `prefixo + base` (ex.: `api.gov.br`), tenta resolver registro A.
  - Grava IPs únicos, ordenados, em `allow-ip-wildcards.txt`.
  - Commita `allow-ip-wildcards.txt` se houve alteração.
  - Faz o merge com `allow-ip-domains.txt` para gerar/atualizar `allow-ip.txt` e commita se mudou.

## Arquivos importantes

- `allow`  
  - Onde são cadastrados os domínios e wildcards de interesse.

- `wordlist.txt`  
  - Prefixos usados para “bruteforce” de subdomínios em cima dos wildcards (ex.: `www`, `api`, `portal`, etc.).
  - Pode ser ajustado conforme necessidade do ambiente.

- `allow-ip.txt`  
  - Arquivo que deve ser publicado/consumido pelo firewall (por exemplo, via URL raw do GitHub).

## Uso típico com firewall

1. Configurar o firewall (ex.: Fortigate) para consumir uma External IP List / Threat Feed apontando para a URL raw de `allow-ip.txt`.  
2. Manter o arquivo `allow` atualizado com novos domínios e wildcards.  
3. Ajustar `wordlist.txt` conforme o tipo de ambiente (corporativo, governamental, etc.).  
4. Os workflows se encarregam de:
   - Resolver domínios.
   - Descobrir subdomínios a partir de wildcards.
   - Consolidar tudo em `allow-ip.txt` automaticamente.
