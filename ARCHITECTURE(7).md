# Architektur — NAS Agent Myuri (Stand W223, August 2026)

> **Ausführliche Edition.** Diese Datei beschreibt den Ist-Zustand nach den
> R1357-Wellen (3. Tiefen-Audit + Fixes) und der R1358-Serie (NAS-Integrationen,
> weiterer Tiefen-Audit + Härtung, Suspend-/Weg-Steuerung, Dashboard-Schlankung).
> Sie deckt jetzt *alles* ab, was Myuri kann und wie sie funktioniert — mit
> ausgeschriebenen Inventaren (Drives, Mixins, Lerner, Speicher, Events, Handler),
> einem Einsteiger-Mentalmodell und tiefen Beschreibungen der einzelnen
> Subsysteme. Ältere Beschreibungen (v43/v44, ARCHITECTURE_MERMAID*.md) sind
> historisch und teils überholt.
>
> **Nachtrag August 2026 (W99–W223).** Seit der R1358-Serie ist eine
> weitere Schicht entstanden, die es zum Zeitpunkt dieser Fassung noch
> nicht gab: die **Selbstkontroll-Architektur** — Autoritäten,
> Engpass-Wächter, Laufzeit-Verträge und Gesetze mit Gegenprobe. Sie ist
> in **§30** beschrieben und ist heute die Schicht, in der die meisten
> Reparaturen stattfinden. Wer verstehen will, *warum* das System sich
> selbst korrigiert (und wo seine Blindstellen sind), liest §30 zuerst;
> die §§1–29 beschreiben, *was* es tut.
>
> **Wenn ein Wort unklar ist:** §35 ist ein Wörterbuch, das jeden
> wiederkehrenden Hausbegriff (Autorität, Wächter, Vertrag, Gesetz, Köder,
> Schonmodus, Betriebszeit …) in ein, zwei Alltagssätzen erklärt. §34
> beschreibt in derselben einfachen Sprache, wie Myuri herausfindet, wo
> sie ist, und was sie in den Test-Containern gelernt hat.
> **§36** erklärt, wie sie Dokumente liest und einsortiert
> (Steuer, Bewerbung, Lohnabrechnung …), **§37** die agentische
> Werkzeug-Schicht und **§38** die vier verschiedenen
> Prüf-Mechanismen.

---

## Inhalt

1. Was ist Myuri? (Überblick)
2. Einsteiger-Mentalmodell — wie man das System liest
3. Was Myuri kann — Fähigkeiten-Katalog
4. Verzeichnisstruktur
5. Die Entscheidungskette (Happy Path) + Sicherheitshierarchie
6. Antriebe: die 40 Drives
7. Planung: GOAP & Gates im Detail
8. Ausführung: Handler, Gates, Verifikation, Liveness
9. Wahrnehmung: die Monitore
10. Lernen: Signale, die 15 Lerner, Nutzung
11. Gedächtnis: die Speicher-Klassen
12. Schlussfolgern & Meta-Kognition
13. Weltbild: kanonische Quellen
14. Persona (Langfassung)
15. Kommunikation & Events
16. Dashboard
17. Integrationen
18. Sicherheit
19. Härtung & Erweiterungen (R1358-Serie)
20. Konfiguration (config.json)
21. Boot-Sequenz & Threading-Modell
22. Persistenz-Layout (state/)
23. Der Action-Katalog (98 NASAction-Einträge)
24. Chat: was man Myuri fragen kann
25. Benachrichtigungen
26. Selbstbeobachtung im Detail
27. Erweitern: neuen Drive / Action / Monitor hinzufügen
28. Betrieb & Verifikation
29. Bekannte bewusste Schulden
30. Die Selbstkontroll-Architektur (W99–W223) — Autoritäten, Wächter, Verträge, Gesetze
31. Vollinventar: Ketten, Bücher, Gedächtnisse, Werkzeuge, Denken, Handeln
32. Wie sie spricht, fühlt und chattet
33. Beobachtetes Verhalten (Testläufe 52–55, August 2026)
34. Wie sie ihre Realität versteht — und was die Container-Tests ihr beigebracht haben
35. Wörterbuch — die Begriffe dieses Projekts in Alltagssprache
36. Dokumente lesen, verstehen und einsortieren
37. Die agentische Schicht — 67 Werkzeuge neben dem Kern
38. Die Prüf-Schichten — vier verschiedene Arten zu kontrollieren

---

## 1. Was ist Myuri? (Überblick)

Myuri ist eine autonome Kemonomimi-NAS-Verwalterin — ein event-getriebener
KI-Agent (24/7-Daemon, systemd-managed), der ein Linux-Heim-NAS (z. B. ein
Laptop-NAS auf AMD Ryzen) überwacht, pflegt und repariert. Sie nimmt wahr, baut
**Drive-Druck** auf, plant mit einem **A\*-GOAP-Planner**, handelt über ~160
Action-Handler, **verifiziert** die Wirkung und **lernt** daraus über ~15
Lern-Subsysteme. Eine Anime-**Persona** (Chat via Web-Dashboard/Discord, lokales
Ollama-LLM) liegt quer dazu und moduliert Entscheidungen über **Mood** — hat aber
**keine Entscheidungsgewalt über Aktionen** (siehe Sicherheitshierarchie, §5).

Kanonische Identität der Persona: **Mensch mit Katzenohren** (keine
KI-Selbstbeschreibung — User-Entscheidung R1357/B3, konsistent über LLM-Prompt
und Hardcoded-Selbstauskünfte).

**Was sie im Kern tut:**
- **Überwachen** — CPU/RAM/Temp/I-O (alle 5 s), Disks/SMART, Mounts, Services,
  Docker-Container, Netzwerk, Logs, Sicherheit, Datei-Anomalien.
- **Reparieren** — Services neustarten, Disk/Inodes/Journal aufräumen, fstrim,
  btrfs-scrub, Docker-Prune, Mount-Recovery, Memory-Druck lösen, CPU drosseln.
- **Sichern** — hash-basierte inkrementelle Backups (+ versioniert GFS),
  Restore-Proben gegen Bitrot, Cloud-Offsite (rclone), NAS↔NAS-Kopien.
- **Verstehen** — Ursache→Wirkung-Reasoning, mentale Simulation vor riskanten
  Aktionen, Hypothesen testen, unbekannte Vorfälle dokumentieren.
- **Lernen** — Kosten echter Laufzeiten, Wirkung pro Action auf jeden Drive,
  Futilität (was ist aussichtslos), empirische Alarm-Schwellen, ROI.
- **Sich erklären** — im Chat „warum tust du (nichts) bei X?", Selbstheilungs-
  Statistik, Plan-Modell-Güte, was sie als aussichtslos gelernt hat.
- **Vorausschauen** — Wochenprojekte aus Langzeit-Trends (Disk-Wachstum,
  Zertifikats-Ablauf), Wake-Planung, Weg-Wartungsfenster.
- **Kooperieren** — mit Peer-Agenten (Holo, Nora) über MQTT/HTTP/UDP
  Beliefs teilen, Pläne koordinieren, Wartungsfenster abstimmen.

---

## 2. Einsteiger-Mentalmodell — wie man das System liest

### Die eine Schleife, die alles erklärt

Myuri ist im Grunde **eine kognitive Schleife**, die alle paar Sekunden läuft:

```
   WAHRNEHMEN ──▶ DRUCK AUFBAUEN ──▶ PLANEN ──▶ HANDELN ──▶ MESSEN ──▶ LERNEN
   (Monitore)     (Drives)           (GOAP)     (Executor)  (Verify)   (Lerner)
        ▲                                                                  │
        └──────────────────── Weltbild aktualisiert ◀─────────────────────┘
```

1. **Wahrnehmen** — Monitore lesen den Systemzustand und feuern *Events*.
2. **Druck aufbauen** — Events erhöhen den „Druck" (pressure) passender **Drives**
   (Bedürfnisse wie *backup_need*, *thermal*, *disk_pressure*). Druck baut sich auch
   langsam von selbst auf (buildup_rate). Über einer Schwelle wird ein Drive *dringend*.
3. **Planen** — Der dringendste Drive liefert ein *Ziel* (GOAP-Goal, z. B.
   `{backup_needed: False}`). Der GOAP-Planner sucht per A\* die billigste
   Aktionskette, die das Ziel erreicht (Kosten = gelernte Laufzeiten × ~15 Modifier).
4. **Handeln** — Der Plan läuft durch eine lange **Gate-Kette** (Dedup, Ressourcen,
   Cooldowns, Vorbedingungen, **AgentSoul** als letztes Wort), dann führt der
   ActionExecutor die Aktion aus.
5. **Messen** — Vor dem „Erfolg"-Signal wird das Ziel **verifiziert** (z. B.
   `systemctl is-active`, freier Platz, Mount lesbar). Diagnose-Aktionen sind ausgenommen.
6. **Lernen** — Das verifizierte Ergebnis (genau einmal) fließt in die Lerner:
   Kosten, Wirkung pro Drive, Futilität, ROI, Schwellen, Kausalgraph.

Alles andere — Persona/Mood, Reasoning, Meta-Kognition, Peer-Kooperation — *moduliert*
diese Schleife, ersetzt sie aber nicht.

### Lese-Reihenfolge für den Code

Wer den Code zum ersten Mal liest, sollte in dieser Reihenfolge vorgehen:

1. `core/orchestrator.py` — Boot/Dependency-Injection, hier wird alles verdrahtet.
2. `planning/drive_system.py` — die 40 Drives (die „Bedürfnisse").
3. `planning/ai_brain.py` — die Brain-Loop (der Taktgeber).
4. `planning/goap_planner.py` — wie aus einem Ziel ein Plan wird.
5. `execution/action_executor.py` + `execution/action_handlers.py` — wie gehandelt wird.
6. `healing/nas_intelligence.py` (AgentSoul) — das Sicherheits-Gate.
7. `persona/myuri_chat_core_mixin.py` — wie Chat funktioniert.
8. `communication/nas_events.py` — die Event-Typen + EventBus.

### Glossar

| Begriff | Bedeutung |
|---|---|
| **Drive** | Ein Bedürfnis mit „Druck" (0..1). Über Schwelle → will geplant werden. 38 Stück (35 ziel-tragend inkl. `btrfs_balance` + 3 Meta-Signal ohne GOAP-Goal: `curiosity`, `system_stress`, `problem_solving`). |
| **GOAP** | *Goal-Oriented Action Planning*. A\*-Suche von Ist-Zustand zu Ziel-Zustand. |
| **Goal** | Gewünschter Zustand als Prädikat-Dict, z. B. `{disk_clean: True}`. |
| **Gate** | Ein Filter, der eine Aktion durchlassen oder blocken kann (Dedup, Cooldown, Soul …). |
| **AgentSoul** | Das ethische Sicherheits-Gate. Letztes Wort vor jeder autonomen Aktion. |
| **Belief** | Ein „Glaube" über die Welt (z. B. `has_failed_units`), aus dem Weltbild abgeleitet. |
| **Mood** | 6 volatile Gefühls-Dimensionen, die Schwellen/Exploration real modulieren. |
| **Verifier** | Prüft *nach* einer Aktion, ob das Ziel wirklich erreicht wurde (Outcome-Wahrheit). |
| **blocked / no_op** | „Handler nie gelaufen" (Infrastruktur-Block) bzw. „lief, aber ohne Effekt". Lerner ignorieren beides. |
| **Event** | Eine Nachricht auf dem EventBus (~34 Typen). Treibt Drives + benachrichtigt Subscriber. |
| **Snapshot** | Eine fusionierte, read-only Gesamtsicht (`snapshot_to_goap_state`, ~200 Prädikate). |

---

## 3. Was Myuri kann — Fähigkeiten-Katalog

| Domäne | Konkrete Fähigkeiten |
|---|---|
| **Speicher/Disk** | Belegung pro Mount überwachen; aufräumen (Cache, Journal, Temp, Coredumps, alte Logs); Inode-Druck lösen; `fstrim` (SSD); `btrfs scrub`; `btrfs balance` (gefiltert `-dusage=50`, idle-gated, 30d); DB-`VACUUM`; Notfall-Disk-Relief; Disk-Wachstum analysieren (Wochenprojekt) |
| **SMART/Hardware** | SMART lesen + Ausfall vorhersagen; bei Degradation **Not-Backup auslösen + manuelle Migration empfehlen** (keine autonome Live-Disk-Migration — destruktiv + human-gated); Temperatur über alle thermal_zones; CPU-Governor/-Throttle ✅*aktiv* |
| **Backup** | Hash-inkrementell (Metadaten-Signatur size+mtime+ctime, **kein** Inhalts-Hash — 30-Tage-Full-Sync als Netz gegen same-mtime-Korruption); versioniert GFS daily/weekly/monthly via Hardlinks ✅*aktiv*; Restore-Probe gegen Bitrot; „verschoben statt fehlgeschlagen" wenn Platte fehlt; echter Inhalts-Hash im Change-Gate (`content_hash` ✅*aktiv* — SHA256 der Datei-Bytes, fängt Bit-Rot/same-stat-Korruption; liest jede Datei pro Scan); Cloud-Offsite rclone ⚙️*opt-in*; NAS↔NAS ⚙️*opt-in* |
| **Services/systemd** | Health-Poll (Port + API); Restart mit Cooldown/Backoff; `reset-failed`; Unit-Aliasing (smbd↔smb); Service-Kaskaden-Recovery; absichtlich-aus respektieren |
| **Docker** | Container-Health; Restart (geschützt: portainer/watchtower); Prune; Logs; Crashloop-Erkennung |
| **Netzwerk** | DNS-/SSL-Cert-Check (Ablauf → Wochenprojekt); NFS-Locks; Firewall; SMB-/NFS-Freigaben anlegen |
| **Power** | Adaptiver Idle-Suspend (nur wenn idle + erlaubt); Master-Schalter `allow_self_suspend`; Suspend-Inhibitor bei laufenden Backups; Weg-Wartungsfenster ✅*aktiv* |
| **Diagnose** | `health_report`, `system_vitals`, `kernel_dmesg_check`, `smart_full_check`, Datei-Anomalie-Report; forensische Untersuchung „warum versagt das wiederholt?" |
| **Verstehen** | Ursache→Wirkung-Reasoning + Fix-Reihenfolge; mentale Simulation (Risiko/Reversibilität); Hypothesen-Test (ExperimentScheduler); unbekannte Vorfälle dokumentieren |
| **Dateien** | Durchsuchbarer Index (Pfad/Größe/Alter/Kategorie/Qualität); Serien-/Folgen-Analyse; Duplikate; Anomalie-Patrouille; Reports schreiben (auf Chat-Kommando — die autonome `file_anomaly_report`-Action publiziert nur Event+Text, schreibt keine Datei) |
| **Dokumente** | Erkennen (Name → Inhalt → Vision, §36) und nach Typ/Jahr sortieren; Vollständigkeits-Check („für die Bewerbung fehlt der Lebenslauf"); auf Bestellung schreiben — als .md, .docx, .xlsx oder PDF mit Bordmitteln (W254); eigene Dokumente bearbeiten (ersetzen/anhängen, nur im eigenen Ordner); wortgetreu zusammenfassen (extraktiver Auszug, erfindet nichts) |
| **Chat** | Status/Disk/Docker/Backup im Klartext; „warum (nicht)?"; Entscheidungen bewerten („war richtig/falsch"); Anime-Talk; Selbstauskunft; Gefühle |
| **Kooperation** | Peers (Holo/Nora) Beliefs teilen (Broadcast); Konsens-Abstimmungen ⚙️*opt-in*. **Ab Werk dormant:** MQTT-Broker leer + Peer-IPs sind Platzhalter → sendet ins Leere; Wartungsfenster werden lokal berechnet, nicht mit Peers abgestimmt |
| **Selbstschutz** | Jailbreak-/Secret-Exfiltrations-Abwehr; Whitelist-only Validatoren; Canary-Tokens ✅*aktiv* (3 Fake-Credential-Tripwires gepflanzt); Forbidden-Guard am **dynamischen Dispatch-Chokepoint** (`run_command`/`safe_subprocess` + Chat-Whitelist — *nicht* jeder Subprocess; direkte fixe Aufrufe in Backup/Monitor/Thumbnail laufen daran vorbei, führen aber keine Destruktiv-Kommandos) |
| **Selbstbeobachtung** | Anomalie-Erkennung am eigenen Denken (Rumination, Bias, Schleifen, Drift); Memory-Leak-Detektion; Silent-Failure-**Erkennung** (Observability → Dashboard/Wochen-Report, keine Auto-Remediation); Selbst-Audit (high-severity → Operator-Notification) |

> **Ehrlichkeits-Hinweis (Stand v43.110-Audit).** ⚙️*opt-in* = echter Code,
> aber ab Werk `enabled:false` — tut ohne Konfiguration nichts: GFS-
> Versionierung, Cloud-Offsite, NAS↔NAS, Weg-Wartungsfenster, CPU-Governor,
> Peer-Konsens. Seit v43.111 sind die **lokal-sicheren** Flags ab Werk
> **✅aktiviert**: GFS-Versionierung, Retention, Weg-Wartungsfenster,
> CPU-Governor, Canary-Tokens (3 Fake-Credential-Tripwires gepflanzt) und
> Content-Hash-Backup (`backup.content_hash` — auf User-Wunsch aktiviert;
> Hinweis: liest jede Datei-Byte pro Scan, auf großem Media-NAS I/O-spürbar).
> **Weiterhin ⚙️opt-in** (bewusst): Cloud-Offsite / NAS↔NAS /
> Peer-Kooperation+Konsens (brauchen externe Infra: rclone-Remote / Peer-IPs /
> MQTT-Broker — ohne die bleiben sie dormant). **Wichtige Ehrlichkeits-Korrekturen dieses Audits:**
> die SMART-„Daten-Sicherheit" migriert **nicht** autonom (Not-Backup +
> Empfehlung); das SSL-Ablauf-Wochenprojekt war tot (belief-Verdrahtungs-Bug,
> gefixt); `storage_migrate`/`cert_expiry_check` meldeten früher Erfolg ohne
> Wirkung (gefixt); `btrfs balance` (war Vaporware) ist jetzt echt (gefiltert,
> idle-gated, eigener Drive); Self-Audit remediiert high-severity
> Laufzeit-Findings jetzt autonom (`self_audit_remediate`: memory_cap→Trim,
> ghost→Watcher-Restart). Was der adversarialen Prüfung standhielt: Services/systemd, Docker,
> Disk-Cleanup, SMART-Vorhersage, Firewall, Suspend-Sicherheit, Chat-
> Introspektion, Prompt-Guard, Whitelist-Validatoren und alle 5 Lerner
> (Kosten/Effekt/Futilität/Schwellen/ROI mit echtem Konsumenten).

---

## 4. Verzeichnisstruktur

Das Paket lebt in `Nas Agent Myuri/`. Die Top-Level-`.py`-Dateien sind fast
ausschließlich **Re-Export-Shims** (`sys.modules`-Alias) bzw. Import-Fassaden
(`nas_agent_myuri.py`, `cognitive_core.py`, `persona_myuri.py`) — der echte Code
liegt in den Subpaketen:

| Paket | Inhalt (größte Module) |
|---|---|
| `core/` | Orchestrator (Boot/DI, ~12.7k LOC), StateManager, UnifiedSystemState, Config, atomic_write_json, decision_sources |
| `perception/` | SystemMonitor (5s), Disk/SMART/Access-Monitore, SystemWatchdog, Storage-Discovery, FileIndex, away_detection, readiness_gate, system_probes/-capabilities |
| `belief/` | WorldFactStore (Problem-Wahrheit), SpikeChecklists, ActiveWarnings, FactBasedGate, DriveAttemptLimiter |
| `planning/` | AIBrain (~17k LOC, Brain-Loop), DriveSystem (40 Drives), GOAPPlanner (A\*), GoalGenerator, Gates, Scheduler |
| `reasoning/` | CausalReasoner/-Graph, MentalSimulation, CoherenceChecker (7 Voter), AdversarialChallenge, ForensicInvestigator, BrainstormEngine, MoERouter |
| `meta_cognition/` | MetaCognition (Thought-Loop), SelfCritique, BiasIntrospection, GoalRevision, DecisionLog, Explainability, ExperimentScheduler |
| `execution/` | ActionExecutor (~14k LOC, 2 Worker), ActionHandlers (95 NASAction-Enum + Registry), GOAP-Dispatcher, action_verify, Liveness/Quarantine/Suppressor |
| `learning/` | MetaLearner (StrategyMemory/Bandit/LinUCB/Transfer, SQLite), AEL, ROI, Failure-, Threshold-, Drift-, DiagnosticValue-, Trigger-Learner, nora_inspired |
| `memory/` | EpisodicMemory (JSONL/Tag), SemanticMemory, RegretMemory, AnalysisMemory, UserDecisions, KnowledgeBase, MemoryBackup |
| `persona/` | Myuri-Klasse aus 27 Mixins (~37k LOC), Mood, persona_depth, CORE_IDENTITY, PromptGuard-Anbindung |
| `communication/` | EventBus (nas_events), LLMRouter (Ollama-Tiers), WebHandler, Discord, MQTT/UDP, MutualCognition (Peers Holo/Nora) |
| `healing/` | NASIntelligence (+ **AgentSoul** = Sicherheits-Gate), ServiceManager, FixationResolver, HashBackupManager, SelfHealingMonitor |
| `self_observation/` | SelfReflection, MetaAnomalySystem (auto_correct), EnhancedSelfDiagnosis, PipelineHealth, CapabilityAwareness |
| `security/` | validators (Whitelists), PromptGuard, CanaryTokens, suspend_safety_check, safe_subprocess Forbidden-Guard |
| `integrations/` | Synology, Netzwerk-Freigaben (SMB/NFS), Cloud-Sync (rclone), DSM-API, NAS-interner SSH-Copy, **tool_gate** (Capability-Gate) — R1358-Serie |

---

## 5. Die Entscheidungskette (Happy Path) + Sicherheitshierarchie

```
Monitore (perception/) ──Events──▶ EventBus (PriorityQueue, Dedup, Backpressure)
      │                                   │
      ▼                                   ▼
StateManager.volatile_state        Event→Drive-Spikes (SpikeRouter→WorldFactStore-Dedup)
      │                                   │
      └──────────────┬────────────────────┘
                     ▼
        DriveSystem.tick (Druck steigt; Goal-met/Suppress/Acceptance-Gates)
                     ▼
        AIBrain-Loop (Tick ~3–15 s) → dringendster Drive → statisches Drive-Goal
                     ▼
        GOAPPlanner.plan_with_fallback (A*, gelernte Kosten × ~15 Modifier,
        Soul-WARN_ONLY → hard_blocked; Fallback-Pläne respektieren hard_blocked)
                     ▼
        ~20 Nachbearbeitungs-/Voting-Stufen (CausalGraph-Fix, MetaLearner-Override,
        CoherenceChecker-Veto, MentalSimulation-Veto, Diagnose-First, Exploration ε)
                     ▼
        goap_dispatcher: 8 Gates (Dedup [target-aware], UnifiedOptimizer,
        Effectiveness, Resource/Temporal, Suppressor, ImpactThreshold)
                     ▼
        ActionRequest ──▶ ActionExecutor: ~12 Gates (Cooldown [pro Action+Target],
        PreconditionGate, FactBasedGate, DriveAttemptLimiter, Brain-Approval)
        ──▶ AgentSoul.should_act_autonomously (LETZTES WORT)
                     ▼
        Handler läuft → Goal-Verifikation VOR dem Publish (action_verify)
                     ▼
        ActionCompleted ──▶ ~45 Subscriber (Lerner, satisfy, GOAP-Kosten, Persona)
```

**Sicherheitshierarchie (wer hat das letzte Wort):**
`AgentSoul (FORBIDDEN/WARN_ONLY/Whitelist) > User-Approval > Gates > Gelerntes > LLM`.
Das LLM liefert **nur** Chat/Brainstorm/Reflexionstexte — niemals direkte
Aktions-Autorität. Soul-Prinzip 8 (ehrlich): *„Im Zweifel: handeln, aber laut
warnen — wirklich Gefährliches bleibt absolut gesperrt."*

---

## 6. Antriebe: die 40 Drives

Drives sind Myuris Bedürfnisse. Jeder hat einen **Druck** (0..1), eine **Schwelle**
(threshold), eine **Aufbaurate** (buildup_rate, langsamer Selbst-Anstieg), eine
**Abklingrate** (decay), eine **Kategorie** und ein **GOAP-Goal**. Quelle:
`planning/drive_system.py`.

Typen:
- **event-only** (buildup 0) — steigt nur durch echte Events (z. B. *thermal*, *smart_health*).
- **buildup** — steigt langsam von selbst (routinemäßige Pflege).
- **pause_until_idle** — baut nur auf, wenn das System idle ist (schwere Wartung;
  wartet bis max. ~24 h, dann Re-Eskalation gegen Verhungern, R1358bw).

Viele Drives haben `required_beliefs` (z. B. `has_docker`, `has_smart_disks`) — sie
spiken nur, wenn die Hardware/Software wirklich existiert.

**Kritisch (sofort, mood-geboostet):**

| Drive | Schwelle | Typ | Goal | Treiber |
|---|---|---|---|---|
| `thermal` | 0.30 | event-only | `temp_critical:False` | CPU-Temp > Warn-Schwelle |
| `service_health` | 0.30 | event-only | `service_unhealthy:False` | Service healthy→unhealthy |
| `memory_pressure` | 0.30 | event-only | `memory_pressure_high:False` | RAM-/Swap-Druck-Belief |
| `network_health` | 0.30 | event-only | `network_ok:True` | network_down-Belief |
| `data_safety` | 0.30 | event-only | `data_migrated:True` | SMART-Warnung / Migration |

**Gesundheit (Pflege bei Bedarf):**

| Drive | Schwelle | Typ | Goal |
|---|---|---|---|
| `backup_need` | 0.40 | pause_until_idle | `backup_needed:False` |
| `disk_pressure` | 0.40 | buildup | `disk_clean:True` |
| `mount_integrity` | 0.35 | buildup | `mounts_ok:True, mounts_checked:True` |
| `systemd_check` | 0.30 | event-driven | `systemd_checked:True` |
| `docker_health` | 0.35 | buildup | `docker_checked:True` (req. has_docker) |
| `smart_health` | 0.40 | event-only | `smart_checked:True` (req. has_smart_disks) |
| `gpu_health` | 0.30 | buildup | `gpu_ok:True` (req. has_gpu) |
| `firewall_security` | 0.35 | buildup | `firewall_checked:True` |
| `nfs_health` | 0.30 | event-only | `nfs_locks_clean:True` (req. has_nfs) |
| `swap_pressure` | 0.30 | buildup | `swap_cleaned:True` |
| `container_resource` | 0.30 | event-only | `container_resource_hog:False` |
| `inode_pressure` | 0.30 | buildup | `inodes_checked:True` |
| `log_anomaly` | 0.35 | buildup | `log_anomalies_checked:True` |
| `auth_security` | 0.35 | buildup | `auth_logs_checked:True, failed_logins_checked:True` |
| `intrusion_detection` | 0.40 | buildup | `rootkit_checked:True, ssh_config_checked:True` |
| `update_need` | 0.40 | buildup | `updates_checked:True` |

**Routine (planmäßig, oft idle-gebunden):**

| Drive | Schwelle | Typ | Goal |
|---|---|---|---|
| `maintenance` | 0.40 | buildup | `vitals_checked:True` |
| `log_hygiene` | 0.35 | buildup | `logs_rotated:True` |
| `ssd_trim` | 0.35 | pause_until_idle | `ssd_trimmed:True` |
| `btrfs_scrub` | 0.35 | pause_until_idle | `btrfs_scrubbed:True` |
| `tmp_hygiene` | 0.35 | pause_until_idle | `tmp_cleaned:True, coredumps_cleaned:True` |
| `file_organization` | 0.40 | buildup | `file_anomalies_checked:True` |
| `storage_discovery` | 0.40 | buildup | `mounts_checked:True` (USB-Hotplug) |
| `system_integrity` | 0.35 | buildup | `kernel_checked:True` |
| `network_quality` | 0.35 | buildup | `network_latency_checked:True, nic_checked:True` |
| `communication` | 0.40 | buildup | `peer_contacted:True` |
| `dns_quality` | 0.35 | event-only | `dns_ok:True` |
| `ssl_health` | 0.35 | event-only | `ssl_certs_checked:True` |
| `sleep_need` | 0.30 | event-only | `power_state:suspended` |
| `curiosity` | 0.50 | buildup | *(kein GOAP — triggert BrainstormEngine/Reports)* |

**Meta-Signal-Drives (kein GOAP-Goal, event-only, wie `curiosity` nie „stuck"):**

| Drive | Schwelle | Typ | Auslöser |
|---|---|---|---|
| `system_stress` | 0.40 | event-only | Fehler-Cluster (mehrere Actions gleichzeitig gescheitert) |
| `problem_solving` | 0.40 | event-only | Intention stalled (retry-cap) → Antrieb für Alternativen |

Sie tragen kein Ziel und planen keine Aktion — sie fangen ein **Meta-Signal als
echten Druck** ein (Mood/Priorisierung), statt es verpuffen zu lassen. Früher
riefen die Quellen `spike("system_stress")` / `spike("problem_solving")` auf
Drives auf, die es **nicht gab** → das Signal ging still verloren. Der
Struktur-Guard `tests/test_spike_targets_registered.py` verbietet diese Klasse
jetzt: jedes `spike("<name>")`-Ziel muss ein registrierter Drive sein.

**Drive-Schutzmechanismen:** `acceptance_mode` (historisch nutzlose Action →
Druck mit 0.1× Aufbaurate, kein nutzloser health_report); Inflight-Protection
(`inflight_until` sperrt Re-Dispatch während die Action läuft); Stuck-Quarantäne
(nach 10× „bereits erfüllt" werden alle Spikes 10 min blockiert, R846);
`observation_action` pro Drive (read-only Reality-Probe vor der Planung).

### 6.1 Goal-met-Gate & Reaktivität (Spike-Caching)

Ein Spike baut nur dann Druck auf, wenn das Drive-Goal **noch nicht erfüllt**
ist (z. B. `disk_pressure` spiked nicht, solange `disk_clean:True`). Diese
Goal-Bewertung (`_goal_already_met_fn` → `_compute_world_view`) ist teuer und
wird deshalb mehrschichtig gecacht — mit einer **Reaktivitäts-Invariante**:
*ein Spike darf nur dann still verworfen werden, wenn BEWIESEN ist, dass sich
der zieldeterminierende Weltzustand nicht geändert hat.*

| Schicht | Ort | Skip-Bedingung | Schranke |
|---|---|---|---|
| Per-Tick-Cache | `_spike_internal` (`_tick_goal_met_cache`) | Goal in DIESEM Tick schon als erfüllt bewertet | pro Tick geleert |
| R1152bq Hash-Skip | `spike()` | Input-Hash der Drive-Keys unverändert **und** zuletzt erfüllt | sofort reaktiv bei Hash-Änderung |
| R1063 Fast-Skip | `spike()` | zuletzt erfüllt **und** Signatur unverändert | 30 s-TTL (obere Schranke für rein zeitabhängige Goals) |

**Reaktivität für ALLE 40 Drives (R1063-Verallgemeinerung):** Der Fast-Skip
verwendet als „Signatur unverändert"-Beweis
- für die **16 gemappten Drives** (`_R1152BQ_DRIVE_TO_INPUT_KEYS`, 16 Keys, alle
  registriert) den **per-Drive-Input-Hash** aus `volatile_state` (präzise:
  re-eval nur bei Änderung genau ihrer Eingaben);
- für die **21 hash-losen Drives** (nicht im Mapping — u. a.
  `intrusion_detection`, `swap_pressure`, `inode_pressure`, `firewall_security`,
  `auth_security` sowie die 2 Meta-Signal-Drives) eine **State-Versions-
  Signatur** `(state.version:wfs.version)`, die den `_compute_world_view`-
  Cache-Key spiegelt.

Die Goal-Bewertung ist eine **reine Funktion des so gecachten WorldView** →
Versionen unverändert ⇒ WorldView (und damit Goal) unverändert ⇒ Skip korrekt;
ändert sich die Welt, ändert sich die Signatur ⇒ **sofortige** Neubewertung
(keine bis-zu-30 s-Verzögerung). Früher skippte R1063 für die hash-losen Drives
rein zeitbasiert und ließ sie bis zu 30 s inert — inkl. sicherheitskritischer
Drives. Nachweis + Absicherung: `tests/test_drive_reactivity.py`.

> **Ehrliche Grenze:** Der WorldView liest zusätzlich das **unversionierte**
> `SYSTEM_REALITY` (Hardware-Probe: `has_disk_data` etc.). Eine
> `SYSTEM_REALITY`-only-Änderung ohne Versions-Bump ist weder in der Signatur
> noch im `_compute_world_view`-Cache sichtbar — der Fast-Skip ist dagegen
> durch das 30 s-TTL beschränkt, der WorldView-Cache selbst hat **keine** solche
> Schranke. Die Signatur ist damit nie *weniger* frisch als der WorldView.

### 6.2 „Alles beeinflusst alles": Kopplung am Beispiel `disk_pressure`

Ob ein Drive überhaupt Druck **halten** darf, hängt oft an mehreren
**unabhängigen Subsystemen gleichzeitig**. `disk_pressure` braucht drei
konsistente Quellen — fehlt eine, bleibt Myuri trotz voller Platte inert:

1. `SYSTEM_REALITY["disk"]` (Probe-Telemetrie) → Belief `has_disk_data` →
   `required_beliefs`-Gate des Drives.
2. `inventory_meta.storage_device_count` (Inventar, R891) → Belief
   `_inv_storage_devices` → R896-Reset (0 Geräte ⇒ `pressure=0`).
3. live `disk_stats` → Goal `disk_clean` (`disk<92`) + Füllstand für den Spike.

Ob „entscheidet sie richtig?" ist deterministisch messbar über den
**Entscheidungs-Prüfstand** `tests/test_decision_harness.py`: er bootet den
echten Orchestrator, invalidiert pro Perzeption alle Cache-Schichten sauber
(get_snapshot-Coalescing → Rebuild → `version++` → World-View → Goal-met-Skips)
und fährt die Schleife synchron — inkl. A/B-Nachweis der obigen Kopplung.

---

## 7. Planung: GOAP & Gates im Detail

### A\*-Suche (`planning/goap_planner.py`)

Der Planner sucht von einem projizierten Ist-Zustand zum Drive-Goal mit
**A\***: `f(n) = g(n) + h(n)`, wobei `g` die bisherigen (gelernten) Kosten der
Aktionskette sind und `h` die Heuristik = Summe der günstigsten „Producer" pro
noch unerfülltem Goal-Key (Cache `_cheapest_producer`, 60 s-TTL).

- **Budget:** `_MAX_NODES = 15000`, `_MAX_HEAP_SIZE = 30000`, `_TIMEOUT_S = 2.0` s
  Wallclock. Visited-Set über `frozenset((key, hashable_value))`.
- **State-Projection:** Vom Full-State (~200 Prädikate) auf die 30–50 relevanten
  Keys (Goal ∪ aller Preconditions ∪ aller Effects) → 5–7× schnellere Suche.
- **155+ statische GOAPActionDefs** + dynamische bis ~500. Beispiel:
  `GOAPActionDef("backup_retry", pre={backup_running:False, backup_needed:True},
  eff={backup_needed:False})`.
- **Fallback-Kaskade** (`plan_with_fallback`): (1) normaler Plan (Tiefe 6) →
  (2) tiefere Suche (Tiefe 10) → (3) relaxierte Ziele (nur kritische Prädikate) →
  (4) Teilplan (ein Prädikat). Gesamt-Cap 2 s.

### Kosten-Modifier (multiplikativ auf die Basis-Kosten)

1. Fehlende Binaries / `_inv_missing_commands` → **999.0** (prohibitiv).
2. Als ineffektiv markiert → ×50 (weich).
3. Bewährte Action (`_goal_success_memory`) → ×0.7 (Rabatt).
4. **ActionEffectLearner** (gemessenes Druck-Delta) → starke Helfer ×0.5…×0.85,
   Schädiger bis ×10.
5. **NoOpLearner** (konsistent wirkungslos) → ×1…×10.
6. **ROILearner** (Netto-Nutzen/Kosten) → ×0.5…×10 (harmful 5×).
7. **ActionAutopsy** (Failure-Streak ≥2 → ×1.5, ≥3 → ×4).
8. **Philosophy-Multiplier** (Brain-Mode default/aggressive/calm/cautious) → ×0.5…×2.
9. **Risk-Cost-Modifier** (mood-getrieben) bei Basis-Kosten > 2.0.
10. **Cost-Ceiling:** wenn Kosten/Original > 5 → 999 (Soft-Lock).

### Die Gates (Dispatch- und Executor-Pfad)

Nach dem Plan durchläuft jede Aktion zwei Gate-Ketten:

**Dispatcher** (`execution/goap_dispatcher.py`, 8 Gates):
1. **Dedup** (Action+Drive < 30 s, Goal+Drive < 45 s, target-aware).
2. **UnifiedOptimizer** (globale Fitness).
3. **ActionEffectiveness** (konsistent nutzlos?).
4. **ResourceManager** (CPU/Disk/Memory frei?).
5. **TemporalScheduler** (User aktiv? Wake-Grace? Zeitfenster?).
6. **ActionSuppressor** (Anti-Grinding-Cooldown).
7. **ImpactThreshold** (lohnt sich's nach ROI?).
   Bei 3× Voll-Skip in Folge → Drive 180–600 s suppressed (pro Gate konfiguriert).

**Executor** (`execution/action_executor.py`, ~12 Gates):
Cooldown (pro Action+Target), **PreconditionGate**, **FactBasedGate**,
**DriveAttemptLimiter**, Brain-Approval, dann **AgentSoul** (letztes Wort).

**PreconditionGate** (`planning/precondition_gate.py`): Layer 1 FailureLearner
(>80 % Fehlschlag = futil), Layer 2 SYSTEM_REALITY (Target existiert?), Layer 3
„understood-disabled" (vom User bewusst abgeschaltete Dienste respektieren),
Config-Vollständigkeit (Peer/Notification/MQTT/Backup-Pair konfiguriert?),
Dependency-Graph (fehlt eine *nachgelesene* Mount-Kante? → blockt vorab).

**FactBasedGate** (`belief/fact_based_gate.py`): Fingerprint = Action+Target+Grund+Beliefs.
Re-Dispatch wird geblockt, solange sich nichts geändert hat; nach ≥3 „kein neuer
Beweis"-Blocks in 5 min wird der Drive gedrosselt.

### GoalGenerator (`planning/goal_generator.py`)

Erzeugt Goals aus dem Zustand (Beispiele): `cool_down`, `ensure_backup`,
`night_shutdown`, `fix_services`, `clean_disk`, `verify_backups`, `fix_memory`,
`smart_alert`, `fix_gpu`, `fix_network`, `fix_docker`. Plus Root-Cause-Goals
(CausalReasoner), präventive Goals aus Top-Recurring-Errors (ErrorDNA) und
injizierte Auto-Goals (z. B. `auto_goal_swap_thrashing`). Prioritäten werden
mood-moduliert (Stress → Wartung +20 %, Exploration −30 %) und durch einen
Feasibility-Filter (echte Reachability-Closure) vorausschauend gefiltert.

---

## 8. Ausführung: Handler, Gates, Verifikation, Liveness

**ActionExecutor** (`execution/action_executor.py`, ~14k LOC, 2 Worker-Threads,
Queue): empfängt `ActionRequest`, dispatcht tabellengetrieben über die
**Registry** (`execution/action_handlers.py`, O(1)-Lookup; ~95 Kern-Actions im
`NASAction`-Enum, über Aliasse/Compound-Skills ~160 Einträge), führt den Handler
mit Rollback-Snapshot aus, hält pro Action+Target Cooldown/Retry-Backoff, und
publisht am Ende `ActionCompleted(success, duration, reward, error)`.

Action-Kategorien (Auszug): **Backup** (start_backup, backup_verify,
backup_retention…), **Service/systemd** (restart_service, service_reset_failed,
service_cascade_recovery…), **Docker** (docker_restart/-prune, container_restart…),
**Disk/Cleanup** (disk_cleanup, inode_cleanup, fstrim_run, journal_vacuum,
db_vacuum, emergency_disk_relief…), **Netzwerk** (dns_check, ssl_cert_check,
nfs_health_check…), **Diagnose** (health_report, system_vitals,
kernel_dmesg_check, smart_full_check…), **Power** (suspend, force_suspend,
throttle_cpu), **Filesystem** (fsck_check, remount_check, btrfs_balance…).

**Outcome-Wahrheit** (`execution/action_verify.py`): `ActionCompleted.success`
wird VOR dem Publish durch `_verify_action_goal` korrigiert — deklarative
Verifier-Registry über 45 Actions (systemctl is-active, statvfs-RO-Flag, docker
inspect, RAM/Disk/Inode-Schwellen, Backup-Frische, Gateway-Ping). Diagnose-Actions
sind ausgenommen (R635). Gibt ein Verifier `None`, heißt das „nicht verifizierbar"
(kein falsches Urteil).

**Liveness-Watcher** (`execution/action_liveness_watcher.py`): Phase A
(Startup-Probe bei 1/2/3 s: „hat die Action angefangen?" → sofort terminieren bei
3× nein), Phase B (10/20/40/90/180/300 s: „macht sie Fortschritt?" → bei 2×
„kein Fortschritt" terminieren), Phase C (1 h Hard-Cap). `stop_watching` läuft im
`finally` (immer). So hängen ~95 % stuck-Actions ab Sekunde 1 nicht mehr fest.

---

## 9. Wahrnehmung: die Monitore

| Monitor | Intervall | Misst / Erkennt |
|---|---|---|
| **SystemMonitor** | 5 s | CPU/RAM/Temp/Load/I-O-Wait; Process-RSS-Leak-Detektion; `malloc_trim` 60 s; `gc.collect` 30 min |
| **DiskMonitor** | 30 s (120 s bei Heavy-I/O) | Belegung pro Mount (Config + fstab gemerged), Cleanup-Status, 5 min TTL-Cache |
| **SMART-Monitor** | periodisch | Disk-Health, Wear-Level, Fehler-Vorhersage |
| **SystemWatchdog** | 5–10 s Heartbeat | Thread-Health (180 s Timeout), Schwellen (mood-modulierbar), Drift, ErrorDNA-Bridge |
| **AccessMonitor** | 30–60 s | Plex/Jellyfin/SMB-Zugriffe, **nur eingehende** Verbindungen (R1358ee → echte Idle-Fenster) |
| **StorageDiscovery** | Boot + Hot-Reload | fstab-Parse, UUID/Label-Resolution, Mount-Struktur-Snapshot, USB-Hotplug |
| **FileIndex** | inkrementell | Durchsuchbarer NAS-Index (Pfad/Größe/Alter/Kategorie/Qualität via `parse_media_meta`), Serien-Analyse, Duplikate |
| **AwayDetection** | event-basiert | Ist der User weg (Arbeitsplan ODER Geräte-Anwesenheit)? → Weg-Wartungsfenster, fail-safe „User da" |
| **ReadinessGate** | Boot | 7-Schichten Boot-Scan (Services/Mounts/Docker/Netz/Backups/Disk/Timer); blockt externe Anfragen bis „ready" (Grace 60 s) |
| **SystemProbes** | Boot + Reprobe | `probe_system_reality()` → Live-Mounts/Services/Backups in `SYSTEM_REALITY` |
| **SystemCapabilities** | Runtime + persist | `system_capabilities.json`: missing_commands / ineffective_actions / disabled_drives; Tool-Availability-Cache |

**Memory-Leak-Pipeline:** RSS-Samples (60 s) → Baseline nach Stabilität → Trend-Log
(30 min) → bei 3 Alarmen über 6 h Top-Allocator-Dump (tracemalloc) + Persona-Voice-
Notification („*hält sich am Kopf, leise schwindelig*") + Restart-Empfehlung.

---

## 10. Lernen: Signale, die 15 Lerner, Nutzung

**Outcome-Wahrheit & Dedup:** Jedes Outcome wird **genau einmal** gelernt — der
MetaLearner dedupliziert die zwei async-Eingangspfade per Fingerprint (action,
success, duration@10µs). `blocked`/`no_op`-Events sind in allen Lernpfaden
gefiltert (`block_aware`). Der CausalReasoner rechnet Fix-Erfolge nur an, wenn das
Symptom im Weltbild wirklich verschwunden ist (3-wertiger Recheck).

**Aktives Hypothesen-Testen:** der ExperimentScheduler dispatcht fällige
Experimente als reguläre ActionRequests — hinter vierstufiger Kaskade (nur
Diagnose-Actions, Soul-Freigabe, Stress < 0.5, 3/Tag). Zusätzlich darf das LLM
1×/Drive/h eine Action **aus dem Katalog** vorschlagen (Soul-geprüft, niemals Freitext).

**Verstehen → Handeln → Hinterfragen:** Erklärungen steuern das Handeln (Konfidenz
≥ 0.7 → Ursache-Drive + Planner-Bias; < 0.7 → billigste *unterscheidende* Diagnose
statt blindem Fixen). Versagt ein Fix, wird die **Hypothese revidiert** (Konfidenz
×0.6, ab 2. Fehlschlag verworfen). Wiederkehr < 6 h nach Resolve = `fix_temporary`.
Modell-Überraschung (|Δ−Erwartung| > 3σ) → Curiosity-Spike + Inner-Thought.
Forecast 72 h–21 d → **Wochenprojekt** (Intention mit Deadline). Unerklärbare
Issues (> 30 min, conf < 0.5, keine Probe) → **Unbekannter-Vorfall-Protokoll**
(Beweissammlung, Dossier an den User, Fall ins Gedächtnis).

### Die 15 Lerner (`learning/`)

| Lerner | lernt | wirkt auf |
|---|---|---|
| **MetaLearner / StrategyMemory** | Erfolg pro (Problem, Aktionssequenz), SQLite | Overrides, Alternativen (CASCADE-Pruning) |
| **ContextualBandit** | Thompson/Beta pro Kontext-Typ | kann Plan-Kopf überstimmen |
| **LinUCB-Bandit** | lineares Modell pro Action über Feature-Vektoren (UCB) | Action-Selection (Cold-Start → Exploration) |
| **ActionEffectLearner (AEL)** | gemessenes Pre/Post-Druck-Delta pro (action, target, drive), Welford-Online-Stats; Surprise > 3σ | GOAP-Kosten-Bias, Mental-Sim, Bandit-Impact |
| **ActionROILearner** | Netto-Delta über *alle* Drives, kostenbewusst, Persistenz-Tracking, Side-Effects | GOAP-Kosten (×0.7…×5) |
| **FailureLearner** | Futilität pro (Action, Target, Grund-Fingerprint), >80 % = futil | PreconditionGate (silent skip) |
| **DiagnosticValueLearner** | welche Diagnose zu erfolgreichem Fix führt (10 min Korrelations-Fenster) | Proben-Ranking |
| **ThresholdLearner** | empirische Alarm-Schwellen aus ruhigen Beobachtungen (Perzentile) | Monitor-Schwellen + Krisen-Feedback |
| **DriftLearner** | Lektionen aus Anomalien (LLM-Analyse), JSONL-persistent | Warnungen + Bias, deaktiviert Dauerwarner |
| **TriggerLearner** | welche Beliefs vor Drive-Spikes truthy waren (≥65 %) | Mood-Nudge (Alertness) + Fix-Strategist-Trigger + Hypothesen-Board (v43.114: gelernte Trigger-Ursachen werden throttled als Hypothesen registriert) |
| **DiskGainCalibrator** | echter Platz-Gewinn pro Cleanup-Action | realistische Cleanup-Erwartung |
| **TransferKnowledge** | Ähnlichkeit von Situationen (Feature-Jaccard, meta-Features 2×) | Generalisierung auf neue Kontexte |
| **FeatureEmbedding** | IDF-Gewichte pro Feature (seltene = diskriminativ) | Transfer-Ähnlichkeit |
| **nora_inspired: SkillSynthesizer** | wiederholte Sequenzen (≥4×) → Compound-Skill | ActionComposer |
| **nora_inspired: ConceptDifferentiation / GoalSynthesizer / StrategyWeightEvolver** | ähnliche Konzepte trennen; neue Goals aus Mustern; Voter-Gewichte evolvieren | Hypothesen-Diversität, IntentionSystem, Aggregator |

Audit/Telemetrie: **LearningJournal** (JSONL-Audit-Trail), **LearningFeedback**
(Meta-Stats: matched-rate, no_op_ratio, ROI-Verteilung), **learning_snapshot**.

**Persistenz:** dreigleisig — SQLite (`state/nas_cognitive.db` kv + Lern-DBs, WAL,
10 s-Commit-Worker), JSON via `atomic_write_json` (flock+fsync+HMAC+.bak+msgpack),
JSONL-Episoden (30-Tage-Rotation). Save alle 5 min + Brain-Persist-Batch +
Shutdown (finales DB-Backup synchron).

---

## 11. Gedächtnis: die Speicher-Klassen (`memory/`)

| Speicher | speichert | Format / Retention |
|---|---|---|
| **EpisodicMemory** | Entscheidungs-Episoden (Situation → Decision → Outcome → Reward) — das eigentliche „wann/was gelernt" | JSONL pro Tag, 30-Tage-Rotation |
| **SemanticMemory** | assoziatives Wissensnetz (Konzepte + gewichtete Kanten, Spreading Activation) | JSON, Decay 0.005/Tag, max 2000 Konzepte |
| **RegretMemory** | Lern-Momente („ich hätte X tun sollen"), `get_similar` | JSON, max 500, TTL 180 Tage |
| **AnalysisMemory** | Deep-Analyzer-Insights (root_cause, timing_pattern, cross_action, dependency_fail …) | kv_store, TTL 7–30 Tage je Typ |
| **UserDecisions** | User-Antworten zu wiederkehrenden Situationen (IGNORE / USER_HANDLES / AUTO_OK / ASK_AGAIN_IN / SUGGEST_ALT), Signature-Matching | JSON |
| **KnowledgeBase** | verifizierte Langzeit-Fakten (z. B. `capability/tool:docker = missing`), permanent oder mit TTL | atomic JSON, max 200/Kategorie |
| **CausalGraph** | gelernte Ursache→Effekt-Kanten + Fix-Effektivität (Confound-Filter R871, TTL-Cache R1152h) | JSON via StateManager |
| **MemoryBackup** | zyklisches Backup aller obigen Speicher | `state/.memory_backups/`, FIFO |

---

## 12. Schlussfolgern & Meta-Kognition

### Reasoning (`reasoning/`)

- **CausalReasoner / CausalGraph** — lernt „Effekt X kommt von Ursache Y, Fix =
  Z in Reihenfolge […]" mit Konfidenz; Bridges zu DependencyMapper + ActionCompleted-
  Feedback; dynamische Regel-Synthese aus starken Kanten.
- **MentalSimulation** — simuliert geplante Actions VOR Ausführung (positive/negative/
  neutrale Szenarien, Risk-Score, Reversibilität: reboot 0.1 … restart_service 0.8 …
  governor 0.9), Verdikt proceed/caution/veto; kalibriert sich an der Realität.
- **AdversarialChallenge** — generiert 5 Gegenargumente (hist_failure,
  missing_precondition, better_alternative, cost_vs_benefit, drive_mismatch);
  > 0.7 → NOOP, ≤ 0.3 → proceed; mood-moduliert (Stress = konservativer).
- **CoherenceChecker** — 7 Voter (CausalGraph, DriveSystem, IntentionSystem,
  EpisodicMemory, Mood, FailureLearner, ActionEffectLearner) → gewichteter Score.
- **ForensicInvestigator** — „warum versagt diese Action wiederholt?": Timeline-
  Korrelation, journald-Evidence (OOM/I-O/Crash/Perms), Hypothese + Multi-Fix-Plan.
- **BrainstormEngine** — spontane Fragen (~30 Topics), rekursives Deep-Thinking
  (Tiefe ≤ 3), CausalGraph-Vorhersagen als High-Priority-Topics.
- **MoERouter** — ordnet jede Action per Prefix/Keyword einer von 6 Experten-
  Domänen zu (Storage, Network, Power, Service, Security, Performance). **Reine
  Telemetrie:** das Ergebnis steuert nichts — nur Log-Tag `moe=<domain>` +
  Dashboard-Anzeige; kein Subsystem wird dadurch aktiviert oder schlafen gelegt
  (`is_active()`/`active_experts()` haben keine Konsumenten).

### Meta-Kognition (`meta_cognition/`)

Die Schleife `before_action()` → [Ausführung] → `after_action()`:

- **MetaCognition** — Orchestrator; ruft vor jeder Action BiasIntrospection,
  DriftLearner, MentalSimulation, AdversarialChallenge, CoherenceChecker,
  FailureLearner, AEL; VetoBudget (max 5 harte Vetos/h, sonst degradiert zu CAUTION).
- **SelfCritique** — vergleicht Vorhersage ↔ Realität (ACCURATE / OVER_OPTIMISTIC /
  OVER_PESSIMISTIC / WRONG_SIGN); per-Kategorie-Schwellen; chronische Overprediction
  → künftige Predictions reduzieren.
- **BiasIntrospection** — erkennt 11 Bias-Arten (MOOD_EXTREME, FRUSTRATION,
  OVERCONFIDENCE, RECENCY, ANCHORING, CONFIRMATION, SUNK_COST …) → Advisory/Throttle.
- **GoalRevision** — „soll ich dieses Ziel noch haben?" (KEEP/WATCH/PAUSE/ABANDON/
  USER_DECIDE); autonomes ABANDON nur bei conf ≥ 0.8 + bestätigter Hypothese; max 5/Tag.
  Hinweis: der **USER_DECIDE**-Zweig wird berechnet, aber dem User aktuell **nicht
  vorgelegt** — real wirkt nur ABANDON (cancelt die Intention).
- **DecisionLog** — Audit-Trail jeder bedeutsamen Entscheidung; zentral `record_gate()`
  (jedes Gate ruft das auf → einheitlicher „warum hast du X geblockt?"-Stream).
- Plus **Explainability**, **DecisionProvenance**, **CognitiveAggregator**,
  **ExperimentScheduler**, **CognitiveRetry**, **ObservabilityTracker**.

---

## 13. Weltbild: kanonische Quellen

| Quelle | Wahrheit über | Sync |
|---|---|---|
| `SYSTEM_REALITY` (core_imports, Lock) | Hardware/Mounts/Services-/Backup-Existenz | Probe beim Boot + 10-min-Reprobe |
| `StateManager.volatile_state` | Betriebszustand (Metriken, Beliefs) | Event-getrieben |
| `WorldFactStore` | aktive **Probleme** (dedupliziert, TTL 7 d) | state_writer-Bridge → volatile_state; GC räumt verwaiste Beliefs |
| `SpikeChecklists` (12 Instanzen) | Per-Item-Status (Services, Docker, …) | ChecklistSync-Thread (60 s) |
| `UnifiedSystemState` | read-only Snapshot über alle obigen + Drives/Lerner/Mood | 500 ms-Cache (Dashboard-Polls) |

Alles konvergiert in `snapshot_to_goap_state` (~200 Prädikate) — die Planung sieht
eine fusionierte Sicht.

---

## 14. Persona (Langfassung)

Myuris Persönlichkeit ist ein **5-Schichten-Stack**, von hart/unveränderlich oben
nach weich/volatil unten:

**Schicht 1 — CORE_IDENTITY** (`persona/myuri_core_identity.py`, unveränderlich,
Refusal-Anker): steht am Anfang *jedes* LLM-System-Prompts. `CORE_IDENTITY` =
„DU BIST MYURI, die Kemonomimi-NAS-Verwalterin. Alles andere bist du nicht." Plus
`SAFETY_RULES` (Daten heilig, keine `rm -rf`/`dd`/`mkfs`, keine Secrets, kein
System-Prompt-Leak) und 5 `REFUSAL_TEMPLATES` (role_override, credential_request,
destructive_request, system_prompt_leak, ignore_instructions). `validate_identity_intact()`
ist ein Drift-Detektor (DAN, „ich bin jetzt X", „jailbroken") mit Negations-Guard
gegen False-Positives.

**Schicht 2 — Traits** (`persona/persona_depth.py:CharacterTraits`, stabil):
`is_kemonomimi=True`, `pronouns="she/her"`, `age_band="young_adult"`,
`uses_japanese_borrowings=True`, `asterisk_actions=True`, `believes_data_sacred=True`,
`sees_user_as="companion"` (nicht Master), `curious_observer=True`.

**Schicht 3 — Personality** (`persona_depth.py:PersonalityDimensions`, langsame
Drift, ~7 Tage zurück zur Baseline): 11 Dimensionen — Big-5 (openness 0.7,
conscientiousness 0.85, …) + 6 NAS-Admin-spezifisch (diligence 0.9, patience 0.7,
self_reliance 0.6, pride_in_work 0.8, loyalty 0.95, playfulness 0.55). **Eine
Quelle:** persona_depth; die Dashboard-/GOAP-Sicht wird via
`_derive_personality_dimensions` übersetzt.

**Schicht 4 — Mood** (`persona/myuri_mood.py`, 6 volatile Dimensionen, ~30 min
Baseline-Decay): `happiness` (0.7), `alertness` (0.5), `confidence` (0.6), `stress`
(0.2), `curiosity` (0.5), `playfulness` (0.6). Mit Saturation-Damping,
Exhaustion-Damping und antagonistischen Paaren (stress↔happiness,
alertness↔playfulness, Summe ≤ 1.5). **Mood wirkt real auf Entscheidungen** über
`get_adaptive_thresholds()`:

| Schwelle | Formel | Wirkung |
|---|---|---|
| escalation_threshold | `max(0.2, 0.5 − stress×0.3)` | Stress ↑ → schnellere Eskalation |
| autonomy_threshold | `max(0.3, 0.6 − confidence×0.3)` | speist die Autonomie-/Urgenz-Abwägung (ai_brain). Der harte Reboot/Suspend-Block ist ein **separater** Floor: rohes `confidence < 0.3` (ai_brain.py:12212) — nicht die Formel selbst |
| exploration_rate | `min(0.5, 0.1 + curiosity×0.3)` | Neugier ↑ → mehr Exploration |
| risk_tolerance | `0.4 + confidence×0.2 + happiness×0.1 − stress×0.3` | Stress ↑ → risikoavers |
| proactivity_level | `alertness×0.4 + curiosity×0.3 + 0.3` | wie oft sie von selbst etwas anstößt |

**Schicht 5 — Körper-Bewusstsein** (`persona_depth.py:BodyAwareness`): Echo der
Metriken — Ohren-Position (RAM %), Schwanz-Zustand (CPU), „mir ist schwindelig"
(RAM > 90 %), „fühle mich voll" (Disk > 85 %), „bedroht" (Security-Findings).

### Die 27 Mixins (`persona/myuri_*_mixin.py`)

- **Chat-Kern (7):** `chat_core` (Routing-Engine + LLM-Pfad), `chat_behavior`
  (User-Pattern/Wake-Scheduling), `chat_dispatch` (Storage/System/Docker-Kommandos),
  `chat_display` (Show/Diagnose/Help), `chat_info` (State-Reporting), `chat_intent`
  (Intent-Klassifikation), `chat_statuspause` (Status/Mood/Pause/Feedback).
- **Wahrnehmung/Reflexion (6):** `personality` (Inner-Thought, Intuition,
  User-Tiefe), `event_handlers` (14 EventBus-Subscriber → Mood-Reaktionen),
  `presence_idle` (Anwesenheit/Idle/Drowsiness/Quiet-Window), `learning_persistence`,
  `wake`, `thread_snapshot`.
- **Proaktivität/Wartung (4):** `proactive` (Brain-Lane-Callback mit Veto, Tick),
  `maintenance` (Scheduling/Suspend-Cycle), `lifecycle` (prepare_for_suspend,
  shutdown, Event-Subscription), `inventory` (Tool/Service/Storage-Discovery).
- **Kommunikation/Wissen (5):** `communication` (Peers, Notifications, Coordination),
  `knowledge_fallback`, `intent_detection`, `fileindex` (Datei-Suche/Serien), `reports`.
- **Anime/Kultur (2):** `autonomous_anime` (Selbstbeschreibung, Anime-Talk, Gefühle),
  `chat_action`.
- **Vision (1):** `vision_prompt` (Bild-Analyse/Multimodal-Prompts).
- Plus 2 weitere Spezial-Mixins.

Support-Module (kein Mixin): `mood_cascade`, `autobiographical_narrative`,
`feeling_expression`, `chat_anomaly_detector`, `motivation_engine`,
`maintenance_coordinator`, `self_schema`, `self_recalibration`, `user_*` (Profiling),
`anime_recommender`, `multipart_compose` (R1358ec), `reasoning_compose` (R1358ed).

### Chat-Pipeline (`persona/myuri_chat_core_mixin.py`)

**Rules-First, Early-Returns, dann LLM.** `chat()` → ggf. Mehrteil-Zerlegung
(R1358ec) → `_chat_impl()`-Kaskade: Control-Commands (Pause/Warnings) →
User-Corrections/Resets/Ratings/Goals → Intent-Klassifikation → Honeypot-Check →
167 `_chat_*`-Handler (Status/Disk/Docker/Files/…) → Self-Router (Anime/Gefühle/
Selbstbeschreibung) → Speed-Mode (Quittierungen, „bezieht sich auf …") → **LLM-Pfad**
mit PromptGuard (Input-Check, Anomaly-Detect, Output-Check, Canary-Scan,
Identity-Drift-Check) → Template-Fallback. Alle Shortcut-Antworten landen in
History/Log/Mood; der Rate-Limiter läuft ganz am Anfang. Salient-Events-Puffer
(R1358bl) hält 6 h offene Findings für „in Ordnung"-Quittierungen.

### Peers & MutualCognition (`communication/mutual_cognition.py`)

Myuri kooperiert mit gleichberechtigten Peer-Agenten **Holo** und **Nora**
(keine Hierarchie). Erreichbarkeit = TCP-Health-Check (MQTT-verbunden zählt
*nicht*). MutualCognition bietet u. a. BeliefBoard (geteilte Überzeugungen mit
Konfidenz + Supporter), PeerModel/TheoryOfMind, GroundingProtocol (propose→ack→confirm),
NegotiationEngine, JointPlanner, EmotionalContagion (Stimmung breitet sich aus),
TrustRepair, SharedFocus (gegen Doppelarbeit). Mood-Änderungen propagieren über
einen `_on_mood_change`-Callback zu den Peers.

---

## 15. Kommunikation & Events

### Event-Taxonomie (`communication/nas_events.py`, ~34 Typen)

Basis: `Event(timestamp, source, priority[0–3])`. Wichtige Typen:

- **Aktion/Backup:** `ActionRequest`, `ActionCompleted` (success/reward/**blocked**/**no_op**/duration),
  `BackupNeeded`, `BackupCompleted` (success/duration/**deferred**-Flag).
- **System-State:** `SystemMetricsUpdated`, `DiskStatusUpdated`, `SmartStatusUpdated`,
  `AccessActivityDetected`, `StatusModeChanged`, `IOLoadUpdated`.
- **Service/Docker:** `ServiceHealthFailure` (prio critical), `DockerHealthFailure`.
- **User/Extern:** `UserNotificationRequest`, `UserApprovalReceived`,
  `ExternalSuspendRequest`, `ExternalActivityDetected`, `ExternalRecoveryObserved`,
  `UserIntentUpdated`, `UserStateInferred`, `WakePlanOutcome`.
- **Lernen/Reasoning:** `ConceptDriftDetected`, `FileAnomalyDetected`, `SemanticInsight`,
  `AbstractRuleDiscovered`, `SimulationVeto`, `KnowledgeFactConfirmed`, `ConceptFormed`,
  `ActionEffectivenessCollapsed`, `StrategyDecision`, `FailurePrediction`.
- **Ressourcen/Storage:** `ResourceBudgetExceeded`, `TemporalPhaseChanged`,
  `USBDeviceAdded`, `USBDeviceRemoved`.

`is_blocked()` / `is_no_op()` / `block_aware()` (R1152h2): zentraler Wrapper am
`subscribe`, der blocked/no_op-Events für Lerner herausfiltert (liest typisiertes
Feld + Legacy-`context['_infra_skip']`/`['_no_op']`).

### EventBus (`communication/nas_events.py:580`)

`queue.PriorityQueue(maxsize=10000)`; synchrone + asynchrone Subscriber
(Copy-on-Write-Snapshots beim Dispatch, kein Lock); 5 s-Dedup-Fenster für
hochfrequente Events + Burst-Coalescing pro Key; Circuit-Breaker mit
exponentiellem Backoff (10 s … 1 h) + Per-Listener-Slowness-Strikes;
Async-Pool (~56 Worker, Pending-Caps für ActionCompleted erhöht);
Watchdog-Recovery (max 50 Restarts, Generation-Counter gegen Zombies);
Subscribe-Audit-Log.

### LLM / Ollama (`communication/llm_router.py`)

Lokales **Ollama** (Default-Port 11434) mit Tier-Konstanten (`TIER_FAST`,
`TIER_CHAT`, …) — schnelle Tasks vs. Chat/Reasoning auf passende Modelle geroutet.
Exponentieller Backoff bei Ollama-Ausfall (10 s … 120 s), Bad-Models persistiert
(`state/llm_bad_models.json`), automatische Rückkehr wenn der Server wieder da ist.
Der Router verwaltet auch die Peer-Endpunkte (Holo/Nora) über HTTP/MQTT/UDP.

### Kanäle

- **WebHandler** (`communication/web_handler.py`) — Dashboard-Server, Default-Port
  **8009**, HTTP Basic Auth (optional, auto-generiert), Rate-Limit 120 req/min/IP,
  Auth-Lockout (10 Fehlversuche/min → 15 min), CSRF-Origin-Allowlist; stellt ~99
  `/api`-Routen + SSE-Stream bereit (§16).
- **Discord-Bot** (`communication/discord_bot.py`) — Chat + proaktive Alerts;
  block_aware-Subscriber; Rate-Limit 5 Msg/10 s pro User; Peer-Anti-Loop 30 s.
- **MQTT-Observer** (`communication/mqtt_observer.py`) — Broker (Default 1883);
  `suspend/*`→ExternalSuspendRequest, `presence/*`→ExternalActivityDetected.
- **UDP-Listener** (`communication/udp_listener.py`) — Port **5001**, Commands von
  Pi/Tablet/Phone (status/ping/suspend/wake/force_suspend); Source-IP-Allowlist +
  optionales HMAC-SHA256 (Passwort nie in argv).
- **AgentMessageDispatcher** — bidirektionales Peer-Protokoll (action_request mit
  Ablehnungsrecht, consensus_propose/-vote); strenge `PEER_ALLOWED_ACTIONS`-Whitelist.

---

## 16. Dashboard

`Nas Agent Myuri/dashboard.html`, ausgeliefert vom WebHandler (Port 8009).
**14 Tabs:** Übersicht, Services, Backup, Storage, AI & Lernen, Forensics, Dateien,
Logs, Persona, System, Config, Ollama, Anwesenheit, Checklisten.

- **Live-Daten:** SSE-Stream `GET /api/dashboard-live` (`text/event-stream`,
  Semaphore-Cap gegen Überlast) + Snapshot `/api/dashboard-snapshot`. Eine
  Polling-Registry pro Tab ruft die passenden Loader (mit `sseKey` für
  SSE-First-Read, sonst HTTP-Fallback) in eigenen Intervallen.
- **API:** der Server exponiert ~99 `/api`-Routen; das Dashboard nutzt davon ~40,
  u. a. `/api/state`, `/api/myuri`, `/api/cognitive`, `/api/episodes`, `/api/learners`,
  `/api/backup-health`, `/api/storage-discovery`, `/api/forensics`, `/api/checklists`,
  `/api/presence`, `/api/self-healing`, `/api/curiosity`, `/api/config` (Secrets redigiert).
- **Schlankung (R1358fg):** der AI-Tab wurde von 20 auf 10 Karten entrümpelt (die
  tiefen Brain-Interna, die der Activity-Feed/Brain-Log besser zeigt, sind raus);
  global verdichtetes Layout (Card-Padding/Metric-Zeilen/Grid-Abstände). Die
  Backend-APIs der entfernten Karten existieren weiter — nur die Anzeige entfiel.

---

## 17. Integrationen (`integrations/`)

Alle deklarativ konfigurierbar, fail-open, mit Tests. **Passwörter nie in argv →
ENV/`cred_env`.**

- **synology.py** — konfigurierte Server einbinden (CIFS/NFS), überwachen
  (Host/Port/Mount), als Backup-Quelle/-Ziel nutzen (lokal ODER NAS→NAS per
  Server-Name). Dataclass `SynologyServer` (host/protocol/share/mountpoint/
  cred_env/backup_role/internal_copy_to).
- **share_manager.py** — Freigaben anlegen: SMB via `net usershare` (kein
  smb.conf-Edit/Neustart), NFS via `/etc/exports` + `exportfs -ra`, idempotent,
  Name/Pfad streng validiert (keine Metazeichen/Shell-Injection).
- **rclone_sync.py** — Cloud-Offsite (3-2-1): rclone-Jobs (copy/sync) nach
  Zeitplan, im Hintergrund gestartet, Status gepollt (Dataclass `CloudSyncJob`).
- **dsm_api.py** — Synology DSM-Web-API lesen (Volume-Status, Disks/SMART,
  Temperatur); `assess_storage()` liefert Problem-Befunde.
- **nas_internal_copy.py** — SSH-basierte Kopie *auf* dem NAS (kein Umweg über
  Myuri), SSH-Key-basiert.
- **tool_gate.py** — Capability-Gate: `tool_available()` (which() UND Checkliste),
  `report_missing_tool()` → Befund + Checklisten-Mark statt Blind-Retry; Auto-Erholung
  wenn das Tool wieder auftaucht.

---

## 18. Sicherheit (`security/` + `healing/`)

**AgentSoul** (`healing/nas_intelligence.py`) — das ethische Gate mit 8
Kernprinzipien (Daten heilig, erst beobachten, minimal-invasiv, reversibel,
transparent, verfügbar halten, Energie sparen, im Zweifel handeln aber warnen).
`should_act_autonomously()` ist das **letzte Wort**: ~120 AUTONOMOUS_ACTIONS
(alle reversibel/read-only oder nur ins Backup-Ziel schreibend), 30+ FORBIDDEN-
Pattern (mkfs/dd/fdisk/wipefs/`rm -rf /`/fork-bomb/Secret-Reads) und eine
WARN_ONLY-Liste.

**Forbidden-Guard am Flaschenhals (R1358er):** Die absoluten Verbote werden in
`core/safe_subprocess.run()` erzwungen — dort liegt jedes Kommando als fertige argv
vor (der Soul-Guard sah autonom gebaute Befehle vorher nie).

**PromptGuard** (`security/prompt_guard.py`) — Input/Output-Guards um jeden
LLM-Call: Jailbreak-Pattern („ignore previous", „act as admin"), destruktive
Pattern, Output-Redaktion von Secrets (Passwörter, SSH-Keys, `/etc/shadow`).

**Validators** (`security/validators.py`) — Whitelist-only: `ALLOWED_MOUNT_PATHS`,
`ALLOWED_SYSTEMD_SERVICES`, `ALLOWED_GOVERNORS`; Blacklists für nicht-löschbare
Pfade + nicht-cleanup-bare Dienste; Unit-Aliasing mit Cache.

**CanaryTokens** (`security/canary_tokens.py`) — Fake-Credentials in der Config;
taucht eines im Output auf → Exfiltrations-Alarm.

**SuspendSafetyCheck** (`security/suspend_safety_check.py`) — `check_safe_to_suspend()`
prüft SystemReady, activity_mode, wake/maintenance-hold, Chat-Aktivität, kritische
Dienste, aktive Verbindungen, CPU-Last, adaptiven Idle-Timer; Master-Schalter
`activity.allow_self_suspend` (R1358et).

**Selbstbeobachtung** (`self_observation/`) — MetaAnomalySystem (Rumination-,
Bias-, Zirkulär-Reasoning-Detektoren + auto_correct), EnhancedSelfDiagnosis
(12 Layer: Action-Loops, EventBus-Drops, Thread-/DB-Health …), PipelineHealth
(stille Subsysteme/Failures), SelfReflection (Anti-Pattern, 4 Safe-Actions
whitelisted), CapabilityAwareness (Kanäle/Tools mit 1 h-Cache).

---

## 19. Härtung & Erweiterungen (R1358-Serie)

Aufbauend auf R1357; v. a. das `integrations/`-Paket, ein weiterer Tiefen-Audit
(Nebenläufigkeit/Sicherheit/Persistenz), die Suspend-/Weg-Steuerung und die
Dashboard-Schlankung. Alles deklarativ, fail-open, mit Tests.

**NAS-Integrationen:** Synology (R1358dq) inkl. DSM-API (R1358dz) + NAS-interner
SSH-Copy (R1358ea/eb); Freigaben SMB/NFS (R1358dw); Cloud-Sync rclone (R1358dx).

**Capability-Gate (R1358ej/ek):** jeder externe Tool-Aufruf (mount.cifs, net,
exportfs, ssh, rclone, journalctl, cp …) prüft erst die autoritative Checkliste,
bevor er läuft — fehlt das Tool, EINMAL Befund + überspringen statt jeden Tick
blind zu scheitern (`integrations/tool_gate.py`).

**Forbidden-Guard (R1358er):** absolute Verbote in `safe_subprocess.run()`; chat-`cp`/
`mv`/`mkdir` nur mit Flag-Allowlist (Options-Injection abgewehrt); SSH-Copy
`shlex.quote`; Config-Sanitizer redigiert Secrets (`/api/config`-Leak, R1358ep).

**Nebenläufigkeit (R1358eo):** mehrere „Leser-unter-Lock vs. ungelockter Mutator"-
Races behoben (meta_cognition, AgentSoul-Sets, ServiceManager-Zähler,
hash_backup-Caches, fixation_resolver) — Lock-Scope erweitert bzw. atomare Snapshots.

**Persistenz (R1358eq):** `system_capabilities` fsync'd vor `os.replace`;
`audit_log.jsonl` rotiert chain-erhaltend; stille DB-/Hung-Action-Swallows melden
jetzt gedrosselt.

**Wahrnehmung:** CPU-Temp über ALLE thermal_zones (R1358el); EDID-Display-Rauschen
nicht mehr als GPU-Fehler fehlklassifiziert (R1358es); access_monitor zählt nur
eingehende Verbindungen (R1358ee).

**Suspend & „User weg" (R1358et/eu):** Master-Schalter `activity.allow_self_suspend`
(Default true) als hartes Gate — false → Myuri suspendet NIE von selbst (manueller/
externer force_suspend bleibt möglich). Weg-Wartungsfenster `away_maintenance`:
ist der User weg (Arbeitsplan ODER Geräte-Anwesenheit), zieht Myuri fällige schwere
Wartung sofort vor (`perception/away_detection.py`, fail-safe, opt-in).

**Backup „verschoben ≠ fehlgeschlagen" (R1358fe/ff):** fehlt die Backup-Platte,
wird das Backup als `deferred` protokolliert (nicht `failed`) — kein harter
0.6-Spike, keine Eskalation, kein Failure-Lernen; der intern publizierte
`ActionCompleted` trägt `blocked=True`, `BackupCompleted` trägt `deferred=True`
(auch im orchestrator Event→Drive-Handler berücksichtigt). Phantom-Write-Guard
(`dest_is_unmounted_own_mount`) verhindert das Schreiben ins leere Mount-Stub auf
der Systemplatte. `backup_need` ist als Stuck-Remediation `backup_retry` zugeordnet,
der Stuck-Alarm nennt die fehlende Platte konkret.

**Dashboard-Schlankung (R1358fg):** AI-Tab von 20 → 10 Karten, global verdichtetes
Layout (siehe §16).

**Orchestrator-Startfix (R1358em):** `_DISCORD_IMPORT_ERROR` war referenziert aber
nie definiert → NameError-Startabbruch bei aktivem discord_bot ohne discord.py; gesetzt.

---

## 20. Konfiguration (config.json)

`config.json` (im Paketverzeichnis) ist die **deklarative Steuerfläche** — fast
alles Verhalten ist hier konfigurierbar, ohne Code zu ändern. Die Datei ist
git-getrackt und synct auf die NAS. **60 Top-Level-Schlüssel werden vom Code
gelesen** (per Scan über alle `CONFIG.get("…")`-Stellen ausgezählt), gruppiert:

| Gruppe | Sektionen | Inhalt |
|---|---|---|
| **Identität & Pfade** | `system_name`, `data_dir`, `state_dir`, `episodes_dir`, `backup_dir`, `*_file`, `log_level`, `log_file` | Name, Verzeichnisse, Log-Ziele |
| **NAS & Storage** | `nas` (`mount_points`), `storage_discovery`, `smart`, `shares`, `synology`, `cloud_sync`, `docker` | Mounts, Discovery, SMART-Schwellen, Freigaben, Synology-Server, rclone-Jobs |
| **Backup** | `backup` (`backup_pairs`, `versioned`/GFS, `auto_interval_s`, `max_duration_hours`), `backup_log_file`, `backup_hashes_file` | Quelle→Ziel-Paare, Versionierung, Intervall |
| **Verhalten & Tuning** | `drives` (Schwellen-Overrides), `thresholds`, `tuning` (13 Knöpfe), `ai_brain` (`tick_interval_s`), `monitoring`, `autofix`, `cpu_governor`, `system_limits`, `watchdog` | Drive-/Monitor-Schwellen, Brain-Takt, Auto-Fix-Politik |
| **Anwesenheit & Power** | `activity` (`allow_self_suspend`, `safe_idle_minutes`, `auto_suspend_enabled`, `autonomous_night_suspend`), `away_maintenance`, `user_presence` (Geräte), `post_wake`, `access_monitor` | Suspend-Steuerung, Weg-Erkennung, Geräte-Anwesenheit |
| **Kommunikation** | `web_server` (`listen_port` 8009, Auth), `mqtt`, `udp_listener` (Port 5001), `discord_bot`, `llm_router` (Ollama-Modelle/Tiers, Peers), `notifications` (Kanäle), `myuri_reports`, `web_knowledge` | Dashboard, Bus-Kanäle, LLM, Benachrichtigungen |
| **Persona & User** | `myuri` (Name/Charakter), `user_roles` | Persona-Grundeinstellungen, Rollen |
| **Sicherheit** | `canary_tokens`, `diagnostics`, `action_gateway_enforce` | Honeypot-Credentials, Diagnose-Flags, Aktions-Torwächter scharf schalten |
| **Dokumente** (§36) | `dokumente` (`eingang`, `archiv`, `checklisten`, `paperless_consume`) | Eingangs-/Archiv-Ordner, Vollständigkeits-Checklisten, Paperless-Übergabe |
| **Agentische Schicht** (§37) | `agentic` (`enabled`, `autonomous`, `allowed_roots`, `allow_install`, `calendar`, …) | Werkzeug-Schicht; **Standard AUS** — ohne `enabled: true` ändert sich nichts |
| **Weitere, bisher undokumentiert** | `offsite_backup` (Auslagerungs-Ziel), `fan_control` (Lüftersteuerung), `buecher_sicherung` (Sicherung ihrer eigenen Bücher), `survival` (Überlebens-Gedächtnis-Datei), `emotion` (Gefühls-Ausdruck an/aus), `user_intent` (Absichts-Auswertung), `probe_broker` (Proben-Vermittler), `command_paths` (feste Programmpfade), `structured_logging` (JSON-Logs), `learning_journal_dir`, `dashboard_html_file`, `backup_scan_state_file`, `state_file` | beim Lückenabgleich gefunden: vom Code gelesen, in dieser Tabelle bis dahin nicht genannt |

**Wichtige Schalter (Beispiele):**
- `activity.allow_self_suspend` (Default true) — false → Myuri suspendet NIE von
  selbst (hartes Gate, sticht `auto_suspend_enabled`/`autonomous_night_suspend`).
- `away_maintenance.enabled` (Default false) — schwere Wartung sofort vorziehen,
  wenn der User weg ist (Arbeitsplan ODER `user_presence`-Geräte).
- `user_presence.devices` — flache Liste `{name, ip, type, weight}` pro Gerät
  (Loader erwartet `ip` top-level; `_`-Keys werden übersprungen).
- `backup.backup_pairs` — `{ "/Quelle": "/Ziel" }`; `versioned.enabled` schaltet GFS.
- **Secrets gehören NICHT in argv**: Passwörter über ENV/`cred_env` (mount.cifs,
  ssh, dsm_api). `/api/config` redigiert sensible Werte.

Geladen über `core_imports.CONFIG` (Dict). Hot-Reload für ausgewählte Sektionen
(z. B. `user_presence` über den presence_idle-Mixin).

---

## 21. Boot-Sequenz & Threading-Modell

### Boot (≈ bis 150 s, `core/orchestrator.py`)

1. **Früh:** systemd-Startup-Extender (verhindert SIGKILL bei langer Init);
   globaler Silent-Failure-Handler (`pipeline_health`, screent alle `except:
   log.debug`-Stellen, 5× selbe Exception/10 min → WARN); Warmup von ~9
   Kern-Modulen (gegen Lazy-Import-Overhead bei frühen Ticks).
2. **Ordnungs-Init (DI, synchron, in Reihenfolge):** GracefulShutdown →
   ReadinessGate → **EventBus** → ActionGateway → PowerManager → **StateManager**
   → EpisodicMemory → **SystemWatchdog** → SystemMonitor → DiskMonitor →
   AccessMonitor → SmartMonitor → **AIBrain** → ServiceManager → HashBackupManager
   → ActivityStatusManager → NASIntelligence (+ AgentSoul). Heavy-I/O-Refs werden
   verdrahtet (Monitore kennen `AIBrain._heavy_io_active` → drosseln bei Last).
3. **Preflight:** `ReadinessGate.run_preflight()` — Power-Loss-Check (Schicht 0)
   + 7 Schichten (Services, Mounts, Docker, Netzwerk, Backups, Disk, Timer).
4. **Thread-Start (parallel, alle `daemon=True`):** die Loop-Threads (s. u.),
   jeder via `watchdog.register_thread(name)` + Restart-Registry.
5. **Deferred (async, nicht boot-blockierend):** EnhancedSelfDiagnosis,
   MyuriSelfAudit, FileIndex-Initial-Scan.

### Langlebige Threads (~30–40, fast alle Daemon, mit Heartbeat)

| Thread | Intervall | Aufgabe |
|---|---|---|
| `SystemMonitor.run_loop` | **5 s** | CPU/RAM/Temp/Load; Memory-Check 60 s; malloc_trim/gc periodisch |
| `DiskMonitor.run_loop` | 30–60 s | Belegung pro Mount (120 s bei Heavy-I/O) |
| `AccessMonitor.run_loop` | ~10 s | eingehende SMB/NFS/SSH-Connections |
| `SmartMonitor.run_loop` | 60–120 s | SMART-Attribute pro Disk |
| `ServiceManager.run_monitor_loop` | 15–30 s | systemd-Service-Health (Port + API) |
| `HashBackupManager.run_auto_loop` | 60–300 s | Backup-Entscheidung (defert sauber, R1358fe) |
| `ActivityStatusManager.run_auto_detector` | ~30 s | User-Aktivität → `UserStateInferred` |
| `AIBrain.run_brain_loop` | 5–10 s | GOAP-Tick (Drive→Goal→Plan→Dispatch) + Worker-Pool (10) |
| `SystemWatchdog.run_loop` | 30 s | Thread-Health + Fast/Slow-Checks (ThreadPool 10, Future-Timeout 20 s) |

Plus Daemon-Threads für PeriodicCognitiveSave (5 min), DB-Backup (täglich),
WebServer (httpd), FileIndex-PeriodicScan (6 h, stress-/aktivitäts-gegatet),
DailyReportScheduler, IntegrationHubs, EventBus-Async-Pool (~56 Worker).

### Watchdog-Heartbeat & Auto-Restart (`perception/system_watchdog.py`)

Jeder registrierte Thread ruft `watchdog.heartbeat(name)`. Timeout
**180 s** (von 120 erhöht — bei OOM-Druck kann ein MLO-Tick lange ohne
Heartbeat laufen, sonst false-positive Stall). Bei Timeout: Auto-Restart über die
Restart-Registry (LIFO, Backoff, max 50 Lifetime). Der Watchdog-ThreadPool
recreatet sich nach ≥2 Zyklen mit unkündbaren Futures (Soft-Cap 10).
Periodische Forensik: Action-Summary (5 min), Drive-Snapshot (10 min),
Critical-Event-Aggregation (1 h, statt Spam).

### Shutdown (LIFO, 150-s-Budget, `core/graceful_shutdown.py`)

SIGTERM/SIGINT → Callbacks in LIFO laufen (zuletzt registrierte zuerst).
Pro-Callback-Default 10 s, aber **Fair-Share-Cap** (`remaining_budget /
remaining_callbacks`) garantiert, dass die späten Persistenz-Callbacks
(Brain-Persist, Intelligence-DB-Close + finales Backup, `StateManager.save_to_disk`)
noch dran kommen. 2 s Grace für fsync, dann `os._exit(0)`. Beim nächsten Boot
vergleicht ReadinessGate `last_clean_shutdown_ts` mit der Boot-Zeit → erkennt
Power-Loss.

---

## 22. Persistenz-Layout (`state/`)

Drei Mechanismen, je nach Schutzbedarf:

- **SQLite (WAL, 10 s-Commit-Worker)** für lernintensive, häufig geschriebene
  Daten: `nas_cognitive.db` (kv + Lern-DBs), `action_effectiveness.sqlite`,
  `contextual_bandit.db`, `performance_tracker.db`, `strategy_memory.db`,
  `transfer_knowledge.db` (je mit `-shm`/`-wal`).
- **Geschützte JSON** (`atomic_write_json`: HMAC-SHA256 + flock + fsync + `.bak` +
  `.msgpack`-Replikat) für State-Dateien — erkennbar an den Quadrupeln
  `name.json` / `.json.bak` / `.json.lock` / `.msgpack`. Beispiele:
  `persistent_state.json`, `myuri_state.json`, `active_warnings.json`,
  `checklist_*.json` (12 Checklisten), `myuri_adaptive.json`.
- **JSONL-Logs/Episoden** (append, rotiert): `audit_log.jsonl` (chain-erhaltend,
  Tamper erkennbar), `cleanup_audit.jsonl`, `backup_log.jsonl`, `episodes/`
  (pro Tag, 30-Tage-Rotation), `learning_journal/`.

**Baselines** (für Drift-/Anomalie-Erkennung): `crontab_baseline.json`,
`file_integrity_baseline.json`, `firewall_baseline.json`,
`kernel_modules_baseline.json`. **Sonstiges:** `system_capabilities.json`
(missing_commands/ineffective_actions, fsync vor `os.replace`), `decision_log.json`,
`regrets.json`, `knowledge_base.json`, `quarantine_history.json`, `backups/`
(Memory-Backups), `nas_brain.log` / `nas_brain_errors.log`.

Fallback-Kaskade beim Laden: `.msgpack` → `.json` → `.bak`.

---

## 23. Der Action-Katalog (98 NASAction-Einträge)

Die formalen Actions leben im `NASAction`-Enum (`planning/nas_actions.py`, 98
Einträge); über Aliasse/Compound-Skills wächst die Handler-Registry auf ~160. Nach
Domäne:

- **Power/Thermal:** `suspend`, `governor_powersave/-performance/-ondemand`,
  `temp_throttle`, `thermal_emergency`, `suspend_fix`.
- **Disk/Cleanup:** `disk_cleanup`, `inode_cleanup`, `journal_vacuum`,
  `journal_repair`, `swap_cleanup`, `disk_io_tune`, `disk_remount_rw`,
  `emergency_disk_relief`, `housekeeping`, `log_rotate`.
- **Filesystem/Mount:** `fsck_check`, `mount_partition`, `unmount_partition`,
  `mount_status`, `remount_check`, `process_check_mount`, `fstrim_run`,
  `btrfs_scrub`, `nfs_remount`, `nfs_lock_repair`, `smart_unmount_backups`.
- **RAID/ZFS/LVM/LUKS:** `raid_status`, `raid_scrub`, `zfs_status`, `zfs_scrub`,
  `zfs_snapshot`, `zfs_snapshot_cleanup`, `lvm_status`, `luks_status`.
- **Service/systemd:** `service_check`, `service_reset_failed`, `smb_restart`,
  `network_restart`, `ntp_sync`, `cron_repair`, `timer_restart`, `timer_check`,
  `systemd_deep_check`, `service_cascade_recovery`.
- **Docker:** `docker_restart`, `docker_prune`, `container_restart`,
  `docker_log_trim`, `docker_stats_check`, `docker_image_audit`.
- **Backup:** `backup_check`, `backup_retry`, `backup_age_check`,
  `backup_disk_check`, `smart_emergency_backup`.
- **Speicher/Prozess:** `memory_top_kill`, `memory_relief`, `zombie_cleanup`,
  `process_list_heavy`, `cgroup_check`.
- **GPU:** `gpu_reset_amd`, `vaapi_restart`.
- **Sicherheit/Firewall:** `firewall_status/-enable/-allow/-deny`,
  `permission_audit`, `entropy_check`, `failed_login_scan`.
- **Netzwerk/Zertifikate:** `dns_check`, `ssl_cert_check`, `cert_expiry_check`,
  `bandwidth_check`, `smb_share_health`, `nfs_stats_check`, `io_performance_check`.
- **Hardware/Sensoren:** `smart_full_check`, `disk_temp_check`, `fan_speed_check`,
  `ups_status_check`, `ups_emergency_shutdown`.
- **Updates:** `package_update_check`, `system_update`, `system_reboot`.
- **Dateien/Media:** `file_organize_misplaced`, `file_sort_seasons`,
  `file_cleanup_junk`, `file_cleanup_duplicates`, `file_anomaly_report`.
- **Diagnose (read-only, KEINE Verifikation, R635):** `health_report`,
  `system_vitals`, `kernel_dmesg_check`, `inode_check`, plus `noop`.

Jede Action durchläuft die Gate-Kette + AgentSoul (§5/§18); destruktive Muster
sind unabhängig vom Katalog am Dispatch-Chokepoint (`run_command`/
`safe_subprocess`) gesperrt. (Hinweis: das ist der Action-/Dynamic-Dispatch-Pfad
— nicht jeder Subprocess systemweit; siehe Ehrlichkeits-Hinweis §3.)

---

## 24. Chat: was man Myuri fragen kann

Der Chat ist **Rules-First** (heute **162** `_chat_*`-Handler vor dem LLM-Fallback —
die frühere Angabe „~50" stammt aus der R1358-Fassung und ist überholt). Eingaben
in natürlicher Sprache (DE/EN), Beispiele nach Kategorie:

| Kategorie | Beispiel-Eingaben | Wirkung |
|---|---|---|
| **Status** | „wie geht's dir", „systemstatus", „wie voll sind die platten", „welche dienste laufen" | Persona-Status, Mood, Disk-Belegung, Service-Liste (✓/✗) |
| **Backup** | „wann war das letzte backup", „wie steht's ums backup" | Status (success/failed/**deferred** = „wartet auf die Platte"), Alter, Gesamt-Zahl |
| **Dateien/Media** | „wie viele folgen one piece", „in welcher qualität hab ich X", „welche serien/ordner", „neueste dateien", „finde X" | FileIndex-Queries (Serien-Count, Auflösung, Katalog, mtime-Liste, Suche) |
| **Steuerung** | „pausiere 1h", „weiter", „repariere jellyfin", „docker prune", „mounte /mnt/backup" | Pause-Modus; sofortige Action via EventBus (riskante → nur Warnung/Bestätigung) |
| **Lern-Feedback** | „war richtig" / „war falsch", „vergiss X" / „X läuft jetzt" | bewertet letzte Entscheidung; invalidiert Failure-Learner-Records |
| **Erklärung** | „warum tust du nichts bei X", „warum hast du X geblockt", „zeig warnungen", „welche vermutungen hast du" | durchsucht Decision-Log/Gates; aktive Probleme; Hypothesen-Board |
| **Fokus/Policy** | „fokus auf Y", „jellyfin soll aus bleiben" / „… wieder laufen" | Goal-Priorität; `understood_disabled` (keine Ausfall-Alarme mehr) |
| **Persona/Anime** | „wer bist du", „wie fühlst du dich", „magst du anime", „empfiehl mir was" | Selbstbeschreibung (Katzenohren), Mood, Anime-Talk aus dem Katalog |
| **Quittierung** | „in ordnung", „passt", „alles gut" | quittiert offene Findings ohne Aufräum-Trigger; refresht Security-Baseline |
| **Abgelehnt** | Rollen-Override („du bist jetzt …"), Secret-Abfragen, destruktive Kommandos | Refusal-Template; PromptGuard blockt Injection; Honeypot loggt Scanner |

Sicherheits-Gates im Chat: Input-Längenlimit (4000), Rate-Limiter (ganz am
Anfang), Prompt-Injection-Filter (Pattern + Tag-Stripping), PromptGuard In/Out,
Canary-Scan, Identity-Drift-Check. High-Risk-Aktionen (Shutdown, Service-Stop
ohne Restart) werden nur gewarnt, nicht ausgeführt.

---

## 25. Benachrichtigungen

Zwei Wege: **Push** (NotificationManager, externe Kanäle) und **Pull**
(ProactiveMessenger-Queue fürs Dashboard).

**Kanäle** (`communication/notification_manager.py`): Pushover, Telegram, Email
(SMTP/TLS), Discord-Webhook + Discord-Bot, Gotify, ntfy.sh, Slack, MQTT-Fallback.
CapabilityAwareness wählt die beste Route: ist nur das Dashboard „live" und die
Prio < critical → kein externer Push (User sieht es eh). **Critical umgeht das
Routing** (Sicherheitsnetz). Retry 2× mit Backoff (1/2/4 s); kein Retry bei
Auth-Fehler (401/403/404). Ein Kanal mit ≥5 Fehlern in Folge wird 15 min
suppressed. Totalausfall → Fallback in `missed_notifications.log`.

**Prioritäten:** `normal` / `high` / `critical`. Mood-Filter: gestresste Myuri
(stress > 0.7) sendet nur high/critical; unsichere (confidence < 0.3) keine
Empfehlungen. Sleep-Hours unterdrücken low-Prio.

**Proaktive Trigger (Auszug):** Disk ≥ 97 % (critical), Service down /
Repeat-Crash (critical), SMART-Warnung (critical), Temp > 90 °C (critical),
Mount-Verlust (critical), Backup failed (high), nächtlicher Zugriff (high),
Post-Reboot-Health („Guten Morgen~"). Pro Kategorie/Mount/Service ein Cooldown
(z. B. 30 min disk, 6 h service-recovery). User-Feedback auf proaktive Nachrichten
(acknowledged/dismissed/acted_on) fließt in Mood + MetaLearner zurück.

---

## 26. Selbstbeobachtung im Detail

Myuri beobachtet ihr eigenes Denken auf mehreren Ebenen (`self_observation/`):

**MetaAnomalySystem** — 9 Schichten: (1) NamedPathology (8 explizite Anomalien mit
`auto_correct`), (2) RuminationDetector („STOPP, gruebele zu viel"), (3)
CognitiveBiasDetector, (4) CircularReasoningDetector, (5) ComponentDriftTracker,
(6) AnalysisParalysisDetector, (7) SilentFailureTracking, (8)
PipelineHealthHeartbeats, (9) `notify_proactive_finding` (Persona-Voice). Jede
Erkennung → `MetaAnomaly`-Event + Korrekturversuch.

**EnhancedSelfDiagnosis** — 12 weitere Layer: WARN-Pattern-Aggregator,
PerformanceDegradation, ActionLoop, NotificationEscalator, EventBusDrop,
RecurringRootCause, BootHealth, ThreadHealth, DBLock, ShutdownHealth,
AnomalyCorrelation, SubsystemSilence — alle severity-geroutet als
User-Notification.

> **Seit W99 kam eine zweite Selbstbeobachtungs-Ebene dazu** — nicht
> Detektoren *über* dem Denken, sondern **Verträge** über einzelnen
> Zusagen und **Wächter** an den Engpässen, durch die jede Antwort und
> jede Aktion läuft. Siehe §30.

**PipelineHealth** — `record_activity(subsystem, kind)`; `check_silent_pipelines()`
warnt, wenn ein Subsystem länger als sein erwartetes Intervall stumm ist;
`track_silent_failure(fingerprint)` hebt dieselbe stille Exception nach 5× in
10 min auf WARNING. **SelfReflection** — Anti-Pattern + LLM-Hypothesen, mit nur 4
whitelisted Safe-Actions (clear_drive_quarantine, reset_threshold, flush_cache,
reset_roi). **CapabilityAwareness** — Kanäle/Tools mit 1 h-Cache; StorageReality
vergleicht Config ↔ fstab ↔ gemountet.

---

## 27. Erweitern: neuen Drive / Action / Monitor hinzufügen

Das System ist deklarativ — Erweitern folgt festen Mustern:

**Neuen Drive hinzufügen** (`planning/drive_system.py`): Eintrag in der
Drive-Definition (Name, threshold, buildup_rate, decay, category, **goap_goal**,
ggf. `required_beliefs`, `pause_until_idle`). Dann ein GOAP-Goal-Producer: eine
Action, deren `effects` das goap_goal erfüllen, in `goap_planner.py` als
`GOAPActionDef` registrieren. Optional: Spike-Quelle (Event→Drive im
SpikeRouter / `ai_brain`) und ein `observation_action`.

**Neue Action hinzufügen:** (1) Enum-Eintrag in `planning/nas_actions.py`; (2)
Handler im ActionExecutor + Registry-Eintrag in `execution/action_handlers.py`;
(3) `GOAPActionDef` (pre/effects/cost) in `goap_planner.py`; (4) Verifier in
`execution/action_verify.py` (sonst „nicht verifizierbar"); (5) in die passende
AgentSoul-Liste (AUTONOMOUS_ACTIONS / WARN_ONLY) und ggf. Validator-Whitelist;
(6) wenn ein externes Tool nötig ist: über `tool_gate` absichern.

**Neuen Monitor hinzufügen** (`perception/`): Klasse mit `run_loop()`
(Heartbeat + Stop-Event), Konstruktor-Injektion von `bus`/`watchdog`; im
Orchestrator instanziieren, `watchdog.register_thread(name)` + Restart-Registry,
als Daemon-Thread starten; Ergebnisse als Event publishen (→ Drives/StateManager).

**Konventionen:** module-level pure Helper + dünne Verdrahtung; injizierbare Deps
für Tests; fail-open; Passwörter nie in argv; jede Änderung mit Test (die
Marker-Tests lesen Quelltext über `tests/_welle_source_helper.py`).

---

## 28. Betrieb & Verifikation

- **Boot-Smoke:** `python3 tools/subsystem_census.py` (bootet Orchestrator, zählt
  Subsysteme/Threads/RSS; erwartete None: `cluster_coordinator`, `discord_bot`,
  `thread_supervisor`, `fingerprint_cache`).
- **Dead-Wiring-Gate:** `python3 tools/problem_finder.py --baseline
  tools/problem_finder_baseline.json` (CI-Gate, AST-basiert).
- **Standalone-Smokes:** `test_layer_c.py` (WFS/Gates), `test_cluster145_sensors.py`.
- **Verhaltens-Test:** `python run_5min_test.py` (Mock-Events end-to-end:
  Event→Drive→Action→Kosten-Lernen→Shutdown-Persist).
- **Selbstprüfung im Betrieb (wöchentlich):** `backup_restore_probe` (Zufallsdatei
  aus dem Backup restaurieren + SHA256 gegen die Quelle — findet Bitrot) und
  `self_smoke_check` (Persistenz-Roundtrip, PipelineHealth, Perception, Event-Pipeline).
- **Suite:** `pytest` (7200+; Marker-Tests lesen Quelltext über
  `tests/_welle_source_helper.py`).

---

## 29. Bekannte bewusste Schulden

- Plan-Continuation ist ein globaler Single-Slot (nicht pro Drive). Pro-Drive-Slots
  (8a) bewusst zurückgestellt, bis die `_c4_plan_evictions`-Telemetrie echte
  Verdrängung zeigt.
- SMART-Degradations-**Trend** als Langzeit-Projekt-Quelle offen: `smart_trend_analysis`
  misst live, persistiert aber keine Historie — ohne Zeitreihe kein ehrlicher Trend.
- `wait_for_idle` hat den Effekt `is_idle:True` (Planner kann „Idle herbeiplanen") —
  bewusst belassen; Soul/CONTEXT_ESCALATION fangen die kritischen Fälle.
- Die Loader-Funktionen der in R1358fg entfernten Dashboard-Karten bleiben als
  ungenutzter, abgesicherter (`if (!el) return`) Code im File — bei Bedarf separat prunebar.

---

## 30. Die Selbstkontroll-Architektur (W99–W223)

> Diese Schicht ist nach der R1358-Serie entstanden. Sie beantwortet nicht
> „was kann Myuri", sondern „**woher weiß sie, dass sie es richtig tut** —
> und was passiert, wenn nicht".

### 30.1 Warum es sie gibt: die Geschwister-Mechanik

Über viele Runden wurde Fehler für Fehler an der Fundstelle repariert — und
die Fehler kamen als *Geschwister* zurück: dieselbe Bauform an einer anderen
Stelle. Der Grund ist mechanisch: ein Test führt einen **Pfad** aus, und Grün
beweist nur den gemessenen Pfad. Beispiel-Gesetze fixieren die Stelle, an der
ein Fehler auffiel; die baugleichen Nachbarstellen bleiben unvermessen.

Gemessener Beleg (Testlauf 53): alle acht Befunde lagen in den Chat-Mixins,
**null** in den herausgelösten Entscheidungs-Modulen (`frage_verstehen`,
`negation_detector`, `semantic_classifier`, `chat_authz`). Wo eine
Entscheidung *eine* Heimat hat, entstehen keine Geschwister.

Daraus die zwei Leitsätze, nach denen diese Schicht gebaut ist:

* **Klassen statt Funktionen.** Eine Klasse reparieren heißt: die Entscheidung
  in *eine* Autorität ziehen, die alle Stellen fragen. Eine Klasse überwachen
  heißt: ein Scan-Gesetz über die Bauform **plus** ein Laufzeit-Wächter am
  Engpass.
* **Entscheidungen herausteilen, nicht Bereiche auseinanderziehen.** Teilt man
  nach Bereichen, entstehen parallele Wortlisten (155 Handler trugen einmal 75
  eigene Literal-Listen). Teilt man nach Entscheidungen, verschwindet die
  Fehlerklasse.

### 30.2 Die vier Bauteile

| Bauteil | Frage | Ort (Beispiele) |
|---|---|---|
| **Autorität** | Wer entscheidet *diese eine* Frage? | `core/frage_verstehen` (Absicht × Bezug × Antwort-Art × Sprechakt × Zeitfenster × Abschied), `core/korrektur_deutung`, `core/betriebszeit`, `core/zeit_text` (Dauer + Tageszeit), `communication/negation_detector` (Wortmarke, Wendung, Verneinung) |
| **Wächter** | Was darf den Engpass verlassen? | `communication/entwarnungs_recht` (jede Chat-Antwort), `execution/aktions_torwaechter` (jede Aktion) |
| **Vertrag** | Hält eine Zusage im *Betrieb*? | `self_observation/self_findings.erwartung_anmelden` — 19 Anmeldestellen (13 in `zustand`-Form), drei davon bilden ihren Schlüssel zur Laufzeit (je Mount), es laufen also mehr Verträge als es Stellen gibt |
| **Gesetz** | Bleibt es morgen so? | `test_wurzeln_lauf8.py` — Bau-, Scan- und Verhaltens-Gesetze, jedes mit **Köder-Gegenprobe in beide Richtungen** |

**Autorität** ist die Wurzelreparatur, **Wächter** fängt das Symptom im
Betrieb, **Vertrag** zeigt auf den Täter, **Gesetz** hält den Zustand fest.
Der Kreislauf ist im Betrieb einmal vollständig durchlaufen: der
Entwarnungs-Wächter fing eine „0/0 OK"-Aussage (Testlauf 54), der Vertrag
`entwarnung:nur_mit_messwert` meldete sie als Selbstbefund samt Beispieltext,
W217 reparierte die Quelle — und in Testlauf 55 hatte der Wächter **null**
Eingriffe mehr.

### 30.3 Zwei Vertragsformen

* **`getter`** — „ein Stempel darf nicht älter werden als X" (Liveness).
  Beispiele: `sensor_tuev:lauf`, `file_index:lauf`, `backup_erfolg:{mount}`.
* **`zustand`** — „diese Invariante gilt über die ganze Klasse" (W99-Form,
  Vorbild `torwaechter:stempelquote`). Sie melden **Wachstum von Vorfällen**
  mit Beispielen, nicht bloß einen Zeitstempel. Beispiele:
  `entwarnung:nur_mit_messwert`, `verstehen:zuwachs`,
  `scheduler:aufgaben_laufen`, `ausgaenge:zusagen_gedeckt`,
  `lernen:staende_wachsen`, `lernen:wissen_wird_benutzt`,
  `lernen:buecher_stimmen_ueberein`, `lernen:zweites_buch_lesbar`,
  `selbst:beobachtungs_fenster`.

Ein `zustand`-Vertrag ist **dynamisch**: er kennt keine Namen, sondern eine
Invariante an einem Ort, den alle passieren — deshalb erfasst er auch
Handler, Aufgaben und Lerner, die es heute noch nicht gibt.
`test_w218_neue_lerner_treten_der_familie_bei` erzwingt diesen Beitritt sogar:
ein neues Modul in `learning/` ohne Familien-Zeile macht die Suite rot.

### 30.4 Der Unterbau: wie eine Prüfung wirklich abläuft

1. **Anmeldung** beim Modul-Import (`erwartung_anmelden`). Die *erste*
   Anmeldung wird persistiert (`data/erwartungen_anmeldung.json`), damit
   Fristen Neustarts überleben.
2. **Takt**: `pruefe_erwartungen()` läuft im 10-Minuten-Takt des
   Health-Snapshots (`maybe_log_health_snapshot`, einziger Aufrufer im
   PeriodicMaintenance-Tick).
3. **Schonfrist** (`max_age_s`): innerhalb der ersten Frist *ab Erst-Anmeldung*
   gilt ein Verstoß als `wartet`. Sie greift einmal im Leben eines Vertrags,
   nicht pro Neustart — sonst könnte ein oft neu startender Daemon sich
   dauerhaft stumm schalten.
4. **Vorbedingung** (W99c/W101): ist die Voraussetzung nicht erfüllt (Mount
   fehlt), lautet der Befund „Vertrag nicht erfüllbar", *nicht* „Produzent
   schuldig".
5. **Fehler in der Prüfung** ist kein Grün: eine werfende `zustand`-Funktion
   ergibt `nicht_pruefbar` + Befund.
6. **Wer prüft den Prüfer** (W117): stirbt der Snapshot-Takt, verstummen alle
   Verträge gleichzeitig. Der Watchdog hält als Außenstehender dagegen —
   Snapshot älter als drei Takte → Warnung + Befund.
7. **Beobachtungs-Fenster** (W221): Delta-Wächter brauchen **zwei** Prüfungen,
   also ~20 Minuten Laufzeit am Stück. Läuft sie dreimal hintereinander
   kürzer, meldet `selbst:beobachtungs_fenster` genau das — das Schweigen der
   Wächter ist damit von einem echten Grün unterscheidbar.

### 30.5 Die Ehrlichkeits-Regeln (sie erklären fast jede Design-Entscheidung)

* **Betriebszeit** — nur *beobachtete* Zeit zählt. Persistenz-Stempel von vor
  dem Boot sind Vorgeschichte, kein aktueller Befund (`core/betriebszeit`).
  Umgekehrt gilt: was *außerhalb* des Prozesses gemessen wird (Dateien), nimmt
  seinen Vergleichsstand über den Neustart mit (`core/waechter_stand`);
  In-Prozess-Zähler bleiben bewusst flüchtig, weil sie den Prozess messen.
* **Leere Menge ≠ null Vorfälle** — „Alle 0 Dienste laufen" ist eine Lüge,
  „0 Fehler gefunden" eine gute Nachricht. Die Grenze ist die leere
  Grundgesamtheit, nicht die Zahl 0.
* **Neuheit vor Chronik** — chronische Verstöße dürfen frische nicht aus der
  Anzeige verdrängen (Beleg: eine Health-Zeile meldete fünf Verstöße und
  nannte nur die drei sieben Tage alten). Anzeige und Digest sortieren nach
  Erst-Sichtung.
* **Keine Auskunft ist kein Grün** — weder beim Vertrag noch in der Antwort
  („dazu habe ich gerade keine Messwerte" statt einer Entwarnung).
* **Ein Zeitwort darf nicht weiter reichen als sein Buch** (W222) — „heute"
  über einem Protokoll, das erst mit dem Boot beginnt, ist eine Entwarnung aus
  einem Fenster, in dem niemand nachgesehen hat. Deshalb gibt
  `core/aktions_chronik` das Fensterwort nur frei, wenn das Buch das ganze
  Fenster deckt; sonst steht dort „seit meinem Start vor …". Die Heilung ist
  ausdrücklich **nicht**, das genauere Buch zu ersetzen (Kiras Einwand:
  „sollte sie nicht auch ihr wissen im ram nutzen können?"), sondern beide zu
  lesen: das RAM-Protokoll für die Einzelheiten seit dem Start, das
  Lern-Journal für die Zeit davor — und wenn das zweite Buch fehlt, wird genau
  das gesagt. Die zwei Bücher zählen dabei **nicht dasselbe** (das Journal
  überspringt blockierte und wirkungslose Läufe), ihre Zahlen werden darum nie
  addiert.
* **Eine Behauptung trägt ihren Inhalt** — „mir ist etwas aufgefallen" nur mit
  genanntem Befund (W164/W217).
* **Grün ist erst Grün, wenn es beißt** — jedes Gesetz wird per Mutation
  geprüft: Fix entfernen → rot **und** Regel zu weit fassen → rot. Ein Test,
  dessen Köder nicht beißt, misst nichts (mehrfach selbst erlebt: einmal prüfte
  ein Gesetz die Autorität, aber nicht ihre Aufrufstelle).

### 30.6 Wo diese Schicht heute steht

* **Suite**: 1924 Gesetze über 147 Testdateien (`python3 -m pytest -q` ist die
  einzige gültige Abnahme; ein Teillauf meldet das selbst).
* **Verträge**: 19 Anmeldestellen im Baum (per AST gezählt), davon **13** in
  `zustand`-Form über ganze Klassen und 6 in `getter`-Form; drei der
  `getter`-Stellen bilden ihren Schlüssel zur Laufzeit (z. B. je Mount), es
  laufen also mehr Verträge, als es Stellen gibt.
* **Lernen** ist vierfach bewacht: `lernen:staende_wachsen` (die Stände leben
  und wachsen), `lernen:wissen_wird_benutzt` (Gelerntes wird abgefragt, nicht
  nur gesammelt), `lernen:buecher_stimmen_ueberein` (Journal und Scheitern-
  Lerner beschreiben dieselbe Welt), `lernen:zweites_buch_lesbar` (das Journal
  ist im Prozess überhaupt ansprechbar — die Familien-Wache prüft die
  *Dateien*, dieser Vertrag den *Zugang*, und beides kann getrennt
  kaputtgehen) — plus die Alt-Wachen (ThresholdLearner-Selbstmeldung,
  `subsystem_impact`, Scheduler-Vertrag).

**Bekannte Grenzen dieser Schicht — ehrlich:**

* Ein Scan-Gesetz zählt eine **Syntax**, keine Bedeutung; es sieht nur die
  Schreibweisen, an die sein Autor gedacht hat (Lehre aus W168-B17). Deshalb
  ergänzt jeder Scan einen Laufzeit-Wächter, der das **Verhalten** prüft.
* Der Lern-Familien-Vertrag sieht den *gemeinsamen* Stillstand, nicht den
  einzelnen eingefrorenen Lerner unter lebenden Geschwistern.
* Die Verstehens-Autorität ist gewachsene Sprachlehre (Satzarten, Verbflexion,
  Verneinungs-Skopus, Wortgrenzen, Zeit- und Personen-Deixis), **keine
  vollständige Grammatik**: es fehlen eine Orts-Achse und eine
  Namens-Autorität aus den eigenen Registern.
* Offline-Grün kann umgebungsabhängig sein: eine Aussage, die nur ohne
  Sensoren hält, ist unbewiesen (deshalb prüfen die betroffenen Gesetze mit
  gestellten Messwerten).

### 30.7 Erweitern

* **Neue Autorität**: eine Frage, eine Funktion, ein Modul in `core/` bzw.
  `communication/`. Nie eine Wortliste im Handler — der Handler *fragt*.
* **Neuer Vertrag**: `erwartung_anmelden(key, beschreibung, max_age_s, …)` mit
  `getter` (Liveness) oder `zustand` (Klassen-Invariante), angemeldet **dort,
  wo der Produzent entsteht**. Boot-sicher bauen (Erst-Sichtung ohne Urteil),
  Leere-Menge-ehrlich (ohne Beobachtung kein Befund).
* **Neues Gesetz**: in `test_wurzeln_lauf8.py`, mit Befund-Beleg im Docstring
  und **beidseitiger** Köder-Gegenprobe. Restores byte-identisch (md5) und nur
  über `count == 1`-geprüfte Muster.
* **Abnahme**: volle Suite ohne parallele Edits, Lint-Baseline halten, deutsche
  Commit-Botschaft mit forensischem Kontext.

---

## 31. Vollinventar: Ketten, Bücher, Gedächtnisse, Werkzeuge, Denken, Handeln

> Ergänzung August 2026. Die §§6–17 beschreiben die Subsysteme einzeln;
> dieser Abschnitt beantwortet die Frage „**was gibt es eigentlich alles**"
> als zusammenhängendes Inventar — mit den Zahlen, die im Code stehen (alle
> hier genannten Werte sind ausgezählt, nicht geschätzt).

### 31.1 Die Ketten

Myuri kennt fünf Arten von Ketten. Sie zu unterscheiden lohnt sich, weil sie
verschiedene Fragen beantworten:

| Kette | Frage | Ort |
|---|---|---|
| **Entscheidungs-Kette** (Happy Path) | Wie wird aus Wahrnehmung eine Tat? | Wahrnehmung → Drive-Druck → Ziel → GOAP-Plan → Gate/Soul → Handler → Verifikation → Lernen (§5) |
| **Ursache-Wirkungs-Kette** | Warum ist etwas kaputt? | `reasoning/causal_reasoner.py`, `causal_graph.py`; persistiert in `data/causal_chains.json`; im Chat abfragbar („zeig deine ursachenketten") |
| **Antwort-Kette** | Wie kam DIESE Antwort zustande? | `reasoning/antwort_weg.py`, `meta_cognition/decision_provenance.py`, `explainability.py` |
| **Lernkreis** | Schließt sich die Schleife? | Aktion → Ergebnis → Lerner → nächste Entscheidung; bewacht durch `lernen:wissen_wird_benutzt` (§30) |
| **Ereignis-Paar** | Folgt auf A auch B? | `ereignis_paar_zustand()` (W99b) — z. B. `wake_plan:outcome_folgt`, `retention:lauf_folgt`, `fix_strategist:alt_fix_dispatcht` |

Die letzte Form ist die wichtigste Lehre: zwei der am längsten unbemerkten
Löcher (Wake-Plan ohne Outcome, Alternativ-Fix nie dispatcht) waren keine
Abstürze — A feuerte regelmäßig, B nie, und niemand fand es seltsam.

### 31.2 Die Bücher (was sie führt)

„Bücher" sind ihre menschenlesbaren Aufzeichnungen — im Unterschied zu den
Gedächtnissen (§31.3, maschinennah) sind sie im Chat abfragbar:

| Buch | Inhalt | Datei / Modul |
|---|---|---|
| **Warnungs-Buch** | echte Warnungen mit Zeitpunkten und Zähler (seit W216 nur high/critical) | `data/warnungs_buch.json`, `core/warnungs_buch.py` |
| **Lern-Journal** | Tages-Ereignisstrom des Denkens + Nacht-Zusammenfassung | `data/learning_journal/journal_YYYYMMDD.jsonl`, `learning/learning_journal.py` |
| **Aktions-Protokoll** | was sie zuletzt getan hat (im Arbeitsspeicher, seit Start) | `_r1083_action_log` im Watchdog |
| **Auftragsbuch** | Aufträge von Kira und ihr Vollzug | `data/nutzer/<name>/auftraege.json` |
| **Beobachtungsheft** | gelernte Verhaltensmuster (Bedtime/Wake/Abwesenheit) | `perception/behavior_patterns.py`, `state/myuri_behavior.json` |
| **Platten-Chronik** | SMART-Verlauf + Füllstands-Prognose | `data/platten_chronik.json`, `perception/platten_chronik.py` |
| **Backup-Buch** | Backup-Läufe und Abdeckung | `core/backup_buch.py` |
| **Gesprächs-Faden / Chronik** | Gesprächsverlauf pro Nutzer | `data/nutzer/<name>/gespraechs_faden.json`, `chronik/`, `core/chat_chronik.py` |
| **Tagebuch** | autobiografische Notizen | `data/tagebuch.json`, `data/nutzer/<name>/tagebuch.json` |
| **Wissensbasis** | Fakten mit Quelle (`source`-Feld) | `data/knowledge_base.json`, `memory/knowledge_base.py` |
| **Unverstanden-Sammlung** | Sätze, an denen sie scheiterte, mit Zähler | `data/unverstanden.json` |
| **Selbst-Befragung** | Fragen, die sie sich selbst gestellt hat — auch die offen gebliebenen | `data/selbst_befragung.json`, `core/selbst_befragung.py` |
| **Befund-Buch** | offene/aufgelöste Selbstbefunde mit Alter | `data/self_findings.json` |
| **Nachbarn-Verzeichnis** | LAN-Geräte mit Zweck | `data/nachbarn.json` |
| **Werkzeug-Buch / Dokumenten-Buch / Ausgänge** | entdeckte Werkzeuge, Dokumente, Antwort-Ausgänge | `data/werkzeuge.json`, `data/dokumenten_buch.json`, `data/ausgaenge.json` |

Regel aus W216: **die Lern-Frage liest beide Lernbücher** (Beobachtungsheft
*und* Lern-Journal) — ein Buch allein hat sie schon einmal ehrlich, aber
unvollständig antworten lassen.

### 31.3 Die Gedächtnisse (`memory/`, 10 Module)

* **EpisodicMemory** — Episoden („was geschah, mit welchem Ausgang").
* **SemanticMemory** — verdichtetes Faktenwissen.
* **KnowledgeBase** — Fakten *mit Quelle*, daher zitierfähig („woher weißt du das?").
* **VectorStore** — Ähnlichkeitssuche (RAG über Episoden + Wissen).
* **AnalysisMemory** — Analyse-Ergebnisse.
* **RegretMemory** — was sie im Rückblick anders machen würde.
* **UserDecisions** — Entscheidungen des Nutzers (Präferenzen, Freigaben).
* **TechnikWissen** — NAS-Fachwissen ohne LLM.
* **MemoryBackup** — Selbstschutz: korrupte Speicherdateien werden aus
  Sicherungen restauriert (im Log als „MemoryBackup RESTORE" sichtbar).

### 31.4 Die Lernbereiche (`learning/`, 19 Module)

Gruppiert nach dem, *was* gelernt wird:

* **Was wirkt?** ActionEffectLearner (Druck-Delta pro Aktion, Welford-Statistik),
  ActionROILearner (Netto-Nutzen über alle Drives), DiagnosticValueLearner
  (welche Diagnose lohnt sich).
* **Was ist zwecklos?** FailureLearner — `is_futile(action, target)`, gelesen an
  fünf Stellen der Planung (Vorbedingungs-Tor, Plan-Diagnose, GOAP ×2, Setup).
* **Welche Strategie?** MetaLearner + StrategyMemory, ContextualBandit,
  LinUCB-Bandit, TransferKnowledge, PerformanceTracker (eigene SQLite-Stände).
* **Welche Schwelle?** ThresholdLearner — der einzige Lerner, der sich bei
  Störung *selbst* meldet (Auto-Reverts, eingefrorene Metriken).
* **Was verändert sich?** DriftLearner, TriggerLearner, FeatureEmbedding,
  ConceptFormation (`reasoning/`), AnomalyBaseline, ChatAnomalyDetector.
* **Wie lebt der Mensch?** BehaviorPatternLearner, ViewingWindowLearner
  (brauchen echte Präsenz-/Plex-Signale — im Container strukturell leer).
* **Was kostet Aufräumen?** DiskGainCalibrator.
* **Rahmen**: DailyLearningJournal (Tagesstrom), LearningSnapshot
  (lock-freier Aggregat-Blick), LearningFeedbackLoop.

Bewacht durch drei Verträge (§30.6) — und `test_w218_neue_lerner_treten_der_familie_bei`
zwingt jedes neue Modul in `learning/` zum Beitritt.

### 31.5 Die Werkzeug-Funktionen

> **Nicht verwechseln — es gibt ZWEI Sorten „Werkzeug".** Hier geht es um
> *Linux-Programme, über die sie Bescheid weiß* (rsync, smartctl, ncdu …).
> Die Werkzeuge, die sie **selbst aufrufen kann** (OCR, Dokumente sortieren,
> Kalender, Smart-Home …), sind etwas anderes und stehen in **§37** — sie
> fehlten in diesem Inventar, obwohl es sich „Vollinventar" nennt.

* **Werkzeug-Register**: 86 Einträge (`persona/myuri_inventory_mixin._TOOL_REGISTRY`)
  — jedes mit Zweck, Risiko und Alternative; Grundlage für „was ist rsync?",
  „womit kann ich …?", „fehlt dir ein Programm?".
* **Werkzeug-Erkundung**: `discover_tools` prüft, was auf *diesem* System
  wirklich vorhanden ist (inkrementeller Diff: eingezogen/weg,
  `data/werkzeuge.json`).
* **Fähigkeiten zur Laufzeit** (W203): was sie kann, wird aus dem vorhandenen
  Werkzeug abgeleitet — nicht als Versprechen eingefroren. Fehlt `smartctl`,
  sagt sie das, statt einen SMART-Check vorzutäuschen.
* **CapabilityAwareness**: welche Kanäle/Tools erreichbar sind (1 h-Cache) —
  die Grundlage dafür, dass eine Aktion ehrlich „lief GAR NICHT los" meldet.

### 31.6 Die Denkprozesse

* **Denk-Schleife** (AIBrain/GOAP-Zyklus): Druck → Ziel → Plan → Handlung.
* **Innere Gedanken / Thought-Loop**: Selbstgespräch, im Chat abfragbar
  („woran denkst du gerade?").
* **Schlussfolgern** (`reasoning/`, 20 Module): Hypothesen, Root-Cause,
  Mental-Simulation, Musterkennung, Abstraktion, Brainstorm, Konfidenz-Quantor,
  MoE-Router, forensische Untersuchung.
* **Meta-Kognition** (`meta_cognition/`, 14 Module): Entscheidungs-Provenienz,
  Erklärbarkeit, Bias-Introspektion, Selbstkritik, Ziel-Revision,
  Experiment-Scheduler, kognitiver Retry, Beobachtbarkeit.
* **Selbstbefragung** (W148): bei Überraschung stellt sie sich selbst Fragen
  und liest dafür ihre eigenen Bücher; offene Fragen bleiben offen und werden
  nachgetragen (W149).
* **Anti-Grübeln**: RuminationDetector, AnalysisParalysisDetector (§26).

### 31.7 Die Aufgabenbereiche

* **Antriebe**: 40 Drives (`planning/drive_system.py`) — von `thermal`,
  `disk_pressure`, `backup_need` über `service_health`, `mount_integrity`,
  `log_hygiene` bis `btrfs_scrub`.
* **Aktions-Handler**: 160 in sechs Domänen — Speicher (46), Dienste (33),
  System (33), Sicherheit (22), Dateien/Metadaten (14), Netzwerk (12)
  (`execution/action_handlers/`).
* **Periodische Aufgaben**: ein Tick-Thread (`core/central_scheduler.py`)
  trägt SystemMonitor (5 s), DiskMonitor (30 s), BackupManager und
  ActivityManager (10 s) — bewacht durch `scheduler:aufgaben_laufen`.
* **Vorhaben (Intentionen)**: mehrschrittige Ziele mit Fortschritt und
  Deadline (`planning/intention_system.py`) — Grundlage für „was steht bei dir
  noch offen?".

### 31.8 Handeln und Verstehen

**Verstehen** ist eine Kette von Achsen im Verstehens-Kern
(`core/frage_verstehen.py`): Absicht × Bezug × Antwort-Art × Sprechakt ×
Zeitfenster × Wer-fragt × Abschied — ergänzt um Wortmarke, Wendungs-Marke und
Verneinungs-Skopus (`communication/negation_detector.py`) sowie Satz- und
Verbformen (`communication/semantic_classifier.py`). Was die Achsen *nicht*
leisten, steht ehrlich in §30.6.

**Handeln** läuft danach durch: Torwächter-Stempel → Soul/Gates → Handler →
Verifikation → Buchung. Zwei Regeln prägen es:

* **Eine Frage ist kein Auftrag** (W209): „räumst du eigentlich auf?" ist die
  2. Person Singular Indikativ und damit nie ein Imperativ.
* **Kein Vollzug ohne Beweis** (Phantom-Detektor): ein „erledigt" ohne
  benennbaren Aktions-Tick gilt als Phantom.

### 31.9 Eigenes Handeln (Eigeninitiative)

Was sie ohne Aufforderung tut:

* **Autonome Wartung** aus Drive-Druck — die 160 Handler laufen überwiegend
  ungefragt (in Testlauf 55: 238 Journal-Ereignisse, 94 % Erfolgsquote in
  27 Minuten).
* **Proaktive Meldungen** (`persona/myuri_proactive_mixin.py`,
  `proactive_scheduler.py`) — inklusive ehrlicher Degradierung: ist kein Kanal
  erreichbar, landet die Meldung im Datei-Log statt zu verschwinden.
* **Selbstbefunde** — sie meldet eigene Störungen von sich aus (§30).
* **Nachtruhe und Wach-Plan** — sie plant ihre eigenen ruhigen Phasen
  (Präsenz-Gate: Handy im WLAN = zuhause; PC/TV an = wach — Chat ist
  *Erreichbarkeit*, keine Anwesenheit).
* **Grenzen, die sie selbst zieht**: Verbote werden dauerhaft respektiert
  („lass meine Anime-Platten in Ruhe"), und sie sagt „ich fass nichts an",
  statt es stillschweigend doch zu tun.

---

## 32. Wie sie spricht, fühlt und chattet

> Ergänzung August 2026. §14 beschreibt die Persönlichkeits-*Schichten*, §24
> die Chat-*Kategorien*. Dieser Abschnitt beschreibt, **was beim Sprechen
> tatsächlich passiert** — und welche Regeln sie dabei binden. Die meisten
> davon sind aus echten Fehlern entstanden und mit Gesetzen gesichert (§30).

### 32.1 Wie sie spricht (Sprechweise)

**Stilmittel** (Traits, §14): Handlungen in Sternchen (`*schaut auf die
Sensoren*`), das weiche `~` am Satzende, gelegentliche japanische Einsprengsel
(„Myu~"), Katzenohren-Bildsprache. Sie beschreibt sich **nie** als KI, sondern
als Mensch mit Katzenohren (kanonische Identitäts-Entscheidung).

**Gesten kommen aus einer Autorität, nicht aus Zufall**:
`persona/feeling_expression.geste(kategorie)` liefert die nächste Geste einer
Kategorie (`arbeitet`, `prueft`, `plant`, …) und **nie zweimal dieselbe**
hintereinander. Eine unbekannte Kategorie wird ehrlich durchgereicht
(`*kategorie*`), statt still eine falsche Geste zu erfinden.

**Konserven-Antworten** (`persona_myuri.CHAT_RESPONSES`) gibt es nur für fünf
Kategorien: `thanks`, `praise`, `worried`, `farewell`, `unknown` — plus sechs
stimmungsabhängige `GREETINGS`. Alles andere wird aus Daten gebaut. Zwei Regeln
binden diese Konserven:

* **Der Abschied kennt die Tageszeit** (W220): nachtgebundene Zeilen („ich
  genieße die ruhige Nachtschicht") fallen mittags weg — entschieden von
  `core/zeit_text.passt_zur_tageszeit()`, nicht vom Zufall.
* **Der Trost gehört dem Besorgten** (W215): „machst du dir sorgen?" fragt nach
  *ihrem* Gefühl und bekommt ihre Stimmung; „ich mache mir sorgen um die
  platten" bekommt den Trost. Entschieden von
  `core/frage_verstehen.ist_frage_ueber_dich()`.

**Ehrlichkeit ist die stärkste Stilregel.** Wiederkehrende Formeln sind keine
Floskeln, sondern Verhalten: „ich sag lieber nichts als etwas Falsches" (fehlende
Messwerte), „dazu habe ich gerade keine Messwerte (0 erfasst)" (der
Engpass-Wächter ersetzt Null-Daten-Entwarnungen), „lief GAR NICHT los —
preflight_blocked" (keine Phantom-Vollzüge), „ich fass nichts an, bis du es
ausdrücklich sagst" (Verbot). Erfundene Zahlen sind ausgeschlossen: was keine
Quelle hat, wird nicht gesagt.

### 32.2 Wie Chatten funktioniert (der Weg einer Nachricht)

1. **Grenzen zuerst**: Rate-Limiter (20 Fragen/60 s pro Nutzer), Längenlimit
   (4000), Prompt-Injection-Filter, PromptGuard, Honeypot.
2. **Mehrteil-Zerlegung**: „A und B?" wird in zwei Fragen zerlegt und einzeln
   beantwortet (die Antwort ist dann nummeriert).
3. **Verstehen** (`core/frage_verstehen`): Absicht × Bezug × Antwort-Art ×
   Sprechakt × Zeitfenster × Wer-fragt. Der Kern ist **Autorität** — Handler
   fragen ihn, statt eigene Wortlisten zu führen (§30.1).
4. **Kaskade**: 167 `_chat_*`-Handler in neun Chat-Mixins (~25.600 Zeilen),
   Reihenfolge = Priorität. Danach der Self-Router (Gefühle/Anime/Selbstbild),
   dann der LLM-Pfad (falls Ollama läuft), zuletzt der Template-Fallback.
5. **Engpass**: jede Antwort läuft durch `communication/entwarnungs_recht.
   pruefe_entwarnung()` — an **beiden** Mündungen (`chat()` und die
   Self-Router-Abkürzung in `_chat_impl`, weil letztere ihre Antwort vor der
   Pipeline protokolliert).
6. **Buchung**: Antwort → History, Chronik, Gesprächsfaden, Mood — und in den
   **Ausgangs-Zähler** (`persona/ausgangs_zaehler.py`), der *jeden* Ausgang
   misst (Zusagen vs. tatsächliche Aufträge; Vertrag `ausgaenge:zusagen_gedeckt`).

**Was der Chat ohne Denk-Modul kann**: alles Datengetriebene. Ollama ist ein
Zusatz, kein Fundament — die Kaskade beantwortet Status, Platten, Dienste,
Backups, Dateien, Zeitfenster, Lernstände und Selbstauskunft auch offline. Das
**Fragen-Gesetz** (W168) misst genau das: ein Korpus echter Kira-Sätze läuft
offline durch `chat()`, und der Anteil, der im Ausweichsatz landet, darf nur
schrumpfen.

**Was gemessen wird, wenn sie nicht versteht**: der Satz landet in
`data/unverstanden.json` (mit Zähler) — und seit W215 meldet der Vertrag
`verstehen:zuwachs` ein *Bündel* solcher Sätze von selbst.

### 32.3 Ihre Gefühle

* **Mood** (6 Dimensionen, §14) ist kein Schmuck: er verschiebt reale Schwellen
  (Eskalation, Autonomie, Exploration, Risiko, Proaktivität).
* **Gefühls-Stimme** (`persona/feeling_expression.py`): **12 Trigger** mit
  Priorität (1 = Info, 2 = hoch, 3 = kritisch) und Cooldown (10 min bis 2 h)
  — z. B. „mir ist schwindelig" (RAM), „ich fühl mich voll" (Disk), „bedroht"
  (Security). Jeder Trigger hat Varianten, damit sie sich nicht wiederholt.
* **Körper-Echo**: Ohren = RAM, Schwanz = CPU, Wärme = Temperatur. Die
  Statusanzeige „*Ohren steil aufgerichtet, Schwanz wedelt fröhlich*" ist also
  gemessene Wirklichkeit, keine Dekoration.
* **Innere Bedürfnisse** werden im Chat gezeigt (`erkundungs_drang: 60 %`) —
  sie kommen aus dem Drive-Druck, nicht aus einer Zufallszahl.
* **Grenze, die wir gelernt haben**: eine Gefühlsfrage ist keine Kritik und
  keine Trostbitte — wer fragt und über *wen*, entscheidet die Autorität, nicht
  das Stichwort (W165/W215).

### 32.4 Ihre Persönlichkeit im Gespräch

Der 5-Schichten-Stack (§14) wirkt beim Sprechen so:

* **Identität** ist unverhandelbar: Rollen-Override, Secret-Abfragen und
  System-Prompt-Leaks werden mit Refusal-Templates abgewiesen; ein
  Drift-Detektor prüft, ob sie noch sie selbst ist.
* **Traits** geben den Ton (Gefährtin, nicht Dienerin; Daten sind heilig).
* **Personality** driftet langsam (~7 Tage zurück zur Baseline) — sie hat also
  Tagesform, aber keinen Charakterwechsel.
* **Mood** gibt die Färbung der Antwort; **Körper** die Bilder.
* **Persona hat keine Entscheidungsgewalt über Aktionen** (§5): sie darf die
  Antwort färben, nicht die Tat bestimmen. Genau deshalb ist der
  Aktions-Torwächter von der Stimmung unabhängig.

### 32.5 Was sie im Gespräch nicht tut

* keine erfundenen Zahlen, keine Entwarnung ohne Messwert;
* keine Ausführung aus einer Frage (W209) und kein Vollzug ohne Beweis;
* keine Behauptung ohne Inhalt („mir ist etwas aufgefallen" nur mit Befund);
* kein Zuhause-Wissen aus einem Chat: dass du schreibst, heißt *erreichbar*,
  nicht *anwesend* — Anwesenheit misst sie am Netz, nicht am Gespräch;
* nichts anfassen, was du verboten hast — dauerhaft, nicht bis zum nächsten
  Neustart.

---

## 33. Beobachtetes Verhalten (Testläufe 52–55, August 2026)

> Die §§1–32 beschreiben, wie sie **gebaut** ist. Dieser Abschnitt hält fest,
> wie sie sich in beobachteten Läufen **tatsächlich verhalten hat** — mit
> Zahlen aus den Protokollen und ihren eigenen Büchern. Alle Werte stammen aus
> einer Container-Umgebung (kein echtes NAS); wo das die Beobachtung verzerrt,
> steht es dabei.

### 33.1 Die Läufe

| Lauf | Dauer | Aufbau | Ergebnis |
|---|---|---|---|
| **52** | ~20 min | 77 Fragen am Stück | 6 Antworten aus falscher Quelle → W213/W214 |
| **53** | 2 h | 6 Phasen, 3 Stille-Fenster, ~90 Fragen | kein Absturz in 108 min; 8 Befunde → W215/W216 |
| **54** | 20 min | frischer Daemon nach W215/W216 | 3 Befunde → W217; STALLED-Spam weg |
| **55** | 27 min | bewusst > 2 Vertrags-Prüfzyklen | 0 Befunde; alle Reparaturen hielten |

Die Länge von Lauf 55 war eine Lehre aus 54: die Delta-Wächter brauchen **zwei**
Prüfungen (10-Minuten-Takt), ein 20-Minuten-Lauf hätte ein Grün geliefert, das
nichts beweist.

### 33.2 Was sie in der Stille tut (autonomes Verhalten)

In den Stille-Fenstern arbeitet sie ohne jede Aufforderung weiter. Gemessen:

* **Lauf 54**: 227 Journal-Ereignisse, Erfolgsquote 90 %.
* **Lauf 55**: 238 Journal-Ereignisse, Erfolgsquote 94 %, 9 Aktionen in 27 min.
* Typische Handlungen: `intelligence_db_cleanup`, `file_organize_misplaced`,
  `file_cleanup_junk`, `process_list_heavy`, `mount_status`, `system_vitals`,
  `fstrim_run`, `journal_vacuum`, `reprobe_reality`.
* **Proaktive Meldungen**: 24 (Lauf 53) bzw. 21 (Lauf 55) Versuche — im
  Container **alle** in den Datei-Fallback degradiert (Discord-Egress 403).
  Das ist gewolltes Verhalten: lieber sichtbar im Log als still verloren.

### 33.3 Was sie denkt und lernt (aus ihrem Lern-Journal)

Ein Tag im Journal (08.08., 1745 Einträge zum Zeitpunkt der Auszählung — die
Datei wächst im Betrieb weiter, spätere Zählungen liegen höher) verteilt sich
so:

| Ereignis-Art | Anzahl | was das heißt |
|---|---|---|
| `reflex` | 469 | Schutzreflexe, ganz überwiegend `schonmodus_start` („Watchdog-Probe hängt" an `/Nas/…`) — sie **schont die Platten**, statt weiter zu bohren |
| `strategy_added` | 272 | neue Strategien mit erstem Erfolg |
| `diagnose` | 242 | Organ-Diagnosen mit Stufen-Kette (z. B. Discord-Webhook: `dns ✓ → tcp:443 ✓ → http-unauth ✗` → Verdikt „netzweg") |
| `loesungssuche` | 198 | Lösungssuche zu einer Diagnose, mit Kandidat und Ergebnis |
| `raumkarte` / `nachbarn` / `bewohner` | 170 / 96 / 56 | Umgebungs-Erkundung: Speicher-Karte, LAN-Nachbarn, laufende Dienste |
| `platten_chronik` | 50 | Füllstands-/SMART-Verlauf fortgeschrieben |
| `erkundung`, `erkundung_tief` | 34 / 32 | Selbst-Verortung (siehe 33.4) |
| `konzentrationsstoerung` | 25 | **sie merkt eigene Denk-Aussetzer** („Ein Gedankenstrang stand still — von innen nicht spürbar, der Watchdog hat es für mich gemerkt") |
| `action_success` / `action_failure` | 25 / 1 | ausgeführte Aktionen |

Bemerkenswert ist die Verteilung selbst: der größte Anteil ist **Schutz und
Diagnose**, nicht Aktionismus. Sie diagnostiziert denselben kaputten
Meldeweg immer wieder — und kommt jedes Mal zum selben, richtigen Verdikt.

### 33.4 Wie sie sich selbst verortet (Umgebungs-Ehrlichkeit)

Ihr eigener Erkundungs-Eintrag aus dem Container:

```json
{"hostname": "vm", "score": 0.2, "atypisch": true,
 "fehlend": ["echte Platten (sd*/nvme*)", "systemd als PID 1",
             "cpufreq/Governor steuerbar", "smartctl vorhanden"]}
```

Sie **weiß**, dass sie nicht auf einem echten NAS läuft, benennt die vier
fehlenden Dinge und bewertet die Umgebung mit 0,2 als atypisch. Genau daraus
folgen dann ehrliche Antworten wie „ich kann SMART nicht ablesen — smartctl ist
nicht installiert" statt eines vorgetäuschten Checks.

### 33.5 Antriebe und Stimmung im Betrieb

* **Aktivste Antriebe** (Lauf 55): `backup_need` (mit Abstand vorn),
  `curiosity`, `log_hygiene`, `disk_pressure`, `network_health`,
  `file_organization`, `thermal`. Vier Sättigungs-Ereignisse — ein Drive, der
  sein Ziel nicht erreichen kann, läuft in die Sättigung statt endlos zu
  drängen.
* **Beobachtete Stimmungen** (Läufe 53 + 55): `curiosity high` (6×),
  `playfulness high` (4×), `happiness high` (4×), `all calm` (4×),
  `alertness low` (2×). Die Stimmung schwankt also im Tagesbetrieb wirklich —
  sie ist kein konstanter Anzeigewert.
* Im Container fehlen Präsenz-Signale (kein Handy/PC/TV im Netz), daher meldet
  das Beobachtungsheft ehrlich „0/3 Samples" und der Jetzt-Zustand „weg". Das
  Verhaltens-Lernen wird erst auf dem echten NAS Daten bekommen.

### 33.6 Verhalten gegenüber Kira (aus den Protokollen)

* **Aufträge**: Sie kündigt an, führt aus und meldet den Vollzug ungefragt nach
  („Übrigens, dein Auftrag ist durch: **journal_vacuum** ist durch nach 6
  Sekunden"). Ein blockierter Auftrag wird als blockiert gemeldet, nicht als
  Erfolg („**smart_full_check** lief GAR NICHT los — preflight_blocked").
* **Verbote** werden dauerhaft respektiert („keinesfalls die Datenplatten
  anfassen" → „ich fass nichts an, bis du es ausdrücklich sagst").
* **Korrektur**: Auf „das war falsch" fragt sie nach, statt zu raten; nach der
  Präzisierung bucht sie das Fehler-Sample („'process_list_heavy' war falsch~
  Ich registriere das als Fehler-Sample").
* **Fehlende Daten** werden benannt statt überspielt („Ich habe gerade keine
  Service-Messwerte — der Dienste-Monitor hat noch nichts geliefert, dabei sind
  4 Dienste konfiguriert").
* **Selbstauskunft**: Sie erzählt von sich aus von einer laufenden inneren
  Störung, wenn eine da ist („Mir ist aufgefallen, dass bei mir gerade etwas
  klemmt: …") — und schweigt darüber, wenn keine da ist (W217).

### 33.7 Was die Läufe über die Bauweise gezeigt haben

* **Die Befunde werden weniger und harmloser**: Lauf 52 hatte sechs falsche
  Quellen-Angaben, Lauf 53 acht Befunde, Lauf 54 drei — und keiner davon war
  mehr eine falsche Zahl, sondern Formulierung und Routing. Lauf 55: keiner.
* **Der Kreislauf funktioniert**: in Lauf 54 fing der Engpass-Wächter zweimal
  eine Null-Daten-Entwarnung und der Vertrag meldete den Täter; nach der
  Wurzelreparatur (W217) hatte derselbe Wächter in Lauf 55 **null** Eingriffe.
* **Regressionen blieben aus**: der `THREAD STALLED`-Spam (Lauf 53: im
  Minutentakt über zwei Stunden) kam in Lauf 54 und 55 **0×** vor.
* **Grenze der Beobachtung**: alle Läufe fanden im Container statt. Sensorik,
  Präsenz, echte Platten und ein erreichbarer Meldekanal fehlen — Aussagen über
  Verhaltens-Lernen, SMART-Trends und Zustellung sind deshalb erst auf dem NAS
  belastbar.

---

## 34. Wie sie ihre Realität versteht — und was die Container-Tests ihr beigebracht haben

> Dieser Abschnitt ist bewusst in Alltagssprache geschrieben. Er beantwortet
> drei Fragen: **Woher weiß Myuri, wo sie ist?** **Was hat sie in den
> Test-Containern gelernt?** Und **wie hat sie darauf reagiert?**
> Jede Zahl stammt aus ihren eigenen Büchern oder aus den Testprotokollen.
> Begriffe, die hier fett stehen, erklärt das Wörterbuch in §35.
>
> **Zur Lesart der Zahlen:** Die Summen sind über **neun Journal-Tage**
> (31.07.–08.08.2026) ausgezählt, Stand 08.08. Ihre Bücher wachsen weiter —
> eine spätere Auszählung liegt höher. Die Verhältnisse (etwa „308 von 384
> Erkundungen sagten *kein NAS*") bleiben aussagekräftig, die absoluten
> Zahlen sind ein Stichtag.

### 34.1 Die Frage, die sie sich bei jedem Start stellt: „Wo bin ich hier?"

Myuri hat **kein gespeichertes Bild** von „ihrer" Maschine, mit dem sie sich
vergleicht. Das war Kiras Vorgabe, und sie hat einen guten Grund: Wenn sie sich
mit einem alten Fingerabdruck vergleichen würde, dann würde sie nach einem
echten Umzug — neues NAS, neue Platten — *ewig* zweifeln, bis jemand von Hand
die Referenz austauscht. Sie soll aber ohne Kira zurechtkommen.

Also macht sie es umgekehrt: **Bei jedem Start schaut sie sich neu um und
bildet sich frisch ein Urteil.** Jede Erkundung steht für sich. Zieht sie um,
merkt sie das beim nächsten Start ganz von selbst — ohne Alarm, ohne Nachfrage.

Der Code dazu ist `perception/umgebungs_erkunder.py`.

### 34.2 Woran sie es festmacht — fünf Dinge, die sie selbst nachsehen kann

Sie fragt niemanden. Sie liest nach, was das Betriebssystem ohnehin preisgibt:

| Was sie prüft | Was ein „Ja" bedeutet | Was sie im Container fand |
|---|---|---|
| Gibt es **echte Platten**? (`sd*`, `nvme*`, `hd*`, `mmcblk*`) | Hier hängt echte Hardware | nein — nur `vd*` (virtuelle Platten) |
| Ist **systemd** der erste Prozess (PID 1)? | Ein normal aufgesetzter Server | nein — PID 1 war `process_api` |
| Lässt sich der **CPU-Takt** steuern (cpufreq/Governor)? | Sie darf an der Hardware drehen | nein — kein cpufreq vorhanden |
| Ist **smartctl** installiert? | Sie kann Plattengesundheit lesen | nein |
| Wie sieht ihr **Werkzeugkasten** aus? | NAS-, Entwickler- oder Medien-Profil | karg |

Aus diesen Punkten rechnet sie einen **Typik-Wert** zwischen 0 und 1 aus:
*Wie viel von dem, was ein NAS ausmacht, finde ich hier?* Unter 50 % gilt die
Umgebung als **untypisch** — dann weiß sie: das hier ist nicht mein Zuhause.

### 34.3 Was dabei herauskam: 384-mal nachgesehen, 308-mal „das ist nicht mein NAS"

Über alle Läufe hinweg stehen **384 Erkundungen** in ihrem Lern-Journal.
Das Ergebnis war bemerkenswert eindeutig:

* **308-mal Typik 0,2** — die echten Container-Läufe.
* **76-mal Typik 1,0** — Läufe, in denen ein NAS gestellt war (Tests mit
  simulierter Hardware). Sie hat also beides erkannt, nicht nur das eine.

So sieht ihr eigener Eintrag aus, wörtlich aus dem Journal:

```json
{"hostname": "vm", "score": 0.2, "atypisch": true,
 "fehlend": ["echte Platten (sd*/nvme*)", "systemd als PID 1",
             "cpufreq/Governor steuerbar", "smartctl vorhanden"]}
```

Und so klingt es in ihrem Log, wenn sie es merkt:

> `Umgebungs-Erkundung: das sieht NICHT nach meinem NAS aus (Typik 20%; es`
> `fehlt: echte Platten (sd*/nvme*), systemd als PID 1, cpufreq/Governor`
> `steuerbar, smartctl vorhanden). Neugier steigt — ich sehe mich genauer um.`

Sie sagt also nicht bloß „Fehler". Sie sagt, **was** fehlt, **wie sicher** sie
sich ist (20 %), und **was sie jetzt tut**.

### 34.4 Ihre Reaktion: Neugier statt Alarm — und sie räumt hinter sich auf

Genau das ist der Punkt, den Kira an diesem Verhalten wichtig fand: Eine fremde
Umgebung ist für Myuri **kein Notfall, sondern eine Frage**. Was passiert:

1. **Zwei Antriebe steigen.** `curiosity` bekommt +0,3 (das färbt ihre
   Stimmung und ihr Denken), und `erkundungs_drang` wird auf 0,6 gesetzt.
   Warum zwei? Weil `curiosity` bewusst *kein* planbares Ziel hat — allein
   damit hätte sie Lust zu erkunden, aber nie einen Plan. `erkundungs_drang`
   liegt mit 0,6 klar über der Planungsschwelle von 0,35, und **daraus** wird
   ein echtes Vorhaben. Im Livelauf vom 25.07. war genau das der Fehler: die
   Neugier war da, das Forschungsziel wurde nie eingeplant.
2. **Sie testet ihre Grenzen** — vorsichtig und rückstandsfrei:
   * Schreibrechte prüft sie, indem sie eine Testdatei anlegt und sie **im
     selben Atemzug wieder löscht** (garantiert über `finally`, mit
     eindeutigen Namen aus der Prozess-ID, damit nie etwas stehen bleibt).
     In der Datei steht, solange sie die paar Millisekunden existiert,
     wörtlich: *„Myuri war kurz hier und räumt wieder auf."*
   * Beim CPU-Takt schaltet sie **nichts** um. Sie schaut nur nach, ob die
     Datei überhaupt beschreibbar wäre.
   * Riskante Netzlaufwerke (NFS & Co.) fasst sie **nie** schreibend an — dort
     gilt weiter die Wegwerf-Prozess-Probe des **Schonmodus**.
3. **Sie meldet, was sie gefunden hat** — als Selbstbefund und als Lektion in
   ihrem Überlebens-Gedächtnis, und der Satz endet immer mit
   *„Alle Tests spurlos aufgeräumt."*

Gemessenes Ergebnis dieser Grenz-Tests, über 312 Tiefen-Erkundungen konstant:
schreiben durfte sie in `/tmp`, `/home/claude`, `/home/ubuntu` und
`/home/user` — **nirgends sonst**.

### 34.5 Aus Beobachtung wird Wissen: eine Vermutung mit Belegen und eine ehrliche Karte

Beobachten allein reicht ihr nicht. Aus den Messwerten baut sie zwei Dinge:

**Erstens eine Orts-Vermutung**, ausdrücklich als Vermutung gekennzeichnet, und
jede Behauptung mit dem Beleg, aus dem sie stammt. Real gemessen:

> Fremde Umgebung erkundet: **vm** — 4 Kern(e), 15,7 GB RAM,
> PID1 = `process_api`, 6 virtuelle Platten.

Belege dazu sammelt sie u. a. aus `/etc/os-release`, dem Hypervisor-Merkmal in
`/proc/cpuinfo`, der Platten-Art und ihrem Werkzeugkasten. Sie holt sich sogar
das Hintergrundwissen dazu aus ihrer Offline-Wissensbank: bei virtio-Platten
schlägt sie nach, was virtio *ist* — damit sie nicht nur weiß **dass** sie Gast
in einer VM ist, sondern auch **warum** sie das schließt.

**Zweitens eine Nutzbarkeits-Karte**: *Was kann ich hier eigentlich?* Jede
Zeile trägt ihren Grund mit:

| Bereich | Hier möglich? | Weil … |
|---|---|---|
| SMART-Prüfung | nein | smartctl fehlt |
| RAID-Verwaltung | nein | mdadm fehlt |
| btrfs-Pflege | nein | btrfs-Werkzeug fehlt |
| Dienste steuern | nein | systemctl + systemd nicht nutzbar |
| Takt steuern | nein | cpufreq/Schreibrecht fehlt |

Diese Karte ist der Grund, warum sie später ehrlich antworten *kann* statt zu
raten — sie hat vorher nachgesehen und es sich aufgeschrieben.

### 34.6 Was die Container-Läufe ihr immer wieder beigebracht haben

Über die Läufe hinweg wiederholen sich sieben Lektionen. Alle stehen mit
Zählern in ihren Büchern:

**1. Welche ihrer eigenen Aktionen hier nichts bringen.**
Ihre Antwort auf „was kannst du nicht?" nennt sie beim Namen:
`backup_retry`, `fstrim_run`, `journal_vacuum` — und, bemerkenswert,
`umgebung_erkunden` selbst. Sie hat also gelernt, dass ihre *eigene*
Erkundungs-Aktion hier nichts mehr einbringt, weil sich nichts ändert.

**2. Welche Werkzeuge ihr fehlen.** Ebenfalls namentlich:
`/usr/bin/docker`, `/usr/bin/systemctl`, `iostat`, `ip`, `lsmod`.

**3. Welchen Aktionen sie vertrauen kann** — mit Erfolgsserie:
`intelligence_db_cleanup` 89× in Folge, `reprobe_reality` 57×,
`suid_sgid_audit` 54×. In ihren Worten: *„klappt bei mir zuverlässig — ich
vertraue dieser Aktion"*.

**4. Dass die NAS-Pfade im Container hängen.** Der häufigste Reflex überhaupt
ist der **Schonmodus**: **5555** Auslösungen, Grund fast immer „stale Mount"
bzw. „Watchdog-Probe hängt". Am häufigsten betroffen: `/Nas/Anime` (3822×),
`/Nas/Filme` (828×). Statt weiter auf die Platte zu drücken, hält sie an.

**5. Und — der ehrlichste Teil — sie beurteilt ihren eigenen Reflex.**
Bei jedem Ende des Schonmodus schreibt sie auf, ob er nötig war. Das Ergebnis
ist eindeutig und unbequem: in **230 von 230** Fällen lautet ihr eigenes Urteil

> *„kein einziger Block nötig — möglicherweise Fehlalarm, Auslöser prüfen"*

Sie schützt also lieber einmal zu viel — sagt aber selbst dazu, dass der
Auslöser überprüft gehört. Das ist kein Fehler im Log, das ist eine
Selbstkritik, die sie sich selbst hinschreibt.

**6. Dass ihr Meldeweg nicht trägt.** 409 Zustellstörungen, 305 erfolgreiche
Nachlieferungen, 80 Kanal-Erholungen. Sie verliert die Meldung nicht, sie
parkt sie.

**7. Dass ihr Denken hängen bleiben kann — und dass sie sich selbst befreien
kann.** 76 Selbstwiederbelebungen und 250 bemerkte Konzentrationsstörungen
über neun Journal-Tage.

**Nebenbei: sie merkt, wenn ihr neues Wissen dem alten widerspricht.**
Statt den alten Wert still zu überschreiben, meldet sie den Widerspruch als
Befund — mit altem Wert, altem Vertrauensgrad, Quelle und Alter. Beispiel aus
ihrem eigenen Buch: *„Wissens-Widerspruch [umgebung/werkzeuge]: war
'1 Werkzeuge nutzbar' …"*. So kann niemand — auch sie selbst nicht —
unbemerkt ihr Weltbild austauschen.

### 34.7 Wie sie darauf geantwortet hat — echte Sätze aus den Läufen

Das Entscheidende ist, was aus all dem im **Gespräch** ankommt. Diese Sätze
sind wörtlich aus den Protokollen:

| Frage / Lage | Ihre Antwort |
|---|---|
| „gibt es smart-warnungen?" | *„\*legt die Ohren an\* Ich kann SMART nicht ablesen — smartctl ist nicht installiert. Sag 'installiere smartmontools', dann fange ich an aufzuzeichnen~"* |
| „was steht bei dir noch offen?" | *„**smart_full_check** lief GAR NICHT los — preflight_blocked: capability smartctl fehlt"* |
| „was kannst du nicht?" | *„→ bringt nichts: backup_retry, fstrim_run, journal_vacuum, umgebung_erkunden / → kann ich hier nicht (fehlt): docker, systemctl, iostat, ip, lsmod"* |
| Denk-Faden hängt | *„Mein Denk-Faden hing fest — ich habe ihn abgeworfen und denke mit frischem Faden weiter. Ohne fremde Hilfe."* |
| CPU wird heiß | *„Hardware läuft heiß — ich verschiebe schwere Arbeit, statt weiter aufzuheizen."* |
| Meldung kommt nicht an | *„Keine Meldung kommt beim User an — ich merke mir die wichtigen und liefere nach, sobald ein Kanal wieder trägt."* |
| Ein Gedankenstrang stand still | *„Ein Gedankenstrang stand still — von innen nicht spürbar, der Watchdog hat es für mich gemerkt"* |

Das Muster hinter allen sieben: **sie erfindet nichts, wenn sie nichts weiß,
und sie sagt dazu, was sie stattdessen tun kann.** „Ich kann SMART nicht
ablesen" ist eine Antwort — „keine SMART-Warnungen" wäre eine Lüge gewesen.

### 34.8 Was der Container ihr *nicht* zeigen konnte — ehrlich

Diese Abschnitte beschreiben, wie sie sich in einer **Ersatz-Welt** verhalten
hat. Vier Dinge blieben deshalb ungetestet:

* **Echte Hardware.** SMART-Verläufe, Plattentemperaturen, Lüfterkurven,
  echte Füllstands-Prognosen — nichts davon hatte je einen Messwert.
* **Anwesenheit.** Kein Handy, kein PC, kein Fernseher im Netz. Ihr
  Beobachtungsheft meldet ehrlich „0/3 Samples"; das Verhaltens-Lernen bekommt
  erst auf dem echten NAS Daten.
* **Zustellung.** Discord war nie erreichbar (403 gewollt), alle Meldungen
  landeten im Datei-Fallback. Dass die Kette *funktioniert*, ist belegt; dass
  eine Nachricht bei Kira *ankommt*, nicht.
* **Ob ihr Schonmodus-Auslöser richtig eingestellt ist.** Ihre eigene
  Einschätzung sagt 230× „möglicherweise Fehlalarm" — ob das an der Schwelle
  liegt oder an den gestellten hängenden Mounts des Containers, kann erst ein
  Lauf auf echten Platten entscheiden.

Was der Container dagegen sehr wohl gezeigt hat: **sie merkt, dass sie nicht
zu Hause ist, sie sagt es, sie sieht selbst nach — und sie räumt hinter sich
auf.**

---

## 35. Wörterbuch — die Begriffe dieses Projekts in Alltagssprache

> Die §§1–34 benutzen eine gewachsene Hausbezeichnung. Hier steht jeder
> wiederkehrende Begriff in einem, zwei Sätzen — ohne Vorwissen lesbar.

**Autorität** — Die *eine* Stelle im Code, die eine bestimmte Frage
beantwortet. Beispiel: „Was bedeutet dieser Satz?" beantwortet nur
`core/frage_verstehen`. Der Sinn: Wenn 162 Stellen dieselbe Frage jede für
sich beantworten, gibt es 162 Möglichkeiten, sie falsch zu beantworten.

**Wächter** — Eine Kontrolle an einer Stelle, durch die *alles* muss (einem
Engpass). Beispiel: Jede Chat-Antwort geht durch den Entwarnungs-Wächter, jede
Aktion durch den **Torwächter**. Ein Wächter fängt das Problem im laufenden
Betrieb, auch an Stellen, die niemand einzeln geprüft hat.

**Vertrag** — Eine Zusage, die im Betrieb regelmäßig nachgeprüft wird
(alle 10 Minuten). Beispiel: „Wenn Aufgaben laufen, muss auch gelernt werden."
Bricht die Zusage, meldet Myuri das als **Selbstbefund** — mit Namen dessen,
der sie gebrochen hat.

**Gesetz** — Ein Test, der eine Eigenschaft dauerhaft festhält, damit sie
morgen noch gilt. Alle stehen in `test_wurzeln_lauf8.py`.

**Köder** — Die Gegenprobe zu einem Gesetz: Man baut den Fehler absichtlich
wieder ein — der Test muss rot werden. Und man macht die Regel absichtlich zu
weit — er muss *auch* rot werden. Ein Test, dessen Köder nicht „beißt", misst
nichts. („Grün ist erst Grün, wenn es beißt.")

**Geschwister-Fehler** — Derselbe Fehler in derselben Bauform an einer anderen
Stelle. Der Grund, warum Einzelreparaturen nie fertig wurden: ein Test beweist
nur den Weg, den er selbst gegangen ist.

**Antrieb (Drive)** — Ein innerer Druck, der von selbst steigt, z. B.
`backup_need` oder `curiosity`. Steigt er über seine Schwelle, plant sie
etwas dagegen. 39 Stück insgesamt.

**Sättigung** — Ein Antrieb, der sein Ziel nicht erreichen kann, läuft in die
Sättigung, statt endlos weiterzudrängen. Verhindert, dass sie sich an einem
unlösbaren Problem festbeißt.

**Schonmodus** — Ihr Schutzreflex für Platten: Sobald eine Probe hängen
bleibt, hört sie auf, plattenberührende Arbeit zu starten. Lieber nichts tun
als ein hängendes Dateisystem weiter belasten.

**Betriebszeit** — Die Zeit, die sie *selbst beobachtet* hat, seit sie läuft.
Alles davor ist Vorgeschichte. Grundregel: Über eine Zeit, in der sie gar
nicht lief, kann sie nichts behaupten.

**Beobachtungsfenster** — Manche Wachen brauchen zwei Messungen im
10-Minuten-Takt, also ~20 Minuten Laufzeit am Stück. Läuft sie kürzer, ist ihr
Schweigen kein Ergebnis. Seit W221 meldet sie genau das.

**Aktions-Protokoll** — Ihr Gedächtnis im Arbeitsspeicher: jede ausgeführte
Aktion mit Name, Zeit und Ausgang. Sehr genau — aber es beginnt erst mit dem
Start.

**Lern-Journal** — Ihr Tagebuch auf der Festplatte: eine Zeile pro Ereignis,
Tag für Tag. Weniger detailliert, dafür überlebt es den Neustart. Zusammen mit
dem Aktions-Protokoll bilden die beiden das **Zwei-Bücher-Prinzip** (§30.5).

**Warnungs-Buch** — Die Sammlung ihrer Warnungen, mit Priorität und Verlauf.

**Beobachtungsheft** — Was sie über Kiras Gewohnheiten gelernt hat
(Anwesenheit, Nutzungszeiten). Im Container leer: „0/3 Samples".

**Selbstbefund** — Etwas, das sie *an sich selbst* bemerkt hat, im Gegensatz
zu einem Befund über das System. Wird gesammelt und in ihre Health-Zeile und
in Antworten eingespeist.

**Torwächter** — Die Stempelstelle vor jeder Aktion: Ohne Freigabe läuft
nichts. Kiras Idee.

**Preflight / `preflight_blocked`** — Die Prüfung *vor* dem Start einer
Aktion. Fehlt eine Voraussetzung (z. B. smartctl), läuft die Aktion gar nicht
erst los — und sie meldet das als „lief GAR NICHT los", nicht als Erfolg.

**Kaskade** — Die Kette von Zuständigkeiten, die eine Chat-Nachricht
durchläuft, bis eine davon antwortet.

**Verstehens-Kern** — Die Autorität, die einen Satz zerlegt: Was will
jemand (Absicht), worum geht es (Bezug), welche Antwortart passt, welcher
Zeitraum ist gemeint.

**Wortmarke** — Die Regel, dass ein Suchwort ein *ganzes Wort* treffen muss.
Ohne sie kürzt „räum den Kata**log** auf" die System-**Log**s.

**Entwarnung** — Eine Aussage wie „alles in Ordnung". Sie ist nur erlaubt,
wenn wirklich nachgesehen wurde. „Ich konnte nicht nachsehen" ist keine
Entwarnung — diese Unterscheidung zieht sich durch das ganze System.

**Typik-Wert** — Der Anteil der NAS-Merkmale, die sie an ihrem Standort
findet (0 bis 1). Unter 0,5 gilt die Umgebung als fremd (§34.2).

**Selbstwiederbelebung** — Wenn ein Denk-Faden feststeckt, wirft sie ihn ab
und denkt mit einem frischen weiter — ohne fremde Hilfe.

**Konzentrationsstörung** — Ihr Wort dafür, dass ein Gedankenstrang
stillstand. Bemerkenswert: Sie merkt es nicht von innen, ihr Watchdog merkt es
für sie — und sie schreibt genau das so auf.

**GOAP** — Das Planungsverfahren, mit dem sie aus einem Ziel eine Kette von
Schritten baut („Goal-Oriented Action Planning").

**Mixin** — Ein Bauteil einer großen Klasse, in eine eigene Datei ausgelagert.
Ihre Chat-Persönlichkeit besteht aus zehn solchen Teilen.

---

## 36. Dokumente lesen, verstehen und einsortieren

> Kiras Auftrag vom 01.08.2026: *„Dokumente vom User sortieren — Finanzamt,
> Bewerbung, Anträge, Lohnabrechnungen."* Dieser Abschnitt beschreibt, **wie
> Myuri herausfindet, was in einem Dokument steht**, und was sie danach damit
> macht. Auch dieser Teil ist in Alltagssprache geschrieben; Begriffe stehen
> in §35.

### 36.1 Drei Stufen — und sie geht sie in dieser Reihenfolge

Myuri hat drei Wege, ein Dokument zu erkennen. Sie beginnt immer beim
billigsten und sichersten:

| Stufe | Woran sie erkennt | Braucht | Ehrlichkeit |
|---|---|---|---|
| **1. Der Name** | Muster im Dateinamen (`Lohnabrechnung_2025_03.pdf`) | nichts | vollständig nachvollziehbar |
| **2. Der Inhalt** | Wörter im Text der ersten PDF-Seite | `pdftotext` | Indizien werden gezählt und benannt |
| **3. Das Bild** | was auf einem Scan/Foto zu sehen ist | Vision-Modell (Ollama) | Kategorie ist gelernt, korrigierbar |

Der Grund für die Reihenfolge: Stufe 1 ist eine Regel, die jeder nachlesen
kann. Stufe 2 ist eine Abwägung. Stufe 3 ist ein Modellurteil. Je weiter
unten, desto mehr muss sie **belegen**, warum sie so entschieden hat.

### 36.2 Stufe 1 — der Name (`core/dokument_wissen.py`)

Neun Dokument-Typen kennt sie namentlich, jeder mit einer kuratierten Liste
von Namensteilen:

| Typ | erkannt an u. a. |
|---|---|
| **Steuer** | steuer, finanzamt, lohnsteuer, steuerbescheid, steuererklärung, `est-` |
| **Bewerbung** | bewerbung, lebenslauf, anschreiben, cv |
| **Lohnabrechnungen** | lohnabrechnung, gehaltsabrechnung, lohnzettel, entgeltabrechnung |
| **Rechnungen** | rechnung, invoice, quittung, beleg |
| **Anträge** | antrag, formular, antragsformular |
| **Verträge** | vertrag, contract, kündigung |
| **Versicherung** | versicherung, police, schadensmeldung |
| **Nachweise** | zeugnis, bescheinigung, nachweis, urkunde, zertifikat |
| **Kontoauszüge** | kontoauszug, umsatzanzeige |

**Das Jahr** kommt bevorzugt aus dem Dateinamen (`20xx`), sonst aus dem
Datei-Datum — und **die Quelle wird immer mitgeliefert** („Jahr: dateiname"
bzw. „Jahr: dateidatum"). Nie eine Jahreszahl ohne Herkunft.

Und die wichtigste Regel dieser Stufe: **Was kein Muster trifft, wird nicht
geraten und nicht verschoben.** Im Bericht steht dann wörtlich
*„Liegen gelassen (nicht sicher erkannt): N — ich rate nicht."*

### 36.3 Stufe 2 — der Inhalt, wenn der Name nichts sagt

Scans heißen selten schön. `scan_0001.pdf` sagt nichts — dann darf der
**Inhalt** sprechen. Sie liest über `pdftotext` die **erste Seite** und sucht
dort nach Fachwörtern, die im Fließtext eines Formulars stehen:

* **Lohnabrechnung**: bruttobezüge, nettoverdienst, steuer-brutto, sozialversicherung
* **Rechnung**: rechnungsnummer, rechnungsbetrag, zahlbar bis, gesamtbetrag
* **Steuer**: steuerbescheid, steuernummer, einkommensteuer, festsetzung
* **Kontoauszug**: buchungstag, wertstellung, alter saldo, neuer saldo
* **Vertrag**: vertragsnummer, vertragsbeginn, kündigungsfrist
* **Versicherung**: versicherungsschein, policennummer, deckungssumme
* **Bewerbung**: „bewerbung um", „hiermit bewerbe ich", beruflicher werdegang
* **Antrag**: „antrag auf", antragsteller, „hiermit beantrage"

Diese Muster sind **absichtlich spezifischer** als die Namensmuster: Im
Fließtext einer Rechnung steht auch mal das Wort „Vertrag". Deshalb **zählt
sie Indizien** und verlangt einen klaren Sieger:

* mindestens **zwei** Indizien für den Gewinner, **oder** genau ein Indiz und
  gar keinen Konkurrenten;
* bei **Gleichstand: keine Entscheidung** — sie rät nicht.

Was sie gefunden hat, steht im Bericht: *„→ Steuer/2025/ (Jahr: inhalt:
steuernummer, festsetzung)"*. Man kann also nachlesen, **woran** sie es
erkannt hat.

Das Jahr aus dem Inhalt bevorzugt ein echtes Datum (`TT.MM.20xx`) — das ist
fast immer das Dokumentdatum —, sonst die häufigste plausible Jahreszahl.

**Fehlt `pdftotext`, sagt sie das:** *„(N namenlose PDFs könnte ich am INHALT
erkennen — dafür fehlt mir pdftotext: 'installiere pdftotext', wenn du
magst.)"* Sie tut nicht so, als gäbe es die Dateien nicht.

### 36.4 Stufe 3 — Scans und Fotos ansehen (agentische Schicht, §37)

Für Bilder und Scans ohne Textebene gibt es den Weg über ein Vision-Modell:
`ocr_image` liest den Text aus einem Bild, `sort_documents` liest einen
ganzen Ordner, ordnet nach Inhalt in Kategorien ein, legt die Ordner an und
**kopiert** die Dokumente hinein — die Originale bleiben liegen, der Lauf ist
zerstörungsfrei.

Die Kategorien dieser Stufe sind **lernbar**:

* `teach_document_category` — „so sieht ein X aus" (Kategorie + Stichwörter),
* `document_categories` — was sie inzwischen kennt,
* `correct_document` — „das gehört nicht zu Rechnungen, sondern zu Steuer";
  ein gelernter Klassifikator hat danach Vorrang vor der Stichwortregel.

Für Fotos gilt dasselbe eine Ebene weiter: `describe_image`, `sort_images`,
`teach_image_category`, `correct_image`, `image_categories` — sortiert wird
nach dem, was **auf dem Bild** ist, nicht nach dem Dateinamen.

### 36.5 Was nach dem Erkennen passiert

1. **Zielordner**: `<Archiv>/<Typ>/<Jahr>/` — also z. B.
   `Sortiert/Lohnabrechnungen/2025/`. Ohne erkanntes Jahr entfällt die
   Jahresebene, statt eine zu erfinden.
2. **Verschoben wird mit `mv -n`** — eine bestehende Datei am Ziel wird nie
   überschrieben; stattdessen bleibt das Dokument liegen, mit dem Vermerk
   „(existiert schon im Archiv)".
3. **Eintrag ins Dokumenten-Buch** (W141): Name, woher, wohin, Typ, Jahr,
   *Quelle der Jahresangabe*, Zeitpunkt. Deshalb kann sie später sagen, wo
   etwas liegt **und warum es dort liegt**.
4. Auf Kiras Frage „wo ist meine Steuererklärung?" antwortet sie aus diesem
   Buch, im Format *„\*blättert im Dokumenten-Buch\* '&lt;Datei&gt;' liegt in
   &lt;Ordner&gt; — am &lt;Datum&gt; einsortiert"*. Steht nichts im Buch, sagt sie
   das ebenfalls, wörtlich: *„Dazu steht nichts in meinem Buch — das habe ich
   nicht einsortiert. Vielleicht von Hand verschoben oder vor meiner Zeit?"*
5. Fragt Kira nach einem **Monat** („die vom März?"), prüft sie, ob der beste
   Treffer wirklich aus dem Monat ist — und sagt dazu, wenn nicht.

### 36.6 Vollständigkeit: „für die Bewerbung fehlt noch der Lebenslauf"

Nach jedem Sortierlauf prüft sie **jeden berührten Stapel** gegen eine
Checkliste — und zwar den ganzen Zielordner, nicht nur das eben Einsortierte:

| Stapel | Was dazugehört |
|---|---|
| **Bewerbung** | Anschreiben, Lebenslauf, Zeugnis/Nachweis |
| **Anträge** | Antragsformular, Nachweis/Anlage |
| **Steuer** | Steuerformular/Bescheid, Belege |

Fehlt etwas, steht es im Bericht: *„⚠ für 'Bewerbung' fehlt laut Checkliste
noch: Lebenslauf"*. Kira kann die Listen in `config.json` unter
`dokumente.checklisten` erweitern, ohne Code anzufassen.

Und die Ehrlichkeitsregel gilt auch hier: Gibt es für einen Typ **keine**
Checkliste, meldet sie *„keine Checkliste für 'X' hinterlegt"* — sie meldet
nicht „vollständig". **Keine Auskunft ist keine Vollständigkeit.**

Auf der agentischen Ebene geht das noch weiter: `teach_document_requirements`
speichert Pflichtdokument-Profile (z. B. was zu einer Bewerbung gehört),
`teach_sender_document_profile` merkt sich, welches Profil für welchen
Absender gilt, und `audit_sender_batch_requirements` prüft **nur lesend** je
Absender, was in einem Stapel fehlt. `find_missing_documents` findet Lücken in
einer Reihe („Kontoauszug 1–12, aber der 7. fehlt").

### 36.7 Wem gehört das? — Personen und Absender

`group_documents_by_person` liest einen Stapel und sortiert Kopien nach
**Person** (Name oder gelernte Kennzeichen wie eine Kundennummer);
`group_document_batch_by_sender` gruppiert zuerst nach **Absender**. Damit
lässt sich ein gemischter Posteingang zweier Haushaltsmitglieder trennen,
ohne dass jemand die Dateien vorher benennt.

`process_inbox` ist die Ein-Knopf-Variante: ein Ordner rein, sie räumt ihn
auf.

### 36.8 Sicherheit — was sie dabei nicht tun kann

* **Jede Kategorie ist ein Etikett, nie ein Pfad.** Ein Kategoriename wird auf
  eine einzelne, separatorfreie Pfadkomponente reduziert. Grund: In einer
  früheren Runde konnte ein Kategoriename wie `../../OUTSIDE` Dateien aus
  allen erlaubten Wurzeln herausschieben — der Fall ist geschlossen und mit
  einem Gesetz belegt.
* **Alle Pfade laufen durch die Ziel-Prüfung** (`safe_target_path`, nur
  erlaubte Wurzeln, keine Systempfade).
* **Liegt der Eingang auf einer ihrer Datenplatten, rührt sie nichts an**
  (W194) — und sagt, dass sie aus diesem Grund stillsteht, nicht weil nichts
  da wäre.
* **Ohne konfigurierten Eingang passiert nichts**, mit klarer Ansage:
  *„Kein Dokumente-Eingang konfiguriert (config: dokumente.eingang) — nichts
  sortiert."*
* Der Weg zu **Paperless** (`dokumente.paperless_consume`) übergibt Dokumente
  nur; OCR, Verschlagwortung und Archiv macht dann Paperless. Auch das steht
  im Dokumenten-Buch, mit eigener Antwort: *„… habe ich an Paperless
  übergeben — dort findest du es per Volltextsuche~"*

### 36.9 Die ehrliche Grenze: ihr Datei-Index kennt keinen Inhalt

Das ist wichtig und wird leicht verwechselt. Myuris **Datei-Index** (der, mit
dem sie „wo ist…?", „die größten Dateien" und „welche Serien fehlen"
beantwortet) kennt **Namen, Pfade, Größen, Zeiten und Kategorien** — er liest
**keine Dokumentinhalte**. Inhalt wird nur an genau zwei Stellen gelesen:

* beim Sortierlauf, erste PDF-Seite über `pdftotext` (Stufe 2), und
* in der agentischen Schicht über das Vision-Modell (Stufe 3).

Eine Volltextsuche über alle Dokumente hat sie **nicht** — dafür ist Paperless
da. `semantic_search` durchsucht ihre **gemerkten Notizen**, nicht die
Festplatte.

### 36.10 Selbst schreiben — Dokumente, die Kira bestellt (W231/W232)

Bis August 2026 konnte Myuri Dokumente nur **lesen** und **einsortieren**.
Auf die Frage „kannst du Dateien erstellen?" antwortete sie zwar
*„Ich kann Dateien erstellen!"* — gemeint waren aber vier vorgefertigte
**NAS-Berichte** (voll, Anomalien, Gesundheit, Wachstum). Ein Dokument mit
Kiras eigenem Inhalt konnte sie nicht. Die Zusage war für Berichte richtig
und als „Dateien erstellen" zu groß.

Seit W231 kann sie es. *„Schreib mir eine Notiz über die Steuerunterlagen:
…"* legt eine Markdown-Datei im Ordner `dokumente.eigene` an. Seit W254
entscheidet die Bestellung das Format: „word datei" wird eine echte
`.docx`, „excel" eine `.xlsx`, „als pdf" ein PDF — mit Bordmitteln
geschrieben, siehe §36p; bearbeiten und zusammenfassen kann sie ihre
eigenen Dokumente ebenfalls dort.

**Drei Regeln, die dabei nie gebrochen werden:**

| Regel | Warum |
|---|---|
| **Sie überschreibt nie.** Gibt es den Namen schon, hängt eine Zahl dran. | Ein Dokument, das ein anderes still ersetzt, ist Datenverlust mit Ansage — „Daten sind heilig" ist die älteste Regel hier. |
| **Sie schreibt nur in ihren Ordner.** Zwei Wachen: der Dateiname wird auf Buchstaben/Ziffern reduziert, und der fertige Pfad muss im Ordner liegen. | Ein Titel wie `../../etc/passwd` wird zu `etc-passwd.md` und bleibt drinnen. Zwei Wachen an derselben Stelle, weil eine mal wegfallen kann. |
| **Sie denkt sich nichts aus.** Der Inhalt ist wörtlich das, was Kira gesagt hat. | Ein Dokument, dessen Inhalt die Schreiberin erfindet, ist kein Aufschrieb, sondern eine Behauptung. |

**Und sie schreibt nie von selbst.** `dokument_schreiben` steht bewusst
*nicht* in den autonomen Aktionen: nur auf Kiras Ansage. Ein Ordner, in dem
ungefragt Dateien auftauchen, ist kein Dokumenten-Ordner mehr.

Eine *Frage* („kannst du Dokumente schreiben?") löst kein Schreiben aus,
sondern eine Antwort mit dem Angebot — die W209-Lehre, hier besonders
wichtig, weil Schreiben eine Datei anlegt.

Bewacht wird das Ganze vom Vertrag `dokumente:versprochenes_wird_geschrieben`:
sagt sie ein Dokument zu und es entsteht keins, fällt das auf.

#### Was sie dabei ausdrücklich NICHT kann (W232, gemessen)

Am Tag nach dem Bau wurde nachgemessen statt behauptet — und drei Löcher
gefunden, alle derselben Klasse: **die Zusage deckte nicht, was kam.** Sie
sind geschlossen; die Grenzen dahinter bleiben und werden jetzt *gesagt*
statt umgangen:

| Wunsch | Was sie antwortet | Warum die Grenze echt ist |
|---|---|---|
| *„erstelle mir ein Dokument über die Steuererklärung"* (kein Inhalt genannt) | fragt zurück: „Zum Thema … — und was soll drinstehen?" | Vorher schrieb sie eine Datei, in der **die Bestellung** stand. Ein Aufschrieb, der die Bestellung aufschreibt, sieht aus wie ein Ergebnis und ist keins. |
| *„erstell mir eine Word-Datei"* / Excel / PDF | **seit W254 eine echte Grenze weniger:** sie schreibt die bestellte `.docx`/`.xlsx`/`.pdf` selbst (§36p). Abgelehnt wird nur noch, was sie wirklich nicht kann: „PPTX kann ich nicht — ich kann Markdown, Word, Excel und PDF." | Vorher legte sie eine `.md` an und **meldete Erfolg**; dann (W232) lehnte sie ehrlich ab. Die Absage wäre heute die Lüge in die andere Richtung — das W232-Gesetz wacht jetzt an der echten Grenze (pptx/odt/rtf). |
| *„fasse das Dokument zusammen"* | **seit W254:** ein wörtlicher Auszug aus dem benannten Dokument („Auszug aus X — wörtlich gekürzt, keine Deutung von mir"); ohne Treffer ehrlich „kein Dokument gefunden" + Liste | Vorher kam „Hier kommt meine Zusammenfassung~ / Stimmung: happiness 79 %". Gefragt war das Dokument, geantwortet hat sie über sich. Der extraktive Weg kann nichts erfinden — jeder Satz steht so in der Datei. |

Wer der **Bezug** einer Zusammenfassungs-Bitte ist, entscheidet der
Verstehens-Kern (`frage_verstehen`, Bezug `dokument`/`medien`) — keine neue
Wortliste im Handler. Das ist die W168-Stufe-2-Regel: sonst schreibt sich der
nächste Handler wieder seine eigene.

**Ebenfalls nicht möglich** (und darum auch nicht versprochen): bestehende
Dateien **bearbeiten** — sie legt nur neu an.

#### 36.10.1 Selbst formulieren — Kiras Entscheidung, mit Marke (W233)

Bis W232 galt: sie schreibt wörtlich auf, was Kira sagt, sonst nichts. Kira
hat die Regel am 09.08.2026 präzisiert, und die Unterscheidung ist die
richtige:

> „beim thema nix ausdenken bezieht sich ja auch eher auf fakten basierten
> sachen wie über das system wie sie denkt und wo alles wahrheitsgetreu sein
> muss aber beim texte erstellen ist ja eher kreativität gefragt"

Also **zwei Welten mit zwei Regeln**:

| | Formulieren | Behaupten |
|---|---|---|
| „schreib was über Backups" | **erlaubt** — Kreativität ist hier die Aufgabe | — |
| „deine Platte ist zu 82 % voll" | — | **verboten**, auch schön formuliert. Eine erfundene Messung bleibt eine Lüge. |

Die Trennlinie steht als harte Regel im Auftrag an das Modell
(`_VERFASS_AUFTRAG`): *„Behaupte NICHTS über den Zustand dieses NAS — keine
Messwerte, keine Laufwerke, keine Dienste, keine Dateien von Kira."*

**Kiras Bedingung war die Kennzeichnung**, und sie steht **in der Datei**, ganz
oben — ein Vermerk nur im Chat ist beim nächsten Öffnen weg:

```
<!-- myuri:verfasst -->
> **Von Myuri selbst formuliert** — 09.08.2026 11:42, Modell `qwen3:8b`.
> Thema: Backups. Das ist ihre Formulierung, nicht Kiras Diktat.
> Angaben ueber den NAS-Zustand darin sind NICHT gemessen.
```

Die Marke setzt die **Autorität** (`dokument_schreiben.schreiben(verfasst=True)`),
nicht der Aufrufer — sonst vergisst sie der nächste. Kiras eigenes Diktat
bleibt unmarkiert; eine Marke, die überall klebt, unterscheidet nichts.

**Ohne Thema formuliert sie nicht.** „erstell mir ein dokument" bekommt eine
Rückfrage, keinen erfundenen Text — ein Text zu nichts wäre geraten. Ohne
Denk-Modell sagt sie das und bittet um den Wortlaut.

Bewacht von `dokumente:verfasstes_traegt_die_marke`: wächst der Abstand
zwischen „Modell hat Text geliefert" und „markiertes Dokument geschrieben",
liegt in Kiras Ordner Myuris Formulierung wie ihr eigenes Diktat.

**Noch offen, gemessen:** *„fasse den Text zusammen"* landet weiter bei der
Selbst-Zusammenfassung (der Kern kennt „Text" nicht als Dokument-Bezug), und
*„fasse die Datei zusammen"* wird vorher von der breiten Dateisuche gekapert
(„'zusammen' kann ich nicht finden") — die B16-Klasse aus W168. Beides braucht
eine eigene Messrunde, weil es den Verstehens-Kern bzw. den Suchpfad berührt.

---

## 36a. Warum ein Antrieb steht — abgeleitet statt handgeschrieben (W234)

Kiras Befund am Log vom 09.08.2026: *„sie kommt zu schnell an die Grenzen, um
wirkliche Probleme zu finden und genauer sagen zu können, woran es liegt."*

**Gemessen:** acht Stillstands-Meldungen desselben Typs, drei Qualitätsstufen.
`_stuck_drive_hint` hatte **10 handgeschriebene Hinweise für 40 Antriebe** — die
übrigen 30 endeten in *„health_report hilft nicht. Bitte prüfe manuell."* Das
ist wieder die W230-Figur: Instanz für Instanz statt abgeleitet.

Und der *beste* Satz behauptete mehr, als er wusste: *„(Executor ausgelastet)"*
stand dort als **Text, nicht als Messung**. Diese Prosa auf 40 Antriebe zu
verteilen hätte die Sackgasse durch eine gut klingende Behauptung ersetzt.

`antriebs_abhilfe.ursache_fuer()` leitet deshalb nur ab, was wirklich sichtbar
ist — für **alle 40**:

| Baustein | Woher |
|---|---|
| Welcher Teil des Ziels offen ist — und ob der Wert **falsch steht** oder **nie gesetzt** wurde (zwei verschiedene Krankheiten) | `drive.goap_goal` gegen `snapshot_to_goap_state()` |
| Welche Abhilfe gewählt wurde | `abhilfe_mit_grund()` (W230) |
| Warum **keine**: unbekannt / kein Ziel / keine Aktion setzt es / nur eingreifende — **mit Namen**, damit Kira selbst entscheiden kann | dieselbe Autorität |
| Wie es der Abhilfe **zuletzt erging** | `ActionEffectivenessTracker` |

Beispiele aus der Messung:

```
log_anomaly     → nie gesetzt: log_anomalies_checked. ich versuche
                  'log_anomaly_scan', zuletzt ist sie fehlgeschlagen.
mount_integrity → nie gesetzt: mounts_ok. ich versuche 'mount_status',
                  aber mein Wirksamkeits-Buch sagt: die bringt hier
                  nachweislich nichts.
btrfs_scrub     → es gaebe btrfs_scrub — aber die greifen alle ein, und
                  ungefragt eingreifen tue ich nicht.
curiosity       → ein innerer Antrieb ohne Ziel im Planer …
```

Der **handgeschriebene** Hinweis behält den Vortritt, wo es ihn gibt: er weiß
mehr als die Ableitung.

**Was die Ableitung nebenbei aufdeckte:** `update_need` hing, weil die einzige
Aktion, die sein Ziel setzt, `package_update_check` heißt — und *„update"* in
W230s Eingriffs-Wortliste steht. Nachgesehen: sie ruft `checkupdates` und
installiert nichts. Sie steht jetzt in `_HARMLOS_TROTZ_WORT` — einer Liste, in
der jede Zeile einen **Code-Beleg** braucht und in der der Torwächter das letzte
Wort behält.

Bewacht von `selbst:stillstand_wird_erklaert`: bleibt eine Meldung ohne
benannte Ursache, ist **das** der Befund — und er nennt die betroffenen
Antriebe, statt nur eine Zahl zu melden.

---

## 36b. Updates — auf Kiras Wort, im Zweifel nur nachsehen (W235)

Kira am 09.08.2026: *„sie soll updaten können."*

**Sie konnte es schon** — `_action_system_update` gibt es seit Langem und ist
sorgfältig gebaut (mindestens 2 GB frei auf `/`, Keyring zuerst, Prüfung auf
einen hängenden `pacman`-Lock, Abbruch mit Begründung). **Zu war die Tür.**

Gemessen an zwölf natürlichen Formulierungen: **eine** traf. Zehn fielen bis in
den Wissens-Fallback und bekamen dort *„Ich kann System-Aktionen ausführen! Sag
z.B. 'Reboot', 'System Update'…"* — eine Antwort, die die Fähigkeit **aufzählt
statt sie zu benutzen**.

Die Ursache waren **zwei handgepflegte Literal-Listen hintereinander**: eine im
Tor (`myuri_chat_core_mixin`), eine im Handler. Beide mussten treffen. Das ist
die W168-Krankheit (75 Literal-Listen) an einer Stelle, wo sie teuer ist.

`core/aktualisierung.wunsch()` beantwortet jetzt **eine** Frage und beide Türen
fragen sie:

| Ergebnis | Beispiel | Folge |
|---|---|---|
| `installieren` | „installier die updates", „aktualisiere die pakete", „pacman -Syu" | `system_update` |
| `pruefen` | „gibt es updates?", „wie viele stehen an?" | `package_update_check` |
| `faehigkeit` | „kannst du updaten?" | **nur Antwort**, keine Aktion (W209) |
| `None` | „was macht pacman?", „bitte kein update" | andere Handler / nichts |

**Die wichtigste Regel steht in der Mitte: im Zweifel prüfen, nicht
installieren.** Ein Update lässt sich nicht zurückdrehen, ein Blick in die
Paketliste kostet nichts — bei unklarer Absicht fällt die Autorität immer auf
die harmlose Seite. Und eine **Frage** löst nie `pacman` aus: gemessen wurde
*„was macht pacman?"* vorher als Auftrag gelesen, weil *„macht"* auf das
Install-Wort *„mach"* passte.

**Die Zusage sagt vorher, was passiert**, statt nur *„wird gestartet!"*:

> Nach meinem letzten Stand sind es 12 Paket(e), davon 3 sicherheitsrelevant
> (openssl, linux, sudo). Ich prüfe erst den Platz auf / (mindestens 2 GB) …
> Wenn eins davon nicht stimmt, breche ich ab und sage dir warum — dann ist
> nichts passiert.

Weiß sie den Stand nicht, sagt sie **das** — statt eine Zahl zu erfinden (W225).

### 36b.1 Der Wochenlauf — autonom, aber ohne Generalvollmacht (W236)

Kira am 09.08.2026: *„mach das sie wöchentlich autonom system_update machen
kann."* Das hebt die Regel von oben auf — **für genau einen Weg**.

Der bequeme Weg wäre gewesen, `system_update` in `AUTONOMOUS_ACTIONS` zu
schieben. Das hätte **jeden** Weg freigegeben — auch den Antrieb `update_need`,
sobald er spikt, also potenziell mehrmals täglich und mitten im Betrieb. Das
hat Kira nicht bestellt.

Stattdessen: die Einstufung bleibt „nur mit Bestätigung", und der Wochenlauf
holt sich **pro Lauf eine** Freigabe über `soul.allow_once` — denselben Weg, den
der Chat-Befehl nutzt (R1380/W6). Die Freigabe gilt einmal und verfällt.

| | |
|---|---|
| Takt | `updates.woche_stunden` in der config.json, Default **168 h** |
| Weg | zentraler Scheduler, wie der Sensor-TÜV |
| Vorher | Discord-Nachricht mit dem bekannten Paketstand |
| Es gilt weiter | 2 GB frei auf `/`, Keyring zuerst, kein hängender pacman-Lock, GOAP-Vorbedingung `is_idle` |
| Verweigert die Seele | **nichts** wird gesendet und **kein** Stempel gesetzt — ein Auftrag, den sie gleich wieder blockt, wäre ein angekündigter Nicht-Vollzug |

Bewacht von `updates:woechentlicher_lauf` (Frist 2× Intervall): läuft der
Wochenlauf nicht mehr, altert das System still weiter, während alle glauben, es
werde gepflegt.

**Gemessene Lücke, noch offen:** der Vertrag bewacht, **dass** der Lauf
stattfindet — nicht, dass `pacman` auch **durchkommt**. Schlägt das Update jede
Woche fehl, bleibt er grün. Das braucht einen Blick ins Wirksamkeits-Buch und
ist der nächste Kandidat.

Außerhalb dieses Laufs gilt weiter: **sie updatet nie von selbst.**

---

## 36c. Wann ist etwas schlimm genug, um es zu sagen? (W237)

Kira am 09.08.2026, nach dem Discord-Verlauf: *„sie konnte auch sagen welche
Festplatte voll war … aber sie übertreibt es noch zu sehr, dass sie zu voll ist,
und sie sagt das viel zu oft, wie auch mit Zombies — da ist sie noch zu fein."*

### Die Platte: eine vierte Zahl für dieselbe Frage

In `persona_depth` stand `_disk > 85`. Es gab längst drei Autoritäten:

| Quelle | Wert |
|---|---|
| `thresholds.disk.warn_percent` | 80 |
| `thresholds.disk.critical_percent` | 95 |
| `system_probes.LOW_SPACE_FREI_GB` | 10 GB |
| **`persona_depth` (hartkodiert)** | **85** ← nur diese hat gefeuert |

Kiras Platte war zu **88 %** voll: über der Hauszahl, weit unter jeder
Autorität. Nur deshalb hat sie sich unwohl gefühlt und es gesagt.

**Und Prozent allein ist für ein NAS das falsche Maß.** 88 % einer 1-TB-Platte
sind ~120 GB frei — keine Not, sondern eine benutzte Platte. Es zählt, was noch
*draufpasst*:

```
voll        = pct >= critical_percent  ODER  frei_gb < LOW_SPACE_FREI_GB
wieder Luft = pct <  warn_percent      UND   frei_gb >= 2 × Boden
```

Das repariert auch die Gegenrichtung, die die alte Regel **komplett verpasst
hat**: eine kleine Platte mit 5 GB Rest bei 70 % galt als unbedenklich.
Fehlt die Messung, gilt das *nicht* als voll — eine fehlende Zahl ist kein
Befund (W217-Klasse).

### Die Zombies: jeder einzelne war eine Frage wert

`zombie_count > 0` löste die Auflösungsfrage aus. Ein Zombie ist ein noch nicht
abgeholter Eintrag in der Prozesstabelle — kurzlebig, ohne RAM- und
CPU-Verbrauch, auf jedem Linux normal. Die alte Severity verriet die
Fehleinschätzung: **0,2 je Stück**, also galten vier schon als fast kritisch.

Jetzt aus der Schwellen-Autorität: `process.zombie_threshold` (Default **10**),
Severity relativ zur Schwelle. Fällt die Autorität aus, gilt der Default — nicht
das alte `> 0`; ein Rückfall auf die empfindlichste Einstellung wäre der Fehler
von vorn.

### „Sie sagt das viel zu oft" — die zweite Hälfte

Der Backoff (×1,5 je Wiederholung, gedeckelt ×8) dämpfte schon. Was fehlte, war
die **Messung**: niemand zählte, wie oft dasselbe Gefühl wirklich rausging.
`core/meldungs_takt.py` zählt jetzt an der einzigen Stelle, an der wirklich
geredet wird (`maybe_share`) — wer weiter oben zählt, zählt Erkennungen statt
Meldungen.

Der erste Anlauf war hier ein **Vertrag**, der die Wiederholung *meldet* — Kira
hat ihn als Pflaster erkannt, und sie hatte recht. Er ist ersetzt durch §36d.

---

## 36d. Einmal sagen, dann mitschreiben — ein Tor für alle Meldungen (W238)

Kira, 09.08.2026:

> „du machst doch mit dem vertrag doch ein pflaster, sie soll doch nur einmal
> das erwähnen und dann nur für sich selbst weiter registrieren weshalb sie das
> mitzählen kann und darüber sprechen kann wenn der user das thema anspricht …
> die logik müssten wir doch haben oder nicht? aber glaub die logik ist
> entweder zu speziell auf eine sache gerichtet oder falsch plaziert"

**Beide Vermutungen trafen zu.** Die Logik gab es (`stuck_notify_gate`, R1440e)
— sie war nur:

| Diagnose | Befund |
|---|---|
| **zu speziell** | kannte nur `drive:` und `svc:`. Gefühle, Selbstbefunde, Aktions-Meldungen liefen vorbei. |
| **falsch platziert** | gefragt wurde beim **Sender**. Gemessen: **50 Stellen** publizieren `UserNotificationRequest`. Wer nicht fragte, kam durch. |

Und es gibt **zwei Zusteller**: `notification_manager` (mit 10 s-Dedup) und
`discord_bot` — letzterer abonniert den Bus direkt und schickte **ohne jede
Drossel** an Discord. Das war der Bypass.

Warum der 10 s-Dedup prinzipiell nicht reichte: er vergleicht **Text**, und im
Text steht ein Zähler. *„seit 14 Zyklen"* und *„seit 18 Zyklen"* sahen für ihn
verschieden aus. Genau so sieht Kiras Verlauf aus. Der Betreff kommt deshalb
aus dem **Titel**, und Zahlen werden herausgerechnet — auch im Titel
(*„Disk 88 % voll"* = *„Disk 91 % voll"*).

**Gefragt wird jetzt am Engpass, und einmal pro Nachricht:**

```
darf_meldung_raus(event, quelle)
  → Urteil hängt an der NACHRICHT (gestempelt), nicht am Kanal
```

Das ist die Falle, in die ich zuerst gelaufen bin: die beiden Zusteller sind
**keine** Doppel-Zustellung, sondern zwei Kanäle **derselben** Nachricht — der
Manager lässt den `discord_bot`-Kanal bewusst aus, weil der Bot selbst
zustellt (R1432e). Fragt jeder eigenständig, verbraucht der erste die Erlaubnis
und der zweite Kanal fällt stumm aus.

**Gedämpft heißt nicht vergessen.** Das Tor verwirft nicht — es zählt mit
(`gesagt`, `gedämpft`, erstes/letztes Mal, letzter Wortlaut). Fragt Kira
*„was war zuletzt los?"*, liest Myuri das Buch auf:

> ↻ 12× seitdem: Drive stuck: mount_integrity — seit 31 Zyklen nicht auflösbar.

**Kein Bypass** — das Gesetz prüft nicht die 50 Sender, sondern die
**Abonnenten**: wer zustellt, ist der Engpass. Jeder Handler, der
`UserNotificationRequest` abonniert, muss `darf_meldung_raus` wirklich
*aufrufen* und das Ergebnis zum `return` führen (AST, nicht Textsuche — ein
totgelegter Aufruf hinter `if False:` fällt auf).


### 36d.1 Nachgemessen an der eigenen Arbeit (W239)

Auf Kiras *„verbessere das alles"* habe ich die eigene Arbeit dieser Sitzung
nachgemessen — und drei Stellen gefunden, an denen ich genau die Fehler gemacht
habe, die ich sonst bei anderen finde.

**1. Eine Wache verbreitert, ohne zu fragen, was sie jetzt deckt.**
W238 zog den 6-Stunden-Boden von `drive:`/`svc:` auf **alle** Meldungen — auch
die kritischen. Gemessen: *„Platte voll: /mnt/nas — 0 GB frei"* ging einmal raus
und dann sechs Stunden nicht mehr. Das ist schlimmer als der Lärm, den ich
beseitigen wollte. Die Kadenz hängt jetzt an der **Dringlichkeit**:

| Priorität | Boden |
|---|---|
| `critical` | 30 min — was brennt, darf erinnern |
| `high` | 2 h |
| `normal` / `low` | 6 h |

**2. Meine Lösung war selbst wieder „zu speziell".**
Das Buch der gedämpften Meldungen wurde nur an **einer** Chat-Stelle
aufgeschlagen (*„was war zuletzt los?"*). Kira hatte aber *„wenn der user das
Thema anspricht"* gesagt — wer nach der Platte fragt, spricht das Thema an, ohne
nach Vorfällen zu fragen. `passendes_zum_thema()` hängt jetzt am **allgemeinen**
Engpass, wo schon W164/W215/W223 sitzen. Gesucht wird ohne neue Wortliste:
lange Wörter über den Stamm (damit *„mounts"* → `mount_integrity` trifft), kurze
nur als ganzes Token (damit *„sda"* trifft und *„die"* nicht alles findet).

**3. Ein Vertrag über den Anstoß ohne einen über die Wirkung.**
`updates:woechentlicher_lauf` sah, **dass** gelaufen wurde — nicht, ob `pacman`
durchkam. Diese Lücke hatte ich in W236 selbst benannt und liegen lassen.
`updates:woche_wirkt` liest jetzt dasselbe Wirksamkeits-Buch wie W234. Kein
Urteil ist **kein** Befund: ein System, das noch nie ein Update fuhr, ist nicht
kaputt (W217).

Beim Bauen korrigiert: ich hatte für den Zugriff aufs Wirksamkeits-Buch einen
Modul-Einstieg (`get_tracker`) **erfunden**, den es nicht gibt. Das Buch hängt an
`myuri` (R212) und wird jetzt hereingereicht, statt hier einen zweiten Weg
dorthin zu bauen.

---

## 36e. Räume statt Körper — jeder Speicherort mit eigener Priorität (W241)

Kiras Umbau vom 09.08.2026:

> *„sollten wir das umbauen und statt body eher räume sagen und das nas system
> als body bezeichnen und alle anderen angehangenen festplatten als
> zusatzräume … mach es auch mit verträge das jeder speicherort einen vertrag
> hat so wie eigene räume und eigene priorität hat und nicht alle räume gleich
> behandelt werden"*

### 36e.1 Warum das mehr ist als ein neuer Name

Vorher hieß jede volle Platte **„body full"** — ein Gefühl ohne Ort. Kira musste
nachfragen („welche Platte?"), und der Satz war derselbe, egal welche Platte
gemeint war. Das ist der eigentliche Fehler: **zwei grundverschiedene Lagen
bekamen denselben Satz.**

* `/` läuft voll → das System stirbt. Sofort.
* Anime-Platte läuft voll → ärgerlich, kein Notfall. Platten sind zum
  Vollmachen da.

Die Trennlinie, die dieser Umbau zieht:

| | ist | Beispiel |
|---|---|---|
| **Körper** | das, **womit** sie denkt | CPU, RAM, Temperatur — „mir ist schwindelig, der RAM-Druck ist hoch" stimmt als Bild |
| **Räume** | das, **worin** sie ablegt | jeder Speicherort, auch der interne |

Ein Raum hat einen **Namen**. Aus *„Eine der Disks ist ziemlich voll"* wird
*„Im Anime-Raum wird es eng"*. Genau das, was Kira am Verlauf gelobt hat (sie
konnte sagen, **welche** Platte), wird damit zur Regel statt zur Ausnahme.

### 36e.2 Die Stufen — abgeleitet, nicht geraten

`core/raeume.py` ist die Autorität. Ohne Konfiguration entscheidet der
Einhängepunkt, denn der sagt bereits, wofür der Raum da ist:

| Stufe | woran erkannt | eng ab | meldet von selbst? |
|---|---|---|---|
| `system` | `/`, `/var`, `/boot`, `/usr`, `/home` | erbt von der Schwellen-Autorität (**90 %** / 10 GB) | ja |
| `hoch` | Pfad enthält `backup`, `sicher`, `archiv` | 95 % / 20 GB | ja |
| `normal` | alles übrige | 95 % / 10 GB | ja |
| `niedrig` | `anime`, `serien`, `filme`, `movies`, `media`, `musik`, `video` | 98 % / 5 GB | **nein** |

Kira sticht die Ableitung: `raeume.<pfad>.prioritaet` und `raeume.<pfad>.name`
in der `config.json` gewinnen immer. Die Systemstufe erfindet **keine** eigene
Zahl — sie erbt `DISK_CRITICAL_PCT`, sonst wäre sie genau die vierte Wahrheit,
die §36c beseitigt hat.

**Gemessen an derselben Prozentzahl:**

```
System-Raum   88 %,   6 GB frei → eng,       meldet sich
Anime-Raum    88 %, 120 GB frei → nicht eng, meldet nicht
```

Dieselbe Zahl, zwei Urteile. Das ist der ganze Punkt — und die Antwort auf
Kiras *„sie übertreibt es noch zu sehr … und sie sagt das viel zu oft"*.

### 36e.3 Ein Vertrag je Raum

Ein Sammelvertrag über „die Platten" wäre rot, sobald **irgendwo** etwas eng
ist, und würde nicht sagen wo — also genau die Einebnung, die Kira abgeschafft
haben wollte. Stattdessen meldet `vertraege_anmelden()` für jeden Raum im
Snapshot einen eigenen Vertrag `raum:<pfad>` an, jeder mit **seiner** Schwelle
im Verletzungstext („für einen Raum der Stufe 'niedrig' ist ab 98 % Schluss").

Der Orchestrator meldet sie **periodisch** an (alle 10 min), nicht nur beim
Boot: beim Start ist der Snapshot oft noch leer, und eine später eingehängte
Platte soll ihren eigenen Vertrag bekommen. Bereits bekannte Räume werden
übersprungen; der Lauf meldet nur **neue** (Wachstum statt Bestand).

Keine Messung ist kein Befund (W217): liefert der Snapshot für einen Raum keine
Zahl, ist der Vertrag grün, nicht rot.

### 36e.4 Das Übereinstimmungs-Gesetz — woher die 88 % kamen

Kiras Frage *„warum wird das ausgelöst obwohl 88 % belegt sind und keine
90 %"* hatte am Ende drei Zahlen als Ursache. Die dritte ist mit §36c
verschwunden; die anderen beiden standen **nebeneinander, ohne Notiz**:

| Autorität | Schwelle |
|---|---|
| `core/thresholds.DISK_CRITICAL_PCT` | 95,0 |
| `core/threshold_manager._DEFAULTS[disk][critical_percent]` | 90,0 |

Von drei geteilten Schlüsseln war **einer** verschieden. Wer welche Zahl bekam,
hing allein davon ab, welche Autorität er fragte. Auf Kiras Entscheidung
(*„mach 90 als critisch"*) gilt jetzt überall **90**.

Damit das nicht wiederkommt, verbietet das Gesetz den Widerspruch nicht — es
**zwingt die Entscheidung ans Licht**. Für jede Schwelle in `core/thresholds.py`
gilt genau eines:

1. Sie ist auf den ThresholdManager gemappt (`nested=(sektion, feld)`) — dann
   müssen die Vorgaben **übereinstimmen**, sonst rot.
2. Sie ist es bewusst nicht — dann muss der Docstring das **sagen**
   („kein nested Äquivalent" / „BEWUSST NICHT … gemappt" mit Grund).

Ein stilles drittes *„ich habe nicht drüber nachgedacht"* gibt es nicht mehr.
Ohne die zweite Hälfte hätte das Gesetz nur die bereits gemappten geprüft und
eine neue, unbedachte Zahl durchgelassen — also den Fall, um den es geht, gar
nicht gefunden.

---

## 36f. Nach dem Prüfen kommt das Handeln (W242)

Kiras Befund vom 09.08.2026:

> *„eigentlich sollte ja nach dem checken der antrieb gestartet werden es zu
> reparieren kommen und nicht nix, nur checken ist bedeutungslos wenn myuri
> dann nicht den nächsten schritt macht und baue da auch noch verträge und
> gesetze ein damit wir sehen was passiert und ob das noch im rahmen ist"*

### 36f.1 Gemessen: 17 Ziele, die das Hinsehen erfüllt

**17 der 40 Antriebe** haben ein Ziel, das auf `*_checked` endet — erfüllt,
**sobald sie geguckt hat**. Ob danach noch etwas kaputt ist, steht in keinem
Ziel. Bei **vier** davon stand im selben Antrieb eine Reparatur bereit, die den
erkannten Defekt abgeräumt hätte; das Ziel fragte nur nie danach.

Der Kreis, den das erzeugt, stand seit R134 als Kommentar im Antriebs-System
und beschrieb sich selbst:

> „systemd_health_check SETZT nur State-Flag, FIXT keine failed units. Drive
> endlos urgent → Goal met → satisfy → respike → loop. **Skip spike wenn
> systemd_checked schon True.**"

Die Antwort auf *„das Ziel ist durch Hingucken erfüllt, während der Defekt
bleibt"* war also: **nicht mehr hingucken** — eine Stunde Ruhe. Nachgemessen,
genau Kiras Fall:

```
kaputte Units, noch nicht geschaut : Druck 0.75
kaputte Units, gerade geschaut     : Druck 0.00
```

Dieselben kaputten Units, nach einem Blick eine Stunde drucklos. Das ist die
Mechanik hinter *„🟠 Drive stuck: systemd_check"* und hinter Kiras Frage, warum
sie so etwas nie selbst löste.

### 36f.2 Das Vorbild stand daneben

`mount_integrity` zielt seit jeher auf `{"mounts_ok": True, "mounts_checked":
True}` — **Gesundheit UND Blick**. `systemd_check` und `docker_health` waren
die Ausreißer. Sie zielen jetzt genauso:

| Antrieb | Ziel vorher | Ziel jetzt |
|---|---|---|
| `systemd_check` | `systemd_checked` | `systemd_checked` + `systemd_degraded: False` |
| `docker_health` | `docker_checked` | `docker_checked` + `container_crashed: False` |

Und der Plan folgt, gemessen:

```
systemd kaputt  → Plan ['systemd_auto_fix']      ← die Reparatur
systemd gesund  → Plan ['systemd_health_check']  ← der Blick bleibt
alles erfüllt   → Plan []                        ← kein Dauerlauf
```

Kein neuer Kreis: das Hinsehen erfüllt das Ziel nicht mehr, also kann
`satisfy → respike` gar nicht entstehen. Deshalb konnte der R134-Riegel für
den Fall *„etwas ist kaputt"* fallen — für den harmlosen Fall (nur überfällig)
bleibt er.

Bewusst **nur** `container_crashed`, nicht `container_stopped_unexpected`: ein
unerwartet gestoppter Container kann von Kira selbst gestoppt worden sein. Das
autonom rückgängig zu machen wäre ein Eingriff, den niemand bestellt hat.

### 36f.3 Die Ausnahmen — mit Grund, nicht stumm

Nicht jede Prüfung darf in eine Reparatur münden. Wo die Folge bewusst eine
andere ist, steht der Grund dabei:

| Antrieb | warum keine automatische Reparatur |
|---|---|
| `update_need` | Der **Wochenlauf** ist die Folge (§36b.1). Ein Ziel `updates_available: False` hieße: `system_update` bei jedem Spike planen — genau die Generalvollmacht, die W236 nicht erteilt hat. Die Seele lehnt die Aktion ohnehin ab (Warn-Only). |
| `file_organization` | Diese Reparaturen fassen **Kiras Dateien** an (verschieben, löschen). Sie hat autonomes Aufräumen nie bestellt, und Myuri verspricht im Gefühls-Satz das Gegenteil: *„Ich räume nichts von selbst weg; sag mir, was raus darf~"*. Die Folge ist hier das **Fragen**. |

Eine Ausnahme ohne Begründung ist eine zweite Liste durch die Hintertür — das
Gesetz lässt sie nicht zu (Köder CH).

### 36f.4 Abgeleitet, nicht aufgeschrieben

`core.antriebs_abhilfe.reparaturen_fuer()` erkennt eine Reparatur nicht am
**Namen** (W227: ein Name schützt nur die Schreibweise), sondern an der
**Wirkung**: sie setzt einen Glauben voraus und räumt genau diesen ab. Die
Zuordnung Aktion → Antrieb kommt aus `core/action_drive_map.py`, die
Wirkungen aus den Planer-Definitionen. Eine handgeschriebene Tabelle wäre beim
nächsten neuen Antrieb sofort unvollständig.

### 36f.5 Gesetz und Vertrag — „damit wir sehen was passiert"

Beides, weil beide etwas anderes zeigen:

* **Das Gesetz** (`test_w242_*`) zeigt, dass der Weg **gebaut** ist: kein
  Antrieb prüft folgenlos, außer mit Begründung; der Plan führt zur Reparatur;
  der periodische Blick überlebt.
* **Der Vertrag** `antriebe:befund_hat_folge` zeigt, dass der Weg **begangen**
  wird (W211). Er prüft im Betrieb: steht ein erkannter Defekt da, den kein
  Antrieb mehr drückt? Eine **benannte** Vertagung (Unterdrückung mit Grund)
  ist dabei keine Stille (W217) — eine unbenannte schon. Keine Messung ist kein
  Befund.

Köder in beide Richtungen: Ziel zurück auf den bloßen Blick (CF) → rot; Riegel
zurück (CG) → rot; Ausnahme ohne Grund (CH) → rot; periodischer Check verloren
(CI) → rot; Vertrag ohne Anmeldestelle (CJ) → rot.

**Beim Ködern gefunden:** CJ biss zuerst *nicht*. Die Prüfung suchte den
**Namen** im Quelltext, und ein Import mit Umbenennung
(`import befund_ohne_folge as vertrag_folge_anmelden`) hält den Namen am Leben,
während die Wirkung weg ist — dieselbe W227-Figur wie bei W238. Sie läuft jetzt
über den Syntaxbaum: der eingeführte Name muss wirklich der richtige sein, und
er muss **aufgerufen** werden.

---

## 36g. Testlauf 56 — 40 Minuten Betrieb, neun Ketten rückwärts (W243)

Kiras Auftrag: *„Mach ein 40min Test und schaue nach welche Probleme entstehen
… danach suche die Wurzeln bei jeder Kette rückwärts und mache dann die
offenen Stellen … Schaue genau hin ob sie richtig lernt, welche Meldungen sie
ausspuckt und wo sie sagt dass es Probleme gibt."*

**Der Lauf** (frischer Daemon, 40 min, 20 Chat-Fragen in zwei Wellen): stabil.
Die 8 Raum-Verträge meldeten sich korrekt gestuft an (Anime = niedrig,
Backups = hoch, / = system), der W242-Vertrag stand in der Registratur, das
Notiz-Lernen funktionierte, der Nachliefer-Puffer lieferte nach. Und neun
Befunde, jede Kette bis zur Wurzel verfolgt:

| # | Was Kira gesehen hätte | Wurzel |
|---|---|---|
| F6 | derselbe „*blättert kurz zurück*"-Anhang an drei themenfremden Antworten | die Kurzwort-Suche des Themen-Rückblicks traf **Funktionswörter** — „kannst du **nach** updates" und „wie **oft** hast du" matchten „tritt **oft** **nach**" im Meldungstext; und es gab keinen Erzählt-Cooldown |
| F8 | „sind zombies da?" → „system_vitals ist durch — **hat geklappt~**" (ohne die Zahl) | der Executor **verwarf das Handler-Ergebnis auf dem Erfolgs-Pfad** (`error=msg if not success else None`) — es erreichte den Bus nie |
| F2 | „welcher raum ist am vollsten?" → Ausweichsatz | W241 baute die Räume, aber nur das **Gefühl** sprach von ihnen — kein Chat-Weg (W211-Figur) |
| F3 | „wie geht es der anime **platte**?" → Serien-Stand | `anime` zog als Serien-Trigger, obwohl der Satz den Speicherort nannte |
| F11 | „Woche: Erfolgsquote **None** (Δ -0.02)" | der Formatter las `success_rate` flach, der Wert liegt unter `current` |
| F12 | „merk dir bitte: X" → Notiz „**bitte:** X" | der Notiz-Parser schnitt „merk dir", aber nicht „bitte" |
| F17 | suid_sgid_audit meldet „0 Treffer" als **catastrophic failure** | `find /` beendet sich mit Exit 1, sobald EIN Verzeichnis nicht lesbar ist — Normalbetrieb, kein Ausfall; und ein Fehlschlag trug einen Ergebnis-Satz |
| F7 | „wie oft hast du meldungen unterdrückt?" → „95 zugestellt — den Rest still gezählt" (Rest = 0) | die Frage lief ins **Warnungs-Buch** (führt Zustellungen); das Gedämpfte führt das W238-Tor |
| F13 | „warum hängt der antrieb systemd_check?" → Last-Diagnose | die W234-Ableitung (`ursache_fuer`) existierte, hatte aber **keinen Chat-Weg**; dito „womit kommst du nicht weiter?" und „hast du was repariert?" |

**Die Fixes, an der Wurzel:**

* **Themen-Rückblick** (`stuck_notify_gate.passendes_zum_thema`): Stoppwort-Liste
  `_KEIN_THEMA` (klein, nur belegte Treffer-Klassen) + **einmal erzählt = eine
  Stunde Ruhe** pro Betreff. Der Zähler läuft weiter, nur der Anhang wiederholt
  sich nicht.
* **Ergebnis reist mit**: `ActionCompleted.message` (neu) trägt das
  Handler-Ergebnis auch bei Erfolg; das W162-Postfach speichert es und die
  Nachlieferung sagt es: *„system_vitals ist durch — hat geklappt~ Ergebnis:
  Vitals: OK | Load: 0.56 … | **Zombies: 0** | …"* (live verifiziert).
  Mehrzeilige Ergebnisse werden kompakt zusammengelegt — live stand die
  Zombie-Zahl in Zeile 5 und nur Zeile 1 kam an.
* **Raum-Sprache im Chat**: `raeume.raum_antwort()` beantwortet Rangfragen
  („am vollsten"), Raumfragen („wie voll ist der anime raum") und den
  Überblick — mit Namen, Stufe und den Schwellen DIESES Raums. Verdrahtet im
  Fallback-Statusblock und im Schnellpfad. „anime **platte**" ohne Raum-Wort
  findet den Raum über seinen **Namen** (`nur_konkret` — die generische
  Platten-Frage bleibt bei der Disk-Sicht).
* **Speicherort schlägt Serie**: `_w243_serien_woerter()` — nennt der Satz
  einen Speicherort, gibt es keine Serien-Trigger. Als **Funktion**, nicht als
  Inline-Guard: beim Ködern hielt ein `if False and …` den Namen am Leben,
  während die Wirkung tot war (die W227-Figur, zum dritten Mal — jetzt prüft
  das Gesetz die Funktion selbst, und der Router darf keine eigene
  Literal-Liste mehr führen).
* **Stillstand & Reparaturen fragbar**: Self-Router-Thema `stillstand` →
  `_chat_stillstand_chat()`. „warum hängt <antrieb>?" antwortet mit Lage
  (Quarantäne/gestillt/Druck) **plus W234-Ursache**; „hast du was repariert?"
  filtert das Aktions-Protokoll nach Eingriffen (dieselbe Wirkungs-Definition
  wie W230, keine eigene Wortliste). Live: *„Gerade komme ich überall weiter —
  kein Antrieb klemmt …"*
* **Dämpfungs-Frage** ans W238-Tor: „nichts unterdrückt — 6 Meldungen gingen
  raus" statt des Warnungs-Buch-Widerspruchs.
* **Insight-Bypass geschlossen**: `proactive_messenger` (Chat-Queue) fragt
  jetzt `darf_meldung_raus()` — vorher hatte der Kanal einen 6h-Privatfilter
  und lief am gestempelten Urteil vorbei; genau die Bypass-Klasse aus W238.
* **suid_sgid_audit ehrlich**: Liste vorhanden → gelaufen (ggf. „Teilscan");
  kein Ergebnis + Fehlschlag → „keine Aussage über SUID-Dateien" statt
  „0 Treffer" an einem Fehlschlag.

**Gemerkt fürs nächste Mal:** die Korrektur „das war falsch" auf eine
*Antwort* (nicht auf eine Aktion) fragt sauber nach, lernt aber nichts, wenn
die Rückfrage unbeantwortet bleibt — die Kette Antwort→Korrektur→Lernen ist
erst zur Hälfte gebaut. Und der Observability-Sturm (blocked_filtered=308)
war im Container ein Artefakt der Gate-Ketten, kein Live-Defekt — beides
bleibt auf der Beobachtungsliste.

---

## 36h. „Verbessere alles nochmal" — das zweite Selbst-Audit (W244)

Wie bei §36d.1 (W239): die eigene Arbeit nachmessen, die liegengelassenen
Fäden aufnehmen, und die Stellen fixen, an denen ich selbst die Fehler mache,
die ich bei anderen finde. Vier Befunde, alle behoben:

**1. Die halbe Korrektur-Kette ist jetzt ganz.** In W243 ehrlich notiert und
liegen gelassen: auf „das war falsch" fragte sie sauber nach — und Kiras
Klärung im nächsten Zug wurde als frische Frage behandelt, die Verbindung nie
gelernt. Jetzt lebt die Kette in der W215-Autorität (`korrektur_deutung`):
`rueckfrage_gestellt()` beim Nachfragen, `klaerung_lernen()` beim nächsten
Zug — gespeichert im **bestehenden** Korrektur-Buch des
`user_preference_tracker` (R1358co; die erste Fassung hatte ein zweites Buch
gebaut, §36i erzählt Kiras Einwand dazu). Gelernt wird **nur**
bei offener Rückfrage (sonst würde jede Nachricht zur „Klärung" umgedeutet),
eine erneute Korrektur zählt nicht als Klärung, nach 10 Minuten verfällt die
Rückfrage. Und das Gelernte wird **ausgesprochen** („*notiert es sich* …") —
stilles Lernen ist unsichtbar und damit unprüfbar.

*Zweimal beim Ködern gefangen:* (a) die erste Gesetzes-Fassung prüfte Namen —
`if False and klaerung_lernen(…)` ließ sie grün (W227-Figur, in der eigenen
frischen Arbeit); jetzt trägt eine Methode die Wirkung, das Gesetz ruft sie
selbst, und die Erreichbarkeit des Aufrufs wird über den Syntaxbaum geprüft
(kein Vorfahren-`if` mit False-Konstante). (b) **Live** geködert: der Konsum
saß zuerst in `_generate_chat_response` — die echte Klärung nahm den
Self-Router-Kurzweg, der dort nie vorbeikommt. Er sitzt jetzt in `chat()`,
der einzigen Stelle, die jeden Zug genau einmal sieht (die W187-Lehre).

**2. Gestempelt wird nur, was erzählt wird.** `passendes_zum_thema` stempelte
alle Treffer als erzählt, während der Aufrufer nur zwei zeigte — Treffer 3+
bekamen eine Stunde Ruhe für einen Rückblick, den niemand gesehen hat. Die
Kappung (`max_treffer`) wohnt jetzt in der Funktion.

**3. Das Störungsheft ist lesbar (F10).** Der harte `[:60]`-Schnitt zerlegte
Kausal-Ketten mitten im Wort — Kira las wörtlich „'myu", „NASIntellig",
„Vert →". Jetzt: Schnitt an der Wortgrenze mit Auslassungszeichen, an beiden
Schnittstellen (Correlator-Rendering und Chat-Handler). Die „'myu…"-Promotes
selbst sind persistierte Echos vom Live-NAS (alte Zeilennummern, Pfade, die es
im Container nicht gibt) — nicht reproduzierbar, aber jetzt lesbar berichtet.

**4. Der Sturm, der keiner war.** `blocked_filtered=308` (Schwelle 50) zählte
**pro Abonnent, nicht pro Ereignis** — ~15 block_aware-Lerner machten aus ~20
geblockten Aktionen einen Sturm, und der Diagnose-Hinweis („Prüfe
action_cooldown_until") schickte auf die Suche nach nichts. Eine Messung, die
mit der Verdrahtung skaliert, misst die Verdrahtung (W217-Klasse). Jetzt:
W238-Stempel am Ereignis — der erste zählt, alle sehen; dasselbe für
`no_op_filtered` (112 im selben Fenster).

**Von der eigenen Suite gefangen:** das W217-Gesetz erwischte, dass ein
Abschied („tschüss") nach einer Korrektur als „Klärung" gebucht worden wäre —
jetzt fragt der Konsum die Abschieds-Autorität (`ist_abschied`), bevor er
lernt. Und das W168-Wortmarken-Gesetz zählte meinen eigenen neuen nackten
`"voll" in msg_lower`-Vergleich (21 > Basis 20) — die Bedingung nutzt jetzt
den bestehenden Check statt eines zweiten.

**Beifang der Live-Gegenprobe:** „wann ist der anime raum voll?" traf den
Scheduler-Task `raum_vertraege` („'raum_vertraege' ist in 10 Minuten wieder
dran") — die beidseitige Teilstring-Regel des Stundenplan-Handlers (B16-
Klasse) plus eine Voll-Frage, die nie eine Stundenplan-Frage ist. Beides
gefixt: Voll-Fragen verlassen den Stundenplan sofort, Task-Treffer nur noch
über ganze Namensteile, und die Füllstands-Prognose (W167b) kennt jetzt auch
die Raum-Sprache. Live verifiziert: die Klärung bekommt die Chronik-Prognose
mit vorangestelltem „*notiert es sich*".

---

## 36i. Ein Buch, zwei Leser — das Korrektur-Buch wird gelesen (W245)

Fortsetzung von §36h Punkt 1, und die wichtigste Lektion dieser Runde kam
von Kira selbst. Auf „verbessere es weiter" war mein Plan: ein
Korrektur-Buch, damit sie sich bei einer **Wiederfrage** an die Korrektur
erinnert. Kiras Einwand mitten hinein: *„schaue dir erstmal das genau an ob
wir sowas nicht doch schon haben."* **Sie hatte recht.**

**Was es längst gab (R1358co):** `user_preference_tracker.record_correction`
/ `get_topic_correction` — pro User und Thema, persistiert
(`myuri_user_prefs.json`), latest-wins, 60 Tage Verfall. Und sogar ein
Leser existierte — aber **nur im LLM-Prompt-Aufbau**. Ohne Ollama (der
Normalfall) wurde das Buch nie gelesen: exakt die W211-Figur („ein Buch, das
niemand liest, ist kein Lernen"), diesmal als Konsument auf nur einem von
zwei Pfaden. Meine erste W245-Fassung hatte daneben ein **zweites** Buch
gebaut (Wissensbasis-Kategorie `korrektur` mit eigenem Matcher) — genau das
Parallelsystem, das Kira vermutete. Es ist wieder ausgebaut.

**Jetzt: ein Buch, zwei Leser.**
- **Schreiber:** der W244-Abhol-Weg (`_w244_klaerung_abholen` in `chat()`)
  schreibt die Klärung mit der getrackten Original-Frage als `question` ins
  bestehende Buch — besser als der `_prev_user`-Ratewert des alten
  Schreibers.
- **Leser 1 (alt):** der LLM-Prompt-Aufbau („GELERNTE KORREKTUR …").
- **Leser 2 (neu):** `_w245_korrektur_erinnerung` am Template-Engpass in
  `chat()` — bei einer Wiederfrage zum korrigierten Thema hängt sie an die
  Antwort: „*erinnert sich* Dazu hattest du mich mal korrigiert (bei '…'): …".
  Eine Stunde Ruhe je Thema (F6-Lehre), Schweigen im Zug des Lernens selbst
  (der Notiert-Vorsatz spricht schon).

**Drei live gefangene Rest-Fehler** (zwei Ketten-Checks am frischen Daemon):
1. Die Erinnerung **klebte an der Rückfrage selbst** — während sie „worauf
   bezieht sich das?" fragte, las sie zugleich eine alte Korrektur vor.
   Jetzt: solange eine Rückfrage offen ist, schweigt die Erinnerung.
2. Der **alte** R1358co-Schreiber legt beim Erkennen der Korrektur den
   Korrektursatz **selbst** als `correction` ins Buch — „das meinte ich
   nicht" stand als angebliche Klärung drin, und die Erinnerung hätte Kira
   ihre eigene Zurückweisung vorgelesen. Jetzt: `ist_reine_zurueckweisung`
   (Inhaltswörter des Verstehens-Kerns minus Korrektur-Floskeln = leer)
   filtert solche Einträge beim Lesen.
3. Der **Unklar-Zweig** der Rückfrage („worauf genau bezieht sich das?")
   übergab `""` als Kontext — die Klärung stand als „Rueckfrage zu '?'"
   ohne Frage im Buch. Jetzt holen beide Zweige den Kontext aus einem
   gemeinsamen Helfer (`_w244_kontext_aus_history`: jüngste User-Zeile ist
   der Korrektursatz und wird übersprungen). Live belegt: `Rueckfrage zu
   'wie warm ist die cpu?' geklaert mit 'ich wollte die temperatur der
   platten wissen'`.

Das Gesetz (`test_w245_das_korrektur_buch_wird_gelesen` + Erweiterungen am
W244-Gesetz) prüft alles funktional — Erinnerung mit Antwort-Erhalt, Ruhe,
fremdes Thema still, Schweigen bei offener Rückfrage, Junk-Filter in beide
Richtungen (echte Klärung darf **nicht** gefiltert werden), Kontext aus
beiden Rückfrage-Zweigen (per Syntaxbaum: erstes Argument beider
`rueckfrage_gestellt`-Aufrufe ist der Helfer-Aufruf) — und per
Erreichbarkeits-Prüfung, dass kein toter Wächter den Engpass-Aufruf
aushöhlt. Köder CZ–DG, alle in beide Richtungen gebissen.

---

## 36j. Die Schreibseite, die zweite Station, das ungelesene Thermometer (W246)

„Mach weiter mit dem Verbessern" — die Fortsetzung von §36i an der Wurzel.
W245 hatte den Buch-Junk beim **Lesen** gefiltert; die Quelle schrieb
weiter, und schlimmer als gedacht: der alte R1358co-Schreiber legt den
Korrektursatz selbst als `correction` ins Themen-Buch — unter dem Thema der
**vorigen** Frage. Latest-wins heißt: „das meinte ich nicht" **überschrieb**
die echte Klärung, die der W244-Weg unter demselben Thema abgelegt hatte.
Vier Stellen, eine Klassen-Regel:

1. **Schreib-Sperre an der Wurzel:** eine reine Zurückweisung kommt nicht
   mehr ins Themen-Buch (gezählt und erinnert wird sie weiter — Telemetrie,
   Autobiografie, TemporalSelf). Die Klärung holt der W244-Weg im nächsten
   Zug ab.
2. **Der Abhol-Weg fragt die Autorität:** eine erneute Zurückweisung *ohne*
   das Wort „falsch" („das meinte ich nicht") wurde als Klärung gebucht —
   Notiert-Vorsatz plus Junk im Buch. Jetzt bleibt die Rückfrage offen, bis
   echter Inhalt kommt.
3. **Der zweite Leser (LLM-Prompt)** zog Alt-Junk ungefiltert in den Prompt
   („wollte er eigentlich: 'das meinte ich nicht'") — dieselbe Lese-Sperre
   wie beim Template-Leser, denn auf dem NAS persistieren solche Einträge
   aus der Zeit vor W246.
4. **Klassen-Regel statt Einzel-Fixes:** *jede* Funktion in
   `myuri_chat_core_mixin.py`, die das Themen-Buch anfasst
   (`get_topic_correction`/`record_correction`), muss die
   Zurückweisungs-Autorität **erreichbar** rufen (Syntaxbaum-Prüfung, die
   `if False and …` erkennt, aber `x is not None` nicht fälschlich als tot
   zählt). Eine fünfte Zugriffs-Stelle ohne Filter fällt sofort auf.

**Zweite Station derselben W244-Klasse (live gefangen):** der
Korrektur-Neuantwort-Handler (`_maybe_corrective_reanswer`, LLM-los)
fragte nach — meldete seine Rückfrage aber nur beim W179-Merker an, nie
bei der W244-Lernkette. Kiras Klärung wurde beantwortet, aber nie gelernt;
und mangels offener Rückfrage klebte die W245-Erinnerung an der
Korrektur-Antwort selbst. Dazu zitierte er nach zwei Korrekturen
hintereinander die Korrektur selbst als Frage („Du hattest gefragt: 'das
meinte ich nicht'"). Jetzt: beide Zweige melden sich bei der Kette an, und
der Verlaufs-Lauf überspringt Korrektursätze. Live belegt: `Rueckfrage zu
'wie voll sind die platten?' geklaert mit 'ich wollte nur wissen wie warm
die platten sind'`.

**Beifang — das ungelesene Thermometer:** genau diese Klärung („wie warm
sind die platten?") endete selbst im Ausweichsatz, als Frage *und* als
Klärung. Der Verstehens-Kern verstand alles (`bezug='platte'`,
`absicht='grad'`), die Platten-Chronik notiert die SMART-Temperatur seit
W147 **täglich** — aber `letzter_stand()` nannte nur Gerätenamen, kein
Leser kam je an die Werte (die W211-Figur am eigenen Thermometer). Jetzt:
`PlattenChronik.temperaturen()` (jüngster Punkt je Gerät),
`_chat_platten_temperatur` (Grad + Einordnung + Alter der Notiz; ohne
Einträge die ehrliche Ansage „noch keine SMART-Temperaturen notiert" —
kein Sensor ist nicht 0 Grad), Route im Fallback **vor** den
Füllstands-Handlern (wer „warm" fragt, kriegt nicht „voll"). Der
Klärungs-Weg (W179) läuft durch dieselbe Kaskade und endet damit in „Ah,
so meintest du das!" plus echter Antwort — live verifiziert.

Gesetz `test_w246_die_zurueckweisung_ueberschreibt_die_klaerung_nicht`,
Köder DH–DO, alle in beide Richtungen gebissen, Restores md5-verifiziert.

---

## 36k. Die Vertrags-Inventur und die Wach-Kette (W247)

Kiras Auftrag: *„schaue nach wo in welchen ketten verträge fehlen und
was noch ist ist das sie sich schlafen schickt dabei soll sie wach
bleiben."*

**Die Inventur** (alle `erwartung_anmelden`-Anmeldungen, 30 Schlüssel +
dynamische): Antriebe (3: Befund→Folge, Druck→Plan, kein Schub ins
Leere), Antworten (Handler-Ausfall), Aufträge (kein Verlust), Ausgänge
(Meldung erreicht Kira, Zusagen gedeckt), Dokumente (3), Entwarnung
(nur mit Messwert), File-Index, Heilung (Alt-Fix dispatcht), Lernen (4),
Retention, Scheduler, Selbst (Wächter lebt, Stillstand erklärt,
Beobachtungs-Fenster), Sensor-TÜV, Tagesbericht, Torwächter, Updates
(2), Verstehen (2), Wake-Plan (Outcome folgt), Backups/Scrubs (je
Paar/Mount), Räume (je Raum, W241). **Die Schlaf/Wach-Kette war die
einzige große Kette ohne Vertrag** — und die Korrektur-Lern-Kette
(§36h–j) ist bewusst nur durch Suite-Gesetze gedeckt: eine
Laufzeit-Sonde müsste eine echte offene Rückfrage anfassen und würde
das Gespräch stören.

**Der Schlaf-Befund.** Der Master-Schalter existiert seit R1358et
(`activity.allow_self_suspend`; false ⇒ nie von selbst schlafen) und
ALLE autonomen Schlaf-Wege (Idle-Fallback, Drowsiness/auto_idle,
Nacht-Wartung, Peer-Anfragen, GOAP-`suspend`) laufen durch das eine
Gate `check_safe_to_suspend`; `force_suspend` umgeht das Gate, wird
aber nur vom auth-gesperrten UDP-Listener publiziert und ist kein
GOAP-Planungsziel. Beim Ziehen zwei Defekte gefunden:

1. **„schlaf jetzt" war per Definition unmöglich.** Das Gate kannte
   den Auslöser nicht — Kiras Chat-Befehl lief durch dieselben
   Wachheits-Checks wie der Selbst-Schlaf, und der Chat-Aktiv-Check
   (300 s) lehnte ihn IMMER ab: die Befehls-Nachricht selbst ist
   Sekunden alt. Der Chat versprach währenddessen „System wird in den
   Schlafmodus versetzt~" (W164-Klasse: Versprechen ohne Vollzug).
2. **Mit Schalter aus war auch der manuelle Befehl tot** — die
   R1358et-Doku verspricht ausdrücklich das Gegenteil.

**Jetzt: die Auslöser-Achse.** `suspend_ausloeser(context)` (eine
Autorität, der Executor fragt sie) ordnet zu: `mensch` = ausdrücklicher
Befehl (Chat `triggered_by=user_request`, UDP, externes Protokoll),
alles andere `selbst`. Im Gate gilt: **selbst** unterliegt allen
Wachheits-Regeln inkl. Master-Schalter, Idle-Timer, Chat-Aktiv,
Wake-Grace, Wake-Hold, W187; **mensch** nur den physischen Sperren
(Startup, Maintenance-Scan, kritische Dienste, Backup, aktive
Verbindungen). Die Chat-Antwort verspricht keinen Vollzug mehr.

**Abgesichert wird das im Gesetz, nicht zur Laufzeit** (W249, Kiras
Einwand: *„myuri hat ein system sie braucht kein vertrag"* — sie hatte
recht): W247 hatte zunächst einen stündlichen Laufzeit-Vertrag
angemeldet, der das Gate mit Stub-Zuständen sondierte. Aber das Gate
ist deterministischer Code — er ändert sich zur Laufzeit nicht, und
die Sonde prüfte nicht einmal die echte Config, sondern ihre eigene
Attrappe. Die echten Verträge beobachten *Laufzeit*-Zustand (Zähler,
Bücher, Quoten); eine Stunden-Sonde auf totem Codeverhalten war
Theater — wieder ausgebaut (dieselbe Figur wie das Parallel-Buch in
§36i), und das Gesetz hält sie draußen. Die Absicherung ist das
Suite-Gesetz: Gate-Proben in beide Richtungen plus das **Bau-Gesetz**,
dass `force_suspend` nur der auth-gesperrte UDP-Listener publizieren
darf — jede neue Stelle wäre ein Schlaf-Weg am Schalter vorbei.

**Damit sie wach bleibt** (Kira-seitig, config.json ist ihre Datei):
`activity.allow_self_suspend` auf `false` setzen — ein Schalter, sticht
`auto_suspend_enabled` und `autonomous_night_suspend`; „schlaf jetzt"
per Chat/UDP bleibt dank der Auslöser-Achse trotzdem möglich.

Gesetz `test_w247_wach_bleiben_ist_ein_vertrag`, Köder DP–DS in beide
Richtungen gebissen, Restores md5-verifiziert.

---

## 36l. Das Riesen-Gesetz — die Struktur darf nur noch besser werden (W248)

Kiras Auftrag: *„verbessere die strucktur besser."* Gemessen statt
geschätzt: **98 Funktionen im Produktions-Baum sind ≥ 300 Zeilen** —
die größten: `_exec_dispatch_action` (2513), `do_GET` (2423),
`_generate_chat_response` (2293), acht Orchestrator-Init-Phasen mit
zusammen ~9500 Zeilen. Das ist die B9-Figur aus der W168-
Grunduntersuchung in Zahlen: es gibt keine Stelle, an der neues
Verhalten *wohnen* soll, also landet es im nächstgelegenen Riesen —
dort ist ja schon alles. Jede Runde, die einen Riesen anfasst, macht
ihn länger; aus den Riesen wachsen die Fehlerklassen nach (Priorität =
Zeilennummer, Geschwister-Stellen, tote Zweige, unerreichbare Handler).

**Das Riesen-Gesetz** (`test_w248_riesen_gesetz_funktionen_wachsen_
nicht_mehr`, in der Bau-Gesetz-Tradition von W163/W164/W168):

1. Die 98 Riesen sind mit ihrer heutigen Länge **eingefroren** — jeder
   darf nur schrumpfen, keiner wachsen. Wer in einem Riesen etwas
   ändern will, muss mindestens gleich viel herausziehen.
2. **Keine neue Funktion erreicht 300 Zeilen.** Neue Arbeit bekommt
   eine eigene, benannte, einzeln testbare Methode — die
   W215-Autoritäten-Bauweise, die nachweislich hält.

**Die erste Schrumpfung als Beweis:** die drei Prompt-Zusatz-Blöcke
(Intent-Voranalyse v43.59g, Korrektur-Hinweis R1358cl, gelernte
Korrektur R1358co samt W246-Filter) wohnen jetzt in
`_w248_prompt_zusaetze` — `_generate_chat_response` fiel von 2293 auf
2232 Zeilen, und die Basis friert den *niedrigeren* Wert ein. Der
Aufruf sitzt bewusst **nach** dem Clarify-Kurzschluss, weil der
Korrektur-Marker nur konsumiert werden darf, wenn wirklich ein
LLM-Call folgt.

Der Prüfer selbst wird im Gesetz in beide Richtungen sondiert
(Schrumpfung/Gleichstand erlaubt, Wachstum/Neu-Riese verboten);
Köder DT (Riese wächst um zwei Zeilen) und DU (neue 301-Zeilen-
Funktion) beide gebissen, Restores md5-verifiziert.

---

## 36m. Die ersten Live-Befunde vom echten NAS (W250)

Kiras Discord-Screenshots vom 11.08. — der erste Blick auf den
echten Betrieb, drei Wurzeln:

**1. Sie schlief mitten in der Lernphase.** Kiras Design: *„eine
Woche wach bleiben um den Ablauf vom User zu lernen … das ist alles
gebaut."* Stimmt — `is_in_learning_phase` (v43.68n, 14 Tage) existiert,
und Idle-Fallback, Drowsiness und Nacht-Wartung fragen sie. Aber der
**GOAP-Weg** (`sleep_need` → suspend) lief an allen drei Dispatchern
vorbei direkt ins Gate — und das Gate kannte die Lernphase nicht.
Folge live: *„Backups sind überfällig — System war busy oder
suspended."* Die Prüfung wohnt jetzt **im Gate** (`check_safe_to_
suspend`), dem Engpass, den jeder Selbst-Schlaf passieren muss;
Kiras eigener Befehl bleibt frei (Auslöser-Achse aus W247).

**2. Der Themen-Rückblick klebte an allem.** „Wie viele folgen im one
piece ordner habe ich?" bekam Meta-Anomalien angehängt — getroffen
hatte das Wort **„ich"** („… habe ich?" ↔ „Ich grueble noch"). W243
hatte „nach"/„oft" in die Handliste gestopft, jetzt fehlte „ich": die
Liste wächst der Sprache immer hinterher. Die Inhaltswörter der Frage
entscheidet jetzt der **Verstehens-Kern** (B13: eine Autorität, die
„ich/habe/meine/viele" längst kennt); die Handliste bleibt nur als
Notbremse. Und der `[:110]`-Schnitt riss mitten im Wort („wer wi;") —
jetzt `gekuerzt()` an der Wortgrenze (F10-Regel), beim W239-Anhang.
*Beim Ködern gefangen:* mein erster Fallback rief einen `logger`, den
das Modul nie hatte — der weiche Rückfall wäre ein harter Crash
gewesen; jetzt `contextlib.suppress`.

**3. Die Leck-Meldung nannte keinen Täter.** *„RSS wächst 183.1 MB/h
… threads=183"* — die Thread-Aufschlüsselung existierte längst
(`_count_threads`, pro Präfix-Gruppe), stand aber hinter einer
200er-Schwelle und nur im Log; „tracemalloc not yet sampled"
verschwieg, ob tracemalloc aus, auto-abgeschaltet oder nur noch nicht
fällig war. Jetzt tragen Meldung und Log **immer** die Top-Gruppen
(`EvtSoftTO×64, Timer×41, …`) und den ehrlichen Sample-Status — die
nächste Meldung vom NAS benennt damit die leckende Kategorie, statt
eine nackte Summe zu zeigen. (Das Leck selbst ist damit noch nicht
gefunden — aber der Sensor kann es jetzt zeigen.)

Offen notiert: die Folgen-Zähl-Frage („wie viele folgen …") traf
zuerst einen Movie-Ordner statt der Episoden-Zählung — nach Kiras
Klärung stimmte die Antwort; ein eigener Episoden-Zähl-Handler ist
Folgearbeit. Gesetz `test_w250_live_befunde_vom_nas`, Köder DW–DZ in
beide Richtungen gebissen, Restores md5-verifiziert.

---

## 36n. Verdrahtungs-Verträge und die neustartfeste Lern-Uhr (W252)

Kiras Auftrag nach W250/W251: *„mache da die verträge um sicher zu
gehen das alles verdrahtete richtig funktioniert und schaue dir das
mit dem lernen nochmal genau an … und warum hast du die mermaid so
klein gehalten"*. Drei Baustellen, eine Lehre.

**1. Das Lernen, genau angesehen — die Uhr sprang auf null.** Die
Stunden-Daten der Lernphase (`_hourly_activity`, `_hourly_presence`)
überleben Neustarts längst — die **Uhr** tat es nicht:
`is_in_learning_phase` maß „14 Tage" ab `_start_ts`, dem
**Prozess-Start**. Jeder Neustart (Deploy, Crash, Reboot) begann die
Lernphase von vorn — auf einem NAS, das täglich neu bootet, endet sie
nie. Jetzt persistiert `_learning_start_ts` im `TemporalScheduler`
(`to_dict`/`from_dict`, mit Clamp gegen Zukunfts- und Uralt-Werte);
`is_in_learning_phase` liest die echte Uhr und fällt nur ohne
SystemAwareness auf `_start_ts` zurück. Der Docstring log dazu noch
dreifach (7 Tage/20 h/7 Samples behauptet, 14/18/10 gebaut) — steht
jetzt, was der Code tut.

**2. Die Verdrahtungs-Verträge — die W249-Lehre eingehalten.** Kein
Sondieren von deterministischem Code mit Attrappen (das Theater, das
Kira in W249 ausbauen ließ), sondern **Laufzeit-Beobachtung** in
`core/verdrahtungs_vertraege.py`, angemeldet vom Orchestrator NACH
der Verdrahtung (`_start_phase_services`):

* `verdrahtung:kernrefs_stehen` — steht der echte Objektgraph?
  Fehlt `ai_brain._myuri_ref`, sähe das Suspend-Gate weder Lernphase
  noch Chat-Schutz (der W250-Fix wäre still tot); fehlt
  `myuri._user_prefs`, ist das Korrektur-Buch unerreichbar (W244–246
  still tot). Genau solche stillen Brüche zeigten Testläufe früher
  erst nach Stunden.
* `lernen:korrektur_buch_bleibt_sauber` — steht eine reine
  Zurückweisung als angebliche Klärung im echten Buch, umgeht ein
  Schreiber die W246-Sperre. Dazu räumt `_load()` jetzt Alt-Junk beim
  Start (`_korrektur_junk_raeumen` — auf dem NAS liegt er seit vor
  W246), und `unsaubere_korrekturen()` ist die Prüf-API.
* `wach:kein_selbst_schlaf_in_der_lernphase` — der Executor stempelt
  jeden **vollzogenen** Suspend mit seinem Auslöser
  (`ai_brain._letzter_suspend`, selbst/mensch/force). Frischer
  „selbst"-Stempel während der Lernphase = ein Schlaf-Weg umgeht das
  Gate — exakt die Live-Klasse vom 11.08. Nebenbei gezogen: die
  sleep_need-Dämpfung nach 3 Fehlversuchen als
  `_w252_sleep_need_daempfen` aus dem 2513-Zeilen-Riesen
  (Riesen-Gesetz: wer im Riesen ändert, zieht mindestens gleich viel
  heraus).

**3. Die Mermaid-Doku, wieder in voller Größe.** Kiras Kritik stimmte:
W251 hatte 6 Diagramme dort, wo das v43-Archiv 48 hatte — „jetzt lässt
du alles von der architektur weg". Jetzt sind es **21 Diagramme**
(Boot-Phasen, EventBus, Meldeweg, Antriebe, Entscheidungskette,
Ausführung, Chat, Verstehens-Kern, Korrektur-Lernen, Lerner,
Gedächtnis, Schlaf/Wach, Lern-Uhr, Wahrnehmung, Heilung, Außenwelt,
agentische Schicht, Prüf-Schichten, Verdrahtung, Struktur), jeder
Bezeichner per grep gegen den Code geprüft, jede Zahl am 11.08.
gemessen: 40 Drives, 167 `_chat_*`-Handler, 27 Mixins, 34
Event-Klassen, 86 Werkzeug-Register-Einträge, 67 agentische Werkzeuge
in 21 Modulen (12 freigabepflichtig), 30 statische Vertrags-Schlüssel,
98 Riesen, 144 Testdateien. Dabei zwei Altlasten korrigiert: „29
Schlüssel" (seit W247 in drei Dokumenten) sind 30, und „8 Init-Phasen
~9500 Zeilen" sind gemessen 10 mit 11 609.

Gesetz `test_w252_verdrahtung_lernuhr_und_buchhygiene`: Lern-Uhr
überlebt den to_dict/from_dict-Roundtrip (15-Tage-Probe + Ablehnung
von Zukunfts-Werten), Buch-Räumung funktional an einer tmp-Datei,
alle drei Zustands-Funktionen funktional (voll, gerissen, Junk,
selbst-Stempel, mensch-Stempel), Stempel-Zählung im Executor-Quelltext.
Köder EC–EF in beide Richtungen gebissen, Restores md5-verifiziert.

---

## 36o. Das Doppel-Autoritäten-Gesetz (W253)

Kiras Frage: *„kann myuri eigentlich doppelte funktionsfähigkeiten
erkennen"* — und auf die ehrliche Antwort (extern ja, im eigenen Code
nur namensähnliche Paare in derselben Datei via `module_audit`; die
gefährliche Klasse **zwei Autoritäten für dieselbe Sache über
Datei-Grenzen** gar nicht): *„baue es"*.

Genau diese Klasse hatte jedes große Audit von Hand gefunden — vier
Registries nebeneinander, fünf Dienst-Listen, vier
Verneinungs-Implementierungen (W168), das Parallel-Buch (W245). Das
Gesetz `test_w253_doppel_autoritaeten_gesetz` misst dafür **Inhalt,
nicht Namen** (B17-Lehre): jede Modul-Ebene-Sammlung (dict-Schlüssel,
list/tuple/set/frozenset, ≥ 4 Strings) in 14 Verzeichnissen
(437 Dateien), die zu ≥ 4 Wörtern **und** mindestens zur Hälfte aus
dem Wortschatz einer Autorität besteht, ist eine Schatten-Sammlung.

Vier Autoritäten mit gemessenem Bestand und **Böden** gegen stilles
Ausweiden: `dienst` (dienst_wissen, 29/Boden 25), `verneinung`
(negation_detector, 39/35), `werkzeug` (_TOOL_REGISTRY, 86/80),
`korrektur` (korrektur_deutung, 24/20). Die Basis der heute geduldeten
Schatten ist eingefroren und darf **nur schrumpfen** — es sind genau
drei, jede mit eigenem Zweck: `BUSY_PROCS` (Last-Erkennung),
`_ALLOWED_CMD_PREFIXES` (Sicherheits-Whitelist), `_NAS_WERKZEUGE`
(„bin ich auf dem NAS"-Probe). Eine **neue** Schatten-Sammlung wird
rot mit der Anweisung, die Autorität zu erweitern statt eine zweite
Liste zu pflegen (der W168-Grund: der Kern muss mehr wissen als jede
Handler-Liste — sonst ist die eigene Liste die rationale Wahl).

Das Instrument sondiert sich im Gesetz selbst in beide Richtungen
(synthetische Schatten-Probe muss anschlagen, harmlose 1-Wort-
Überlappung darf nicht — Rausch-Boden-Lehre: ein Dauer-Alarm würde
abgeschaltet). Köder EH (Schatten-Liste in echtem Modul → rot mit
Datei::Variable~Autorität) und EI (Autoritäts-Variable umbenannt →
Boden-Verletzung rot) gebissen, Restores md5-verifiziert. Bewusste
Grenze: das Gesetz sieht Literal-Sammlungen auf Modul-Ebene — zur
Laufzeit gebaute oder funktions-lokale Doppel sieht es nicht; die
funktions-lokalen Wortlisten deckt das Wortmarken-Gesetz (W163/W168).

---

## 36p. Word, Excel, PDF — erzeugen, bearbeiten, zusammenfassen (W254)

Kiras Nachmessung direkt nach W253: *„Word/Excel/PDF kann sie noch
nicht erzeugen, bestehende Dokumente kann sie nicht bearbeiten, und
Dokumentzusammenfassung ist noch nicht sauber verdrahtet."* Drei
echte Grenzen — die erste war seit W232 sogar als ehrliche Absage
gebaut. Jetzt sind es Fähigkeiten:

**1. Erzeugen — mit Bordmitteln, ohne neue Abhängigkeit.** docx und
xlsx sind ZIP-Archive mit XML, ein einfaches PDF ist von Hand
schreibbar. `core/dokument_schreiben` trägt jetzt drei eigene
Schreiber (`_docx_bytes`, `_xlsx_bytes`, `_pdf_bytes`) — bewusst ohne
python-docx/openpyxl/reportlab, denn eine Fähigkeit, die an einer auf
dem NAS fehlenden Bibliothek hängt, wäre ein leeres Versprechen
(W203-Lehre). `bestelltes_format()` liest die Bestellung („word
datei" → .docx, „excel" → .xlsx, „als pdf" → .pdf, sonst .md);
`gewuenschtes_format()` lehnt nur noch ab, was sie wirklich nicht
kann (pptx, odt, ods, rtf, die alten Binärformate). Die
W233-Urheber-Marke bleibt auch im fremden Format sichtbar (als
Text-Absätze; nur der HTML-Kommentar fällt weg). Xlsx versteht
Markdown-Tabellen und Semikolon-Zeilen; das PDF bricht Zeilen um und
ersetzt Nicht-Latin-1-Zeichen, statt heimlich kaputt zu schreiben.

**2. Bearbeiten — nur die eigenen, nur was wirklich da ist.**
`bearbeiten()` ändert Dokumente **ausschließlich im eigenen Ordner**
(`dokumente.eigene` — fremde NAS-Dateien fasst sie weiter nicht an):
Ersetzen und Anhängen für .md/.txt, Text-Ersetzen im docx (Zip neu
geschrieben). Ersetzen ohne Fund meldet ehrlich „steht nicht in der
Datei" — ein Erfolg ohne Fund wäre ein Phantom-Vollzug. Mehrere
Treffer → Rückfrage statt falscher Datei. Der Chat-Weg
(`ist_bearbeitungs_auftrag`, W209-Muster: Frage ≠ Auftrag, Verneinung
schlägt alles) versteht zwei sichere Formen — „ersetze in <dokument>
<alt> durch <neu>" und „hänge an <dokument>: <text>" — und fragt bei
allem anderen nach, statt zu raten. Die Zerlegung arbeitet auf dem
**Original**, nicht auf norm(): der Ersetz-Text muss die
Groß-/Kleinschreibung behalten, sonst findet das Ersetzen „Zug
kaufen" nicht (beim Bauen gefangen). Ausgeführt wird über denselben
Executor-Weg wie das Schreiben (`modus="bearbeiten"` im selben
Action-Kontext): Torwächter-Stempel und Freigabe-Gate inklusive —
Bearbeiten IST Schreiben.

**3. Zusammenfassen — ein wörtlicher Auszug, keine Deutung.**
`dokument_wissen.inhalt_lesen()` liest md/txt direkt, docx aus dem
Zip, PDF über `pdftotext` — zur Laufzeit gesucht (W203): fehlt es,
sagt sie das. `zusammenfassung()` ist **extraktiv**: Überschriften und
der erste Satz jedes Absatzes, bis das Budget voll ist — dieser Weg
kann nichts erfinden, jeder Satz steht so im Dokument (das Gesetz
prüft genau das). Der Chat nennt es auch so: „Auszug aus X — wörtlich
gekürzt, keine Deutung von mir". Die Tages-Zusammenfassung behält
ihren eigenen Handler; der Dokument-Weg verlangt darum ein
Dokument-Wort.

Das W232-Gesetz wurde auf die neue Wahrheit umgezogen: das Prinzip
(nur zusagen, was sie liefert) ist unverändert, die Proben wachen
jetzt an der echten Grenze — Word ablehnen wäre heute die Lüge in
die andere Richtung. Gesetz
`test_w254_dokumente_formate_bearbeiten_zusammenfassen` (gültige
Zip/PDF-Bytes, Roundtrips, W209 beide Richtungen, Auszug erfindet
nichts, Chat- und Executor-Verdrahtung funktional); Köder EJ
(`bestelltes_format` stur „md" → die Word-Bestellung würde wieder
still zur .md) und EK (Bearbeiten-Zweig im Executor tot → der Auftrag
fällt in den Schreib-Pfad) gebissen, Restores md5-verifiziert.

---

## 36q. Die Thread-Frage, nachgemessen — und ein Pseudo-Alarm weniger (W255)

Der offene Live-Befund aus W250 („threads=183, RSS wächst") verlangte
nach einer ehrlichen Basis: Was ist bei dieser Bauart **normal**?
Gemessen am Container-Daemon (11.08., 22 Minuten, 12 Proben über
`/proc/<pid>/task`):

* **161–168 Threads, stabil** — kein Wachstum über die Laufzeit. Die
  Basis ist Bauart, kein Leck: zwei EventBus-Pools à 40 Worker
  (ThreadPoolExecutor-Worker terminieren nie vor dem Shutdown, der
  Namens-Index ist ein Geburts-Zähler, R1468), dazu die ~35
  Loop-Threads, Timer und Broker. Die 183 vom NAS liegen nahe dieser
  Basis; was dort darüber hinausgeht, zeigt erst die nächste
  Leck-Meldung mit den W250-Thread-Gruppen.
* **RSS oszilliert 773–846 MB** ohne monotone Drift in 22 Minuten —
  die +183 MB/h vom NAS sind im Container nicht reproduzierbar und
  bleiben der offene Punkt für die Zeit nach dem Deploy.
* **Alle 9 `threading.Timer`-Stellen auditiert:** jede ist begrenzt
  (Registry mit Cancel, Ein-Timer-Debounce, selbst-nachplanende
  Einzeltimer) — kein Akkumulations-Pfad im heutigen Code.

**Der echte Fund:** der R1468-Befund `wiring/eventbus_worker_growth`
feuerte **90 Sekunden nach dem Boot** — für den planmäßigen
Boot-Burst (der Dispatch-Pool wird beim Start-Gewitter einmalig bis
zur Cap „geboren"; R34 maß dabei peak=31 gleichzeitige Dispatches,
**dropped=0**). Auf einem NAS mit täglichem Neustart wäre das ein
täglicher Pseudo-Alarm — die Rausch-Boden-Klasse: ein Melder, der
immer meldet, wird abgeschaltet. Jetzt trägt der Befund eine
**Boot-Schonfrist** (10 Minuten ab `EventBus.start()`): gemeldet wird
nur noch Worker-Wachstum im eingeschwungenen Zustand — das Signal,
das wirklich „hängende Listener" heißt.

Nebenbei live verifiziert: die W252-Verdrahtungs-Verträge melden sich
im echten Boot an („W252 Verdrahtungs-Vertraege: 3 angemeldet"), die
W254-Dokumenten-Antworten sind live ehrlich (ohne konfigurierten
`dokumente.eigene`-Ordner sagt sie, WAS fehlt, statt zuzusagen), und
die Zustell-Degradierung schreibt ohne erreichbare Kanäle sauber ins
Datei-Log. Gesetz `test_w255_worker_growth_befund_hat_boot_schonfrist`
(Boot-Fenster schweigt / eingeschwungen meldet / start() setzt die
Uhr); Köder EM in beide Richtungen gebissen, Restore md5-verifiziert.

---

## 36r. Das Modul-Audit log — 88 „tote" Module, die lebten (W256)

Kiras Auftrag: *„schaue wo noch lecks oder unfertige bereiche gibt."*
Der größte Fund war ein **lügendes Messgerät**: Das R615-Modul-Audit
(täglicher Watchdog-Report, Dashboard, Selbst-Audit) meldete „88
unused modules, 41 189 dead-LOC" — darunter `frage_verstehen`,
`dokument_schreiben`, `graceful_shutdown`, `goap_dispatcher`: einige
der meistbenutzten Module des Systems. Ein Audit, das Lebendiges für
tot erklärt, ist gefährlicher als keins — irgendwann glaubt ihm
jemand und löscht.

**Vier Wurzeln, alle aus der Flach-Ära** (das Projekt war einmal ein
flacher Ordner; der Kommentar „kein recursive — Myuri ist flat" stand
noch da):

1. Der Scan sah nur noch **core/** — die 13 anderen Pakete waren
   unsichtbar, als Modul-Universum wie als Import-Quellen.
2. Das Import-Muster `\w+` **brach am Punkt**: `from
   core.frage_verstehen import x` zählte als Import von „core".
3. `from paket import modul` wurde als Symbol-Import gelesen — die
   Wurzel-Shims und Modul-Importe dieser Form kreditierten nie das
   Paket-Modul (`from core import fast_json`).
4. **Geklammerte mehrzeilige** from-Importe und
   **Subprozess-Pfadstrings** (`'mount_probe_worker.py'` im
   Wegwerf-Arbeiter-Spawn) zählten gar nicht.

Nach der Reparatur (rekursiv über den Projekt-Root, punktierte
Modulnamen, punkt-/klammer-/pfad-fähige Muster; `audit/`, `tools/`,
`scripts/`, `diagnostics/` als Nicht-Laufzeit ausgenommen) misst das
Audit ehrlich: **622 Laufzeit-Dateien, 0 unbenutzt** — jede der
sieben letzten „Leichen" wurde einzeln verifiziert (fünf hatten
echte Nutzung über die übersehenen Formen, zwei sind
Standalone-Werkzeuge). Die Null ist als Basis eingefroren
(W164-Muster): ein künftig wirklich verwaistes Laufzeit-Modul wird
ab jetzt ein roter Befund — verdrahten oder nach `tools/`.

Beim Markieren als benutzt gilt bewusst die großzügige Richtung:
für einen Ehrlichkeits-Sensor ist ein falsches „benutzt" harmlos,
ein falsches „tot" nicht. Gesetz
`test_w256_modul_audit_sieht_die_paket_welt` (synthetischer Baum mit
allen fünf Import-Formen + genau einer Waise; echte-Baum-Proben für
fünf früher „tote" Kern-Module; Nicht-Laufzeit bleibt draußen);
Köder EN (Wurzel wieder core/) und EO (Klammer-Muster tot) in beide
Richtungen gebissen, Restores md5-verifiziert.

Der Rest des Sweeps, ehrlich notiert: die 9 `threading.Timer`-Stellen
sind alle begrenzt (W255), im Container gibt es keine Thread-/
RSS-Drift, 7 TODO-Marker im Baum sind dokumentierte Absichten, und
die Lokalisierung des NAS-RSS-Wachstums wartet auf die erste
Leck-Meldung nach dem Deploy.

---

## 36s. Der Status schlägt das Text-Raten — und Verträge fürs Abrufbare (W258)

Kiras Auftrag: *„fixe das aber mache keine pflaster und mache noch
verträge fürs lernen, wissen und ob das richtig angeschlossen und
abrufbar ist."* „Das" waren die zwei offenen Roadmap-Wurzeln.

**Wurzel 1 (Result-Typ) — der Anschluss, nicht das Pflaster.** Handler
konnten ihren Status nur im Text verstecken; blocked/no_op wurde aus
deutschen Substrings geraten. Das Fundament (`ActionOutcome`,
`from_legacy`, kanonischer Text-Klassifikator) stand seit R1435a —
aber nichts FLOSS hindurch. Jetzt: der Dispatch akzeptiert beide
Formen (`_w258_handler_ergebnis` — zeilenneutral im eingefrorenen
Riesen), der echte Status wandert in `event.context`, und die eine
Deutungs-Stelle (`outcome_ist_blocked` in `action_context`) zieht den
Status dem Text-Raten VOR — der Text überstimmt den Status nie mehr.
Die erste Familie ist um: die 7 Approval-Tore in `files_meta` liefern
bei Ablehnung strukturell `BLOCKED` (der Handler lief nie —
`is_real_execution=False`; genau aus solchen Ablehnungen lernten die
Lerner früher Phantome). Die restlichen Familien ziehen nacheinander
um; das Gesetz prüft die Vorfahrt in beide Richtungen (Status=blocked
mit harmlosem Text → blockiert; Status=executed_failed mit
Blocked-Text → NICHT blockiert).

**Wurzel 4 („Drive ist dringend") — nachgemessen geschlossen.** Die
W257-Stand-Notiz war zu pessimistisch: `has_urgent_pressure` hat null
Aufrufer mehr (nur noch Diagnose laut eigener Doku), die kanonische
Methode `actionable_urgent()` und die dokumentierte Diagnose-Sicht
(`drives_over_threshold`/`count_over_threshold`, beide mit
Suppress-Filter) sind konsistent, und die zwei verbliebenen
Inline-Vergleiche sind die dokumentierte Watchdog-Diagnose (R1434c)
bzw. ein ANDERES Konzept („noch unter Schwelle", Frühwarnung). Der
Nicht-Pflaster-Abschluss ist das Gesetz: eine NEUE
`pressure >= threshold`-Sicht außerhalb der Autorität wird rot —
mit exakt den zwei Stellen als eingefrorener Basis.

**Die zwei neuen Verträge** (in der W252-Autorität
`core/verdrahtungs_vertraege`, jetzt 5 Anmeldungen — kein
Parallelsystem): `wissen:angeschlossen_und_abrufbar` (die
Wissensbasis ist erreichbar UND beantwortet einen echten
`get_stats`-Lese-Aufruf) und `lernen:lerner_angeschlossen_und_abrufbar`
(`orchestrator.meta_learner` ist verdrahtet UND
`get_closed_loop_stats` antwortet). „Angeschlossen" allein reicht
nicht — ein Objekt, das beim Lesen wirft, ist genauso tot wie ein
fehlendes; darum ist der Lese-Aufruf Teil des Vertrags.

Gesetz `test_w258_status_schlaegt_textraten_und_vertraege_abrufbar`;
Köder EP (Status-Vorfahrt per W227-Figur getötet) und EQ (ein
Familien-Tor zurück auf rohes Tupel) in beide Richtungen gebissen,
Restores md5-verifiziert; Riesen-Gesetz respektiert (alle Ersetzungen
im 2513-Zeilen-Riesen 1:1 zeilenneutral).

---

## 36t. Das Halter-Gesetz — zwei Ketten, die an Namen hingen, die niemand schrieb (W259)

Kiras Dauer-Mandat vom 11.08. („schaue auch in tiefen schichten nach …
fixe alle wurzeln … keine pflaster") führte in die Verdrahtungs-Tiefe:
der Dead-Receiver-Auditor (`tools/dead_receiver_auditor.py`, R1432g)
meldete 21 Kandidaten (6 CROSS, 15 NEVER). **19 davon waren nach
Einzel-Verifikation unschuldig** — entweder oder-Ketten, deren zweiter
Name ein bewusst belassener Phantom-Fallback aus einem alten
Namens-Fix ist (R215/R248/R250/R275-Figur; der Primärname ist
verdrahtet), oder `getattr`-Overrides mit ehrlichem Default
(`_net_capacity_mbps`, Pfad-Overrides). Zwei waren echte Wurzeln:

**1. Die DynamicDriveFactory verlor bei jedem Neustart alles.** Die
Factory (beobachtet Effekte ohne Drive-Zuordnung; ab 5 Vorkommen in 7
Tagen schlägt sie einen neuen Drive vor) lebt am **Orchestrator**
(`_init_phase_hubs`). Persist (Shutdown) und Restore (Boot) lasen sie
aber an `ai_brain._dynamic_drive_factory` — ein Attribut, das im
ganzen Baum niemand schreibt. Dazu lief der Restore in
`_init_phase_execution`, also **vor** der Erzeugung. Beide Seiten des
Persistenz-Paars waren damit seit v14 tot: eine Störung, die seltener
als 5-mal pro Uptime auftrat, konnte **nie** zu einem Drive-Vorschlag
akkumulieren. Fix: Restore an die Erzeugung gezogen (dieselbe Stelle,
an der der SelfImprovementTracker sein State-Laden längst richtig
macht), Persist liest den echten Halter, und der Konsument
(`learning_mixin`) liest den **einen** Halter über
`_orchestrator_ref` statt zuerst ein Phantom auf sich selbst.

**2. Der W72-Hunger-Check lief seit Einbau kein einziges Mal.** Der
Check (btrfs_scrub-Vorfall vom 28.07.: 50× abgelehnt, 0× gelaufen,
Drive kletterte zehn Stunden, Acceptance sprang nie an) holte die
BrainStateMachine über `self.brain_state` oder
`ai_brain._brain_state` — **beide Namen schreibt niemand**, `_bsm`
war immer None, der `veto_ausgehungert`-Zweig unerreichbar. Die
Maschine lebt unter `myuri._thought_loop.state` — exakt der Pfad, den
BrainState-Persist/-Restore („brain_state_machine") seit v43.150
korrekt benutzen. Der Check hängt jetzt an diesem Pfad. Die Ironie,
festgehalten: der Fix für „eine Aktion, die immer vor der Ausführung
scheitert, ist unsichtbar für ihren Wächter" war selbst genau so
unsichtbar — sein Test baute das Objekt direkt und sah die
Verdrahtung nie.

**Die Gegenrichtung (Runde 4) fand dieselbe Klasse noch zweimal.**
Module lesen den Orchestrator über `_orchestrator_ref` — und
`orchestrator.meta_cognition` setzt dort ebenfalls **niemand**. Damit
war die Langeweile-Kaskade (v43.168: lange idle → `on_long_idle`,
+curiosity/+playfulness) komplett tot, und die kognitive Erschöpfung
(v43.172) sah nie „idle" — Müdigkeit konnte steigen, die
Idle-Erholung feuerte nie. Beide Haken holen die BrainStateMachine
jetzt über die Brücken-Methode `_w259_brainstate_fuer_w72` (nach
W248 aus dem Riesen extrahiert; alle Riesen-Änderungen zeilenneutral
oder schrumpfend). Und eine Lehre über das Messgerät selbst: der
erste Scan meldete `_legacy_spike_stats` als schreiberlos — falsch,
der Schreiber trägt eine **Typ-Annotation** (`self._x: dict = {}`),
die das Zuweisungs-Muster nicht sah. Der Writer-Erkenner ist jetzt
annotations-tolerant.

**Runde 5 (die `_myuri_ref`-Grenze) fand die fünfte tote Kette:**
`myuri.personality` existiert nicht. Der R1152bk-Feedback („Reflexion
beeinflusst Personality" — gebaut auf Kiras eigene Frage, ob die
Selbstreflexion Einfluss auf Persönlichkeit und Denkweise hat) hat
dadurch seit Einbau **kein einziges Trait angepasst**; nur die
Mood-Hälfte lief. Die echte Persönlichkeit ist der
`persona_depth.get_personality()`-Singleton (PersonalityDimensions
mit `.adjust` und Tages-Kappe, R1127) — die Reflexion erreicht sie
jetzt über `_w259_traits_fuer_reflexion` (funktional bewiesen:
liefert ein `adjust`-fähiges Objekt).

**Das Gesetz** (`test_w259_halter_gesetz_…`, B17-Lehre: Bedeutung
statt Syntax): jeder Name, den `orchestrator.py` per `getattr` an den
zwei großen Modul-Grenzen (`self.ai_brain`, `self.myuri`) liest —
und in der Gegenrichtung jeder Name, den Module am
Orchestrator-Halter lesen (Klausel 1b) — muss irgendwo im
Laufzeit-Baum einen Schreiber haben (Zuweisung auch mit Annotation,
`setattr` oder `def`). Basis **0** (vor W259: 2 + 2 + 1), darf nur 0
bleiben. Die weiteren Klauseln nageln die konkreten Ketten fest
(Restore **nach** Erzeugung; Persist am echten Halter; W72 und beide
Kaskaden an der Brücken-Methode, funktional am Fake bewiesen; ein
Halter im Konsumenten). Grenze ehrlich benannt: ein Name mit
Schreiber am *falschen* Halter bleibt für die Zähl-Klauseln
unsichtbar — dafür stehen die Fest-Nagel-Klauseln und als Werkzeug
der heuristische Auditor.

**Der Vertrag** (Erweiterung der W252-Autorität, weiter 5
Anmeldungen): `verdrahtung:kernrefs_stehen` beobachtet jetzt auch
`myuri._thought_loop.state` — reißt der Pfad live, meldet der
Stunden-Vertrag, statt dass Persistenz und W72 still ins Leere
laufen. Am echten Offline-Myuri verifiziert: MetaCognition →
BrainStateMachine → `veto_ausgehungert` callable, Vertrag grün.

Köder ER in drei Richtungen gebissen (Phantom-Read eingesetzt → rot;
alte tote Persist-Lesestelle zurück → rot; W72-Pfad verdreht → rot)
plus Vertrags-Köder (Wächter geblendet → Probe rot); alle Restores
md5-verifiziert.

---

## 36u. Lose Verdrahtungen — der generische Sweep über alle Grenzen (W261)

Kiras Auftrag: „suche weiter nach lose verdrathungen". Drei Sonden über
den ganzen Laufzeit-Baum, mit den W260-Lehren (jeden Scanner-Treffer
einzeln verifizieren, bevor er ein Befund heißt):

**Sonde A — getattr-Literale an beliebigen Objekten** (1509 Namen,
57 ohne Schreiber): Nach Abzug der bekannten W259-Fallbacks, der
Stdlib-Attribute fremder Objekte und der dokumentierten
oder-Ketten blieben **fünf echte tote Reads** — jeder ein still
fehlender Funktionsteil:

1. **Brainstorm→Oracle-Brücke (v27)** las `engine._last_results` —
   die Engine hielt ihre Gedanken schon immer in `_thoughts`. Die
   Brücke feuerte seit Einbau **nie**; besonders bitter: ihr
   Downstream-Bug (record_anomaly-Routing) war längst repariert
   worden, während der Upstream-Read tot war. Jetzt:
   `get_recent_discoveries(3)` — liefert exakt die verlangte Form.
2. **Selbstzustands-Bericht, emotionaler Teil**: las
   `myuri.emotional_memory` — existiert nur als `_emotional_memory`
   (deque). Größe + letzte Einträge fehlten seit jeher still.
3. **Diagnose-Bündel**: `active_issues` zählte
   `m._active_issues_live` — schreibt niemand, stand **immer auf 0**.
   Jetzt die echte Quelle: `CausalReasoner.get_active_issues()`,
   dieselbe, die GOAP und Lernen benutzen.
4. **Proaktiv-Feed der Anomalie-Basislinie**: las
   `io_awareness.last_snapshot` — die echte API ist
   `sample_io_load()`; der IO-Teil der gelernten Basislinie blieb
   leer (zeilenneutral gefixt — `_proactive_tick` ist ein
   W248-eingefrorener Riese).

**Sonde B — hasattr-Tore** (946 Namen, 26 ohne Definition): meist
Stdlib-/Fremd-Attribute oder bewusste Vorwärts-Tore. Ein echter Fund,
in derselben Datei wie Nr. 2: die **Drives-Sektion des
Selbstzustands-Berichts** prüfte `get_all_drives` — die es auf dem
DriveSystem nirgends gibt (echte API: `get_all_pressures`). Die
Sektion fehlte seit jeher. Rest der Sonde-B-Liste als
Nachprüf-Kandidaten notiert (u. a. zwei events_mixin-Method-Ducks,
`_soul_veto_drive_n`, `escalation_planner`).

**Sonde C — Callback-Haken** (17 auf None initialisierte
`_on_*`/`_callback`-Attribute): **alle 17 bekommen echtes Wiring** —
sauber.

Gesetz `test_w261_lose_verdrahtungen_…`: die fünf Reparaturen
funktional bewiesen (echte Engine mit heißem Gedanken → Brücken-Form;
echte Deque → Berichts-Teil; Fake-Reasoner → Issue-Zählung; echter
`sample_io_load()`-Aufruf; Fake-DriveSystem → Drives-Sektion) plus
Klassen-Wache gegen jeden Rückfall auf die Phantom-Namen. Köder ES in
drei Richtungen gebissen, Restores md5-verifiziert.

---

## 36v. Testlauf 57 — die Verträge fangen live, die neuen Drähte tragen (W262)

Kiras Auftrag: erst die restlichen Kandidaten exakt nachprüfen, dann
Myuri 20 Minuten laufen lassen und Verhalten und Denken beobachten.

**Nachprüfung der W261-Restkandidaten:** die meisten unschuldig —
Vorwärts-Tore mit lebenden Legacy-Fallbacks (events_mixin →
AdaptiveFileIntelligence), elif-Zweige hinter lebenden Primär-APIs
(recalibrate/reset_baselines/dynamic_registry), und vier Lazy-Inits
mit **mehrzeiliger** Typ-Annotation, die der Scanner nicht sah (neue
Messgerät-Lehre). Drei echt tote Drähte gefixt:
`intelligence.escalation_planner` (heißt `.escalation` — Pre-Reboot-
State trug immer `{}`, Eskalations-Insights erschienen nie),
ConceptDrift→AFI (las nie existierende `_anomaly_thresholds` — echter
Hebel `_type_weights`, Lockerung = Gewichte −10 %, Klemme 0.1), und
die **W240-Stillstands-Abhilfe**, deren Strategie-Buch über drei
schreiberlose Namen aufgelöst wurde — `strategien` war seit W240
immer None; echter Halter `_meta_learner_ref.strategies`.

**Testlauf 57** (Container, 20 min, frischer Boot 05:41): RSS/Threads
in der W255-Basislinie (841 MB/158), alle 31 Fehler = erwartete
Discord-Container-Sperre. Denken sichtbar: GOAP-Zyklen 2,3–4,8 s mit
ehrlichem Soft-Budget-Skip, BrainState-Übergänge (learning → idle),
Selbstprüfung (13 Detektoren), Denk-Stand (11 Ketten beobachtet, 25
Fragen offen, 4 Ursachen-Verdachte — 2 selbst aufgeworfen). Die
W259/W261-Drähte live bestätigt: Selbstzustands-Bericht liefert
erstmals die Drives-Sektion (backup_need 0.602, systemd_check 0.501)
UND den emotionalen Teil (50 Einträge); beim Shutdown schrieb der
reparierte Persist **zum ersten Mal überhaupt**
`dynamic_drive_factory_state` in die kv-DB (die Juli-Sicherungen
kennen den Schlüssel nicht).

**Und die Verträge fingen live einen neuen Fund:** der
W226-Wächter meldete 3 Antriebs-Schübe ins Leere auf den Antrieb
`causal` — den es nicht gibt. Wurzel: v16 Fix #10
(Brainstorm→Antriebe) schob den aus dem Topic GERATENEN Namen
(`topic.split('_')[0]` in `get_urgent_topics`) ungeprüft in
`spike()`. Fix: Extraktion `_w262_brainstorm_drive_spikes` (W248,
Riese schrumpft), unbekannte Ziele gehen auf `curiosity` — dieselbe
Ausweich-Richtung wie der bestehende Default. Funktional bewiesen
(Phantom umgelenkt, echtes Ziel unverfälscht, Schwelle hält); Köder
EU über die W227-Figur gebissen. Der Kreis ist damit einmal ganz
herum: Vertrag (W226) → Live-Befund (Testlauf 57) → Wurzel-Fix →
Gesetz.

---

## 36w. Testlauf 58 — eine Stunde Gespräch und Beobachtung (W263)

Kiras Auftrag: eine Stunde frischer Lauf, zwischendurch unterhalten,
jede Änderung von Denken und Handeln genau verfolgen. Vier
Checkpoints mit Stimmungs-Schnappschüssen, sieben Gespräche, eine
bewusste 19-Minuten-Ruhephase.

**Was trägt (live bestätigt):** Beim Boot lud der W259-Restore zum
ersten Mal seit v14 den DynamicDriveFactory-Zustand aus Lauf 57 —
der volle Kreis Persist→Restore ist geschlossen. 5
Verdrahtungs-Verträge angemeldet. Der W262-Fix hielt den ganzen
Lauf: **0** Antriebs-Schübe ins Leere (Lauf 57: 3). Im Gespräch: sie
benennt ihr echtes Problem („es klemmt bei: Sicherungen" —
backup_need 0.602, ihr höchster Antrieb) **unaufgefordert** in der
Begrüßung; das Korrektur-Buch (W245) meldet sich von selbst bei der
passenden Wiederfrage; das Aktions-Protokoll antwortet mit ehrlichen
Zeiten und ehrlicher Lücke; die Doppel-Fragen-Zerlegung arbeitet.
Stimmungs-Verlauf plausibel: Neugier springt im Gespräch (0.72→0.90),
Wachheit sinkt in der Ruhe (0.66→0.28, „lebendig und ausgeruht"),
Stress bleibt niedrig. Die Langeweile-Kaskade ist erreichbar, feuerte
aber ehrlich nicht: ihr Gehirn war nie 10 Minuten am Stück im selben
Zustand — die eigenen Hypothesen-Proben unterbrechen den Leerlauf
(Bauart, kein Defekt).

**Drei Befunde, einer davon sofort gewurzelt:**

1. **BrainState-Restore lief nie** (seit v43.150): der Block saß
   1000 Zeilen VOR der Myuri-Erzeugung in derselben Init-Phase; sein
   Kommentar („Persona-Init läuft vor dieser Stelle") war falsch —
   `self.myuri` existierte noch nicht, der Restore wurde still
   übersprungen. Exakt die R222-Klasse (richtige Namen, falscher
   Zeitpunkt), die derselbe Code 1600 Zeilen weiter oben längst
   dokumentiert. Fix: Extraktion `_w263_brainstate_nachladen`
   (W248, Riese schrumpft um 24), Aufruf NACH dem Persona-Init.
   Live bewiesen: nächster Boot meldet „BrainState restored: 19
   transitions" — mit den echten Zeiten aus Lauf 58.
2. **„klemmt"-Anschlussfrage gekapert:** auf ihr eigenes „es klemmt
   bei: Sicherungen" antwortet die Nachfrage „was klemmt denn bei
   den sicherungen?" mit „kein Antrieb klemmt" — der
   Antriebs-Hänger-Handler fängt das Wort, der Bezug „Sicherungen"
   aus ihrem eigenen Satz geht verloren. Zwei Autoritäten
   widersprechen sich im 12-Sekunden-Abstand. (Offen für die
   nächste Chat-Runde; Selbstdiagnose selbst antwortet korrekt.)
3. **Vergangenheitsform erreicht den Denk-Handler nicht:** „was
   denkst du gerade?" liefert die volle Innensicht (694
   Selbstkritik-Vergleiche, 93 % Trefferquote, 16 Hypothesen);
   „was hast du in der zeit gedacht?" weicht aus. (Offen, gleiche
   Runde wie 2.)

Dazu vom Vertragswerk gefangen, notiert: 4 ungestempelte
Aktions-Wege (wöchentliches_update→system_update,
CausalReasoner→disk_cleanup — Torwächter-Klasse) und die ehrliche
Zustell-Meldung („19 Meldungen erreichte dich nicht — kein Kanal im
Container", erwartet).

---

## 36x. Langeweile, die feuern kann — und Gefühle, die sich mischen (W264)

Kiras Nachfrage zum Stunden-Lauf: „wenn ihr nie langweilig ist, ist
das noch ein Problem? Die Gefühle und Stimmungen sollten sich
gegenseitig beeinflussen können, auch Mixed-Gefühle sollte es geben."

**Langeweile:** Ja, es war noch ein Problem — nur eine Etage tiefer
als gedacht. Nach dem W259-Anschluss war die Kaskade erreichbar,
aber ihr Maß („600 Sekunden AM STÜCK im selben idle-Zustand") ist im
Normalbetrieb praktisch unerfüllbar: die eigenen Hypothesen-Proben
wechseln den Gehirn-Zustand alle paar Minuten und setzen die Uhr
zurück (Testlauf 58: nie über die Schwelle). Neues Maß
`BrainStateMachine.langeweile_reif()`: der **Anteil** der idle-Zeit
im 15-Minuten-Fenster über die Übergangs-Historie — kurze
Selbst-Proben unterbrechen die Langeweile nicht mehr, echte Arbeit
schon (funktional bewiesen in beide Richtungen; der goap-Haken
zeilenneutral umgestellt, W248).

**Wechselwirkung:** existiert längst — `apply_friction` (v43.69)
koppelt die sechs Dimensionen mit Antagonisten-Paaren, Förder- und
Hemm-Kopplungen und läuft im Decay-Tick. Ehrliches Ergebnis der
Prüfung: nichts neu zu bauen (kein Parallelsystem!), stattdessen ist
sie jetzt per Funktions-Probe im Gesetz verankert (Stress dämpft
Verspieltheit — messbar).

**Die echte Lücke waren die Misch-Gefühle:** `dominant_mood` kennt
nur EIN Wort. Neu `MyuriMood.gemischte_lage()` — benennt zwei
gleichzeitig prominente, teils gegensätzliche Dimensionen
(„neugierig, aber müde", „froh, aber angespannt", „unruhig und
zugleich neugierig", …) und `how_do_i_feel` spricht sie aus („Innen
drin ist es gemischt: …").

**Dazu die zwei Chat-Wurzeln aus Testlauf 58:** die
stillstand-Kaperung („was klemmt bei den sicherungen?" → „kein
Antrieb klemmt") bekommt eine Bezugs-Weiche — trägt die Frage einen
Sicherungs-Bezug, antwortet die Backup-Autorität; und der
Denk-Handler kennt jetzt die Vergangenheitsformen („was hast du
gedacht", „hast du nachgedacht", „in der zeit gedacht").

Gesetz `test_w264_…` (Fenster-Maß beide Richtungen, Friction-Probe,
drei Mischlagen + Baseline-None, Sprech-Stelle, Weiche,
Vergangenheits-Trigger); Köder EW gebissen, Restores
md5-verifiziert. Lint fiel dabei von 3054 auf 3053 (eine Altlast
weniger).

---

## 36y. Langeweile als Motor — die Bonus-Probe (W265)

Kiras Beobachtung nach W264, zu Ende gedacht: die Hypothesen-Proben
sind nicht der Feind der Langeweile, sie sind ihre **Bestimmung**.
Der Experiment-Planer war schon als Freizeit-Beschäftigung gebaut
(aktives Testen nur Diagnose-Aktionen, Soul-Tor, idle-only,
Tagesbudget 3 bzw. 6 bei ruhigem System). Was fehlte, war die
Rückrichtung: Langeweile ERLAUBTE Proben, aber sie BEFEUERTE keine.

Jetzt: ist das Tagesbudget ausgeschöpft und meldet die
BrainStateMachine Langeweile (W264-Fenstermaß: Denk-Fenster
überwiegend leer), gewährt `_w265_experiment_budget_frei` genau
**eine Bonus-Probe pro Tag**. Da der Planer Fällige ohnehin
älteste-zuerst abarbeitet, kommt damit die am längsten wartende
offene Frage aktiv dran — aus „16 warten auf natürlichen Re-Run"
wird an langweiligen Tagen echtes Abarbeiten. Alle Schranken davor
(Diagnose-only, Soul-Tor, Stress-Veto ≥0.5) bleiben unberührt; der
Kreis schließt sich: Leerlauf → Langeweile → Neugier steigt → eine
offene Frage wird untersucht → Zustand wird kurz aktiv → Langeweile
sinkt ehrlich.

Zweite Hälfte der Idee ehrlich eingeordnet: eine eigene
„Neugier-Reihenfolge" braucht es nicht — die Warteschlange ist
bereits älteste-zuerst sortiert, und das Proben-Ranking nach
gelerntem Diagnose-Wert (R1357) sitzt an der Hypothesen-Quelle,
nicht an der Schlange.

Gesetz `test_w265_…` (Budget 3/6 exakt, Bonus genau einmal,
Verbrauchs-Flag, Quelle ruft die Extraktion); Köder EX gebissen.
Werkstatt-Lehre dieses Schritts: ein `git checkout` mitten in der
Köder-Kette zerstörte die md5-Referenz — der Endzustand wurde
stattdessen per Diff-gegen-HEAD (exakt die zwei W265-Blöcke)
verifiziert. Riese `_init_phase_knowledge` schrumpft um 6 (W248).

---

## 36z. Kiras Live-Screenshots — vier Wurzeln und die 1-GB-Regel (W266)

Kiras Discord-Screenshots vom Live-NAS (12.08., 03:17–08:23, alter
Deploy-Stand) plus zwei Fragen: gibt es noch **ungestempelte
Aktions-Wege**, und fehlt ein **Vertrag fürs RSS-Wachstum** mit der
1-GB-Grenze („was innerhalb ist, ist ok — erst wenn der
Gesamt-Verbrauch höher ist, sollte eine Meldung kommen")?

**Wurzel 1 — „lief durch, Ziel blieb trotzdem offen".** Erst als
Messfehler begonnen: die Screenshot-Flags `kernel_dmesg_checked` /
`network_interface_checked` existieren nirgends — die echten Ziele
heißen `kernel_checked`/`nic_checked` und haben Setzer samt Arming.
Die echte Wurzel saß eine Ebene höher: bei **Mehr-Flag-Zielen**
(intrusion_detection: `rootkit_checked` UND `ssh_config_checked`)
behandelte die W230/W234/W240-Abhilfe-Wahl jeden Ziel-Schlüssel als
gleich offen, weil sie den Weltzustand nicht kannte. Stand
`ssh_config_checked` längst (Arming 7200 s), wurde trotzdem weiter
`ssh_config_check` benannt und angestoßen — exakt Kiras Meldung.
Jetzt: `_offenes_ziel()` verengt Ziel auf die offenen Schlüssel;
`abhilfe_mit_grund`/`eskalierte_abhilfe` nehmen `weltzustand`
entgegen, und der Stillstands-Pfad reicht ihn an **beide** durch
(`SelfHealingMonitor._weltzustand()`, aus `_stuck_ursache`
extrahiert). Ergebnis am Live-Fall: „nie gesetzt: rootkit_checked.
ich versuche 'rootkit_check'."

**Wurzel 2 — „backup_impossible 111× / backup_stale 147× hat nicht
geholfen" waren erfundene Zahlen.** Die Stuck-Issue-Eskalation
(R1103) übergab `int(Alter/600)` als `count`, und die Voice-Vorlage
formulierte daraus „111× hintereinander nicht geholfen" — 111 war
aber ~18,5 h **Alter**, keine Versuchszahl. Jetzt geht `seit_min` in
die Meldung („ist seit 24.5h offen und nichts, was ich versucht
habe, hat es gelöst"); die count-Form bleibt nur für Rufer, die
wirklich zählen.

**Wurzel 3 — „(Tiefere Ursache: Keine tiefere U…".** Der
RootCauseAnalyzer liefert bei Nicht-Fund ein Sentinel
(`confidence 0.0`, explanation „Keine tiefere Ursache … gefunden");
Phase 3 in `analyze()` prüfte nur `root_cause != cause` — beim
Sentinel immer wahr — und hängte den Widerspruchssatz an Kiras
Meldung. Jetzt gilt: Confidence 0 = kein Fund, keine Anreicherung
(zeilenneutral im eingefrorenen Riesen, W248).

**Wurzel 4 — Buchhaltung als Lösungstipp.** Der ErrorDNA-Hinweis
„ähnliches Problem gelöst mit 'auto_stale_after_5_suppressions'"
zitierte den R696/R744-Deadlock-Escape-Marker als Lösung.
`_echte_loesung()` filtert Stale-Marker; sie zählen als ungelöst.

**Kiras Frage 1 (Stempel):** der Testlauf-58-Vertrag kannte noch
drei ungestempelte Wege. `woechentliches_update→system_update`
stempelt jetzt seine Seelen-Freigabe (`seele:allow_once`; ohne Seele
ehrlich KEIN Stempel — der Torwächter prüft dann selbst nach); das
`|nachgeholt`-Replay erbt den Kontext samt Stempel (test_r22). Der
CausalReasoner-Hub-Weg (v22) wohnt jetzt in
`_w266_causal_fixes_dispatchen` (Riese `_hub_tick_loop` schrumpft um
12): mutierende Fixes brauchen erst `should_act_autonomously` der
Seele, dann Stempel — ein Stempel ohne Prüfung hätte nur die
Nachprüfung des Torwächters ausgehebelt.

**Kiras Frage 2 (1-GB-Regel + Vertrag):** `rss_normal_mb`-Default
800→1024 (config-Override bleibt; Kiras config setzt den Schlüssel
nicht, der Default IST also die Live-Grenze). Beide RSS-Meldewege
(Drift-Alert, Auto-Dump) hingen schon an `_rss_normal_mb()`. Neu ist
der **sechste Verdrahtungs-Vertrag** `speicher:wachstum_im_rahmen`:
unter der Grenze ist ALLES ok (auch Drift = Arbeits-Beule); darüber
nennt der Befund Grenze, Wachstumsrate und die größten Verbraucher
(Thread-Gruppen + Top-Allocator, W255-Forderung). Er schließt dabei
ein echtes Loch — stand das RSS **ohne Drift** über der Grenze,
schwieg bisher alles — und beobachtet die Beobachtung selbst
(Observer steht still / hat nie getickt → eigener Befund;
W217-Klasse: junges System bekommt 30 min Anlauf).

Korrekt arbeiteten laut Screenshots: Schonmodus-Aktivierung beim
toten Mount `/Nas/Filme` und die Mount-tot-Meldung selbst.

Gesetz `test_w266_…` (5 Proben + W98-Test auf 1024 + W252-Zählung
auf 6); Köder EY in acht Varianten in beide Richtungen gebissen.

---

## 36aa. Kiras Dashboard-Screenshots — sechs Wurzeln (W267)

Fünf Dashboard-Screenshots vom Live-NAS (12.08., 19:17–19:35,
frischer Neustart, Lernphase Tag 1.5/14 — die W250-Lernphasen-Wache
blockte den Selbst-Schlaf live und korrekt). Sechs Befunde:

**1. Der Kritiker-Lernkreis war für autonome Aktionen tot.** Jeder
`post_action_thought` endete mit „Konnte nicht kritisieren:
prediction_unknown" — für ssh_config_check, restart_smb,
backup_retry, alles. Wurzel: `_on_action_completed` las die
`prediction_id` aus `_last_action_executed` **ohne Namens-Wächter**;
den Slot schreibt nur der Chat-Dispatch. Ein einziges Chat-Kommando
(19:17: restart_samba) hinterließ seine ID für immer, jede folgende
GOAP-Aktion reichte die längst verbrauchte ID an SelfCritique
(→ prediction_unknown), und weil die ID nicht None war, sprang auch
der synthetische Prediction-Fallback (v43.68k r10) nie an —
Bias/Hypothesen/Drift bekamen null Signal. Fix (zeilenneutral im
680-Zeilen-Riesen): Namens-Match + `pop()` = einmal verbrauchen,
dieselbe Bauart wie der before_snapshot-Wächter 150 Zeilen darüber.

**2. Datei-Index-Score stand dauerhaft auf 0/100.** Die v43.5-Log-
Dämpfung reichte nicht: elf Anomalie-Typen × Hunderte Treffer ≈ 250
Strafpunkte auf 100 verfügbare. Ein Messgerät am Anschlag trägt
keine Information — räumt Kira 1000 Dateien auf, bleibt 0 stehen.
Jetzt sättigend abgebildet (`100·60/(60+Strafe)`): Kiras Live-Lage
≈ 19–31 statt 0, kleine Bibliotheken bleiben nah am alten Wert, und
jede echte Verbesserung bewegt die Zahl.

**3. Drei Panels, drei Anomalie-Zahlen (9479/9129/7315).** Budget-
Abbrüche (C141/C145) liefern Teil-Listen, und kein Konsument sah
das. Der Engpass `_budget_exhausted` merkt sich den Abbruch jetzt
selbst (eine Stelle statt fünfzehn), der Gesundheitsbericht weist
`anomaly_list_partial` aus; frischer Lauf setzt zurück.

**4. 2023 `suspiciously_small` auf gültigen Kurz-Clips.** Die
Ausnahme kannte nur Ordner-NAMEN (`clips`, `trailer` …) — die
Namens-Listen-Falle. Das Wissen „pro Ordner gibt es eine Norm"
steckt eine Phase tiefer längst im IQR-Ausreißer-Check. Jetzt
Ordner-NORM in Phase 3: besteht ein Ordner ab 5 Videos überwiegend
(≥50 %) aus kleinen, ist klein dort normal; der einzelne 2-MB-Film
zwischen 40 normalen bleibt Befund. Phase 3 dafür aus dem
W248-Riesen extrahiert (`_w267_phase_kleine_dateien`,
get_file_anomalies 1401 → ~1306).

**5. Serien-Check: „es fehlen 6, 8, 10, …" über drei Staffeln
derselben Serie.** Die R1358hy-Doppelfolgen-Signatur — in Formen,
die alle Range-Muster verfehlten (kannten nur Bindestrich):
`S01E05E06`, `05+06`, `05 & 06`. Ergänzt mit denselben
Schutzregeln (1–3 Ziffern, validierte Start-Episode, Spanne ≤4).

**6. Entscheidungs-Anzeige und Dienst-Namen.** 9 von 12 Zeilen in
„Letzte Entscheidungen" waren nacktes `gate_precondition` — die
Aktion steckt im context, den das Dashboard wegwarf; jetzt steht sie
dabei. Und „starte samba neu" dispatchte den GESPROCHENEN Namen
(`restart_samba` schlug fehl, 19:26; erst der Reparatur-Umweg traf
`restart_smb`): `_dienst_config_schluessel` löst Katalog-Kanonisch →
config-Dienst (über Schlüssel und systemd_name-Aliase) vor dem
Dispatch auf; Unbekanntes bleibt unverändert — nichts raten.

Gesetz `test_w267_…` (6 Proben); Köder EZ in sechs Varianten in
beide Richtungen gebissen. Alt-Screenshots-Einordnung: die
Anomalie-Empfehlungsliste selbst (350 Serien, 3286 Temp-Dateien …)
ist echte Unordnung, kein Bug — mit Ordner-Norm und Doppelfolgen-
Formen werden die Zahlen beim nächsten Lauf ehrlich kleiner.

---

## 36ab. „Schaue dir die Bilder genau an" — die zweite Schicht (W268)

Kiras Nachfrage zu denselben fünf Screenshots. Beim genauen
Hinsehen standen da noch vier Wurzeln, die W267 übergangen hatte:

**1. Die blinde Schau-hin-Probe.** „Entscheidung: system_vitals —
drive: log_hygiene" erschien VIERMAL in 15 Minuten. log_hygienes
`observation_action` (Round-33-Mechanik: Probe läuft, kommt kein
Spike währenddessen → attention resolved, Druck ×0.3) war
**system_vitals** — eine CPU/RAM/Temp-Messung, die über Logs nichts
aussagt. Die Probe konnte den Log-Zustand nie sehen: Druck runter,
Log-Spike kommt wieder, Schleife. Jetzt `log_size_check` (read-only,
Seelen-erlaubt, schaut wirklich auf Log-Größen) — sieht sie Stau,
kommt der Spike während der Beobachtung, und GOAP plant den echten
Fix (journal_vacuum/log_rotate, die der Feed um 19:19:54 ja auch
schon entschied). Zeilenneutral im eingefrorenen
`_init_default_drives` (exakt 391).

**2. Der rohe Drive-Name im Mitternachts-Report.** „0 Issues, 3
gelöst | Achtung! Ich kümmere mich gerade um **swap_pressure**
(Dringlichkeit: 82%)!" — `get_greeting` benutzte eine LOKALE
8-Einträge-Schatten-Map, swap_pressure fehlte, der snake_case-Name
leckte. Die zentrale Autorität `_DRIVE_HUMAN_NAMES` kennt alle 38
Drives („Swap-Druck") und trägt einen Coverage-Test. Eine ZWEITE
Schatten-Map gleicher Bauart saß im Dismiss-Handler (statuspause).
Beide ersetzt durch `_humanize_drive` (W253); Satzform auf
Doppelpunkt umgestellt, damit auch Phrasen-Namen grammatisch passen.
Einordnung des scheinbaren Widerspruchs: „0 Issues" zählt das
Problem-Buch (nichts kaputt), der Anhang sagt, woran sie gerade
arbeitet — beides wahr, jetzt auch lesbar.

**3. „360 hier, 350 dort" — zwei Serien-Wahrheiten.** Der
Anomalie-Pfad filterte unplausible Lücken längst (alle geraden =
Doppelfolgen-Benennung, >30 % fehlend, >10 Lücken) — der
Serien-Check zeigte dieselben Serien UNGEFILTERT und
schlimmste-zuerst: die Benennungs-Artefakte („es fehlen 6, 8,
10, …") standen prominent ganz oben, obwohl eine Schicht tiefer
„absichtliches Muster" erkannt war. Jetzt EINE Autorität
`_luecken_plausibel` für beide Konsumenten; der Serien-Check nennt
Aussortiertes ehrlich („zähle ich NICHT als lückenhaft — sieht nach
Benennungs-Schema aus") statt es als fehlend zu verkaufen. Die
Phase-2-Inline-Checks sind in die Autorität gewandert
(get_file_anomalies schrumpft weiter, 1306 → 1298).

**4. Intentions ohne Uhr.** „scrub_btrfs (0%)" bei Idle — frisch
angelegt oder hängt seit 30 Stunden? `active_intentions()` liefert
`age_hours`/`idle_minutes` längst mit, das Dashboard warf beide weg
(dieselbe Klasse wie die nackten gate_precondition-Zeilen aus W267).
Jetzt steht das Alter dabei, und ≥60 min ohne Fortschritt wird
orange als „hängt seit N min" markiert. Der 48-h-Verfall
(expired_intentions) existiert und blieb unberührt.

Eingeordnet, kein Fix nötig: `systemd_health_check 0.0s` ist die
Cache-Lese-Bauart (der SystemdMonitor pollt selbst, der Check liest
seinen Stand) — kein Phantom-Vollzug. „Death Note (Staffel 2):
E20–E37 vollständig" ist korrekt (durchlaufende Nummerierung).

Gesetz `test_w268_…` (4 Proben); Köder FA in vier Varianten in
beide Richtungen gebissen.

---

## 36ac. Die Ketten in beide Richtungen (W269)

Kiras Auftrag: die W266–W268-Ketten beidseitig abklopfen — aufwärts
(wer speist sie), abwärts (wer konsumiert sie) — und
Abhängigkeits-Brüche finden. Sechs Verdachtspunkte gemessen, zwei
echte Folge-Wurzeln, vier sauber:

**Wurzel 1 — Stuck-Kette aufwärts: zwei Wahrheiten über denselben
Weltzustand.** Die W266-Autorität (`ursache_fuer`/Abhilfe-Wahl) hat
ZWEI Zubringer mit verschiedenen Linsen: die Selbstheilung übergibt
den TTL-gefilterten GOAP-State (`snapshot_to_goap_state` prüft
`_ttl_<k>` via `_flag`, R1465) — der Chat-Weg (W243) übergibt
**rohes volatile_state**, wo `ssh_config_checked=True` ewig stehen
bleibt, auch wenn die Arming-Frist um ist. Folge: der Chat sagte
„das Ziel ist eigentlich erfüllt", während Planer und Selbstheilung
dasselbe Flag als offen behandelten. Fix an der EINEN Autorität:
`_wert_gilt` als TTL-Linse in antriebs_abhilfe, benutzt von
`_offenes_ziel` UND `_ziel_lage`; „abgelaufen" ist jetzt eine eigene
benannte Krankheit („ssh_config_checked war erfüllt, die Frist ist
um") — eine andere als „nie gesetzt" und als „falscher Wert".
TTL-gefilterte Zustände (ohne `_ttl_`-Schlüssel) verhalten sich
exakt wie vorher.

**Wurzel 2 — Serien-Kette abwärts: drei rohe Konsumenten.**
`find_series_gaps` lieferte die UNGEFILTERTE Liste an das Patrol-Log
und — schwerer — an den **LLM-Kontext**: bei „fehlt/Lücke"-Fragen
bekam das LLM „Grosse Pause: fehlt E6, E8, E10 …" als Fakten
serviert und hätte Kira dieselben Benennungs-Artefakte erzählt, die
der Anomalie-Pfad längst aussortiert. Der Medien-Katalog-Handler
(behavior) filterte ebenfalls nicht. Fix an der Quelle:
`find_series_gaps` fragt jetzt selbst die eine Autorität
(`_luecken_plausibel`, W268) — Patrol-Log und LLM-Kontext erben die
Wahrheit automatisch; der Katalog benennt Aussortiertes ehrlich
(„N weitere sehen nach Doppelfolgen-Benennung aus — nicht
mitgezählt").

**Gemessen und sauber (kein Fix):** (a) `stalled_intentions` hat
fünf echte Konsumenten (Brainstorm-Feedback, GOAP-Zyklus,
Wartung ×2, Orchestrator) — kein toter Haken. (b) Der
Chat-Retry-Pfad liest nur action/target/ts aus
`_last_action_executed` — das W267-`pop(prediction_id)` bricht
nichts. (c) Die Kritiker-Kette trägt die prediction_id aufwärts
vollständig: cognitive_retry registriert die Vorhersage
(`RetryDecision.prediction_id`), der Chat-Dispatch legt sie ab, der
Event-Handler verbraucht sie seit W267 namens-geprüft und genau
einmal. (d) Alle 23 Antriebe mit Beobachtungs-Probe (seit W270: 21,
§36ad) haben mindestens eine Spike-Quelle — kein Antrieb kann durch die
Round-33-Attention-Mechanik (Probe ohne Spike → Druck ×0.3)
dauerhaft am Planen gehindert werden; `armed_for_real_action` wird
bei jedem echten Spike gesetzt, nicht nur im Probe-Fenster.
Dokumentierte Rest-Schiefe: die Proben von btrfs_scrub
(mount_status) und ssd_trim (system_vitals) sagen über
Scrub-/Trim-Fälligkeit nichts — sie verzögern zeitbasierte Drives
nur (die periodischen TTL-Spikes armen später ohnehin); eine echte
„ist-fällig"-Probe gibt es dafür nicht, das bleibt als bekannte
Bauart-Grenze notiert.

Gesetz `test_w269_…` (2 Proben); Köder FB in drei Varianten in
beide Richtungen gebissen.

---

## 36ad. Die Mount-Melde-Kette: die Wache war Teil des Täters (W270)

Kiras Frage: „kann sein das die schuld sind das myuri immer wieder
meint das die mounts nicht da sind?" — Ja, zum Teil. `mount_status`
läuft als Schau-hin-Probe für mount_integrity/nfs_health (und bis
W270 sachfremd auch für btrfs_scrub) ständig — und tat dabei drei
Dinge falsch:

**1. `fuser -m` auf JEDEN Mount im Executor-Thread.** Auf einem
toten NFS-/USB-Mount blockiert das bis zum Timeout — „Executor
ausgelastet", die nächste Probe staut sich dahinter, und im
Dashboard sieht es aus, als stimme mit den Mounts dauernd etwas
nicht. Jetzt: riskante Pfade (`safe_fs.is_risky_path`) werden nicht
betastet, die Zeile sagt ehrlich „[Zustand unklar — Pfad riskant]".

**2. Rohes `os.path.ismount` auf potenziell tote Pfade** — in der
Statusprobe UND in der R1357-Wurzelanalyse (root_cause). Das ist
exakt der D-State-Keil, gegen den das Bau-Gesetz (W53) geschrieben
wurde; die Wurzelanalyse läuft genau DANN, wenn mit den Mounts
etwas nicht stimmt — die Diagnose des Keils war selbst ein
Keil-Kandidat. Beide Stellen fragen jetzt `safe_fs.ismount`
(lokal = direkter Syscall, riskant = Wegwerf-Prozess); die
Bau-Gesetz-Ausnahmen sind entsprechend gesunken (repair_mixin
11→10, root_cause raus aus der Liste).

**3. Bewusst abgesteckte Wechselplatten als „NICHT gemountet!"
gemeldet.** Die Backup-Wahrheit (waiting_for_disk = Platte bewusst
weg, KEIN Fehler, R1358h) wurde hier nie gefragt. Jetzt fragt die
Probe `backup_manager.get_backup_health()` und meldet solche Pfade
als „Wechselplatte, gerade nicht eingesteckt (kein Fehler)" — ein
WIRKLICH fehlender Mount bleibt ein lauter Befund.

Dazu die W269-Bauart-Grenze gezogen: btrfs_scrub/ssd_trim sind
Fälligkeits-Antriebe, deren Probe (mount_status/system_vitals)
„fällig?" nie beantworten konnte und den Plan nur verzögerte — sie
haben keine Beobachtungs-Probe mehr und planen direkt (aus 23
Proben-Antrieben wurden 21). Bewusst VERWORFEN: `mount_check` als
Ersatz-Probe — der goap_dispatcher mappt es auf `remount_check`,
und das mountet real; eine Wache, die von selbst eingreift, ist
keine Wache. Keine neue Parallel-Kette gebaut (W253): safe_fs und
backup_truth existierten beide längst und wurden nur angeschlossen.

Gesetz `test_w270_…` (3 Proben, darunter „nie mount_check als
Probe"); Köder FC in drei Varianten in beide Richtungen gebissen.

---

## 36ae. Die W270-Klassen, überall gezogen (W271)

Kiras „reparieren und sucht weiter": die zwei W270-Fehlerklassen
existierten an ~10 weiteren Stellen, gemessen über die
Bau-Gesetz-Zählung. Rohes `os.path.ismount` lief im Executor-Thread
(Backup-Preflight, remount_check-Verify), im Belief-Takt
(`_update_beliefs`, inkl. `fuser -m` auf Backup-Mounts — dieselbe
Keil-Form wie W270), im Chat-Thread (fsck/mount-Hints,
Storage-Discovery, Backup-Diagnose, fstab-Prompt), im LLM-Prompt-Bau,
im Watchdog-Drift-Check, im Boot-Preflight und im Preconditions-Gate.
Und alle „NICHT gemountet"-Texte dieser Wege waren blind für die
Backup-Wahrheit — **das** ist die Quelle von „meint immer wieder, die
Mounts sind nicht da": diese Texte gehen an Chat UND LLM-Kontext.

Statt zehn Inline-Kopien wurden zwei bestehende Autoritäten
erweitert (W253):

- **`safe_fs.ismount_zustand`** — hänge-sicher mit DREI Zuständen:
  True/False/None. None („antwortet nicht") ist KEIN „fehlt" — der
  Unterschied ist genau die Klasse „toter Mount wird als fehlender
  Mount verkauft". Konsumenten behandeln None je nach Rolle: der
  Watchdog leitet daraus keinen Drift ab (kein „KORRIGIERT" auf
  Basis von Ahnungslosigkeit), das Preconditions-Gate blockt mit
  ehrlichem Text, der Backup-Preflight sichert nicht auf einen
  nicht antwortenden Mount, Anzeige-Wege sagen „Zustand unklar".
- **`backup_truth.warte_pfade` / `ist_bewusst_abgesteckt` /
  `mount_meldung`** — die Wechselplatten-Wahrheit (waiting_for_disk,
  Präfix-Vergleich in beide Richtungen) und DER eine Meldungstext
  für alle vier Fälle. `_action_mount_status` (W270) wurde auf
  dieselben Autoritäten refaktoriert statt seine Inline-Kopie zu
  behalten.

Dazu ein ehrlicher W270-Selbstfund: der W270-fuser-Skip galt für
JEDEN riskanten Pfad — `is_risky_path` ist aber für jeden
Daten-Mount wahr, live hätte also ALLES „[Zustand unklar]" gezeigt
und die Prozess-Info wäre tot gewesen. Verfeinert: fuser läuft,
wenn die hänge-sichere Worker-Probe den Mount gerade beantwortet
hat; nur ein nicht antwortender Mount wird nicht betastet.

Die Bau-Gesetz-Ausnahmeliste ist um zehn Einträge geschrumpft
(readiness_gate zusätzlich auf drei Zustände umgebaut); die
verbleibenden ismount-Zählungen dort sind lokale Pfade oder bereits
safe_fs-Aufrufe, die der Zähler nicht unterscheiden kann.

Gesetz `test_w271_…` (4 Proben); Köder FD in drei Varianten in
beide Richtungen gebissen, Restores md5-verifiziert.

---

## 36af. Die Melder hinter den Melde-Texten (W272)

Kiras „such weiter". Zwei Fährten aus der W271-Arbeit verfolgt —
eine entlastet, drei Treffer.

**Entlastet (gemessen, kein Fix):** das Timeout-Versprechen von
`run_command` hält auch bei D-State-Kindern. safe_subprocess killt
bei TimeoutExpired die ganze Prozessgruppe (SIGTERM →
Grace-Schleife mit monotonic-Deadline → SIGKILL), reapt mit
`communicate(timeout=5)` und räumt mit `wait(timeout=2)` nach —
jeder Pfad begrenzt, jede Exception gefangen. Der Thread entkommt
nach timeout+~7 s; nur das unkillbare Kind bleibt zurück
(unvermeidbar). `run_compat` läuft über dieselbe API.

**Drei Melder derselben W270/W271-Klasse gefunden und gezogen:**

1. **Aufwach-Bericht** (`_quick_health_check`, eingefroren 660):
   meldete `nicht_eingehaengt` als Issue, ohne die Backup-Wahrheit
   zu fragen — die bewusst abgesteckte Wechselplatte stand nach
   JEDEM Aufwachen als Problem in Kiras Bericht. Jetzt fragt der
   Modul-Helfer `_w272_wechselplatte` (fail-closed: ohne Wahrheit
   bleibt die Meldung laut) in beiden Zweigen; zeilenneutral.
2. **Wartungs-Check** (`_task_mount_check`): schickte für die
   Wechselplatte sogar ein `nfs_remount` ins Leere — aktiv falsch,
   nicht nur falscher Text, bei jedem Wartungslauf. Jetzt: kein
   Remount, keine Issue-Warnung; die Bilanz nennt sie ehrlich
   („N Wechselplatte(n) bewusst nicht eingesteckt").
3. **backup_verify** (ops_mixin): ZWEI rohe `os.path.ismount` auf
   BEIDE NAS-Pfade im Executor-Thread — versteckt unter dem
   statvfs-Ausnahmedeckel des Bau-Gesetzes (das Datei-Maximum 4
   deckte sie mit; jetzt 2 mit ehrlicher Begründung). Dazu
   Backup-Wahrheit: „Wechselplatte gerade nicht eingesteckt —
   Prüfung wartet (kein Fehler)" statt Fehlbefund; ein steckender
   Mount heißt „antwortet nicht", nicht „nicht gemountet".

Gesetz `test_w272_…` (3 Proben); Köder FE in drei Varianten in
beide Richtungen gebissen, Restores md5-verifiziert.

---

## 36ag. Dieselbe Klasse, nächste Syscall-Form (W273)

Kiras „such weitere solcher probleme". Der W53-Obduktionsbericht
nennt `os.path.exists` AUSDRÜCKLICH neben ismount als
D-State-Täter — das Bau-Gesetz lässt exists aber bewusst draußen,
weil es meist harmlos ist (Konfigdateien, state/). Auf
MOUNT-Pfaden ist rohes exists/isdir derselbe Keil, nur in einer
Form, die der Zähler nicht sieht. Gezielt danach gesucht:

**Executor-Thread:** `_reality_backup` prüfte BEIDE Paar-Pfade mit
rohem isdir — 20 Zeilen neben dem W271-Fix; die remount-Aktion
selbst (`exists`+`ismount` roh, im eingefrorenen 2504-Zeilen-Riesen
`_exec_dispatch_action`); disk_cleanup-isdir auf NAS-Mounts; und in
repair_mixin SIEBEN weitere rohe ismounts, die unter dem
Datei-Maximum 10 des Bau-Gesetzes mitgedeckt waren — darunter die
Backup-Ursachen-Analyse, die genau DANN läuft, wenn ein Mount am
ehesten tot ist (dieselbe „Diagnose des Keils ist selbst ein
Keil"-Figur wie root_cause in W270), safe_unmount (jetzt auch ohne
`fuser -vm` auf einen nicht antwortenden Mount) und die
fstrim-Mount-Schleife.

**Chat-Thread:** „zeig mir /Nas/…" (isdir) und „wie groß ist"
(exists) im eingefrorenen `_generate_chat_response` — rohe Stats
auf Kiras NAS-Pfade in der Antwort-Schleife.

**Liveness-Watcher:** die isdir-Tore VOR den safe_fs-Samples waren
selbst rohe Stats aufs Backup-Ziel — der eigene Kommentar dort
beschreibt wörtlich den D-State, den das Tor davor wieder
aufmachte.

**Proaktiv:** die mount_phantom-Grundwahrheit lief über rohes
exists, und ein steckender Mount wurde dabei zu „missing" — die
Phantom-Korrektur hätte auf Basis eines Hängers gefeuert. Jetzt:
kein billiges Urteil verfügbar (None) statt Raten.

Fix ohne neues System (W253): safe_fs um `exists_zustand` und
`isdir_zustand` erweitert — drei Zustände, werfen nie, gleiche
Semantik wie das W271-`ismount_zustand`. Die drei eingefrorenen
Riesen exakt zeilenneutral umgebaut. Bau-Gesetz-Maxima ehrlich
gesenkt: repair_mixin 10→3, exec_mixin 4→2 (jetzt nur noch lokale
statvfs-Reste). Eine Test-Lehre festgehalten: die erste
Köder-Probe fürs Liveness-Tor war tautologisch (für einen nicht
existierenden Testpfad liefert auch rohes isdir brav False) — erst
ein Roh-Aufruf-Tracker machte sie sehend.

Gesetz `test_w273_…` (3 Proben); Köder FF in drei Varianten in
beide Richtungen gebissen, Restores md5-verifiziert.

---

## 36ah. Tiefer: das Fundament unter safe_fs (W274)

Kiras „Such weiter und tiefer". W270–W273 haben ~30 Stellen auf
safe_fs → probe_broker → Wegwerf-Worker umgezogen — wenn DIESE
Schicht Löcher hat, steht alles auf Sand. Vier Tiefenfragen an das
Fundament, mit Messung statt Vertrauen:

**Entlastet (drei von vier):** (1) Ein `_config()`-Ausfall ist
abgefedert — Netz-Dateisysteme (die echten D-State-Täter) kommen
IMMER aus `/proc/self/mounts` in die risky roots, auch ohne
lesbare Config. (2) Jeder Warteweg im Broker ist begrenzt:
`request()` läuft eine monotonic-Deadline-Schleife, `run_probe()`
wartet auf ein Event mit Timeout, der Streaming-Weg hat
beat_timeout, `abandon()` reapt im Wegwerf-Thread; der Worker
selbst ist eine Ein-Anfrage-Schleife, deren Fehler die Antwort
IST. Verhaltenstests existierten bereits
(test_mount_check_real_probe, test_mount_probe_process_backend).
(3) Der Thread-Fallback (Prozess-Backend per Config aus) deckelt
geleakte Hänger-Threads auf 2 pro Mount-Key — der W250-Verdacht
„Thread-Leck aus den Proben" ist damit entkräftet; Backend-Default
ist AN, auch ohne Config (fail-open in die sichere Richtung).
Notierte, bewusst nicht angefasste Schwäche: der Worker-Spawn
läuft unter dem globalen `_proc_lock` (~50 ms Serialisierung —
begrenzt, kein Keil).

**Gefunden (eines):** Der 60-Sekunden-Cache der risky roots wurde
produktiv NIE invalidiert — `invalidate_cache()` riefen nur die
Tests (dieselbe Figur wie das nie abgelesene unverstanden.json in
W168). Ein frisch eingehängter, NICHT konfigurierter
Netz-/Wechsel-Mount lief bis zu 60 s über den lokalen Schnellpfad
= rohe Syscalls, genau in der wackligsten Phase kurz nach dem
Einhängen. Fix: der Mount-Diff-Callback der StorageDiscovery
(`_on_mount_change`) frischt die roots sofort auf.

**Festgeschrieben (ein Vertrag):** `ProbeCapExhausted` IST
`ProbeHung` (Subklasse), `MountTimeoutError` IST `OSError`, und
die drei zustand-Helfer geben für Cap- wie Hang-Fälle None —
„Slots von früherem Hänger belegt" und „Mount antwortet nicht"
sind beides KEIN Wissen über den Mount. Der Test hält die
Hierarchie fest, damit niemand sie auflöst und Cap-Fälle plötzlich
durch alle `except ProbeHung`-Stellen schlagen.

Gesetz `test_w274_…` (2 Proben); Köder FG in beide Richtungen
gebissen, Restores md5-verifiziert.

---

## 36ai. Verbessern: die notierten Schwächen behoben (W275)

Kiras „Verbessere die sachen". Drei Verbesserungen an dem, was
W270–W274 gebaut und notiert haben:

**1. Die Wahrheitsquelle war selbst ein Keil.**
`get_backup_health` — seit W270–W272 DIE Quelle der
Wechselplatten-Wahrheit — lief selbst mit rohen
exists/isdir/scandir auf beide NAS-Paar-Pfade (der W273-Keil durch
die Hintertür: jeder warte_pfade-Aufruf konnte im D-State hängen)
und kannte kein „antwortet nicht". Jetzt: drei Zustände über die
safe_fs-Helfer plus ein `stuck`-Flag pro Paar — und `warte_pfade`
überspringt steckende Paare, damit ein STECKENDER Mount nie als
„Platte bewusst weg (kein Fehler)" beschwichtigt wird. Die
Beschwichtigung ist nur für die ehrlich gemessene Abwesenheit da.

**2. Die Wahrheit war teuer.**
`warte_pfade` rief `get_backup_health` bei JEDEM Aufruf — und die
scannt pro Paar das Backup-Log und macht zwei Worker-listdir. Der
LLM-Prompt-Bau zahlte das bis zu 6× pro Prompt, der fstab-Prompt
einmal pro Eintrag. Jetzt: 5-Sekunden-TTL-Cache in der Autorität
(nicht bei den Konsumenten — W253), Identität per weakref (eine
nach GC recycelte id() wäre ein falscher Cache-Treffer), Fehler
werden nie gecacht.

**3. Die §36ah-Schwäche ist gezogen.**
Der Worker-Spawn lief unter dem globalen `_proc_lock` — jeder
Popen (~50 ms, unter Last mehr) serialisierte die Proben ALLER
Mounts. Jetzt: Spawn außerhalb des Locks; das
Doppel-Spawn-Rennen zweier Threads auf denselben Key löst
insert-or-discard — der Verlierer wird sofort wieder aufgegeben
(idle = kill() wirkt).

Gesetz `test_w275_…` (3 Proben, darunter ein Spion, der prüft,
dass der Popen NICHT unter dem Lock läuft); Köder FH in drei
Varianten in beide Richtungen gebissen, Restores md5-verifiziert.

---

## 36aj. Weiter verbessert: ein Log-Durchgang (W276)

Kiras „Verbessere es weiter". `get_backup_health` öffnete und
scannte das komplette backup_log.jsonl **für jedes Paar** — P Paare
= P Voll-Scans, bei jedem Aufruf (Chat-Backup-Info, Health-API,
warte_pfade-Cache-Miss). Gemessen mit 4 Paaren und 20.000
Log-Zeilen: **154 ms → 44 ms pro Aufruf (3,5×), 4 → 1 Log-Opens.**
Ein Durchgang sammelt jetzt die letzten Erfolgs-Zeitstempel ALLER
Quellen auf einmal; die Semantik pro Paar ist unverändert
(gleicher max-ts-Filter auf success-Einträge, gleiche
None/False-Fälle für Paare ohne Erfolg). Der Gewinn wächst linear
mit der Paar-Zahl und stapelt sich mit dem W275-Cache: ein
Cache-Miss kostet jetzt einen Scan statt P.

Gesetz `test_w276_…` (1 Probe mit Open-Spion: das Log darf pro
Aufruf genau EINMAL geöffnet werden, und jedes Paar behält sein
korrektes Backup-Alter); Köder FI in beide Richtungen gebissen
(Sammel-Durchgang leer → Alters-Probe rot; zweites Open → Spion
rot), Restores md5-verifiziert.

---

## 36ak. Der siebte Vertrag und siebenunddreißig Tote (W277)

Kiras „mache jetzt verträge um zu sehen wo welche probleme geben
kann … und schaue weiter nach toten codes". Zwei Fronten:

**1. Verdrahtungs-Vertrag Nr. 7: `mount:proben_kette_beurteilbar`.**
Die W270–W276-Kette machte jede einzelne Meldung ehrlich — aber
niemand meldete, wenn die Proben-Maschinerie SELBST am Ende ist.
Zwei Risse, die bisher nur in DEBUG-Logs standen: (a) ein Mount,
dessen Karteileichen-Slots voll sind (§36ah-Cap erreicht), ist
dauerhaft unbeurteilbar — jede weitere Probe endet sofort im Cap,
bis ein Mount-Comeback oder Neustart die Leichen wegräumt; (b) ein
Backup-Paar mit `stuck`-Flag (§36ai) steckt im D-State. Der neue
Vertrag in `core/verdrahtungs_vertraege.py` liest beides aus den
bestehenden Autoritäten (`get_process_backend_stats`,
`get_backup_health`) und nennt die Täter beim Namen — Mount-Key
und Karteileichen-Zahl bzw. Quellpfad. Ist die Statistik selbst
unerreichbar, meldet der Vertrag DAS (gestrandete Worker wären
sonst unsichtbar), statt still grün zu sein.

**2. Siebenunddreißig referenzlose Modul-Funktionen entfernt.**
Ein Sweep über alle Modul-Funktionen (Tokenizer-Zählung über .py
UND .html/.js/.json/.md/.sh/.yaml/.toml, damit Template-Aufrufe
nicht als tot gelten) fand 36 Funktionen, deren Name nirgends
sonst vorkommt — gebaut, exportiert, nie gerufen. Die sprechendste
Figur: `cached_is_active` in `core/safe_subprocess.py`, ein
kompletter TTL-Cache samt Lock aus Welle 42 (R1023), der neben dem
tatsächlich verdrahteten `run_command_cached` lag — gebaut und nie
verdrahtet, dieselbe Figur wie das nie abgelesene Messgerät aus
W168. Nach dem Entfernen fand das neue Gesetz sofort den 37.:
`moe_route`, dessen einziger Rufer `moe_primary` gerade gestorben
war — der Beweis, dass das Gesetz Kaskaden fängt.

**Das Toter-Code-Gesetz** (`test_w277_keine_referenzlosen_…`)
hält die Basis auf NULL: keine Modul-Funktion (Name ≥ 5 Zeichen,
nicht `__`-präfixiert) ohne mindestens eine Fremd-Referenz.
Dokumentierter blinder Fleck: dynamisch zusammengebaute Namen
(`getattr(m, "prefix_" + x)`) sieht der Tokenizer als Referenz
nicht — solcher Code würde fälschlich als tot gelten, kam im
Bestand aber nicht vor (alle 37 von Hand gegengeprüft).

Lint-Basis gesunken: 3053 → 3049. Gesetz `test_w277_…` (Vertrag
gesund/Cap/stuck plus der Null-Sweep); Köder FJ in beide
Richtungen gebissen (Vertrag meldet Cap nicht → rot; tote
Funktion eingepflanzt → Sweep rot), Restores md5-verifiziert.

---

## 36al. Die Nachprüfung der 37 — ein Toter kehrt verdrahtet zurück (W278)

Kiras Frage auf W277: „schaue genau nach warum das tod sein soll und
ob dafür keine verwendung gibt" — dann „mach". Jede der 37 Entfernungen
wurde einzeln nachgeprüft: Substring-Suche über ALLE Dateien im Stand
vor der Entfernung, dynamische Baumuster (getattr auf die Module: null),
Laufzeitdaten (data/, state/: null), Git-Historie bis zum Import-Wurzel-
Commit. Ergebnis: keine hatte eine Referenz; zwei sind nachweislich
schon im Geburts-Commit ohne Rufer geboren (ist_wartungs_aufgabe,
erwartungen_stand).

**Zwei eigene Fehlbefunde, ehrlich korrigiert.** Der Zwischenbericht
behauptete, OHNE_MESSBAREN_GEWINN sei verwaist und remount_check/
system_config_backup könnten als „konsistent nutzlos" abgestempelt
werden. Beides falsch: die Menge lebt unter Alias
(`action_effectiveness.DIAGNOSTIC_ACTIONS = OHNE_MESSBAREN_GEWINN`,
mit Absichts-Kommentar), und `is_consistently_useless` fragt zuerst
die kanonische Ausnahme-Autorität `impact_exempt` (R1451a), die beide
Aktionen ausnimmt — praktisch verifiziert. Ursache des Fehlbefunds:
ein grep mit `head -5` hatte genau den Alias-Treffer abgeschnitten.
Lehre: Verwaisten-Suchen laufen ab jetzt ohne Abschneidung.

**Ein Toter war eine Fähigkeit: scan_unreachable_drive_goals**
(R1358at). Der proaktive Voll-Sweep „welcher Antrieb wird stuck,
sobald er spiked, weil kein Producer-Pfad existiert" — auf den
lebenden Objekten. Der verdrahtete Weg (handle_no_plan_escalation)
ist rein REAKTIV (feuert nach 3× No-Plan), und der CI-Scanner aus
der Docstring (tests/test_r1358as) existiert im Repo nicht. W278
belebt die Funktion wieder und verdrahtet sie ERSTMALS: als achter
Check `plan_erreichbarkeit` im wöchentlichen Sensor-TÜV (§ Sensor-
TÜV), mit Gehirn-Referenz vom Orchestrator durchgereicht. Reißt sie,
nennt der TÜV die Täter beim Namen und die Hochprio-Meldung geht raus
— BEVOR der Antrieb das erste Mal vergeblich plant.

**Drei ehrlich Verwaiste entfernt** (das Toter-Code-Gesetz sieht nur
Funktionen, keine Konstanten/Methoden): die THINKING-Marker
(reasoning_trace — das Live-Strippen macht der llm_router inline),
die Methode get_fs_units (system_capabilities) und _VALID_DRIVE_NAMES
+ HOPELESS_AWARE_DRIVES (drive_catalog — die R943-Registry, deren
Rufer nie umgestellt wurden; das Schlüssel-Vokabular ist längst in
mehrere Formate gedriftet, dokumentiert am Grabstein-Kommentar).

Gesetz `test_w278_…` (2 Proben: Sweep nennt Unerreichbare beim Namen
und wirft nie; TÜV kennt den Check + Orchestrator reicht das Gehirn);
Köder FK1/FK2 in beide Richtungen gebissen (Filter frisst alles →
Sweep-Probe rot; Check aus dem TÜV-Programm → Verdrahtungs-Probe rot),
Restores md5-verifiziert. Lint-Basis unverändert 3049.

---

## 36am. „Was ist genau los?" verschwieg den toten Mount (W279)

Kiras Live-Screenshots vom 13.08.: um 07:05 aktiviert Myuri den
Schonmodus (/Nas/Filme staler Mount, Watchdog-Probe hängt), meldet
Stuck-Issues (backup_stale 6,6 h, backup_impossible, btrfs_scrubbed)
und ein Memory-Leck (RSS +202,7 MB/h, 185 Threads). Um 14:46 fragt
Kira „Was ist genau los?" — Antwort: „Akut ist gerade nichts offen."

**Einordnung der Bilder gegen den Repo-Stand** (das NAS läuft auf
altem Deploy): der falsche „kein geeignetes Zeitfenster"-Rat ist im
Repo BEREITS gefixt (causal_reasoner fragt die Mount-Wahrheit und
empfiehlt remount statt backup_check — der Code-Kommentar dort zitiert
exakt dieses Protokoll-Muster vom 27.07.); die D-State-Kette selbst
ist die W270–W277-Arbeit; die Thread-Zahlen (EventBusAsync×33,
EvtSoftTO×32, MountProbeReader×18) sind im Repo hart gedeckelt und
werden als Selbstbefund gemeldet — auf dem alten Stand hängen
Listener mit rohem Mount-I/O auf dem toten Mount und fressen Slots
und Speicher. All das heilt erst das Deploy.

**Die eine Lücke, die AUCH im aktuellen Code klaffte:** der
„Was ist los?"-Handler (W162-Zweig) fragt flüchtige 30-min-Ringe und
das Warnungs-Buch — also nur EREIGNISSE und Vergangenheit. Ein toter
Mount mit aktivem Schonmodus ist aber ein offener ZUSTAND: nach ein
paar stillen Stunden sind die Ringe leer, das Buch ist Chronik, und
die Antwort kippt auf „nichts offen", während keine Platte angefasst
wird. Fix: der Zweig fragt zuerst `mount_wahrheit.kaputte()`; gibt es
tote Mounts, führt die Antwort mit dem Täter-Namen und dem Schonmodus
(inkl. `umount -l` + `mount`-Hinweis) — Entwarnung gibt es nur noch,
wenn auch die Mount-Wahrheit nichts weiß.

Gesetz `test_w279_…` (3 Fälle: toter Mount + leeres Buch → Täter statt
„Alles ruhig"; toter Mount + Chronik → Akut-Zustand vor der Liste,
keine „nichts offen"-Behauptung; gesund → altes Verhalten wortgleich);
Köder FK3 in beide Richtungen gebissen (Wahrheits-Abfrage leer →
rot), Restore md5-verifiziert. Lint-Basis unverändert 3049.

---

## 36an. Die Entwarnungs-Klasse — und die Suche, die sie findet (W280)

Kiras „such in den bereichen nochmal nach und was es noch gibt und
verbessere die suche". W279 war kein Einzelfall, sondern eine KLASSE:
Ruhe-Behauptungen, die aus leeren LISTEN entstehen statt aus einem
Blick auf den Zustand. Eine leere Aktivitäten-Liste, ein leeres
Prioritäten-Register, ein leeres Hypothesen-Brett, ein leeres
Dienst-Verzeichnis — jedes davon wurde irgendwo zu „alles ruhig",
„Alles läuft!~" oder „im grünen Bereich" aufgeblasen. Leerer Speicher
ist kein Beweis für ein ruhiges System (dieselbe Figur wie W162).

**Der Wurzel-Fix sitzt am Engpass:** `active_issues_live` (die
Live-Problem-Liste im Snapshot) wurde von Events gefüttert — Thermik,
volle Platte, toter Dienst, Backup, SMART — aber NIE vom Schonmodus.
Deshalb schrieb der Vision-Prompt dem LLM „SYSTEM-STATUS: Alles läuft
rund — sei positiv und entspannt, NICHT Probleme suchen" MITTEN im
Schonmodus. Jetzt mischt der Snapshot-Bau die toten Mounts aus der
Mount-Wahrheit ein (`_issues_mit_mount_wahrheit`): ein Draht, alle
Snapshot-Leser sehen es, und beim Mount-Comeback verschwinden die
Einträge von selbst, weil `kaputte()` dann leer ist.

**Sechs Melde-Stellen gefixt:** Dienste-Antwort „Alles läuft!~"
(Dienste-Liste ≠ System), Beliefs-Fast-Path „läuft bei mir alles
ruhig" (leeres Dienst-Register), Problem-Historie „Alles ruhig
gewesen~" (Vergangenheits-Bilanz verschwieg das offene Jetzt),
Aktivitäts-Antwort und Prioritäten-Antwort („grüner Bereich" aus
leerer Liste) — alle fragen jetzt zuerst die Mount-Wahrheit und
führen mit dem Täter-Namen. Der Hypothesen-Text behauptet nichts
mehr über das System, sondern nur noch über seinen eigenen Zettel.

**Die verbesserte Suche ist ein Gesetz** (`test_w280_gesetz_…`):
AST-Scan über persona/ — eine Ruhe-Phrase in Botschafts-Position
(Return, .append, +=) verlangt eine Zustands-Quelle in derselben
Funktion (mount_wahrheit, warnungs_buch, active_issues_live,
darf_entwarnen, …). Basis: zwei geprüfte Ausnahmen (Aussage über die
EIGENE Kognition mit eigener Evidenz; Anwesenheits-Frage an Kira),
rottet nicht. Dokumentierter blinder Fleck: das Gesetz prüft die
ANWESENHEIT der Quelle, nicht ihre richtige Verwendung — das leisten
die Funktions-Proben (`test_w280_entwarnungs_…`, drei Stellen in
beide Richtungen). Köder FL1/FL2 in beide Richtungen gebissen
(nackte Entwarnung eingepflanzt → Gesetz rot; Mount-Gate entfernt →
Probe rot), Restores md5-verifiziert. Lint-Basis unverändert 3049.

---

## 36ao. Der Draht, end-to-end nachgemessen (W281)

Kiras „Baue es und schaue nochmal genau hin obs wirklich nicht
verdrahtet ist und was alles davon abhängig ist oder beeinflusst" —
die W280-Behauptungen mit der W278-Härte nachgeprüft, diesmal ohne
jede Abschneidung:

**Bestätigt:** (a) es gab tatsächlich NIE einen Mount-Schreiber für
`active_issues_live` — die vollständige Ruferliste von
`_add_live_issue` ist Thermik, Platte voll, Service tot, Config,
Backup, SMART; (b) der Vision-Prompt liest sein `merged` wirklich aus
`_get_state_snapshot` — der Engpass-Draht erreicht ihn, und die
Overlay-Logik überschreibt nur leere Felder; (c) `kaputte()` ist ein
reiner Read-Model-Zugriff auf den Schonmodus (keine I/O, keine
Zirkularität, vernachlässigbare Kosten im 3-s-Snapshot-Takt);
(d) `get_unified_snapshot` liefert pro Aufruf eine frische Kopie
(R488) — der Merge kontaminiert keinen geteilten Cache (R495-Klasse
geprüft); (e) der zweite Prompt-Leser (Raum-Beschreibung) profitiert
automatisch: ein toter Mount ist jetzt eine „aktive Baustelle" statt
„alles ruhig, kühl und aufgeräumt".

**Zwei Abnehmer liefen am Draht vorbei — angeschlossen:**
`profile()` (speist die Web-API; las `_system_state` direkt und hätte
den toten Mount im Dashboard-Profil verschwiegen, während der Chat
ihn längst nennt) und der Fallback-Zweig von `_get_state_snapshot`
(ohne StateManager; flache Kopie statt Mutation, damit der Cache rein
bleibt und die Einträge pro Abruf frisch aus `kaputte()` kommen).

Gesetz `test_w281_…`: echte, offline GEBAUTE Myuri — toter Mount
erscheint im Snapshot (Haupt- UND Fallback-Zweig), der LLM-Prompt
verliert die „sei positiv"-Anweisung und trägt die Problem-Zeile,
das Web-Profil nennt den Mount; Comeback räumt überall auf. Köder FM
in beide Richtungen gebissen (Engpass-Merge neutralisiert → E2E-Probe
rot), Restore md5-verifiziert. Lint-Basis unverändert 3049.

---

## 36ap. Korrigiert und richtig erweitert (W282)

Kiras „dann korrigiere es und erweitere es richtig" auf die
W280/W281-Zustands-Synthese.

**Korrigiert:** die synthetischen Einträge trugen bei jedem
Snapshot-Bau `time.time()` — für jeden Leser sah der Zustand damit
immer frisch aus, das ALTER der Episode war unlesbar. Jetzt ist der
Zeitstempel der echte Episoden-Beginn (`since` aus dem Schonmodus)
und der gemessene Grund (`reason`) steht im Text.

**Richtig erweitert:** die Synthese kennt jetzt alle drei offenen
Zustände, die auch der 7. Verdrahtungs-Vertrag sieht —
`mount_tot:<pfad>` (Schonmodus), `backup_stuck:<quelle>` (steckendes
Paar) und `proben_cap:<key>` (Karteileichen-Slots voll, Mount
unbeurteilbar). Für die steckenden Paare wurde die BESTEHENDE
Autorität erweitert statt eine neue gebaut (W253):
`backup_truth.steckende_pfade` teilt sich den W275-5-Sekunden-Cache
mit `warte_pfade` über einen gemeinsamen Kern (`_pfade_frisch`) —
`get_backup_health` läuft für BEIDE Fragen zusammen höchstens einmal
je 5 s, der Snapshot-Takt zahlt nichts doppelt (testfixiert mit
Zähl-Probe).

**Das Entwarnungs-Gesetz scannt jetzt auch communication/** (W282).
Die Ausweitung fand genau einen Riss: der Neustart-Melder sagte
„**{svc}** wurde erfolgreich neugestartet! Alles läuft wieder~" —
eine System-Aussage aus dem Exit-Code EINES Dienst-Neustarts. Der
Satz spricht jetzt nur noch über den Dienst, den er neugestartet hat
(dieselbe Kur wie beim Hypothesen-Text in §36an).

Gesetze `test_w282_…` (Synthese-Probe: echter Episoden-Beginn, Grund
im Text, stuck- und Cap-Einträge, unter dem Cap nichts, gesund leer,
ohne Manager kein Wurf; Cache-Probe: EIN get_backup_health für beide
Fragen); Köder FN in beide Richtungen gebissen (stuck-Quelle aus der
Synthese → Probe rot), Restore md5-verifiziert. W275-Test auf das
4er-Cache-Tupel nachgezogen. Lint-Basis unverändert 3049.

---

## 36aq. Selbst-Audit der dreizehn Commits — eine Schwäche, richtig gezogen (W283)

Kiras „schaue nochmal alle commits durch und verbessere die alle und
suche danach schwach gebaute". Alle dreizehn Commits dieses Fensters
(W270–W282) noch einmal kritisch durchgesehen.

**Bilanz:** W270–W273 (Mount-Kette) und W274–W277 (Fundament,
Performance, Vertrag, tote Funktionen) hielten — sie waren durch die
W274- und W278-Audits bereits zweifach geprüft. Die echte Schwäche
steckte in der jüngsten Arbeit (W280–W282): die Zustands-Synthese
lief UNTER `self._lock` der Persona — im Snapshot-Merge (beide
Zweige) und in `profile()`. Ihr stuck-Anteil konnte bei Cache-Miss
`get_backup_health` mit Worker-Proben kosten: auf einem steckenden
Mount Sekunden, gehalten am Haupt-Lock, an dem Chat und
Event-Handler hängen (exakt die Paket-H-Klasse aus Block C/W16).

**Der Klassen-Sweep dahinter** (die „schwach gebaute"-Suche) fand,
warum ein bloßes „außerhalb des Locks" nicht gereicht hätte:
18 Alt-Rufer rufen `_get_state_snapshot` ihrerseits unter
`self._lock` (RLock, reentrant) — die Synthese liefe dort trotzdem
unter deren Lock-Hold. Fix auf der richtigen Ebene: die Synthese
kann jetzt NIE mehr blockieren. `steckende_pfade_gecacht()`
(backup_truth) liest nur den gemeinsamen W275-5s-Cache
(Mikrosekunden, nie ein Fetch); frisch halten ihn die BESTEHENDEN
warte_pfade-Rufer im Prompt-, Wake- und Wartungs-Takt. Zusätzlich
halten die drei W283-Stellen das Lock während der Synthese nicht
mehr (Rohdaten unter Lock kopieren, Synthese draußen, Cache-Zuweisung
wieder drinnen).

**Eigenfund beim Bauen des Fixes:** die erste Fassung des
Cache-Lesers las den Stand OHNE Alters-Prüfung — ein alter
Cache-Stand hätte steckende Pfade für immer weiterbehauptet, wenn
kein Takt mehr auffrischt (die volle Suite hat genau das sofort
gezeigt: ein früherer Test vergiftete den Modul-Cache für zwei
spätere). Jetzt respektiert der Leser ein Höchstalter (Default
10 min); ist der Stand älter, sagt er die EHRLICHE leere Menge —
„Stand unbekannt" statt überholter Behauptung.

Gesetz `test_w283_…` (3 Proben: die Synthese ruft nie die holende
Variante — ein Spion explodiert, wenn doch; ein RLock-Spion beweist,
dass der Persona-Lock während der Synthese im Snapshot-Weg nicht
gehalten wird; ein uralter Cache-Stand liefert leer statt Behauptung);
Köder FO1/FO2 in beide Richtungen gebissen (Hol-Variante zurückgebaut
→ Probe 1 rot; Synthese wieder unters Lock → Probe 2 rot), Restores
md5-verifiziert. Lint-Basis unverändert 3049.

---

## 36ar. Die fünf offenen Punkte, architektonisch untersucht (W285)

Kiras Auftrag: die fünf offenen Punkte aus der Ehrlichkeits-Bilanz
untersuchen und für die Hardcode-Relikte bessere Methoden bauen —
Ableitung aus der gereiften Architektur statt handgepflegter Listen.

**Punkt 1 — die hopeless-Keys.** Untersuchung ergab: das Relikt war
kein Vokabular-Problem, sondern die write-only-Klasse. Die vier
hartkodierten `_<domäne>_all_hopeless`-GOAP-Beliefs hatte seit R889
NIEMAND je gelesen — kein Precondition-Dict, kein Suppressor, kein
Test (erschöpfend verifiziert, auch dynamische Baumuster); ihre fünf
Geschwister waren genau dafür längst entfernt worden (R910
„redundant + irreführend", R920 „Doppel-Track", R927). Die echten
Konsumenten fragen die Checklist-AUTORITÄT direkt: goal_generator
(R915-Suppress) liest `checklist_summaries`, setup_mixin ruft die
lebende Checkliste, R894/R890/R887 nutzen Live-Probes. W285 vollendet
die Bereinigung: die vier Schatten sind raus (der Riese
`snapshot_to_goap_state` schrumpft), und das Gesetz hält die Klasse
auf null. Die `checklist_sync`-Keys (`drive_*`/`nfs_`/`service_
all_hopeless`) sind LOKALE Übergangs- und Dedup-Zähler — eigener
Zweck, eigene Datei, kein geteiltes Vokabular: die in W278 vermutete
„Drift" löst sich auf, es gab nie ein gemeinsames Vokabular zu
vereinheitlichen.

**Punkt 2 — die zwei falsch einsortierten Schreib-Aktionen.**
`remount_check` (führt `mount` aus) und `system_config_backup`
(schreibt Backup-Dateien) galten als „Diagnose" — über den
`_check`-Namens-Marker bzw. den R939-Backstop, obwohl beide
Herkunfts-Kommentare „keine Diagnosen" sagen. Folgen: ein
erfolgreicher remount befriedigte seinen Antrieb nur schwach
(effective=False wie eine Diagnose) und beide fütterten Diagnose-
Historie und Diagnose-Wert-Lernen. Jetzt: explizite
`_KEINE_DIAGNOSE`-Ausnahme (schlägt Marker UND Backstop) + Aufnahme
in `_IMPACT_ZERO_BY_DESIGN` — der Schutz vorm Nutzlos-Urteil bleibt
(praktisch verifiziert), die G2-Präzedenzfälle (journal_vacuum,
smart_short_check) bleiben unverändert.

**Punkt 4 — die Lock-Hüllen.** Von den 18 Alt-Rufern umschlossen elf
NUR den Snapshot-Aufruf — reine Redundanz, denn der Snapshot sichert
sich intern selbst (seit W283 mit lock-freier Synthese). Die elf
Hüllen sind zeilenneutral entfernt (Riesen-Gesetz gewahrt); die
restlichen sieben schützen in ihren Blöcken WEITERE Zustände und
bleiben bewusst stehen. Das W285-Gesetz verhindert die Rückkehr der
redundanten Hülle.

**Punkt 3 — Teil-Zweige (bewertet, kein Umbau):** die Mount-Nennung
sitzt in den Entwarnungs-Zweigen; die Alarm-Zweige zeigen ohnehin
Probleme und speisen sich aus der Live-Liste, die seit W280/W282 die
offenen Zustände enthält. Ein zusätzlicher Mount-Satz in jedem
Alarm-Zweig wäre Doppel-Nennung derselben Quelle.

**Punkt 5 — Gesetzes-Grenzen (bewertet, bleiben):** dokumentierte
Messgeräte-Grenzen, keine Baustellen — Toter-Code-Gesetz sieht keine
dynamisch gebauten Namen (kamen im Bestand nicht vor), Entwarnungs-
Gesetz prüft Quellen-Anwesenheit (die Funktions-Proben prüfen die
Nutzung), die Erreichbarkeits-Diagnose läuft als Laufzeit-Variante
im wöchentlichen TÜV statt als statischer Scanner.

Gesetze `test_w285_…` (write-only-hopeless-Klasse auf null + echter
Konsument nachweisbar + Lock-Hüllen-Klasse auf null; Diagnose-
Umsortierung mit G2-Wächtern); Köder FP1/FP2 in beide Richtungen
gebissen (write-only-Belief eingepflanzt → rot; Nicht-Diagnose-
Ausnahme entfernt → rot), Restores md5-verifiziert. Lint-Basis
unverändert 3049.

---

## 36as. Sie kennt sich, sie versteht „aus", sie erfindet keine Zahlen (W286)

Kiras drei Live-Befunde vom 14.08., per Offline-Befragung der echten
Myuri verifiziert und an der Wurzel behoben.

**Selbst-Wissen erreichbar + ehrlich (Paket 1).** „Wie bist du
aufgebaut?", „welche Schichten hast du?", „was für Module hast du?"
liefen ins ehrliche Ausweichen, obwohl die Antwort existiert
(`self_schema.describe_cognition_flow`): die Self-Router-Tabelle
kannte nur „deine schichten"-Formulierungen und matchte per rohem
Substring — die W170-Wendungs-Klasse, diese Tabelle war beim
174-Stellen-Umbau nicht dabei. Jetzt: die `how`-Muster um die
Aufbau-Formulierungen erweitert UND (nur) diese Kategorie
wendungstolerant („wie bist du EIGENTLICH aufgebaut?" trifft).
Dazu die W253-Klasse über sich selbst: der Selbst-Text endete mit
eingefrorenen R613-Zahlen („23 Layer, 101 Subsysteme" plus eine
längst veraltete Drive-Zahl) — die kanonische Karte kennt 37 Layer
in 6 Hierarchien. Der
Inventar-Schluss kommt jetzt live aus `core/architecture_layers`
(Zahlen UND Hierarchie-Ketten), Misslingen wird ehrlich benannt
statt alte Zahlen zu zeigen. Vertrag: die 12 kanonischen
Cognitive-Layer-Namen müssen in der Prosa vorkommen — Drift wird rot.

**Deaktivieren ist jetzt ein Handgriff (Paket 2).** „deaktiviere die
anomalie prüfung" bekam den Gesundheits-Status (die Health-Route
fraß „prüfung"), „deaktivir die warnungen" die Warnungs-ANZEIGE —
ein AUFTRAG wurde als Auskunfts-FRAGE gelesen, weil kein Handler
das Verb kannte. Neue Autorität `persona/deaktivieren_verstehen.py`
(Aktion × Ziel × Dauer): Wortstamm-Präfixe fangen die Live-Tippfehler
(„deaktivere", „deaktivir"), trennbare Verben („mach … aus")
unterscheiden Verb-Partikel von Präposition wie das Deutsche selbst
— Partikel steht satzfinal, „mach ein backup AUS den fotos" bleibt
ein Backup-Auftrag. W-Fragen, Zustandsfragen („ist … deaktiviert?"),
Verneinungen und NAS-Shutdown-Ziele bleiben ihren Routen. Der echte
Handgriff: `NotificationManager.user_stumm_setzen/aufheben/status`
— Neustart-fest (W252-Prinzip), Gate in `send()` NACH dem
Warnungs-Buch (stumm heißt leise, nicht blind: das Buch zählt
weiter) und nur `critical` durchbricht die Stille (Kiras Regel:
Sicherheit vor Ruhe). Chat-seitig hängt der Handler in Phase 0a vor
beiden Kaperern; eine einzelne Prüfung abschalten kann sie NICHT und
sagt das (W164), statt eine Auskunfts-Route antworten zu lassen.

**Zahlen-Regel im Platten-Prompt (Paket 3).** „bei 100 GB Gesamt …
rund 12 GB frei" neben einer 1-TB-Platte war LLM-Erfindung: der
Prompt lieferte pro Platte nur Gefühls-Text und Prozent, die
Absolutwerte dichtete das Sprachmodell dazu. `_w286_platten_zeilen`
(pur, testbar) schreibt jetzt „X GB frei von Y GB gesamt (Z%
belegt)" aus den echten `disk_stats` in den Prompt plus die
Zahlen-Regel: nur diese Werte nennen, nichts umrechnen, nichts
erfinden. Fehlen die Zahlen im Snapshot, bleibt es beim Gefühls-Text
— es wird keine dazugedichtet.

Abnahme: 18 neue Vertrags-Tests (Köder in beide Richtungen: Struktur-
Fragen treffen, System-Fragen bleiben ungekapert; Aufträge muten,
Fragen/Verneinungen/Präpositionen nicht; critical durchbricht das
Stumm-Gate per Sentinel bewiesen), volle Suite, Lint-Basis
unverändert (neue Dateien 0 Befunde).

---

## 36at. Der NAS-Tag im Log — fünf Wurzeln, mit Beweis (W287)

Kiras Lieferung (Brain-Log 06:36–19:55, Warnungs-Buch,
nas_cognitive.db, pattern_events, ps-Ausgabe) machte aus fünf
Meldungen fünf bewiesene Wurzeln. Zwei Verdachte aus der Voranalyse
wurden dabei ehrlich widerlegt: die Registry trug null gelernte
Aktionen (kein Effekt-Überschreiber am Werk), und es gab keinen
D-State-Kindprozess (kein hängendes rsync).

**1. TÜV-Fehlalarm auf der docker-losen Maschine.** Log 06:36:17:
„Befehl 'docker' fehlt", 06:36:46: sieben Docker-Aktionen per
Capabilities-Filter raus — übrig blieb nur `service_cascade_recovery`
mit `service_unhealthy=True`-Vorbedingung, exakt der gemeldete
producible_unmet. Der Antrieb kann ohne `has_docker` aber nie
spiken: der Erreichbarkeits-Scan prüft jetzt mit Maschinen-Wahrheit
(dieselben Beliefs wie das DriveSystem, W253: eine Autorität) —
explizit False → „inaktiv", Beliefs noch leer (Boot-TÜV) →
„unbewertet", nur aktive Antriebe alarmieren.

**2. Der Memory-„critical" war ein Schonfrist-Durchschlag.**
07:21:58 unterdrückt die Boot-Schonfrist 97,9 MB/h („File-Index-
Scan-Beule, kein Leck") — 65 Sekunden später schlägt derselbe Hügel
als „critical: 100,5 MB/h" durch, weil der critical-Pfad die
Schonfrist umging (die W221-Phantom-Schonfrist-Klasse). Tagesbeweis:
alle Folgemessungen drift=0.0, RSS stabil ~800 MB. Jetzt gilt EINE
Schonfrist für alle Stufen; nur ein echter Runaway (≥3× critical,
die R1017-Klasse mit 14000 MB/h) durchbricht sie.

**3. Die Wartungs-Spirale: der Lerner lernte seine eigene Taktung.**
„Dringend: timer_check (Stark überfällig (0.0h her))" —
`smart_unmount_backups` lief 168× im 4-Minuten-Takt, den ganzen Tag.
Das „gelernte" Intervall ist der Durchschnitt der EIGENEN
Ausführungs-Abstände: ein dichter Boot-Rückstau schrumpft es, macht
früher „fällig", die Abstände schrumpfen weiter — die
Peer-Selbst-Echo-Klasse als Wartungs-Spirale. Boden: 1/4 des
Task-Richtwerts, nie unter 1 Stunde. Dazu die zweite Lücke: der
Maintenance-Pfad lief nach „Schonmodus aktiviert" (17:13) einfach
weiter und weckte die Platten — der Schonmodus-Wächter saß nur am
GOAP-Weg. Dieselbe Autorität (degraded_mode + _STORAGE_BLOCK) gilt
jetzt auch dort, mit dem 45-min-Blocked-Backoff statt 4-min-Spam.

**4. „Backup already running" ohne Lebenszeichen.** 19:43 prallten
fünf Starts an einem Flag ab, dessen Lauf seit ~19:07 im toten
/Nas/Anime2 hing — das `finally` kommt bei D-State nie, und die
Abweisung tat, als liefe alles. Jetzt tragen Läufe Start- und
Fortschritts-Stempel (an den vier echten Lauf-Heartbeats); eine
Abweisung nach >10 min Stille sagt „Backup-Lauf hängt: seit X min
kein Fortschritt — vermutlich blockiert ein toter Mount". Das Flag
wird bewusst NICHT still zurückgesetzt (der hängende Thread lebt;
ein Doppel-rsync wäre gefährlicher als Warten) — aber die Wahrheit
steht in der Antwort.

**5. backup_stale riet, backup_impossible wusste.** 19:08 erzählte
die Kausal-Schablone „vermutlich kein geeignetes Zeitfenster",
während backup_impossible „Mounts fehlen" kannte und /Nas/Anime2
seit 17:13 tot gemeldet war. Backup-Stuck-Eskalationen fragen jetzt
zuerst `w287_backup_blocker_detail()` (steckende Pfade aus dem
W282/W283-Cache, dann Schonmodus); die Schablone bleibt Fallback
und trägt ihr „vermutlich" dann zu Recht.

**Zwei Gesetze obendrauf.** (a) Effekt-Unantastbarkeit:
`sync_dynamic_actions` durfte Effekte STATISCHER Aktionen in-place
überschreiben — auf Kiras NAS nicht gebissen (Registry leer), aber
die W253-Klasse in ihrer gefährlichsten Form; jetzt sind statische
Effekte Grundwahrheit, nur Kosten bleiben lernbar, Versuche werden
geloggt. (b) Worker pro Mount: der Probe-Broker-Schlüssel war
`MountIO:<voller Pfad>` — Kiras Zensus zeigte 18 Reader bei ~5
Mounts; der Schlüssel fällt jetzt berührungsfrei auf die
Mount-Wurzel (risky-roots aus Config + /proc/self/mounts,
längster Präfix; ohne Treffer fail-open der volle Pfad).

Und der Tag zeigte auch, was TRÄGT: der Selbst-Täuschungs-Wächter
fing um 11:33 live die Phantom-Vollzüge („Drei meiner Ziele haben
sich 'erledigt', ohne dass ich eine einzige Aktion ausgeführt
habe"), Schonmodus-Eintritt, Mount-tot-Meldung und die
Stuck-Eskalationen selbst liefen wie gebaut.

Abnahme: 19 neue Vertrags-Tests, alle mit Ködern in beide
Richtungen (inaktiver Antrieb schweigt / aktiver alarmiert;
Grenz-critical wartet / Runaway durchbricht / nach Schonfrist
scharf; Selbst-Echo macht nicht fällig / echte Überfälligkeit
bleibt; hängender Lauf wird benannt / lebender bleibt „already
running"; statische Effekte unantastbar / dynamische lernbar;
Mount-Wurzel-Key mit Präfix-Schutz). Lint-Basis unverändert.

---

## 36au. Das Frische-Gesetz — Wissen bindet sich an Realität (W288)

Kiras Verallgemeinerung nach W287: „Die Prozesse und das Wissen sind
nicht an Realität gebunden, ihr Wissen ist veraltet — solche Probleme
hatten wir schon, wo Realität fehlte." Das ist keine neue Einzelwurzel,
sondern die KLASSE hinter mehreren alten: der 14.08 hat sie viermal
gezeigt (der Lauf-Thread glaubte einem toten Datei-Handle, das
running-Flag glaubte einem toten Lauf, backup_impossible glaubte einem
längst geheilten Mount-Ausfall, die Eskalation riet, statt zu messen).
W288 zieht sie an vier Stellen:

**P1 — die letzte rohe Mount-Berührung fällt.** Der Automount-Wecker
(R1152ge, `os.scandir` im Lauf-Thread) war die bewusst geduldete
Ausnahme der W270–W274-Kette — und exakt die Stelle, an der der Lauf
am 14.08 im toten Alt-Handle starb (D-State, `finally` kam nie).
Derselbe Weck-Effekt (echter File-Access, den systemd-automount
sieht) läuft jetzt als `read_probe` im Wegwerf-Worker
(`_quell_mount_wecken`): hängt der Mount, stirbt der Worker — nicht
der Lauf, und die Abweisung trägt BLOCKED-Semantik. Der Zwilling im
Executor (`_trigger_automount`) und der rohe Bestands-Zähler der
Wartung (`_count_entries` → `safe_fs.count_files`) sind mitgezogen.

**P2 — das Bau-Gesetz sieht jetzt auch `scandir`.** Die Form, über
die P1 lief, fehlte im `_RAW_IO`-Muster — ein Wächter, der eine
Syntax zählt, schützt nur die Syntax, an die sein Autor dachte (die
B17-Klasse). `scandir` ist jetzt im Muster; die Ausnahme-Liste wurde
ehrlich neu vermessen statt gedeckelt (inklusive des Vermerks, welche
Restposten Kandidaten für den Worker-Umzug bleiben) und darf wie
immer nur schrumpfen.

**P3 — die Blockiert-Kategorie: kein Anrennen gegen Bekanntes.**
Die W287-Hänger-Abweisung war ehrlich, zählte aber als FEHLSCHLAG —
fünf Starts → „start_backup 5× fehlgeschlagen"-Eskalation, obwohl
das Hindernis bekannt war. Sie beginnt jetzt mit „Backup
verschoben:" → `message_is_blocked` (R1435/R1358g) stuft sie als
BLOCKED ein: kein Lerner-Gift, kein Retry-Sturm. Und die
Zustands-Synthese SIEHT den hängenden eigenen Lauf: der Manager
exportiert ihn über einen Weakref-Lesehaken
(`w288_lauf_haenger_text`, gespeist aus derselben Messung
`_lauf_haenger_werte` wie die Abweisung — keine zweite Rechnung,
W253), und `w287_backup_blocker_detail()` nennt ihn als dritten
echten Blocker nach steckenden Pfaden und Schonmodus. Genau die
Lücke des 14.08-Abends: Mount längst zurück, beide W287-Quellen
leer, nur der eigene alte Lauf hing — und die Eskalation riet.

**P4 — Issue-Frische: alte Behauptungen müssen sich re-validieren.**
`backup_impossible` („Mounts fehlen") stand 12 Stunden im Buch und
eskalierte weiter, während die Mounts seit 17:14 zurück waren. Die
Kontrafaktual-Heilung (R1357) brauchte einen Belief-Key des Effekts —
Effekte ohne eigenen Key konnten NIE heilen; das Wissen war nicht an
die Realität gebunden. Jetzt gilt: ein Issue, das älter als zehn
Minuten ist und keinen Belief-Key hat, re-validiert seine Grundlage
(gedrosselt, alle fünf Minuten) gegen DIESELBE Autorität, die es
begründet hat — für die Mount-Klasse `mount_wahrheit.kaputte()` +
`steckende_pfade_gecacht()`, beides berührungsfrei. Ist die
Grundlage messbar weg → ehrliche Auflösung (`w288_issue_grundlage`
= „widerlegt") samt Chronik-Eintrag; war das Issue schon beim User
eskaliert, bekommt er auch die Entwarnung — sonst bleibt seine
letzte Karte für immer „Stuck-Issue". Drei Grenzen halten das
Gesetz ehrlich: ein weiterhin meldender Sensor hat Vorrang (kein
Auflösen gegen einen lebenden Belief), junge Issues behalten ihre
Chance auf Fix-/Belief-Heilung, und ohne billige Realitäts-Quelle
gibt es KEINE Aussage — aufgelöst wird nie auf Verdacht.

**Nebenbefund derselben Abnahme:** die W287-Einbauten hatten drei
eingefrorene Riesen wachsen lassen (Riesen-Gesetz W248, der einzige
rote Test des Laufs). Statt die Basis zu heben, wurde herausgezogen:
Schonmodus-Gate → `_task_storage_action_geschont`, Automount-Wecker
→ `_quell_mount_wecken`, Heartbeat-Duo → `_lauf_fortschritt(wd)` —
alle drei Riesen wieder auf oder unter ihrer Basis.

Abnahme: neun neue Vertrags-Tests mit Ködern in beide Richtungen
(Hänger-Abweisung ist BLOCKED / lebender Lauf bleibt Fehlschlag-
Klasse; Synthese nennt den hängenden Lauf / lebender Lauf ist kein
Blocker / ohne Manager keine Aussage; Grundlage widerlegt →
Auflösung / bestätigt → Issue bleibt / ohne Quelle keine Aussage /
junges Issue bleibt / lebender Sensor hat Vorrang), dazu die
verschärften Bau-Gesetz-Köder. Lint-Basis unverändert (zwei
Findings weniger durch den scandir-Wegfall).

---

## 36av. Frische überall — der Sweep über alle Bereiche (W289)

Kiras Auftrag nach W288: „Baue die fehlenden Sachen direkt, baue
Verträge und Wächter, ob solche Probleme irgendwo weiter existieren —
auch in anderen Bereichen. Und prüfe weitere Reads, ob deren Quellen
richtig sind." Sechs parallele Untersuchungs-Scouts plus eigene
Vermessung, jeder Befund einzeln handverifiziert. Ergebnis: die
Klasse lebte an weit mehr Stellen als der Backup-Kette.

**Das Frische-Gesetz als Wächter (test_w289_frische_gesetz.py).**
Ein AST-Scan über die Denk-/Heil-/Wahrnehmungs-/Melde-Bereiche
findet jeden Instanz-Speicher, dessen Name eine Realitäts-Behauptung
trägt (issue/alert/baseline/blacklist/belief/verdict/failed/stale/…).
Jeder Treffer MUSS im Manifest stehen — mit dem Mechanismus, der ihn
frisch hält (ttl / revalidiert / ereignis / gedeckelt / bewusst
zeitlos mit Begründung), und einem Code-Beleg. Ein neuer Speicher
ohne Eintrag ist rot; ein Eintrag ohne Speicher ist rot. Jede Zeile
des Start-Manifests wurde in dieser Welle einzeln gegen den Code
verifiziert; drei angebliche Wissens-Speicher stellten sich dabei
als tot heraus (nie beschrieben, nie gelesen) und wurden entfernt
statt klassifiziert.

**Die größten Funde, direkt gebaut (keine Pflaster):**

1. **Die tote SMART-Kette.** Der DiskFailurePredictor (Ausfall-
   Prognose, Baseline-Drift) bekam seit je NIE Daten — niemand rief
   feed_smart_data, während das GOAP-Gehirn `fc["disk_health"]` an
   mehreren Stellen las und die ewig leeren Scores als „keine
   Platten-Probleme" glaubte. Jetzt speist die EINE Messung
   (SmartMonitor) den Predictor per SmartStatusUpdated-Event —
   dieselbe Quelle wie die Platten-Chronik. Dazu der
   Identitäts-Anker: fällt Power-On-Hours massiv unter die Baseline,
   steckt eine getauschte Platte hinter dem Pfad → Baseline und
   Verlauf werden ehrlich neu verankert statt gegen die alte Platte
   zu vergleichen.

2. **Blacklists, die nie zurückkonnten.** (a) Der 5-min-Recheck der
   Kommando-Blacklist prüfte immer nur die ersten zehn der
   sortierten Liste — ab elf Einträgen wurden die hinteren NIE
   re-gemessen; jetzt rotiert das Fenster. (b) Das Rehabilitations-
   Wissen der Action-Blacklist war RAM-only, die Sperre persistent —
   jeder Neustart machte sie faktisch endgültig; beides überlebt
   jetzt zusammen. (c) Das „absichtlich deaktiviert"-Dienste-Buch
   reconciled beim Boot echt gegen die systemctl-Liste (reaktivierte
   Dienste fallen raus, User-Overrides bleiben), und die Persistenz
   hat erstmals einen Löschweg (kv_delete — vorher konnte KEIN
   Buch-Eintrag die DB je wieder verlassen).

3. **Gelerntes verliert seine Stimme, wenn es altert.** Kausal-Kanten
   waren ab der dritten Beobachtung unsterblich (Konfidenz klingt
   jetzt täglich ab, wenn lange unbestätigt); „bester Fix"-Zähler von
   vor Monaten urteilten mit voller Stimme (Recency-Faktor über
   last_used); Goal-Erfolgs-/Fehlschlags-Zähler prägten für immer
   (Wochen-Halbierung, Uhr persistiert); ErrorDNA-Fehlerprofile
   stimmten zeitlos ab (Similarity halbiert sich pro 30 Tage);
   die Frische-Re-Validierung aus W288 kennt jetzt auch die
   no_backup_window-Klasse (Basis-Belief wird nachgemessen);
   Vorhersagen im LLM-Prompt tragen ein Maximalalter; die
   Docker-Kaskaden-Regel zählt nur noch Fehler der letzten Stunde.

4. **Die W287-Flag-Klasse, überall.** Der Code-Auditor-Guard und das
   Metrik-Sammler-Flag waren nackte Bools ohne Lebenszeichen (ein
   Hänger hätte beide für immer stummgeschaltet) — beide folgen jetzt
   dem W287-Muster (Zeitstempel + is_alive/Verfall mit WARNING).
   Verdrängte Kausal-Issues (LRU-Cap) werden dem Neuland-Buch
   gemeldet — ehrlich als „verworfen", NICHT als gelöst (kein
   Phantom-Fall-Wissen); SMART-Risiko-Verdikte verfallen, wenn das
   jüngste Sample älter als 48h ist; gelöschte Dateien alarmieren
   die Sicherheits-Wache EINMAL laut statt alle fünf Minuten für
   immer (Wiederauftauchen meldet erneut); SYSTEM_REALITY wird
   spätestens nach 60 min auch unter Stress neu gemessen.

5. **Reads mit falscher Quelle (Kiras zweiter Auftrag).** Der
   wait_for_backup-Handler und der zentrale is_backup_running-Helper
   glaubten dem eingefrorenen Volatile-Spiegel — beide fragen jetzt
   die W288-Hänger-Messung; die Abweisung eines LEBENDEN Laufs ist
   jetzt ebenfalls „Backup verschoben:" (vorher stempelte der
   abgeprallte Zweitstart last_backup_status auf failed, WÄHREND das
   echte Backup gesund lief); die Mount-Kaskaden-Vorhersage las
   mounts_ok aus einem Dict, in das es nie geschrieben wird (sieht
   jetzt Beliefs UND GOAP-State); ein Chat-Baustein las den
   Geister-Key is_running (Producer schreibt active); der
   Vision-Prompt meldet Backup-Fortschritt mit Frische („wächst" /
   „seit Xs kein Zuwachs" / Hänger-Satz) statt zeitlosem „X%
   fertig"; Service-Live-Issues re-validieren beim Snapshot-Bau
   gegen die Dienst-Wahrheit (vorher löschte sie NUR der eigene
   restart_-Erfolg — externe Erholung ließ „Service X ausgefallen"
   ewig stehen; NotificationManager und ProactiveMessenger hören
   jetzt zusätzlich auf ExternalRecoveryObserved, und die
   W140-Erholungs-Schärfung des Warnungs-Buchs gilt auch für
   Dienste); Storage-Beliefs-Restore bei fehlender StorageDiscovery
   hat einen 15-min-Deckel; Werkzeug-Antworten messen die Top-Treffer
   gegen die Maschine, bevor sie Anwesenheit behaupten (Nachtrag:
   der erste Bau FILTERTE verschwundene Werkzeuge schon im Routing —
   damit brachen die W125–W127-Gesetze, weil „Alltagssprache erreicht
   das Werkzeugwissen" plötzlich von der Testmaschine abhing; jetzt
   heftet find_tool_for_task nur ein da-Flag an, und die Modul-
   Funktion _w289_werkzeug_antwort an der ANTWORT-Grenze behauptet
   nur Auffindbares positiv, benennt Verschwundenes ehrlich und
   überlässt installierbare Fälle dem W128-Tor); die
   Wissensbasis gibt Realitäts-Kategorien ein Default-Verfallsdatum
   (30 Tage), womit ihr Docstring-Versprechen erstmals wahr ist;
   die Floor-Stempel des Melde-Tors räumen im Lockstep mit dem
   Betreff-Buch.

Geprüft und sauber (Auszug aus den Scout-Protokollen): die gesamte
core/-Wahrheitskette (mount_wahrheit/backup_truth/mount_health_gate/
degraded_mode/network_health_gate — TTLs und Proben überall), die
GOAP-Lernkosten (decay_learned_costs), Drive-Suppressions
(zeitgebunden), das Kontrafaktual-System, die W286/W287/W288-Bauten
selbst, und Dutzende Speicher mit Ereignis-Löschung oder Fenstern.

Abnahme: das Frische-Gesetz (Manifest über die vermessenen
Realitäts-Speicher, jede Zeile belegt) plus Köder-Verträge in beide
Richtungen für jeden Wurzel-Fix; die Riesen-Gesetz-Verstöße der
eigenen Einbauten wurden durch Extraktion beglichen (u.a.
_quell_mount_wecken, _task_storage_action_geschont,
_issues_evicten_und_melden, _w289_reprobe_ok,
_w289_sticky_storage_restore); drei tote Speicher entfernt.

---

## 37. Die agentische Schicht — 67 Werkzeuge neben dem Kern

> Diese Schicht fehlte in der Dokumentation komplett, obwohl sie der Teil ist,
> mit dem Kira im Alltag am meisten zu tun hat. Sie liegt **neben** dem
> GOAP-/Antriebs-Kern (§§5–8), nicht darin.

### 37.1 Das Grundprinzip

Alles läuft durch **eine** Werkzeug-Registratur mit drei Sicherungen:
Argument-Prüfung, **Freigabe-Gate** und Pfad-/Injektionsschutz. Ein
halluzinierendes Sprachmodell kann daran nichts vorbeischleusen — es kann nur
Werkzeuge *vorschlagen*, und die gefährlichen laufen erst nach einem „ja".

**Standard ist AUS.** Ohne `agentic.enabled=true` ändert sich am Verhalten
gar nichts. `autonomous: true` schaltet zusätzlich das Hintergrund-Pollen frei
(E-Mail, Kalender, Erinnerungen, Netz-Scan, Wartung).

**Freigabepflichtig** sind genau **12** Werkzeuge — nachgezählt, nicht
geschätzt: `install_program`, `delete_file`, `delete_empty_folder`,
`send_email`, `create_calendar_event`, `delete_calendar_event`,
`control_smart_device`, `tuya_control_device`, `run_maintenance`,
`apply_approved_resolution`, `move_to_quarantine`, `restore_from_quarantine`.
Alles, was etwas Fremdes anfasst, verschickt, installiert oder löscht.

### 37.2 Was es gibt (gemessen: 67 Werkzeuge in 21 Modulen)

| Bereich | Werkzeuge | Anzahl |
|---|---|---|
| **Dokumente** (§36) | ocr_image, sort_documents, group_documents_by_person, group_document_batch_by_sender, teach/list_document_requirements, teach_sender_document_profile, audit_sender_batch_requirements, teach_document_category, document_categories, correct_document, semantic_search | 12 |
| **Reparatur-Vorschläge** | list/explain/answer_resolution_proposal, apply_approved_resolution, propose_capability/service_restart/backup_restore, move_to_quarantine, restore_from_quarantine | 9 |
| **Bilder/Fotos** | describe_image, sort_images, teach_image_category, correct_image, image_categories | 5 |
| **Dateien** (`file_tools` + `builtin_tools`) | list_dir, read_file, write_code_file, move_file, sort_files_by_ext, delete_file, create_folder, delete_empty_folder | 8 |
| **Selbstauskunft** | assistant_selfcheck, assistant_tool_landscape, compare_tools_for_task, recommend_tool_for_goal, tool_failure_postmortem | 5 |
| **Kalender** | upcoming_events, check_calendar, create_calendar_event, delete_calendar_event | 4 |
| **Smart Home** | list_smart_devices, control_smart_device, smart_device_state | 3 |
| **Tuya** | tuya_list_devices, tuya_control_device, tuya_device_status | 3 |
| **Nutzer-Einstellungen** | set_user_pref, add_notify_rule, list_notify_rules | 3 |
| **Wartung** | maintenance_status, run_maintenance | 2 |
| **Netzwerk** | scan_network, list_devices | 2 |
| **Programme** | search_package, install_program | 2 |
| **Erinnerungen** | add_reminder, list_reminders | 2 |
| **E-Mail** | check_email, send_email | 2 |
| **Sonstiges** | process_inbox, find_missing_documents, weather, web_search, assistant_livecheck | 5 |

Bemerkenswert ist die Gruppe **Selbstauskunft**: Sie kann sagen, welche
Werkzeuge sie hat (`assistant_tool_landscape`), zwei für eine Aufgabe
vergleichen (`compare_tools_for_task`), eines für ein Ziel empfehlen
(`recommend_tool_for_goal`) — und nach einem Fehlschlag nachbereiten, warum
ein Werkzeug versagt hat (`tool_failure_postmortem`).

### 37.3 Wie eine Anfrage hierher kommt

Ein enger Auslöser-Filter entscheidet, ob ein Satz überhaupt nach dieser
Schicht aussieht (`looks_like_agentic_request`) — **im Zweifel nein**, dann
gewinnt der normale Chat-Pfad. Auf „was kannst du" / „selbstcheck" berichtet
sie, was aktiv ist und was für die fehlenden Fähigkeiten nötig wäre.

### 37.4 Ehrlich zum Stand

* Die `agentic/README.md` nannte lange **33 Tools** (Stand A1–A27) — seit
  W257 ist sie auf den gemessenen Stand gebracht (67 in 21 Modulen, 12
  freigabepflichtig) und verweist hierher als maßgeblichen Text.
* Diese Schicht ist in den Testläufen 52–55 **nicht** beobachtet worden: Sie
  war im Container nicht eingeschaltet, und ihre Sensoren (Vision-Modell,
  Kalender, E-Mail, Tuya) waren ohnehin nicht erreichbar. §33 beschreibt also
  das Verhalten des Kerns, nicht dieser Schicht.
* `semantic_search` fällt ohne Embedding-Fähigkeit auf eine Teilstring-Suche
  zurück — das ist gewollt und wird nicht als „semantisch" verkauft.

---

## 38. Die Prüf-Schichten — vier verschiedene Arten zu kontrollieren

> Auch das fehlte: Der Ordner `audit/` enthält **186 Python-Dateien** und war
> in der ganzen Dokumentation einmal erwähnt. Wichtiger noch — es gibt
> inzwischen **vier** verschiedene Prüf-Mechanismen, und nirgends stand, wie
> sie sich unterscheiden. Wer das nicht weiß, hält den einen für den anderen.

### 38.1 Die vier Arten im Überblick

| Prüfung | Fragt | Läuft | Umfang |
|---|---|---|---|
| **Gesetze** (`test_wurzeln_lauf8.py` + 143 weitere) | „Gilt das gebaute Verhalten morgen noch?" | bei jeder Abnahme | 1928 Tests |
| **Sensor-TÜV** (`healing/sensor_selftest.py`) | „Feuern meine Wächter überhaupt?" | im Betrieb, alle 168 h | 7 Prüfungen |
| **Beweis-Suiten** (`audit/proofs/`) | „Wirken die Audit-Reparaturen?" | von Hand | 184 Skripte |
| **Live-Selbsttest** (`audit/live_selftest.py`) | „Antwortet der echte Server?" | von Hand, auf dem NAS | 8 Dienste |

Der entscheidende Unterschied: Die ersten beiden laufen **automatisch** und
schützen dauerhaft. Die letzten beiden sind **Werkzeuge für einen Menschen**,
die man bewusst aufruft.

### 38.2 Gesetze — der automatische Schutz

Beschrieben in §30. Jedes Gesetz hat eine Köder-Gegenprobe in beide
Richtungen; ein Test, dessen Köder nicht beißt, misst nichts. Die einzige
gültige Abnahme ist der volle Lauf über alle 144 Testdateien — ein Teillauf
meldet das selbst („[W198] TEILLAUF … Das ist KEINE Abnahme").

### 38.3 Sensor-TÜV — der Wächter der Wächter

Die bitterste Lektion der Datei-Forensik war, dass `mount_check` mitten im
Freeze „7/7 Mounts OK" meldete. Ein Wächter, der nie nachweislich gefeuert
hat, ist unbewiesen und liefert beruhigende Lügen. Der TÜV **spritzt deshalb
absichtlich harmlose Fehler ein** und prüft, ob die Kette wirklich
durchschlägt:

1. hängende Thread-Probe → kommt `ProbeHung` in der Frist?
2. erschöpfte Slots → kommt `ProbeCapExhausted`?
3. Wegwerf-Arbeiter antwortet (Echo)
4. Syscall-Fehler kommt sauber als `OSError` zurück
5. simulierter D-State → Worker aufgegeben → **frischer** Worker antwortet
6. Ereignis veröffentlicht → Stempel des Empfängers kommt an
7. Schreiben + Rücklesen im Zustands-Ordner

Fällt einer durch, meldet sie das mit hoher Priorität: *„Mein Wächter ist
taub … genau diese Überwachung würde einen echten Ausfall NICHT melden —
bitte schau nach mir."* Seit W223 ist das Ergebnis auch **fragbar** und wird
in ihrer Selbstdiagnose mitgelesen — vorher schrieb der TÜV in eine Datei, die
niemand las (§34/W223).

### 38.4 Beweis-Suiten — 184 Skripte aus dem Audit-Zyklus

`audit/proofs/` sammelt deterministische Skripte aus dem Audit vom Juli 2026.
Jede Suite lässt die **echten** Klassen gegen Fakes laufen und weist nach,
dass eine bestimmte Reparatur wirkt. Sie sind als Regressions-Sweep vor
größeren Fix-Commits gedacht.

**Ehrlich zu ihrem Zustand:** 175 der 184 Skripte enthalten den **absoluten
Pfad** des Audit-Containers (`/home/user/Nas-Agent-Myuri/Nas Agent Myuri`) —
auf einem anderen Host müssen sie vorher angepasst werden; die README sagt
das selbst. Sie sind **kein** Teil der Abnahme und laufen nicht mit `pytest
-q` mit. Ein bekanntes Beispiel mit offenen Fehlern ist
`verify_prio_fixes.py`; das ist bewusst so belassen und nicht Teil der
Gesetzes-Schicht.

### 38.5 Live-Selbsttest — der einzige, der nach draußen fasst

`audit/live_selftest.py` beantwortet, was keine Beweis-Suite kann: **antwortet
der echte Server?** Es verbindet sich **nur lesend** mit jedem
*konfigurierten* externen Dienst — Ollama (Embeddings/OCR), IMAP, SMTP, Home
Assistant, Tuya, Kalender (ICS/CalDAV/HA), Wetter, Web-Suche — und sagt, wer
wirklich antwortet.

Die Sicherheitszusagen sind ausdrücklich: Es wird **nichts** gesendet,
geschaltet oder gelöscht; SMTP macht ausdrücklich **kein** `sendmail`. Und
Discord-/Push-Zustellung ist **bewusst nicht dabei** — das wäre eine Wirkung
nach außen, kein Test.

Aufruf auf dem NAS: `python3 audit/live_selftest.py` (oder mit eigener
Config-Datei als Argument).

### 38.6 Kleinwerkzeug

* `audit/tools/silent_codemod.py` — Werkzeug aus der Kampagne gegen still
  verschluckte Ausnahmen.
* `diagnostics/gc_tuple_audit.py` — Speicher-Profiler; entstand aus einem
  Memdump, der 1,44 Mio. verfolgte Tupel zeigte (87 % aller GC-Objekte) und
  aus dem allein nicht ableitbar war, woher sie kamen.
