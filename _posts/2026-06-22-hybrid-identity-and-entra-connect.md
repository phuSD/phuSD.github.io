---
layout: post
title: "Hybrid Identity & Entra Connect"
date: 2026-06-22 21:15:00 +0200
categories: [entra-id, security]
tags: [entra-connect, hybrid]
---

Hybrid Identity verbindet on-premises Active Directory mit Entra-ID.

# Authenzitifierzungsmethoden im Vergleich

| Methode                               | Beschreibung                                           | Passwort-Prüfung | Lizenz               |
| ------------------------------------- | ------------------------------------------------------ | ---------------- | -------------------- |
| **Password Hash Sync (PHS)**          | Hash des Passworts wird in Entra synchronisiert        | In der Cloud     | Free                 |
| **Pass-Through Authentication (PTA)** | Passwort wird on-premises geprüft, Cloud leitet weiter | On-premises      | Free                 |
| **Federation (ADFS)**                 | Eigener Identity Provider (ADFS/Drittanbieter)         | On-premises/ADFS | Eigene Infrastruktur |

## Entra-Connect / Sync Engine

### Dedizierter Windows Server (nicht Domain-Controller!)
- Minimum Win-Server 2016+
- SQL: localDB (bis 100k Objekte) oder SQL-Server (ab 100k)

### Sync-Intervall
 - Standard: alle 30min
 - manuelles Triggern via Powershell: 
 ```powershell 
 Start-ADSyncSyncCycle
 ```

### Was wird synchronisiert?

| Objekt                 | Standard        | Konfigurierbar       |
| ---------------------- | --------------- | -------------------- |
| User                   | ✅               | Attribute wählbar    |
| Gruppen (Security)     | ✅               | Scope konfigurierbar |
| Gruppen (Distribution) | ✅               |                      |
| Contacts               | ✅               |                      |
| Geräte (Computer)      | ❌               | Über Hybrid Join     |
| Passwort-Hashes        | ✅ (bei PHS)     |                      |
| Writeback: Passwörter  | Optional (SSPR) |                      |
| Writeback: Gruppen     | Optional        |                      |
| Writeback: Geräte      | Optional        |                      |

---

### Was soll synchronisiert werden?
```
Entra Connect → Azure AD Connect → Sync Filtering
```

#### OU-basiertes Filtering (empfohlen)
```
Nur bestimmte OUs synchronisieren:
✅ OU=Users,DC=corp,DC=local
✅ OU=Groups,DC=corp,DC=local
❌ OU=ServiceAccounts,DC=corp,DC=local (nur wenn nötig)
❌ OU=Computers,DC=corp,DC=local
```

#### Attribut-basiertes Filtering
```
Nur User mit bestimmtem Attribut synchronisieren:
z.B. extensionAttribute1 = "Sync"
```

### Seamless SSO (Single Sign-On)

Ermöglicht automatische Anmeldung ohne Passwort-Eingabe auf Domänen-verbundenen Geräten.

```
Entra Connect → Optional Features → Seamless Single Sign-On ✅
→ PowerShell auf DC ausführen (setzt AZUREADSSOACC Computer-Account)
→ Intranet Zone im Browser auf *.microsoftonline.com setzen (GPO)
```

---

### Passwort Writeback (für SSPR)

Ermöglicht Benutzern, ihr on-premises AD-Passwort über Entra/SSPR zurückzusetzen.

```
Entra Connect → Optional Features → Password Writeback ✅
Entra Admin Center → Password Reset → On-Premises Integration ✅
```

> [!NOTE]
> Ohne Writeback können Benutzer zwar ihr Cloud-Passwort zurücksetzen, aber das on-premises Passwort bleibt unverändert!

## Entra Connect Health

Monitoring-Dashboard für die Sync-Infrastruktur.

```
Entra Admin Center → Entra Connect Health
→ Sync Errors, Server Status, Latency, Alert
```

### Häufige Sync-Fehler

| Fehler | Ursache | Lösung |
|--------|---------|--------|
| `AttributeValueMustBeUnique` | Doppelter Wert (z.B. UPN, ProxyAddress) | Duplikat im AD beheben |
| `InvalidSoftMatch` | Objekte können nicht gemacht werden | GUID / Source Anchor prüfen |
| `LargeObject` | Attribut zu gross (z.B. ThumbnailPhoto) | Bild verkleinern oder Attribut ausschliessen |
| `ExportedChange-Error` | Entra lehnt Änderung ab | Detailmeldung in Sync Service Manager prüfen |

### Sync-Fehler analysieren
```powershell
# Sync-Fehler anzeigen
Get-ADSyncCSObject -ConnectorName "domain.com" -DistinguishedName "CN=User,OU=..." 
Get-ADSyncMVObject

# Sync-Status
Get-ADSyncScheduler

# Sync manuell starten
Start-ADSyncSyncCycle -PolicyType Delta
```

---

## Entra Connect Cloud Sync (Alternative)

Leichtgewichtige Alternative zu Entra Connect – Agent-basiert, kein eigener Server nötig.

|                      | Entra Connect (Sync)        | Entra Connect Cloud Sync     |
| -------------------- | --------------------------- | ---------------------------- |
| Installation         | Eigener Server              | Leichtgewichtiger Agent      |
| Multi-Forest         | ✅                           | ✅ (einfacher)                |
| Gruppenrückschreiben | ✅                           | Eingeschränkt                |
| Geeignet für         | Grosse, komplexe Umgebungen | Einfache/mittlere Umgebungen |
| Verwaltung           | Lokal                       | Vollständig in der Cloud     |
|                      |                             |                              |

---

## Source Anchor (Unveränderliche ID)

Verbindet on-premises Objekt mit Cloud-Objekt eindeutig.

| Attribut | Empfehlung |
|---------|-----------|
| `ms-DS-ConsistencyGuid` | ✅ Empfohlen (seit 2016) |
| `objectGUID` | ✅ Fallback |
| `userPrincipalName` | ❌ Kann sich ändern |

> [!WARNING]
> Source Anchor nach Erstkonfiguration **nicht mehr ändern** – führt zu Duplikaten!

---

## Best Practices Zusammenfassung

- [ ] **Password Hash Sync** bevorzugen (einfacher, resilienter, Identity Protection)
- [ ] ADFS vermeiden (wenn möglich abbauen/migrieren)
- [ ] Entra Connect auf **dediziertem Server** (kein DC)
- [ ] **Staging Mode Server** einrichten (Hot-Standby)
- [ ] OU-basiertes Filtering konfigurieren (nur nötige Objekte synchronisieren)
- [ ] Seamless SSO aktivieren (User-Experience)
- [ ] Password Writeback aktivieren (für SSPR)
- [ ] Entra Connect Health aktivieren und Alerts einrichten
- [ ] Sync-Fehler regelmässig kontrollieren und beheben
- [ ] Entra Connect regelmässig updaten (EOL-Versionen beachten)
- [ ] `ms-DS-ConsistencyGuid` als Source Anchor verwenden
