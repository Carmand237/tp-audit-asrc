# 🔐 TP Audit de Sécurité Réseaux et Systèmes — ASRC IMIE Paris

> **Formation** : ASRC — Administrateur Sécurité Réseaux & Cloud  
> **Organisme** : IMIE Paris  
> **Formateur** : Paul ZOKOU  
> **Référence mission** : AUDIT-ASRC-2026-001  
> **Date** : Mars 2026  
> **Classification** : Pédagogique — Environnement de test isolé

---

## 📋 Table des matières

1. [Présentation du projet](#-présentation-du-projet)
2. [Architecture de l'infrastructure](#-architecture-de-linfrastructure)
3. [Étape 1 — Cadrage de la mission](#étape-1--cadrage-de-la-mission)
4. [Étape 2 — Cartographie du SI](#étape-2--cartographie-du-si)
5. [Étape 3 — Analyse de risques EBIOS RM](#étape-3--analyse-de-risques-ebios-rm)
6. [Étape 4 — Audit réseau](#étape-4--audit-réseau)
7. [Étape 5 — Audit systèmes & Active Directory](#étape-5--audit-systèmes--active-directory)
8. [Étape 6 — Audit applicatif](#étape-6--audit-applicatif)
9. [Synthèse des constats](#-synthèse-des-constats)
10. [Recommandations](#-recommandations)
11. [Outils utilisés](#-outils-utilisés)
12. [Livrables](#-livrables)
13. [Équipe](#-équipe)

---

## 🎯 Présentation du projet

Ce dépôt documente le **TP final d'audit de sécurité** réalisé dans le cadre de la formation ASRC à l'IMIE Paris. La mission simule un audit professionnel complet commandité par la PME fictive **ALPHA SERVICES**.

### Objectifs pédagogiques

- Réaliser un audit de sécurité de bout en bout sur une infrastructure virtualisée réaliste
- Appliquer la méthodologie **EBIOS RM** pour l'analyse de risques
- Maîtriser les outils d'audit : **Nmap**, **BloodHound**, **rpcclient**, **ldapsearch**, **Lynis**
- Produire des livrables professionnels exploitables par une DSI

### Périmètre de la mission

| Élément | Détail |
|---------|--------|
| Client | ALPHA SERVICES (PME fictive) |
| Auditeur | CHUIGWA Armand — Chef de projet ASRC |
| Environnement | VMware Workstation Pro — réseau isolé |
| Méthodologie | EBIOS RM · ISO 27001/27002 · Tests techniques |
| Durée | 20 mars 2026 |

---

## 🏗️ Architecture de l'infrastructure

### Hyperviseur et technologie

```
VMware Workstation Pro 25H2
├── Réseau : VMnet (Host-only) par zone
├── 5 segments réseau isolés
└── 6 machines virtuelles interconnectées
```

### Plan d'adressage réseau

| Zone | Réseau | VMnet | Passerelle pfSense | Usage |
|------|--------|-------|--------------------|-------|
| WAN | 192.168.8.0/24 | VMnet8 (NAT) | — | Accès Internet |
| LAN | 192.168.10.0/24 | VMnet1 | 192.168.10.254 | Postes utilisateurs |
| SERVERS | 192.168.20.0/24 | VMnet2 | 192.168.20.254 | Serveurs internes |
| DMZ | 192.168.30.0/24 | VMnet3 | 192.168.30.254 | Services exposés |
| ADMIN | 192.168.40.0/24 | VMnet4 | 192.168.40.254 | Administration |

### Inventaire des VMs

| VM | Nom | Rôle | OS | IP | Zone |
|----|-----|------|----|----|------|
| FW | PF-ALPHA | Pare-feu | pfSense 2.7.0 | Multi-zones | Toutes |
| DC | DC-ALPHA | Active Directory | Windows Server 2022 Eval | 192.168.20.10 | SERVERS |
| SRV-LX | SRV-LINUX | Services internes | Ubuntu Server | 192.168.20.20 | SERVERS |
| WEB | WEB-ALPHA | Serveur web DMZ | Ubuntu Server | 192.168.30.10 | DMZ |
| CLT | CLIENT-ALPHA | Poste utilisateur | Windows 10 LTSC 2019 | 192.168.10.50 | LAN |
| AUDIT | AUDIT-ASRC | Poste d'audit | Kali Linux | 192.168.40.10 | ADMIN |

### Interfaces pfSense (PF-ALPHA)

```
em0 (WAN)   → VMnet8 NAT    → 192.168.8.128/24  (DHCP)
em1 (LAN)   → VMnet1        → 192.168.10.254/24
em2 (opt1)  → VMnet2        → 192.168.20.254/24  [SERVERS]
em3 (opt2)  → VMnet3        → 192.168.30.254/24  [DMZ]
em4 (opt3)  → VMnet4        → 192.168.40.254/24  [ADMIN]
```

### Flux inter-zones autorisés

| Source | Destination | Protocole/Port | Statut |
|--------|-------------|----------------|--------|
| LAN | DMZ | HTTP/HTTPS (80/443) | ✅ Autorisé |
| LAN | SERVERS | LDAP/Kerberos (389/88) | ✅ Autorisé |
| ADMIN | LAN | Tous | ✅ Autorisé |
| ADMIN | SERVERS | Tous | ✅ Autorisé |
| ADMIN | DMZ | Tous | ✅ Autorisé |
| DMZ | WAN | HTTP/HTTPS + DNS | ✅ Autorisé |
| DMZ | LAN | Tous | ❌ Bloqué |
| DMZ | SERVERS | Tous | ❌ Bloqué |
| LAN | ADMIN | Tous | ❌ Bloqué |

---

## Étape 1 — Cadrage de la mission

### Documents contractuels

| Document | Objet | Statut |
|----------|-------|--------|
| Lettre de mission | Périmètre, objectifs, dates, responsabilités | ✅ Établi |
| NDA | Confidentialité des constats et livrables | ✅ Établi |
| RoE (Rules of Engagement) | Interventions depuis AUDIT-ASRC uniquement, aucune action destructrice | ✅ Établi |

### Hors périmètre

- Réseaux partenaires ou tiers externes
- Données utilisateurs réelles ou personnelles
- Infrastructure hôte VMware Workstation
- Services cloud tiers
- Tests d'ingénierie sociale (phishing, vishing)

---

## Étape 2 — Cartographie du SI

### Découverte des hôtes actifs

```bash
# Scan de découverte par zone depuis AUDIT-ASRC
nmap -sn 192.168.10.0/24
nmap -sn 192.168.20.0/24
nmap -sn 192.168.30.0/24
```

**Résultats :**

| IP | Hostname résolu | Zone | Identification |
|----|-----------------|------|----------------|
| 192.168.10.50 | — | LAN | CLIENT-ALPHA |
| 192.168.10.254 | — | LAN | pfSense interface LAN |
| 192.168.20.10 | dc-alpha.alpha.lab | SERVERS | DC-ALPHA ✅ DNS résolu |
| 192.168.20.254 | — | SERVERS | pfSense interface SERVERS |
| 192.168.30.10 | — | DMZ | WEB-ALPHA |
| 192.168.30.254 | — | DMZ | pfSense interface DMZ |

> **Note** : DC-ALPHA répond avec son hostname FQDN `dc-alpha.alpha.lab` — le DNS Active Directory est visible depuis la zone ADMIN.

### Dépendances fonctionnelles critiques

```
CLIENT-ALPHA ──[Kerberos/LDAP]──► DC-ALPHA
WEB-ALPHA    ──[Routage DMZ]───► PF-ALPHA
Tous serveurs──[DNS AD]─────────► DC-ALPHA
CLIENT-ALPHA ──[GPO]────────────► DC-ALPHA

⚠️  Indisponibilité DC-ALPHA = Arrêt total du SI
```

---

## Étape 3 — Analyse de risques EBIOS RM

### Atelier 1 — Biens essentiels et supports

| Bien essentiel | Bien support associé | Criticité |
|----------------|---------------------|-----------|
| Annuaire Active Directory | DC-ALPHA (192.168.20.10) | 🔴 Critique |
| Données métiers | SRV-LINUX (192.168.20.20) | 🟠 Élevée |
| Service web exposé | WEB-ALPHA (192.168.30.10) | 🟠 Élevée |
| Continuité réseau | PF-ALPHA (pare-feu) | 🟠 Élevée |

### Atelier 2 — Sources de risque

| Source | Motivation | Capacité | Pertinence |
|--------|-----------|----------|------------|
| Attaquant externe | Financière / opportuniste | Élevée | Très élevée |
| Employé malveillant | Idéologique / revanche | Moyenne | Moyenne |
| Prestataire externe | Opportuniste | Moyenne | Moyenne |
| Erreur humaine | Involontaire | Élevée | Élevée |

### Atelier 3 — Scénarios stratégiques

| ID | Scénario | Source | Impact | Priorité |
|----|----------|--------|--------|----------|
| S1 | Compromission Active Directory | Attaquant externe | Prise de contrôle totale du SI | 🔴 CRITIQUE |
| S2 | Intrusion via DMZ (WEB-ALPHA) | Attaquant externe | Pivot vers réseau interne | 🟠 ÉLEVÉ |
| S3 | Exfiltration de données | Interne ou externe | Fuite données / impact RGPD | 🟠 ÉLEVÉ |
| S4 | Déni de service web | Attaquant externe | Indisponibilité service | 🟡 MOYEN |

### Atelier 4 — Chemin d'attaque opérationnel (S1)

```
[Internet]
    │
    ▼ Reconnaissance (Nmap)
[WEB-ALPHA :80] ──► Exploitation Apache 2.4.52 CVE
    │
    ▼ Pivot DMZ → LAN (segmentation insuffisante)
[CLIENT-ALPHA] ──► Exploitation Flexense RCE (CVE-2018-5267)
    │                   OU  RDP bruteforce (pas de lockout)
    ▼ NTLM Relay (SMB signing not required)
[DC-ALPHA] ──► Credential dumping
    │
    ▼ Compte Administrateur (pwdNeverExpires + 1 seul admin)
[CONTRÔLE TOTAL DU SI] 🔴
```

---

## Étape 4 — Audit réseau

### Scans Nmap détaillés

#### DC-ALPHA (192.168.20.10)

```bash
sudo nmap -sV -sC -p- 192.168.20.10 -oN scan_dc-alpha.txt
```

| Port | Service | Version | Analyse |
|------|---------|---------|---------|
| 53/tcp | DNS | Simple DNS Plus | ✅ Normal DC |
| 88/tcp | Kerberos | Microsoft Windows Kerberos | ✅ Normal AD |
| 445/tcp | SMB | microsoft-ds | ✅ **Signing REQUIRED** |
| 636/tcp | LDAPS | tcpwrapped | ⚠️ Non fonctionnel |
| 3268/tcp | LDAP GC | Microsoft Windows AD LDAP | ⚠️ GC exposé |
| 3389/tcp | — | — | ✅ Non exposé |

#### WEB-ALPHA (192.168.30.10)

```bash
sudo nmap -sV -sC -p- 192.168.30.10 -oN scan_web-alpha.txt
```

| Port | Service | Version | Analyse |
|------|---------|---------|---------|
| 80/tcp | HTTP | Apache httpd 2.4.52 (Ubuntu) | 🔴 Obsolète, version exposée |
| 443/tcp | HTTPS | — | 🔴 **TLS non configuré** |

Scripts NSE détectés :
- `http-title: Accueil - ALPHA SERVICES`
- `http-robots.txt: Disallow: /includes/ /admin/`
- `http-cookie-flags: PHPSESSID — httponly flag not set`

#### CLIENT-ALPHA (192.168.10.50)

```bash
sudo nmap -sV -sC 192.168.10.50 -oN scan_client-alpha.txt
```

| Port | Service | Version | Analyse |
|------|---------|---------|---------|
| 80/tcp | HTTP | Flexense Dup Scout 10.0.18 | 🔴 **CVE RCE connues** |
| 445/tcp | SMB | microsoft-ds | 🔴 **Signing NOT required** |
| 3389/tcp | RDP | Microsoft Terminal Services | 🔴 **RDP exposé** |

### Test SMB inter-zones

```bash
nmap -p 445 192.168.20.10 192.168.10.50 192.168.30.10
```

| Cible | Port 445 | Constat |
|-------|----------|---------|
| DC-ALPHA | open | ⚠️ SMB accessible depuis ADMIN |
| CLIENT-ALPHA | open | 🔴 SMB + signing optionnel |
| WEB-ALPHA | closed | ✅ Linux — SMB non exposé |

---

## Étape 5 — Audit systèmes & Active Directory

### Audit Windows — CLIENT-ALPHA

```powershell
systeminfo
net user
net localgroup
net accounts
```

**Constats système :**

| Paramètre | Valeur | Risque |
|-----------|--------|--------|
| OS | Windows 10 Entreprise LTSC build 17763 | 🔴 Build 2019 |
| Patches installés | 14 KB seulement | 🔴 Très insuffisant |
| Date installation | 08/10/2019 | 🔴 7 ans sans patch complet |
| Seuil de verrouillage | **Jamais** | 🔴 Bruteforce illimité |
| Longueur min. MDP | 7 caractères | 🟠 Insuffisant (min 12) |
| Compte user01 | Présent, non justifié | 🟠 Backdoor potentielle |
| Adaptateur TAP-Windows V9 | Présent (inactif) | 🟡 Résidu OpenVPN |

### Audit Active Directory — DC-ALPHA

#### Enumération LDAP avec credentials

```bash
ldapsearch -x -H ldap://192.168.20.10 \
  -D "Carmand@alpha.lab" -W \
  -b "DC=alpha,DC=lab" "(objectclass=user)" \
  sAMAccountName userPrincipalName memberOf
```

#### Enumération RPC

```bash
rpcclient -U "alpha.lab/Carmand" 192.168.20.10 -c "enumdomusers"
rpcclient -U "alpha.lab/Carmand" 192.168.20.10 -c "enumdomgroups"
```

**Utilisateurs du domaine :**

| Compte | Activé | AdminCount | MDP sans expiration | Dernière connexion | Risque |
|--------|--------|-----------|---------------------|--------------------|--------|
| ADMINISTRATEUR | ✅ Oui | ✅ Oui | 🔴 **OUI** | 18 mars 2026 (22x) | CRITIQUE |
| CARMAND | ✅ Oui | ❌ Non | ✅ Non | 20 mars 2026 | Normal |
| KRBTGT | ❌ Non | ✅ Oui | ✅ Non | Jamais | Normal |
| INVITÉ | ❌ Non | ❌ Non | ⚠️ Oui | Jamais | Faible |

**Groupes privilégiés :**

| Groupe | Membres | Constat |
|--------|---------|---------|
| Admins du domaine | Administrateur (SID-500) uniquement | 🔴 SPOF critique |
| Administrateurs de l'entreprise | Administrateur (SID-500) uniquement | 🔴 Contrôle forêt AD |
| Administrateurs du schéma | Administrateur (SID-500) uniquement | 🔴 Modification schéma |
| GG_IT / GG_finance / GG_Comercial | 0 membres | ⚠️ Groupes vides |

#### Collecte BloodHound

```bash
# Ajout du hostname dans /etc/hosts
echo "192.168.20.10 dc-alpha.alpha.lab dc-alpha" >> /etc/hosts

# Collecte
bloodhound-python -u Carmand -p "[REDACTED]" \
  -d alpha.lab \
  -dc dc-alpha.alpha.lab \
  -c all --zip \
  -ns 192.168.20.10
```

**Résultats BloodHound :**

```
Domaines    : 1  (alpha.lab)
Forêts      : 1  (aucun trust externe)
Ordinateurs : 2  (CLIENT-ALPHA, DC-ALPHA)
Utilisateurs: 5
Groupes     : 56
GPO         : 3
OU          : 2  (Domain Controllers, Departement)
Trusts      : 0
```

---

## Étape 6 — Audit applicatif

### WEB-ALPHA — Tests applicatifs

```bash
# Scan des headers HTTP
curl -I http://192.168.30.10

# Contenu robots.txt
curl http://192.168.30.10/robots.txt

# Test des chemins sensibles
curl -I http://192.168.30.10/admin/
curl -I http://192.168.30.10/includes/
```

**Constats applicatifs WEB-ALPHA :**

| Constat | Détail | Criticité |
|---------|--------|-----------|
| HTTP sans HTTPS | Port 443 fermé | 🔴 CRITIQUE |
| Version Apache exposée | `Apache/2.4.52 (Ubuntu)` dans les headers | 🔴 CRITIQUE |
| Cookie sans HttpOnly | `PHPSESSID` accessible via JS | 🟠 ÉLEVÉ |
| robots.txt sensible | `/admin/` et `/includes/` déclarés | 🟠 ÉLEVÉ |
| Headers sécurité absents | Pas de CSP, HSTS, X-Frame-Options | 🟠 ÉLEVÉ |
| Comportement HEAD≠GET | `/admin/` : HEAD=200, GET=404 | 🟡 MOYEN |

---

## 📊 Synthèse des constats

### Tableau global

| ID | Constat | Domaine | Criticité |
|----|---------|---------|-----------|
| C1 | HTTP sans HTTPS — WEB-ALPHA | Applicatif | 🔴 CRITIQUE |
| C2 | Apache 2.4.52 version exposée | Applicatif | 🔴 CRITIQUE |
| C8 | RDP exposé sur CLIENT-ALPHA | Réseau | 🔴 CRITIQUE |
| C9 | SMB signing non requis — CLIENT-ALPHA | Réseau | 🔴 CRITIQUE |
| C10 | Flexense Dup Scout 10.0.18 — CVE RCE | Réseau | 🔴 CRITIQUE |
| C19 | Compte Administrateur non renommé | AD | 🔴 CRITIQUE |
| C23 | Compte Administrateur utilisé quotidiennement | AD | 🔴 CRITIQUE |
| C27 | Mot de passe Administrateur sans expiration | AD | 🔴 CRITIQUE |
| C31 | Administrateur = seul membre de tous les groupes admin | AD | 🔴 CRITIQUE |
| C3 | Cookie PHPSESSID sans HttpOnly | Applicatif | 🟠 ÉLEVÉ |
| C4 | robots.txt révèle /admin/ | Applicatif | 🟠 ÉLEVÉ |
| C5 | Headers HTTP de sécurité absents | Applicatif | 🟠 ÉLEVÉ |
| C11 | SMB port 445 accessible depuis LAN | Réseau | 🟠 ÉLEVÉ |
| C13 | Windows 10 LTSC — 14 patches seulement | Système | 🟠 ÉLEVÉ |
| C14 | Compte local user01 non justifié | Système | 🟠 ÉLEVÉ |
| C15 | Seuil de verrouillage = Jamais | Système | 🟠 ÉLEVÉ |
| C18 | Enumération AD avec compte standard | AD | 🟠 ÉLEVÉ |
| C30 | 2 comptes actifs — aucune séparation des rôles | AD | 🟠 ÉLEVÉ |
| C6 | Comportement HEAD/GET incohérent Apache | Applicatif | 🟡 MOYEN |
| C12 | Infos AD exposées via RDP sans auth | Réseau | 🟡 MOYEN |
| C17 | Adaptateur TAP-Windows V9 résiduel | Système | 🟡 MOYEN |
| C25 | Windows Server 2022 Evaluation | Système | 🟡 MOYEN |
| P1 | SMB signing required sur DC-ALPHA | Réseau | ✅ BON |
| P2 | LDAP anonyme bloqué sur DC-ALPHA | AD | ✅ BON |

### Répartition par criticité

```
🔴 CRITIQUE : 9 constats  ████████████████████ 40%
🟠 ÉLEVÉ    : 9 constats  ████████████████████ 40%
🟡 MOYEN    : 4 constats  █████████            16%
✅ BON      : 2 points     ████                  8%

Score global estimé : 8.2 / 10 — Niveau CRITIQUE
```

---

## 🛠️ Recommandations

### Phase 1 — Actions immédiates (72h)

| ID | Action | Impact |
|----|--------|--------|
| R1 | Activer HTTPS/TLS sur WEB-ALPHA | 🔴 Critique |
| R2 | Masquer la version Apache (`ServerTokens Prod`) | 🔴 Critique |
| R3 | Désactiver RDP sur CLIENT-ALPHA (ou restreindre à ADMIN) | 🔴 Critique |
| R4 | Activer SMB signing requis sur CLIENT-ALPHA (GPO) | 🔴 Critique |
| R5 | Désinstaller Flexense Dup Scout 10.0.18 | 🔴 Critique |
| R6 | Créer des comptes admin AD dédiés — désactiver Administrateur | 🔴 Critique |
| R7 | Configurer expiration MDP du compte Administrateur (60j) | 🔴 Critique |

### Phase 2 — Court terme (30 jours)

| ID | Action | Impact |
|----|--------|--------|
| R8 | Ajouter headers sécurité HTTP (CSP, HSTS, X-Frame-Options) | 🟠 Élevé |
| R9 | Configurer seuil verrouillage = 5 tentatives (GPO) | 🟠 Élevé |
| R10 | Auditer et supprimer le compte user01 | 🟠 Élevé |
| R11 | Déployer mises à jour Windows sur CLIENT-ALPHA | 🟠 Élevé |
| R12 | Corriger cookies PHP (HttpOnly + Secure) | 🟠 Élevé |
| R13 | Restreindre l'énumération LDAP aux admins | 🟠 Élevé |

### Phase 3 — Moyen terme (90 jours)

| ID | Action | Impact |
|----|--------|--------|
| R14 | Licencier Windows Server 2022 sur DC-ALPHA | 🟡 Moyen |
| R15 | Supprimer adaptateur TAP-Windows résiduel | 🟡 Moyen |
| R16 | Implémenter une solution PAM (LAPS + rotation MDP admin) | 🔴 Critique |

### Estimation de réduction du risque

```
Phase 1 (72h) ──► ~65% des risques critiques traités
Phase 2 (30j) ──► ~25% des risques élevés traités
Phase 3 (90j) ──► ~10% restants + PAM
                   ──────────────────────────────
Total           ──► ~90% de réduction du risque global
```

---

## 🧰 Outils utilisés

| Outil | Version | Usage |
|-------|---------|-------|
| **Nmap** | 7.98 | Découverte hôtes, scan de ports, détection de services |
| **BloodHound** | Legacy + bloodhound.py 1.9.0 | Analyse des chemins d'attaque AD |
| **rpcclient** | Samba suite | Enumération RPC, utilisateurs, groupes AD |
| **ldapsearch** | OpenLDAP | Requêtes LDAP avec et sans credentials |
| **enum4linux** | — | Enumération complète Windows/Samba |
| **curl** | — | Tests applicatifs HTTP, headers, chemins |
| **crackmapexec** | — | Tests SMB, enumération AD |
| **pfSense** | 2.7.0 | Analyse des règles de filtrage et NAT |
| **Wireshark** | — | Analyse de trafic réseau |

### Commandes clés de l'audit

```bash
# Découverte réseau
nmap -sn 192.168.{10,20,30}.0/24

# Scan complet avec scripts
sudo nmap -sV -sC -p- <IP> -oN scan_<nom>.txt

# Test SMB signing
nmap --script smb2-security-mode -p 445 <IP>

# Enumération AD anonyme (test)
ldapsearch -x -H ldap://<DC_IP> -b "DC=alpha,DC=lab"

# Enumération AD authentifiée
ldapsearch -x -H ldap://<DC_IP> -D "user@domain" -W -b "DC=alpha,DC=lab"

# Collecte BloodHound
bloodhound-python -u <user> -p <pass> -d alpha.lab \
  -dc dc-alpha.alpha.lab -c all --zip -ns 192.168.20.10

# Enumération RPC
rpcclient -U "domain/user" <DC_IP> -c "enumdomusers"
rpcclient -U "domain/user" <DC_IP> -c "queryuser 0x<RID>"
```

---

## 📁 Livrables

```
📂 TP-Audit-ASRC-2026/
├── 📄 README.md                          ← Ce fichier
├── 📄 Rapport_Audit_Pro_ALPHA_SERVICES.docx  ← Rapport complet
├── 📂 scans/
│   ├── scan_dc-alpha.txt                 ← Nmap DC-ALPHA
│   ├── scan_web-alpha.txt                ← Nmap WEB-ALPHA
│   └── scan_client-alpha.txt             ← Nmap CLIENT-ALPHA
├── 📂 bloodhound/
│   └── 20260320041119_bloodhound.zip     ← Données BloodHound
├── 📂 enum/
│   ├── ldapsearch_dc-alpha.txt           ← Enumération LDAP
│   └── rpcclient_dc-alpha.txt            ← Enumération RPC
└── 📂 docs/
    ├── plan_adressage.md                 ← Plan réseau détaillé
    └── ebios_rm_ateliers.md              ← Ateliers EBIOS RM
```

---

## 👥 Équipe

| Rôle | Nom | Spécialité |
|------|-----|-----------|
| Chef de projet / Auditeur principal | CHUIGWA Armand | Sécurité réseaux & systèmes |
| Auditeur | | |
| Auditeur | | |
| Auditeur | | |
| Validateur | | |

---

## 📚 Références

- [ANSSI — EBIOS Risk Manager](https://cyber.gouv.fr/securisation/analyse-des-risques/methode-ebios-rm/)
- [ISO/IEC 27001:2022](https://www.iso.org/standard/27001)
- [ISO/IEC 27002:2022](https://www.iso.org/standard/75652.html)
- [CIS Benchmarks — Windows Server](https://www.cisecurity.org/cis-benchmarks)
- [BloodHound Documentation](https://bloodhound.readthedocs.io/)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/index.html)

---

## ⚠️ Avertissement légal

> Ce projet est réalisé dans un **environnement de test isolé et pédagogique** dans le cadre de la formation ASRC à l'IMIE Paris. Toutes les techniques documentées ici sont utilisées **uniquement sur des systèmes autorisés**. La reproduction de ces techniques sur des systèmes réels sans autorisation explicite est **illégale** (Article 323-1 du Code Pénal français).

---

*Formation ASRC — IMIE Paris · Mars 2026 · AUDIT-ASRC-2026-001*
