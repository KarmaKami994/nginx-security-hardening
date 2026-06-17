# 🛡️ Nginx Security Hardening – Defense in Depth Demo

Eine praktische Demonstration wie HTTP Security Header über einen gehärteten
Nginx Reverse Proxy nachgerüstet werden können, auch bei absichtlich
verwundbaren Anwendungen.

Als Testobjekt dient die offizielle OWASP Juice Shop, eine bekannte
Trainings-App mit über 100 dokumentierten Sicherheitslücken.

---

## 🎯 Projektziel

Zeigen wie ein Reverse Proxy als zusätzliche Sicherheitsebene fungiert,
unabhängig davon ob die Backend-Anwendung selbst sicher konfiguriert ist
(Defense in Depth Prinzip).

---

## 🏗️ Architektur

```
Internet
   │
   ▼
Nginx Proxy Manager (SSL Terminierung, Let's Encrypt)
https://juiceshop.kamicorp.ch
   │
   ▼
nginx-hardened (Security Header Layer)
   │
   ▼
juice-shop (isoliert, kein Host-Port)
```

### Warum diese Architektur?

Juice Shop ist nicht direkt erreichbar, da kein Port-Mapping auf den Host
existiert. Es ist nur intern im Docker-Netzwerk ansprechbar.

nginx-hardened dient als dedizierte Sicherheitsschicht und demonstriert
eigenständig geschriebene Nginx Konfiguration statt GUI Klicks in einem
Tool.

NPM übernimmt das SSL und Domain Management mit produktionsreifer SSL
Terminierung via Let's Encrypt.

In einer echten Produktivumgebung würde man diese zwei Proxy Schichten
meist zu einer zusammenfassen, da NPM Security Header auch direkt setzen
kann. Die Trennung hier ist bewusst für Demonstrations und Lernzwecke
gewählt.

---

## 🔒 Vorher / Nachher: HTTP Security Header

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

Die Probleme dieser Konfiguration sind klar erkennbar. Die Nginx Version
ist offen sichtbar, was die Suche nach passenden CVEs erleichtert. CORS
ist komplett offen, Content Security Policy fehlt komplett, ebenso
Strict Transport Security und Referrer Policy.

### Nachher (gehärtet, via nginx-hardened und HTTPS via NPM)

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

| Header | Vorher | Nachher |
|--------|--------|---------|
| Server Version | nginx/1.31.1 sichtbar | versteckt via server_tokens off |
| X-Frame-Options | SAMEORIGIN | DENY (strenger) |
| Content-Security-Policy | fehlt | definiert |
| Strict-Transport-Security | fehlt | aktiv für 1 Jahr |
| Referrer-Policy | fehlt | aktiv |
| Permissions-Policy | fehlt | aktiv |

---

## 🛠️ Tech Stack

| Komponente | Zweck |
|------------|-------|
| OWASP Juice Shop | Absichtlich verwundbare Test App |
| Nginx (gehärtet) | Security Header Layer |
| Nginx Proxy Manager | SSL Terminierung, Domain Routing |
| Docker Compose | Container Orchestrierung |
| Let's Encrypt | Kostenloses SSL Zertifikat |

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
curl -I http://localhost:8090
```

### 4. Optional: Reverse Proxy davorschalten

Für produktionsnahe SSL Terminierung kann ein weiterer Reverse Proxy wie
Nginx Proxy Manager vorgeschaltet werden, der auf nginx-hardened:80
weiterleitet.

---

## 📚 Learnings

### Technisch

Die Direktive server_tokens off versteckt die Nginx Versionsnummer und
ist meist der einfachste erste Schritt jedes Hardenings.

Bei mehreren Proxy Schichten gewinnt der Header der zuletzt gesetzten
Schicht zum Client. NPM überschreibt zum Beispiel den Server Header von
nginx-hardened mit seinem eigenen Wert.

Ohne external: true in Docker Compose wird automatisch ein
Projektpräfix vor den Netzwerknamen gesetzt, etwa
ordnername_netzwerkname.

Ein Container ganz ohne Port Mapping ist nur innerhalb des Docker
Netzwerks erreichbar und damit von aussen komplett isoliert.

### Security

Content Security Policy ist der mächtigste Einzelheader gegen XSS
Angriffe, da er exakt kontrolliert welche Ressourcen Quellen erlaubt
sind.

Auch wenn eine Anwendung selbst verwundbar ist, wie Juice Shop
absichtlich, reduziert eine vorgeschaltete Hardening Schicht die
Angriffsfläche. Dieses Prinzip nennt sich Defense in Depth.

Mehrschichtige Proxy Architektur dient in diesem Projekt der Transparenz
und zusätzlichem Schutz, nicht der Verschleierung. Das unterscheidet sie
fundamental von Proxy Chains die Angreifer zur Anonymisierung nutzen.

---

## 📋 Roadmap / TODOs

- [ ] Rate Limiting gegen Brute Force hinzufügen
- [ ] ModSecurity als Web Application Firewall integrieren
- [ ] Automatisierte Header Checks via GitHub Actions
- [ ] OWASP ZAP Scan gegen die gehärtete Version durchführen
- [ ] Security Header direkt in NPM zeigen als Production Variante

---

## 📖 Referenzen

- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [Mozilla Observatory](https://observatory.mozilla.org/)
