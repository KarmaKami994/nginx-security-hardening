# 🛡️ Nginx Security Hardening – Defense in Depth Demo

Eine praktische Demonstration wie HTTP Security Header über einen gehärteten
Nginx Reverse Proxy nachgerüstet werden können – auch bei absichtlich
verwundbaren Anwendungen.

Als Testobjekt dient die offizielle **OWASP Juice Shop**, eine bekannte
Trainings-App mit über 100 dokumentierten Sicherheitslücken.

---

## 🎯 Projektziel

Zeigen wie ein Reverse Proxy als zusätzliche Sicherheitsebene fungiert –
unabhängig davon ob die Backend-Anwendung selbst sicher konfiguriert ist
(**Defense in Depth** Prinzip).

---

## 🏗️ Architektur

```
Internet
   │
   ▼
Nginx Proxy Manager (SSL-Terminierung, Let's Encrypt)
https://juiceshop.kamicorp.ch
   │
   ▼
nginx-hardened (Security Header Layer)
   │
   ▼
juice-shop (isoliert, kein Host-Port)
```

### Warum diese Architektur?

- **Juice Shop ist nicht direkt erreichbar** – kein Port-Mapping auf den
  Host, nur intern im Docker-Netzwerk ansprechbar
- **nginx-hardened als dedizierte Sicherheitsschicht** – demonstriert
  eigenständig geschriebene Nginx-Konfiguration statt GUI-Klicks
- **NPM für SSL/Domain-Management** – Produktionsreife SSL-Terminierung
  via Let's Encrypt

> **Hinweis zu Production:** In einer echten Produktivumgebung würde man
> diese zwei Proxy-Schichten meist zu einer zusammenfassen (NPM kann
> Security Header auch direkt setzen). Die Trennung hier ist bewusst für
> Demonstrations- und Lernzwecke gewählt.

---

## 🔒 Vorher / Nachher – HTTP Security Header

### Vorher (ungehärtet, direkter Nginx Passthrough)

```http
HTTP/1.1 200 OK
Server: nginx/1.31.1
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
Cache-Control: public, max-age=0
```

**Probleme:**
- Nginx-Version offen sichtbar (`nginx/1.31.1`) → erleichtert CVE-Suche
- CORS komplett offen (`Access-Control-Allow-Origin: *`)
- Content-Security-Policy fehlt komplett
- Strict-Transport-Security fehlt komplett
- Referrer-Policy fehlt komplett

### Nachher (gehärtet, via nginx-hardened + HTTPS via NPM)

```http
HTTP/1.1 200 OK
Server: openresty
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'
Strict-Transport-Security: max-age=31536000; includeSubDomains
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
X-XSS-Protection: 1; mode=block
```

**Verbesserungen:**
| Header | Vorher | Nachher |
|--------|--------|---------|
| Server Version | `nginx/1.31.1` (sichtbar) | versteckt via `server_tokens off` |
| X-Frame-Options | `SAMEORIGIN` | `DENY` (strenger) |
| Content-Security-Policy | ❌ fehlt | ✅ definiert |
| Strict-Transport-Security | ❌ fehlt | ✅ aktiv (1 Jahr) |
| Referrer-Policy | ❌ fehlt | ✅ aktiv |
| Permissions-Policy | ❌ fehlt | ✅ aktiv |

---

## 🛠️ Tech Stack

| Komponente | Zweck |
|------------|-------|
| OWASP Juice Shop | Absichtlich verwundbare Test-App |
| Nginx (gehärtet) | Security Header Layer |
| Nginx Proxy Manager | SSL-Terminierung, Domain-Routing |
| Docker Compose | Container-Orchestrierung |
| Let's Encrypt | Kostenloses SSL-Zertifikat |

---

## 🚀 Setup Anleitung

### 1. Repository klonen
```bash
git clone https://github.com/KarmaKami994/nginx-security-hardening.git
cd nginx-security-hardening
```

### 2. Stack starten
```bash
docker compose up -d
```

### 3. Vorher/Nachher testen
```bash
# Direkt über Nginx (Port 8090)
curl -I http://localhost:8090
```

### 4. Optional: Reverse Proxy davorschalten
Für produktionsnahe SSL-Terminierung kann ein weiterer Reverse Proxy
(z.B. Nginx Proxy Manager) vorgeschaltet werden, der auf
`nginx-hardened:80` weiterleitet.

---

## 📚 Learnings

### Technisch
- **server_tokens off** versteckt die Nginx-Versionsnummer – einfachster
  erster Schritt jedes Hardenings
- **Multi-Layer Proxying** – bei mehreren Proxy-Schichten gewinnt der
  Header der zuletzt gesetzten/nächsten Schicht zum Client (NPM
  überschreibt z.B. den `Server` Header von nginx-hardened)
- **Docker Compose Netzwerk-Namensgebung** – ohne `external: true` wird
  automatisch ein Projektpräfix vorangestellt (`<ordnername>_<netzwerkname>`)
- **Network Isolation** – ein Container ganz ohne Port-Mapping ist nur
  innerhalb des Docker-Netzwerks erreichbar

### Security
- **Content-Security-Policy** ist der mächtigste Einzelheader gegen XSS –
  er kontrolliert exakt welche Ressourcen-Quellen erlaubt sind
- **Defense in Depth** – auch wenn eine Anwendung selbst verwundbar ist
  (wie Juice Shop absichtlich), reduziert eine Hardening-Schicht davor
  die Angriffsfläche
- **Proxy-Layer ≠ Anonymisierung** – mehrschichtige Architektur dient
  hier der Transparenz und zusätzlichem
