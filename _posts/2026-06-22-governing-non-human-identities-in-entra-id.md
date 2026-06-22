---
layout: post
title: "The New Identity Perimeter: Non-Human Identities in Microsoft Entra ID"
date: 2026-06-22
categories: [entra-id, security]
tags: [non-human-identities, conditional-access, zero-trust, workload-identities]
---

Nicht jeder Administrator in eurem Tenant ist ein Mensch. Service Principals, App Registrations und Agent Identities halten Berechtigungen, die ganze Umgebungen kontrollieren können – und die meisten Organisationen haben keine Übersicht darüber. Dieser Beitrag fasst die wichtigsten Erkenntnisse aus dem Playbook *"The New Identity Perimeter"* zusammen und zeigt, wie Non-Human Identities (NHIs) in Microsoft Entra ID governance-gerecht abgesichert werden.

## Das Problem: Shadow Admins unter der Oberfläche

Die sichtbaren Human Admins sind nur die Spitze des Eisbergs. Darunter liegt eine wachsende Schicht von **Shadow Admins** – Service Principals und App Registrations mit weitreichenden Berechtigungen, die oft undokumentiert und unüberwacht existieren.

Ein prominentes Beispiel: Beim **Midnight Blizzard Angriff 2024** nutzten Angreifer eine Legacy-App mit bestehenden Zugriffsrechten in einem unverwalteten Test-Tenant, um sich lateral zu bewegen und die produktive Umgebung zu kompromittieren.

## Warum MFA für Non-Human Identities nicht funktioniert

Das klassische Sicherheitsparadigma **"Security = MFA"** ist für Non-Human Identities ein totes Konzept. MFA basiert auf Latenz und interaktiven Challenges – beides Dinge, die bei AI Agents und Workload Identities schlicht nicht existieren. Diese Identitäten agieren schneller als jeder menschliche Operator.

Die neue Realität erfordert **deterministische, automatisierte Policy-Durchsetzung**, die nicht auf einen menschlichen Prompt angewiesen ist.

## Die Access Equation: Subject vs. Target

Conditional Access steht als Policy-Gatekeeper direkt zwischen zwei Seiten:

| Subject (Apps & Agents) | Target (Resources) |
|---|---|
| Service Principals | Microsoft Graph APIs |
| Copilot Studio Agents | SharePoint Sites |
| Third-Party SaaS Integrations | Exchange Online |

Die Identität, die aktiv Zugriff anfordert (Subject), wird gegen die passive Ressource (Target) geprüft – und Conditional Access entscheidet dazwischen.

## Das Application Permission Paradox

Hier liegt einer der gefährlichsten Fallstricke:

- **Delegated API Permissions**: Die App agiert im Namen eines angemeldeten Benutzers. Der Blast Radius ist auf den Scope des Users beschränkt.
- **Application Permissions**: Die App agiert als sie selbst, ohne Benutzerkontext.

> **Warnung:** Eine App mit `Files.ReadWrite.All` oder `Sites.FullControl.All` via Application Permissions ist de facto ein Shadow Admin mit der Fähigkeit zum vollständigen Tenant-Kompromittierung.

## Die 3-Tier Agent Identity Hierarchy

Im Zeitalter von AI Agents braucht es eine klare Hierarchie:

1. **Agent Blueprint** – Das statische Template. Lebt in einem Tenant und definiert, wie sich der Agent verhalten soll.
2. **Blueprint Principle** – Der Identity-Anker. Repräsentiert das Blueprint innerhalb jedes deployen Tenants. Berechtigungen, die hier vergeben werden, kaskadieren nach unten.
3. **Agent Identity** – Die laufenden Instanzen. Die tatsächlichen Entitäten, die sich authentifizieren, Tasks ausführen und Tenant-Logs generieren.

### Required Resource Access (RRA) ist kein Grant

Ein häufiges Missverständnis: Admins nehmen an, dass das Hinzufügen von Berechtigungen zum **Required Resource Access (RRA)** eines Agent Blueprints automatisch Zugriff gewährt. Das tut es nicht. RRA ist lediglich ein Signal, was der Agent benötigt – **echter Zugriff erfordert expliziten dynamischen Consent**. Berechtigungen auf dem Blueprint Principle kaskadieren nur dann zu Agent Identities, wenn die Zielressource explizit als vererbbare Ressource konfiguriert ist.

## Conditional Access als Zero Trust Policy Engine

Conditional Access fungiert als deterministische Policy Engine, die nach erfolgter Erst-Authentifizierung greift:

**Signals (IF):** Agent/App Identity, IP Location Ranges, Device State, Entra ID Protection Risk

**Decisions (THEN):** Allow oder Block

Die Entscheidung fällt auf Basis evaluierter Signale, **bevor** die App jemals die Ressource erreicht.

### Der fundamentale Unterschied: Challenge vs. Block

| | Human Identities | App / Agent Identities |
|---|---|---|
| **Policy Response** | Challenge State | Missing State |
| **Mechanismus** | MFA, Compliant Device, Terms of Use | Keine interaktive Challenge möglich |
| **Konsequenz** | "Grant Access, but Verify" | Nur deterministische Signale |

**Das Verdict:** Für Non-Human Identities ist die primäre, empfohlene Conditional Access-Kontrolle **BLOCK**.

## Die Architektur des Instant Block

Ein reiner Block bei Token-Ausstellung reicht nicht. **Continuous Access Evaluation (CAE)** überwacht aktive Agent-Sessions in Echtzeit. Bei anomalem Verhalten wird die laufende Session sofort unterbrochen – nicht erst beim nächsten Token-Refresh.

Diese Entscheidung wird durch ein **Unified Risk Model** gespeist, das Signale aus Entra (Identity), Defender (Endpoint) und Purview (Data) zusammenführt.

## Credential Sprawl: Die Wurzel des Übels

Die überwiegende Mehrheit von App-Kompromittierungen und Ausfällen beginnt mit einem vergessenen Client Secret oder einem abgelaufenen Zertifikat. Traditionelle Zugriffskontrollen tun sich schwer, Credential-Ablauf zu tracken. Secrets landen in E-Mails oder sind in Skripten hardcodiert.

Die zentrale Frage bleibt: **Wem gehört diese App eigentlich, und ist ihr Zugriff noch gerechtfertigt?**

### Die Lösung: Managed Identities

Der sofortige Weg weg von manuellen Secrets führt zu **Managed Identities**. Durch deren Einsatz übernimmt Entra ID intrinsisch das Lifecycle-Management, die Rotation und die Sicherheit der Identität einer Applikation. Das eliminiert den operativen Overhead von Zertifikats-Tracking und entfernt das Risiko von hardcodierten Secrets vollständig.

## Cross-Tenant Governance nicht vergessen

Organisationen sichern ihren primären Tenant, ignorieren aber hunderte unverwalteter Test-Tenants, die über einfache Azure-Subscriptions erstellt wurden. Microsoft selbst hat intern 7 Millionen solcher Tenants entdeckt.

**Related Tenants Discovery** nutzt B2B Sign-in Logs und Multi-Tenant App Consents, um die unbekannte organisatorische Exposure automatisch zu mappen.

Die **Golden Configuration** definiert eine Baseline, die aktiv über 200 Ressourcentypen in Entra und Intune auf Policy-Drift überwacht – alle 6 Stunden.

## Der 6-Monats Governance Blueprint

| Phase 1: Discover & Audit | Phase 2: Transition & Vault | Phase 3: Enforce & Monitor |
|---|---|---|
| Monate 1–2 | Monate 3–4 | Monate 5–6 |
| App Governance Accelerators deployen, um über-berechtigte Shadow Admins und ungetrackte Related Tenants zu finden | Kritische operative Applikationen zu Managed Identities migrieren, alle Legacy Secrets vaulten | Conditional Access Optimization Agent deployen, strikte Block-Policies für riskante Agent IDs, Golden Configuration Baseline etablieren |

## Fazit: Ein Unified Risk Model für die AI-Ära

Die Absicherung des App-to-Resource-Pfads ist kein reines IAM-Problem mehr – es ist der Kern moderner Infrastruktur. Durch explizites Verifizieren von Non-Human Identities, Durchsetzen von Least Privilege über vererbbare Ressourcen und die Annahme von Breach via Continuous Access Evaluation und Block-Policies können Organisationen die Geschwindigkeit von Agentic AI sicher freischalten.

Das erfordert ein **Unified Risk Model** über drei Säulen:
- **Identity** (Entra)
- **Endpoint** (Defender)
- **Data** (Purview)

Das vollständige Playbook ist als [PDF verfügbar](/assets/Governing_Non-Human_Identities.pdf).
