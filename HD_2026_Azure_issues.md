# HD_2026_Azure — seznam chyb k opravě (issues)

Připraveno k založení jako GitHub issues. Pořadí kopíruje závislostní osu nasazení
(**Terraform → Config/Secrets → Deploymenty → Ingress/síť → Auth/dokumentace**),
takže řešit shora dolů.

**Severita:** `P0` = blokátor nasazení · `P1` = funkční chyba · `P2` = bezpečnost / kvalita / dokumentace

**Návrh přidělení:**
- Balíček 1 (Terraform) + Balíček 2 (Config) — paralelně, 2 lidé, hned na začátku
- Balíček 3 (Deploymenty) — člověk se znalostí kontejnerů, po B1+B2
- Balíček 4 (Ingress/síť) — síťař, po B3
- Balíček 5 (Auth/docs) — průběžně / na konec

---

## Balíček 1 — Terraform / infrastruktura

### #1 [P0] Nedeklarovaná proměnná `agent_vm_size`
**Soubor:** `Terraform/main.tf:126`, `Terraform/variables.tf`
`azurerm_kubernetes_cluster_node_pool.userpool` používá `vm_size = var.agent_vm_size`, ale proměnná není ve `variables.tf` deklarována. `terraform apply` selže nebo se ptá interaktivně.
**Akceptační kritérium:** Proměnná `agent_vm_size` deklarována s rozumným defaultem (např. `Standard_D2s_v3`); `terraform validate` projde.

### #2 [P0] Špatné výchozí cesty k manifestům a configu
**Soubor:** `Terraform/variables.tf` (`manifests_path`, `config_path`), použití v `Terraform/main.tf:174,188,189`
Defaulty ukazují na `../DockerStack/kubernetes/k8s_manifests` a `../DockerStack/kubernetes`, ale reálná struktura je `UOIS/kubernetes/manifests` a `UOIS/kubernetes`. `file()` a `fileset()` selžou na neexistující cestě.
**Akceptační kritérium:** Defaulty opraveny na skutečné cesty; `terraform plan` načte ConfigMapu i manifesty bez chyby.

### #3 [P1] Autoscaling userpoolu se rozbije pro PROD
**Soubor:** `Terraform/main.tf` (blok `userpool`)
`min_count = var.node_count`, `max_count = 2`. Komentář u `node_count` uvádí „PROD = 3" → `min_count (3) > max_count (2)`, což je nevalidní. DEV (1) projde jen náhodou.
**Akceptační kritérium:** `max_count >= min_count` pro DEV i PROD (např. samostatná proměnná pro max, nebo max odvozený od node_count).

### #4 [P0] Heslo do DB `example` nesplní komplexitu Azure
**Soubor:** `README.md` (pokyn zadat `example`), `UOIS/kubernetes/common.env` (`POSTGRES_PASSWORD=example`)
Azure PostgreSQL Flexible Server vyžaduje heslo splňující komplexitu (délka + kategorie znaků). `example` bude pravděpodobně odmítnuto při provisioningu.
**Akceptační kritérium:** Návod i `common.env` používají heslo splňující požadavky Azure; provisioning DB proběhne.

### #5 [P1] Příliš volné / nekonzistentní verze providerů
**Soubor:** `Terraform/providers.tf`
`helm >= 2.1.0`, ale konfigurace používá syntaxi v3 (`set = [{...}]`, `provider "helm" { kubernetes = {...} }`) — při stažení 2.x se rozbije (vyžaduje blokovou syntaxi). Obdobně `azapi ~>1.5` vs. přístup `...output.publicKey` (v1 typicky vyžaduje `jsondecode`).
**Akceptační kritérium:** Verze providerů přesně připnuty tak, aby odpovídaly použité syntaxi; `terraform init` + `plan` projde reprodukovatelně.

---

## Balíček 2 — Konfigurace a tajemství

### #6 [P1] Dva nekonzistentní způsoby vytvoření ConfigMapy
**Soubor:** `Terraform/main.tf` (`kubernetes_config_map_v1.common_env`), `README.md`, `UOIS/kubernetes/common.env`
Terraform merguje a přepisuje `POSTGRES_HOST`/`POSTGRES_USER` dynamickou FQDN DB. README ale doporučuje `kubectl create configmap --from-env-file=common.env`, kde se merge neprovede a použije se **zastaralý napevno zadaný** `POSTGRES_HOST=uois-db-vmexan5ugbfa2...`.
**Akceptační kritérium:** Jediný zdroj pravdy pro ConfigMapu; dokumentace i Terraform vedou ke stejnému výsledku se správným hostem.

### #7 [P2] Tajemství v plaintextu v ConfigMapě
**Soubor:** `UOIS/kubernetes/common.env`, `UOIS/kubernetes/manifests/pgadmin-deployment.yaml`
`POSTGRES_PASSWORD`, `SALT`, `ADMIN_DEFAULT_PASSWORD` a pgAdmin heslo `example` jsou v ConfigMapě / přímo v manifestu. Patří do `Secret`.
**Akceptační kritérium:** Citlivé hodnoty přesunuty do `Secret` (ideálně mimo git / přes proměnné); pody je čtou přes `secretKeyRef`.

---

## Balíček 3 — Deploymenty podů

### #8 [P1] Init container — neshoda cest k `systemdata.json`
**Soubor:** `UOIS/kubernetes/manifests/*-deployment.yaml` vs. `UOIS/kubernetes/k8s_extras/prebuilt-initContainer.yaml`
Deploymenty: `cp /data/systemdata.json /mnt/systemdata.json` (mount `/mnt`).
Prebuilt varianta: `cp /systemdata.json /data/systemdata.json` (mount `/mnt`).
Varianty si protiřečí, kde soubor v image `tadblack/systemdata` reálně leží — jedna init container shodí.
**Akceptační kritérium:** Ověřena skutečná cesta v image; obě varianty sjednoceny a init container doběhne (`kubectl logs` bez chyby).

### #9 [P1] `gql-*` deploymenty nemají `resources.requests`
**Soubor:** `UOIS/kubernetes/manifests/gql-ug|office|granting|projects-deployment.yaml`
Chybí CPU requesty → CPU-based HPA na nich nemůže fungovat a pody běží v QoS *BestEffort*. `apollo`/`frontend` requesty mají.
**Akceptační kritérium:** Doplněny `requests` (cpu/memory) konzistentně se zbytkem.

### #10 [P2] HPA jen pro 2 z 6 služeb
**Soubor:** `UOIS/kubernetes/manifests/*-hpa.yaml`
HPA existuje jen pro `apollo` a `frontend`, ačkoli README slibuje autoscaling služeb. Předpoklad: běžící metrics-server + requesty (viz #9).
**Akceptační kritérium:** Buď doplněn HPA pro `gql-*`, nebo upravena dokumentace, aby odpovídala realitě.

### #11 [P2] `readinessProbe` frontendu má `initialDelaySeconds: 300`
**Soubor:** `UOIS/kubernetes/manifests/frontend-deployment.yaml`
Pod nebere provoz prvních 5 minut. Ověřit, zda je to záměr (startup nebo nižší hodnota by byly vhodnější).
**Akceptační kritérium:** Hodnota potvrzena jako záměr, nebo snížena / nahrazena `startupProbe`.

### #12 [P2] Ověřit mapování `POSTGRES_DB` ve frontendu
**Soubor:** `UOIS/kubernetes/manifests/frontend-deployment.yaml`
Frontend má `POSTGRES_DB` namapováno na `key: POSTGRES_CREDENTIALS_DB`. Pravděpodobně záměr (auth proti credentials DB), ale chce to explicitní potvrzení, ne odhad.
**Akceptační kritérium:** Záměr ověřen a okomentován v manifestu.

---

## Balíček 4 — Service / Ingress / síť

### #13 [P1] TLS hosty v Ingressu nesedí s pravidly
**Soubor:** `UOIS/kubernetes/k8s_extras/ingress-controller.yaml`
`tls.hosts: ["*.unob", "unob"]`, ale `rules` jsou na hostech `uois` a `pgadmin.uois`. Certifikát nebude odpovídat hostnames → TLS rozbité, přitom `ssl-redirect: "true"`.
**Akceptační kritérium:** TLS hosty sladěny s hosty v pravidlech; HTTPS funguje, redirect nevede na nevalidní cert.

### #14 [P1] Špatný namespace ingress-nginx v dokumentaci
**Soubor:** `README.md`
Helm instaluje ingress-nginx do namespace `ingress-basic` (TF data source to má správně), ale README/příkazy používají `-n ingress-nginx` → `service not found`.
**Akceptační kritérium:** Všechny příkazy v README používají `ingress-basic`.

### #15 [P2] Ingress se nenasazuje automaticky
**Soubor:** `Terraform/main.tf` (`kubectl_manifest.deploy_all`), `README.md`
Terraform aplikuje jen složku `manifests/`; ingress je v `k8s_extras/` a musí se aplikovat ručně, což README zmiňuje jen napůl.
**Akceptační kritérium:** Buď ingress zahrnut do automatického nasazení, nebo jasně zdokumentovaný ruční krok.

---

## Balíček 5 — Auth (Podklady) a dokumentace

### #16 [P2] `auth.py` důvěřuje neověřenému JWT
**Soubor:** `Podklady/EntraID/Auth/auth.py`
`jwt.decode(..., options={"verify_signature": False})` + holý `except:`. Z neověřeného tokenu se bere `oid`.
**Akceptační kritérium:** Buď zdokumentováno, že jde jen o neautoritativní čtení pro kontext, nebo ověření podpisu zapnuto; `except` zúžen.

### #17 [P2] `core.py` neověřuje audience tokenu
**Soubor:** `Podklady/EntraID/Auth/easyauth/core.py`
`"verify_aud": False` a `audience=...` zakomentováno → tokeny se neověřují proti audience. Funkce navíc vypadá oříznutě (ověřit návratovou hodnotu při selhání).
**Akceptační kritérium:** Audience check zapnut s reálnou hodnotou; doplněna jasná návratová hodnota / výjimka při selhání.

### #18 [P2] Překlepy a nesoulady v README
**Soubor:** `README.md`, název složky `Podklady/Legal_requeremtns`
Mj.: `cd /Terraform` (absolutní cesta místo relativní), velké „Kubernetes" vs. malé v reálné cestě (case-sensitivita na Linuxu), `terraform apply -destroy` (přehlednější `terraform destroy`), „Přítup", „nasatvený", složka `requeremtns`.
**Akceptační kritérium:** README projde korekturou; cesty a příkazy odpovídají repu.
