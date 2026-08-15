# Mastermind-Nora: Systemarchitektur

## Ueberblick

Nora ist ein **autonomer KI-Agent** fuer Linux-Servermanagement. Sie ueberwacht Systemmetriken, erkennt Probleme, trifft Entscheidungen und lernt aus Ergebnissen — 24/7, ohne menschliches Eingreifen.

```
┌──────────────────────────────────────────────────────────────────┐
│                         orchestrator.py                          │
│                    (Dependency Injection Hub)                     │
├──────────┬───────────┬────────────┬──────────┬──────────────────┤
│ Monitors │  EventBus │  AIBrain   │ Actions  │ Network/External │
│ (21)     │  (Events) │  (Decide)  │ (Execute)│ (MQTT/Web/UDP)   │
├──────────┴───────────┴─────┬──────┴──────────┴──────────────────┤
│                    cognitive_core.py                              │
│            (213 kognitive Subsysteme: 65 eager + 148 lazy)       │
│    ┌──────────┐  ┌──────────┐  ┌─────────────┐  ┌────────────┐ │
│    │ _decide  │→ │ _learn   │→ │ _background │→ │_persistence│ │
│    │ (4480 Z) │  │ (2890 Z) │  │ (600 Z)     │  │ (300 Z)    │ │
│    └──────────┘  └──────────┘  └─────────────┘  └────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. Einstiegspunkt und Hauptschleife

**Datei:** `mastermind.py` → `orchestrator.py`

```
python3 orchestrator.py
  → Orchestrator.__init__()    # Dependency Injection aller Komponenten
  → Orchestrator.start()       # 5-Phasen Startup
  → Orchestrator.run_health_monitor()  # Endlos-Watchdog-Loop
```

### Start-Phasen

| Phase | Was passiert | Fehlerbehandlung |
|-------|-------------|------------------|
| 0 | Preflight Checks, Dependency Health | Non-fatal, geloggt |
| 1 | EventBus starten | **Fatal** bei Fehler |
| 2 | ThreadPool + 30 Monitor-Threads starten | **Fatal** bei Fehler |
| 2b | LLM Router (Ollama) starten | Non-fatal |
| 3 | UDP, AgentGateway, MQTT starten | Non-fatal |
| 4 | Webserver binden (Port 8009) | Startup-Warnung |

### Hauptschleife

`AIBrain.run_brain_loop()` laeuft als Thread mit **5-Sekunden-Intervall**:
- Alle 5s: Snapshot holen, Zustand pruefen
- **Alle 15s** (`_cycle % 3 == 0`): `cognitive.decide(snapshot)` + `cognitive.learn()`
- Alle 30s: UserCognitive Guidance aktualisieren
- Alle 150s (`_cycle % 30 == 0`): Background Tasks, Cognitive Insights

---

## 2. Datenfluss: Monitor → Event → Decide → Action → Learn

```
     ┌─────────────┐
     │  21 Monitors │  (System, Disk, Docker, Network, GPU, UPS, ...)
     │  run_loop()  │  Jeder mit eigenem Thread + Intervall
     └──────┬───────┘
            │ publish(Event)
     ┌──────▼───────┐
     │   EventBus   │  Thread-safe Queue, max 5000, Dead-Letter-Queue
     │  32 Events   │  Auto-disable nach 10 Listener-Fehlern
     └──────┬───────┘
            │ subscribe() / get_snapshot()
     ┌──────▼───────┐
     │ StateManager │  Aggregiert alle Monitor-Daten in einen Snapshot
     │  + Episodic  │  Volatile + Persistent + MetricsRing (mmap)
     └──────┬───────┘
            │ snapshot dict
     ┌──────▼───────────────────────────┐
     │ cognitive.decide(snapshot)        │
     │ ┌─────┐ ┌──────┐ ┌──────────┐  │
     │ │Encode│→│Health│→│84 Voter  │→ │ consensus → action
     │ └─────┘ └──────┘ └──────────┘  │
     └──────┬───────────────────────────┘
            │ action string
     ┌──────▼───────┐
     │   AIBrain    │  _evaluate_outcome() → ActionExecutor
     │   execute    │  Subprozesse, Service-Restart, Throttle, etc.
     └──────┬───────┘
            │ (success, reward)
     ┌──────▼───────────────────────────┐
     │ cognitive.learn(action, success) │
     │  → 70+ Learner-Updates           │
     │  → Voter-Accuracy Tracking       │
     │  → Strategy Evolution            │
     └─────────────────────────────────┘
```

### Reward-Berechnung

`AIBrain._evaluate_outcome(prev_snapshot, curr_snapshot)` berechnet den Reward aus 5 Dimensionen:

| Dimension | Gut (1.0) | Mittel | Schlecht (0.0) |
|-----------|-----------|--------|----------------|
| CPU-Temp | < 65°C | < 75°C (0.7) / < 85°C (0.3) | > 85°C |
| RAM | < 70% | < 85% (0.6) / < 95% (0.2) | > 95% |
| Services | alle gesund | anteilig | alle kaputt |
| Disk | < 85% | < 95% (0.5) | > 95% |
| CPU-Last | < 50% | < 80% (0.6) | > 80% (0.2) |

`reward = sum(5 Faktoren) / 5` — **Success wenn reward > 0.55**

---

## 3. Monitor-Schicht (21 Monitore)

**Verzeichnis:** `monitors/`

| Monitor | Datei | Intervall | Metriken |
|---------|-------|-----------|----------|
| SystemMonitor | `system.py` | 5s | CPU%, RAM%, Load, Temp |
| DiskMonitor | `disk.py` | 30s | Disk%, SMART, Mount-Status |
| AccessMonitor | `access.py` | 4s | SSH-Sessions, Logins |
| NetworkMonitor | `network.py` | 10s | Interfaces, I/O, DNS |
| ServiceManager | `services.py` | 30s | systemd Services Health |
| DockerMonitor | `docker.py` | 30s | Container-Status, Stats |
| GPUMonitor | `gpu.py` | 120s | NVIDIA Temp, VRAM, Load |
| UPSMonitor | `ups.py` | 120s | USV-Status, Batterie |
| FirewallMonitor | `firewall.py` | 300s | UFW/nftables Regeln |
| CertificateMonitor | `certificates.py` | 3600s | SSL-Ablauf |
| StorageHealth | `storage_health.py` | 300s | ZFS/BTRFS/RAID |
| BandwidthMonitor | `bandwidth.py` | 10s | Netzwerk-Durchsatz |
| PhonePresence | `phone_presence.py` | 30s | Geraete im LAN |
| Watchdog | `watchdog.py` | 10s | Thread-Health, RAM, Disk |

*Intervalle sind Config-Defaults — alle per `config.json` ueberschreibbar.*

Weitere Monitore (optional): `antivirus.py`, `apm.py`, `database.py`, `docker_updates.py`, `hardware_prediction.py`, `kubernetes.py`, `network_deep.py`

**Bibliothek:** `psutil` (optional, Fallback auf `/proc` Parsing)

### EventBus (`core/events.py`)

**32 Event-Typen** als `@dataclass(slots=True)`:

| Kategorie | Events |
|-----------|--------|
| **System** | `SystemMetricsUpdated`, `DiskStatusUpdated`, `SmartStatusUpdated`, `NetworkBandwidthUpdated` |
| **Zugriff** | `AccessActivityDetected`, `ExternalActivityDetected`, `UserPresenceChanged`, `DeviceWokeUp` |
| **Aktionen** | `ActionRequest`, `ActionCompleted`, `ActionOutcomeRecorded`, `HealingActionExecuted` |
| **Services** | `ServiceHealthFailure`, `SystemRebootPending`, `SystemUpdatesAvailable` |
| **Sicherheit** | `ThreatDetected`, `AnomalyDetected`, `IntegrityAlertRaised`, `CascadeFailureDetected` |
| **KI** | `CognitiveDecisionMade`, `CognitiveStateChanged`, `LearningProgressUpdated`, `PersonaStateChanged` |
| **Infra** | `ConfigurationChanged`, `MonitorThresholdAdjusted`, `FleetStatusChanged`, `WakeRequestReceived` |
| **Sonstige** | `BackupNeeded`, `ExternalSuspendRequest`, `LogErrorsDetected`, `AgentReport`, `DashboardInteraction` |

**Technische Details:**
- Max Queue: **5000** Events, danach Backpressure
- Dead-Letter-Queue: **100** Events (fuer fehlgeschlagene Zustellungen)
- Listener-Fehler: Nach **10 aufeinanderfolgenden Fehlern** → Listener automatisch deaktiviert
- Recovery: Alle **3600s** (1h) werden deaktivierte Listener wieder aktiviert (Transient-Schutz)
- Slow-Warning: Listener die > **2s** brauchen werden geloggt

### StateManager (`core/state.py`)

**3-Tier Speicher-Architektur:**

| Tier | Medium | Groesse | Inhalt |
|------|--------|---------|--------|
| **Hot** | In-Memory Dict | ~100KB | `volatile_state` — aktuelle Metriken, letzte Aktivitaet |
| **Warm** | mmap Ring-Buffer | 48B × 720 = ~34KB | `MetricsRingBuffer` — 12h CPU/RAM/Temp/Load/Disk History |
| **Cold** | JSON auf Disk | ~50KB | `persistent_state` — Langzeit-State, System-Profil, Decision-Stats |

**MetricsRingBuffer Format** (48 Bytes pro Eintrag):
```
timestamp(f64) + cpu_temp(f64) + cpu_usage(f64) + ram_usage(f64) +
ram_gb(f64) + load_1m(f64) + load_5m(f64) + load_15m(f64) +
connections(u16) + governor_id(u8) + disk_max_pct(u8) + padding(8B)
```

**`get_snapshot()`** liefert `deepcopy(persistent_state | volatile_state)` — alle Monitor-Daten in einem Dict.

---

## 4. Kognitiver Kern (213 Subsysteme in 45 Layers)

**Datei:** `cognitive_core.py` (1780 Zeilen) → `CognitiveOrchestrator`
**Mixins:** `DecideMixin`, `LearnMixin`, `BackgroundMixin`, `InsightsMixin`, `PersistenceMixin`

### Mixture-of-Experts (MoE) Architektur

```
                    ┌─────────────────────────┐
                    │  CognitiveOrchestrator   │
                    │  (Master-Klasse)         │
                    └────────┬────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
  ┌──────────┐      ┌──────────────┐      ┌───────────┐
  │ EAGER    │      │ LAZY         │      │ ORCHESTR. │
  │ Layer 1-8│      │ Layer 9-45   │      │ Mixins    │
  │ 65 Subs  │      │ 148 Subs     │      │ 6 Dateien │
  │ Immer RAM│      │ On-Demand    │      │ 7600 Z    │
  └──────────┘      └──────────────┘      └───────────┘
```

### Layer-Architektur (Kern — Immer im RAM)

| Layer | Datei | Klassen | Zweck |
|-------|-------|---------|-------|
| **L1 Perception** | `perception.py` | StateEncoder, AnomalyClassifier, PatternMatcher, TimeSeriesPredictor | Zustand kodieren, Anomalien erkennen |
| **L2 Memory** | `memory.py` | SemanticMemory (TF-IDF), EpisodicMemory, KnowledgeGraph, DecisionJournal | Erinnern, Suchen, Audit |
| **L3 Learning** | `learning.py` | BayesianLearning, QLearning (Double-Q + Traces), CausalLearning, ContextualBandit (LinUCB), PolicyGradient, Transfer, Curiosity, Calibrator | 8 parallele Lernalgorithmen |
| **L4 World Model** | `world_model.py` | WorldModel, BeliefSystem (Beta-Binomial), LoadForecaster, PredictiveMaintenance | Zukunft vorhersagen |
| **L5 Planning** | `planning.py` | GOAPPlanner (A*), ReasoningEngine (MCDM), MultiObjectiveOptimizer (Pareto), ActionComposer, ConflictResolver, IntentionSystem | Planen und Priorisieren |
| **L6 Executive** | `executive.py` | CognitiveControl (Attention+WM), MetaDecision, EmotionalState, AttentionScheduler, GoalGenerator, FeedbackController (PID), EmergencyProtocol | Steuerung und Emotion |
| **L7 Meta** | `meta.py` | SelfModel, RegretTracker, Profiler, LearningCurves, NarrativeLogger, ExplanationEngine, ResourceBudget | Selbstbeobachtung |
| **L8 Advanced** | `advanced_cognitive.py` + `causal_reasoning.py` + `cognitive_detectors.py` + `strategy_management.py` | AdvancedCausal, StrategyComposer, CheckpointManager, Constraints, ParalysisDetector, CircularDetector, BiasDetector, ThinkingStrategyMemory | Fortgeschrittenes Denken |

### Lazy-Loaded Layers (On-Demand, 148 Subsysteme)

| Layer | Datei | Klassen | Zweck |
|-------|-------|---------|-------|
| **L9** | `patch_management.py` | PatchImpactTracker, UpdateRolloutStrategy, ConfigDriftDetector, ServiceRestartPolicy | Update-Verwaltung, Patch-Impacts |
| **L9** | `restart_intelligence.py` | RestartScorer, RestartOutcomeTracker, RestartStormProtector, RestartIntelligence | Service-Restart Entscheidungen mit Storm-Schutz |
| **L10** | `decision_framework.py` | ModeSelector, PreActionSimulator, RegretLearningFeedback, ConfidenceCalibrationEngine | Denk-Modus-Wahl (deduktiv/probabilistisch/RL/szenario/evidenz/dialektisch) |
| **L11** | `advanced_intelligence.py` | TemporalPatternEngine, ProblemFingerprinter, CausalChainTracker, CrossCorrelationIntelligence, AdaptiveResponseOptimizer | Temporal Patterns, Fingerprinting, Kausale Ketten |
| **L12** | `predictive_engine.py` | PredictiveAnomalyForecaster, SituationalAwareness, IntelligentPrioritizer, ActionEffectivenessLearner | Anomalie-Vorhersage, Prioritaeten, Situationsbewusstsein |
| **L13** | `narrative_engine.py` | SystemNarrativeEngine, LearningInsightGenerator, DecisionExplainer | System-Story, Insights, Menschenlesbare Erklaerungen |
| **L14** | `semantic_understanding.py` | IntentClassifier, AbstractionLadder, AnalogicalReasoner, ContextFusion, AnalogyDB, ComprehensionPredictor | Intent-Erkennung, Abstraktions-Ebenen, Analogien |
| **L15** | `autonomous_learning.py` | MetaLearner, FailurePostMortem, SuccessAmplifier, KnowledgeDistiller, SelfImprovement | Autonomes Lernen, Wissens-Destillation |
| **L16** | `proactive_intelligence.py` | HealthPredictor, OpportunityDetector, PreemptivePlanner | Gesundheits-Vorhersage, Gelegenheiten, Vorbeugende Planung |
| **L17** | `hypothesis_engine.py` | HypothesisGenerator, EvidenceWeigher, DiagnosticReasoner | Hypothesen generieren + pruefen, Diagnostik |
| **L18** | `wisdom_engine.py` | PatternAbstractor, StrategicMemory, CascadePredictor | Erfahrungs-Synthese, Prinzipien, Kaskadenvorhersage |
| **L19** | `communication_intelligence.py` | AnomalyNarrator, ContextualExplainer, FeedbackIntegrator | Anomalie-Stories, Kontext-Erklaerungen |
| **L20** | `rapid_comprehension.py` | SemanticPatternMatcher, AnalogyDB, ComprehensionPredictor, RapidInsightGenerator | Schnelles Muster-Erkennen, Rapid Insights |
| **L21** | `instruction_intelligence.py` | InstructionDecomposer, InstructionPrioritizer | Aufgaben-Zerlegung und Priorisierung |
| **L22** | `error_anticipation.py` | ErrorPatternRecognizer, FailureProbabilityEstimator, PreventiveAdvisor, CascadePredictor | Fehler-Vorhersage, Service-Risiko, Praevention |
| **L23** | `semantic_acceleration.py` | ConceptCache, SemanticIndex | Verstaendnis-Cache, Konzept-Indizierung |
| **L24** | `deep_comprehension.py` | DeepComprehension, ComprehensionFuser, InsightExtractor, ComprehensionMemory | 5-Schicht-Verstehen, Insight-Extraktion |
| **L25** | `solution_accelerator.py` | SolutionMemory, AccelerationEngine, SolutionRanker, KnowledgeCompressor, PathAccelerator | Bewaehrte Loesungen, Fast-Paths |
| **L26** | `risk_intelligence.py` | RiskMatrixEngine (5×5), ImpactSimulator, RiskMitigationPlanner, UncertaintyQuantifier | Formale Risiko-Matrix, Mitigation, Unsicherheitsquantifizierung |
| **L27** | `adaptive_strategy.py` | StrategyEvolution, ContextualStrategy, StrategyBacktester, MultiHorizonPlanner | Strategie-Evolution, Multi-Horizont Planung |
| **L28** | `consensus_intelligence.py` | VoterReliabilityTracker, DynamicVoterWeighting, DisagreementAnalyzer, ConsensusQualityPredictor | Voter-Zuverlaessigkeit, Gewichtung, Konsens-Qualitaet |
| **L29** | `decision_resilience.py` | AdversarialChallenger, FatigueMonitor, FallbackPlanner, ReversibilityScorer | Post-Konsens Sicherheit, Ermuedungs-Schutz |
| **L30** | `causal_decision_theory.py` | CausalPlanner, ValueOfInformation, CounterfactualRegret, InterventionCalculator | Kausales Planen, VOI, Kontrafaktisches Denken |
| **L31** | `collective_meta_intelligence.py` | CoherenceChecker, LayerPerformanceTracker, SubsystemRecommender | System-Kohaerenz, Layer-Leistung |
| **L32** | `error_forensics.py` | LogForensics, StackTraceIntelligence, TemporalErrorCorrelation, ErrorDNA | Log-Forensik, Stack-Traces, Error-DNA |
| **L33** | `cognitive_search.py` | SemanticErrorSearch, MultiDimensionalTriage, ErrorPatternClusterer | Semantische Fehlersuche, Triage |
| **L34** | `error_resolution.py` | KnowledgeBasedResolver, FixVerifier | Wissensbasierte Reparatur + Verifikation |
| **L35** | `adaptive_learning_engine.py` | CurriculumLearner, ActiveLearning, ContinualRuleLearner, EnvironmentModel | Curriculum-Lernen, Aktives Lernen, Umgebungsmodell |
| **L36** | `adaptive_systems_core.py` | DriftDetector, AdaptiveActivation, AdaptiveExploration | Drift-Erkennung, Adaptive Layer-Aktivierung |
| **L37** | `adaptive_memory_consolidation.py` | DreamConsolidator, MemoryPruner | Schlaf-Konsolidierung, Gedaechtnis-Beschneidung |
| **L38** | `adaptive_intelligence.py` | CrossDomainTransfer, KnowledgeDistillery, SelfImprover | Cross-Domain Transfer, Wissens-Destillation |
| **L39** | `architecture_awareness.py` | ArchitectureMap, SystemProfile, ComponentRegistry | Selbst-Wissen ueber eigene Architektur |
| **L41** | `semantic_knowledge.py` | **SemanticKnowledgeNet** | Echte Konzept-Ontologie mit Relationen, Hierarchien, Spreading Activation |
| **L42** | `concept_formation.py` | **ConceptFormationEngine** | Autonome Konzeptbildung, Clustering, Split/Merge, Taxonomie |
| **L43** | `active_learning.py` | **ActiveLearningEngine** | Unsicherheits-basierte Exploration, Information Gain, Wissensluecken |
| **L44** | `hypothetical_reasoning.py` | **HypotheticalReasoner** | Was-waere-wenn, Kontrafaktische Analyse, Szenario-Simulation |
| **L45** | `abstraction_engine.py` | **AbstractionEngine** | Mehrstufige Regel-Induktion, Prinzip-Generalisierung, Analogie-Bildung |

### Higher Cognition (Layer 41-45) — Detailansicht

Die Layer 41-45 bilden eine **vernetzte hoehere Kognitionsschicht**, die ueber einfaches Lernen und
Entscheiden hinausgeht. Sie operieren nicht als isolierte Silos, sondern sind ueber **7 Cross-Module
Bridges** tief miteinander und mit den bestehenden Layern gekoppelt.

#### L41: SemanticKnowledgeNet — Echtes Semantisches Gedaechtnis

```
Anders als SemanticMemoryEngine (L2, TF-IDF Keyword-Buffer) baut dieses Modul
eine echte Konzept-Ontologie:

  Konzepte:
    concept_id → {properties, type, confidence, access_count}
    Beispiel: "disk_full" → {severity: "high", domain: "storage"}

  10 Relationstypen:
    IS_A, CAUSES, CAUSED_BY, REQUIRES, SIMILAR_TO,
    PART_OF, RESOLVES, CONFLICTS_WITH, PRECEDES, CO_OCCURS

    Beispiel-Graph:
      high_load ──CAUSES──→ high_temp
      restart_nginx ──RESOLVES──→ 502_error
      disk_full ──IS_A──→ resource_problem

  Eigenschafts-Vererbung:
    disk_full IS_A resource_problem
    → disk_full erbt automatisch Properties von resource_problem

  Spreading Activation:
    activate("disk_full") → BFS mit Daempfung (0.6 pro Hop)
    → findet assoziierte Konzepte: {resource_problem: 0.3, slow_io: 0.2, ...}

  Inferenz:
    - Transitive IS_A: A is_a B, B is_a C → A is_a C
    - Gemeinsamer Vorfahre → SIMILAR_TO
    - Gleiche Ursachen → CO_OCCURS

  Loesungs-Suche:
    find_resolution("disk_full") →
      1. Direkte RESOLVES Relationen
      2. Geerbte Loesungen von Eltern-Konzepten
      3. Loesungen aehnlicher Probleme

  Max: 500 Konzepte, 50 Relationen/Konzept, Pruning nach Score
```

#### L42: ConceptFormationEngine — Autonome Konzeptbildung

```
Entdeckt neue Kategorien/Problemtypen automatisch aus Beobachtungen:

  Beobachtung → Klassifikation:
    observe({cpu: 85, ram: 70}, label="high_load") →
      1. Finde naechstes Konzept (gewichtete euklidische Distanz)
      2. Distanz < dynamischer Schwellenwert? → Zuordnen + Prototyp updaten
      3. Zu weit? → Neues Konzept erstellen

  Prototyp-Update (Welford's Online-Algorithmus):
    Inkrementelles Mean + Variance Update ohne alle Daten zu speichern

  Konzept-Split:
    Wenn Feature-Varianz > 1.5 → Split entlang High-Variance Feature
    Beispiel: "high_load" → "high_load_cpu_high" + "high_load_cpu_low"

  Konzept-Merge:
    Wenn Distanz < 3.0 → Gewichteter Merge der Prototypen

  Taxonomie-Generierung:
    Automatische Parent/Children Hierarchie
    → get_taxonomy() zeigt Baum aller entdeckten Konzepte

  Feature-Importance:
    Diskriminative Features werden hoeher gewichtet
    Split-Features bekommen Importance-Boost

  Max: 80 Konzepte, 50 Beispiele/Konzept
```

#### L43: ActiveLearningEngine — Aktives, zielgerichtetes Lernen

```
Statt passivem Lernen: Plant aktiv was gelernt werden sollte.

  Unsicherheits-Modell:
    Pro (state, action): {mean, variance, count}
    Bayesian Online-Update bei jeder Beobachtung

  Information Gain Ranking:
    rank_actions_by_info_gain(state, actions) →
      Info-Gain ≈ variance / √(observations + 1)
      + Bonus fuer Aktionen die Wissensluecken schliessen

  Wissensluecken-Erkennung:
    1. Haeufige Ueberraschungen in einer Domaene → "surprise_cluster"
    2. Unbekannte Domaenen → "unknown_domain"
    3. Niedrige Vorhersage-Genauigkeit → "low_accuracy"

  Experiment-Planung:
    plan_experiment(hypothesis, state, action, metric) →
      Baseline messen → Aktion ausfuehren → Metrik beobachten →
      evaluate_experiment() → confirmed/refuted/inconclusive

  Exploration ↔ MetaController:
    Hohe Unsicherheit → exploration_rate += uncertainty × 0.05
    Niedrige Unsicherheit → exploration_rate -= 0.005

  Max: 2000 Uncertainty-Eintraege, 50 Gaps, 5 aktive Experimente
```

#### L44: HypotheticalReasoner — Was-waere-wenn Denken

```
Ergaenzt WorldModel (L4, nur beobachtete Transitionen) um generatives Denken:

  1. what_if(state, action):
     a) Direkte Beobachtungen (Transitions-Modell)
     b) Feature-Delta Modell: Lernt pro Aktion wie Features sich aendern
        Beispiel: "restart" → {cpu_usage: Δ=-30, ram_usage: Δ=-15}
     c) State-Interpolation: Aehnliche bekannte States finden
     d) Risiko-Analyse: Outcomes mit reward < 0.3 → Risk-Score
     e) Opportunity-Erkennung: Outcomes mit reward > 0.7

  2. counterfactual(state, chosen, alternative, actual_reward):
     "Was waere passiert wenn wir Y statt X gemacht haetten?"
     → Geschaetzter alternativer Reward
     → Regret = max(0, alt_reward - actual_reward)
     → Bei Regret > 0.2: Alternative als RESOLVES-Relation lernen

  3. simulate_scenario(state, action_sequence, n_simulations):
     Monte-Carlo Trajektorien-Simulation:
     → avg_total_reward, best_case, worst_case, risk_score

  4. compare_actions(state, actions):
     Alle Aktionen hypothetisch bewerten:
     → Sortiert nach (expected_reward - risk_score × 0.3)

  5. assess_risk(state, action):
     → risk_score = 0.4×fail_prob + 0.3×(1-worst_reward) + 0.3×√variance

  Risk-Gate in decide():
     Wenn risk_score > 0.7: Riskante Alternativen anderer Voter abschwaechen

  Max: 3000 Transitions, 300 Counterfactuals
```

#### L45: AbstractionEngine — Mehrstufige Abstraktion & Generalisierung

```
3 Abstraktions-Stufen:

  Level 1 — Konkrete Regeln (IF-THEN):
    Erfahrungen sammeln → Feature-Muster + Aktion + Erfolg
    Features abstrahieren: cpu_usage=85 → "high"
    Alle 2er Feature-Kombinationen als Bedingungen testen
    Starke Muster (support ≥ 3, success_rate > 0.6) → Regel

    Beispiel:
      IF cpu_usage=high AND ram_usage=high THEN restart (conf=0.78, support=12)

  Level 2 — Prinzipien (Generalisierung ueber Regeln):
    Aehnliche Regeln mit gleicher Aktion → Gemeinsames Muster extrahieren
    Feste Werte → Wildcards

    Beispiel:
      Regel A: IF cpu=high AND ram=high THEN restart
      Regel B: IF cpu=high AND temp=hot THEN restart
      → Prinzip: IF cpu=high AND *=* THEN restart

  Level 3 — Analogien (Strukturelle Aehnlichkeit):
    Problem A : Loesung A ≈ Problem B : Loesung B
    Jaccard auf Feature-Keys + Wert-Matching

    Beispiel:
      Frueheres Problem {cpu=high, connections=flood} → restart half
      Aktuelles Problem {cpu=high, ram=high} → restart vorgeschlagen
      (Key-Aehnlichkeit + gleicher abstrakter Level)

  Feature-Abstraktion (symbolisch):
    cpu_usage → low/medium/high/critical (Schwellenwerte)
    Auto-Abstraktion: Historische Perzentile → low/medium/high

  Max: 200 Regeln, 50 Prinzipien, 100 Analogien, 1000 Erfahrungen
```

### Cross-Module Bridges (L41-L45 Vernetzung)

Die 5 hoeheren Kognitionsmodule sind ueber **7 bidirektionale Bruecken** vernetzt,
um gegenseitige Beeinflussung und Wissenstransfer zu ermoeglichen:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                   CROSS-MODULE BRIDGE ARCHITEKTUR                            │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                         LEARN PHASE                                    │  │
│  │                                                                        │  │
│  │  ConceptFormation ───Bridge 1───→ SemanticNet                         │  │
│  │    Neue Konzepte → define_concept + IS_A Relation                     │  │
│  │                                                                        │  │
│  │  AbstractionEngine ──Bridge 2───→ SemanticNet                         │  │
│  │    Induzierte Regeln → RESOLVES Relationen                            │  │
│  │                                                                        │  │
│  │  CausalReasoning ───Bridge 3───→ SemanticNet                          │  │
│  │    Kausale Erkenntnisse → CAUSES Relationen                           │  │
│  │                                                                        │  │
│  │  Counterfactual ────Bridge 4───→ SemanticNet + AbstractionEngine      │  │
│  │    Regret > 0.2 → Alternative als RESOLVES lernen                     │  │
│  │    + Alternative als erfolgreiche Erfahrung in Abstraction            │  │
│  │                                                                        │  │
│  │  SelfSupervised ────Bridge 5───→ SemanticNet                          │  │
│  │    Entdeckte Betriebsmodi → Konzepte + CO_OCCURS Relationen           │  │
│  │                                                                        │  │
│  │  ActiveLearner ─────Bridge 6───→ MetaController                       │  │
│  │    Hohe Unsicherheit → exploration_rate += uncertainty × 0.05         │  │
│  │    Niedrige Unsicherheit → exploration_rate -= 0.005                  │  │
│  │                                                                        │  │
│  │  Voter-Accuracy ───Bridge 7───→ SemanticNet + Abstraction + Hypoth.  │  │
│  │    semantic_net richtig → RESOLVES staerken                           │  │
│  │    abstraction richtig → Regel-Confidence erhoehen                    │  │
│  │    hypothetical richtig → Transition bekraeftigen                     │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                        DECIDE PHASE                                    │  │
│  │                                                                        │  │
│  │  SemanticNet ──CAUSES──→ HypotheticalReasoner                         │  │
│  │    Kausale Relationen als Features fuer Simulation injizieren          │  │
│  │                                                                        │  │
│  │  HypotheticalReasoner ──Risk-Gate──→ Alle Voter                       │  │
│  │    risk_score > 0.7 → Riskante Alternativen anderer Voter daempfen    │  │
│  │                                                                        │  │
│  │  ConceptFormation ──classify──→ AbstractionEngine                     │  │
│  │    Erkannte Konzept-Features fliessen in Regel-Matching               │  │
│  │                                                                        │  │
│  │  SemanticNet ──Spreading Activation──→ semantic_assoc Voter           │  │
│  │    Assoziative Suche findet Loesungen via Netzwerk-Traversal          │  │
│  │                                                                        │  │
│  │  ActiveLearner ──Knowledge Gaps──→ active_explore Voter               │  │
│  │    Wissensluecken erhoehen Explorations-Druck                          │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                        DREAM PHASE (idle)                              │  │
│  │                                                                        │  │
│  │  ConceptFormation → Taxonomie konsolidieren (Split/Merge)             │  │
│  │  ActiveLearner → Wissensluecken erkennen + Background Insights        │  │
│  │  Abstraction → Regeln/Prinzipien Inventar pflegen                     │  │
│  │  SemanticNet → HypotheticalReasoner: Dream-Szenarien simulieren       │  │
│  │    "Was passiert wenn Problem X erneut auftritt?"                      │  │
│  │    → Risiken in _bg_insights["dream_risks"] speichern                 │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Lazy Loading Mechanismus

```
Zugriff auf self.risk_matrix
    → __getattr__ → _get_lazy("risk_matrix")
        → Nicht im Cache? → Factory aus _lazy_registry aufrufen
            → Instanz erstellen → State laden (falls frueher evicted)
                → In _lazy_cache speichern → Zurueckgeben
```

**Dynamischer TTL (basierend auf RAM-Druck):**

| RAM-Nutzung | TTL | Verhalten |
|-------------|-----|-----------|
| < 60% | 900s (15min) | Entspannt |
| 60-75% | 600s (10min) | Normal |
| 75-85% | 300s (5min) | Sparsam |
| 85-95% | 120s (2min) | Aggressiv |
| > 95% | 30s | Notfall |

**Eviction-Score:** `priority × -100 + frequency × 0.5 - idle_time`
**Schutz:** Prioritaet-1 Module werden im Fokus-Modus nie evicted.
**Preloading:** Proaktives Laden basierend auf Stunden-Mustern und Co-Access-Patterns.

---

## 5. Decision Pipeline (`_decide.py`, 4482 Zeilen)

### Nummerierte Stufen

| # | Stufe | Zeilen | Was passiert |
|---|-------|--------|-------------|
| 1 | ENCODE | 66-69 | Snapshot → coarse/fine state + numerische Features |
| 2 | HEALTH | 70-76 | HealthScore berechnen, Anomalien erkennen |
| 3 | CONTEXT | 91-99 | Forecasting, Energie, Load-Vorhersage |
| 4 | PERCEPTION | 100-160 | Problem-Typ erkennen, Beliefs updaten |
| 5 | PLANNING | 160-880 | GOAP, Kausale Analyse, Hypothesen, RCA |
| 6 | RISK ASSESSMENT | 920-970 | RiskMatrix, Blocking, Mitigation |
| 7 | RECALL | 1000-1120 | Gedaechtnis: War dieses Problem schon mal da? |
| 8 | STRATEGY SELECTION | 986-1010 | Meta-Learner → Strategie waehlen |
| 9 | VOTE COLLECTION | 1124-1860 | **84 Voter** sammeln Empfehlungen (inkl. L41-L45 Higher Cognition) |
| 9b | VOTER MODULATION | 1860-2650 | Reliability, Risk, Ethics, Feedback anwenden |
| 10 | MOO + CONSENSUS | 3060-3100 | Pareto-Filter, collect_votes() → Aktion |
| 10b | POST-CONSENSUS | 3100-3200 | Adversarial Challenge, Fatigue, Reversibility |
| 10c | COHERENCE | 3240-3310 | System-Kohaerenz pruefen, Governance Gate |
| 11 | FINAL | 3860-3950 | Calibrator, Confidence Gate, Record Decision |

### Voter-System (84 Voter)

Jeder Voter gibt ein Tupel `(action, confidence)` ab. Die 84 Voter gruppiert:

| Gruppe | Voter | Quelle |
|--------|-------|--------|
| **Kern-RL** (6) | `thompson`, `qlearn`, `qlearn_top`, `bandit`, `bandit_uncertain`, `bayesian_empirical` | Bayesian, Q-Learning, LinUCB |
| **Planung** (5) | `goap`, `mcdm`, `pipeline`, `intention`, `execution_plan` | GOAP, MCDM, Pipeline, Intentionen |
| **Gedaechtnis** (8) | `fingerprint`, `fingerprint_similar`, `episodic`, `habit`, `journal_history`, `knowledge_graph`, `knowledge_rule`, `knowledge_strategy` | Erfahrung, Muster, Wissen |
| **Strategie** (5) | `strategic_memory`, `policy`, `action_sequence`, `transfer_learning`, `cross_domain_transfer` | Langzeit, Policy, Transfer |
| **Kausalitaet** (4) | `causal`, `causal_predict`, `causal_intervention`, `cascade_prevention` | Kausale Analyse + Vorhersage |
| **Risiko+Sicherheit** (6) | `risk_safety`, `risk_mitigation`, `reversibility`, `failure_risk`, `resilience`, `fatigue_guard` | Risikobewertung + Schutz |
| **Vorhersage** (7) | `predictive`, `precursor_prevent`, `temporal_prevent`, `cascade_forecast`, `health_trend`, `preemptive`, `preventive_advisor` | Proaktive Praevention |
| **Verstehen** (6) | `effectiveness`, `diagnostic`, `evidence`, `root_cause`, `error_dna`, `hypothesis` | Diagnose + Evidenz |
| **Insight** (7) | `insight`, `deep_insight`, `rapid_insight`, `rapid_insight_hint`, `dream_insight`, `opportunity`, `self_improvement` | Einsichten + Gelegenheiten |
| **Loesungen** (4) | `solution_eval`, `fast_path`, `skill`, `distilled_knowledge` | Bewaehrte Loesungen |
| **Kontext** (6) | `context`, `narrative`, `analogy`, `analogy_early`, `amplified`, `env_model` | Kontextwissen |
| **Fusion** (4) | `fusion_wait`, `fusion_severity`, `energy_optimizer`, `maintenance` | System-Balance |
| **Meta** (5) | `world_model`, `caution_disagree`, `multi_horizon`, `impact_sim`, `info_value` | Simulation + Vorsicht |
| **Sonstige** (6) | `experiment`, `curiosity_voter`, `exploration_ctrl`, `restart_scorer`, `restart_intel`, `continual_rule` | Exploration + Restart |
| **Hoehere Kognition** (5) | `semantic_net`, `semantic_assoc`, `hypothetical`, `abstraction`, `active_explore` | Ontologie, Hypothetisches Denken, Regeln, Exploration (L41-L45) |

#### Hoehere Kognition Voter — Detail

| Voter | Modul | Was er tut | Confidence-Bereich | Besonderheiten |
|-------|-------|------------|---------------------|----------------|
| `semantic_net` | L41 | `find_resolution(problem)` → Ontologie-basierte Loesung | 0.2-1.0 | Erbt Loesungen von Eltern-Konzepten |
| `semantic_assoc` | L41 | Spreading Activation → assoziativ verwandte Aktionen | 0.1-0.5 | Findet Loesungen via Netzwerk-Traversal |
| `hypothetical` | L44 | `compare_actions()` → Risiko-bewertete Aktionswahl | 0.2-1.0 | **Risk-Gate**: Daempft riskante Alternativen anderer Voter |
| `abstraction` | L45 | `suggest_action()` → Regel/Prinzip/Analogie-Vorschlag | 0.2-1.0 | 3-stufig: Regeln > Prinzipien > Analogien |
| `active_explore` | L43 | `suggest_exploration_action()` → Info-Gain maximieren | 0.08-0.55 | Nur aktiv wenn exploration_rate + gap_boost > 0.12 |

### Konsens-Bildung

```
84 Voters  →  DynamicVoterWeighting (Reliability × Performance × Diversity)
    ↓
Pareto-Filter (Multi-Objective: Confidence × Stability × Energy)
    ↓
collect_votes()  →  GEWICHTETE SUMME (nicht Majority!)
    │                 scores[action] += confidence pro Voter
    │                 consensus = winner_score / total_scores
    ↓
Post-Consensus Safety Pipeline:
    ├── Adversarial Challenge (kann Override zu NOOP ausloesen)
    ├── Fatigue Check (Drosselung bei Ermuedung)
    ├── Reversibility Check (bevorzugt reversible Aktionen)
    ├── Causal Intervention (bricht kausale Ketten)
    ├── Coherence Check (prueft Widerspruchsfreiheit)
    ├── Calibrator (korrigiert Confidence aus historischen Fehlern)
    └── Governance Gate (Mindest-Confidence pro Aktion)
            ↓
      FINAL ACTION + reasoning dict
```

**Konsens-Methode:** Gewichtete Summe — jede Aktion sammelt Confidence-Punkte von allen Votern.
Die Aktion mit der hoechsten Summe gewinnt. `consensus`-Wert (0-1) zeigt Einigkeit.
Tie-Breaking: Hoechste Voter-Anzahl → Statische Prioritaet → Alphabetisch.

---

## 5b. Detaillierter Ablauf der Schichten in `decide()`

Die `decide()`-Methode durchlaeuft die Schichten **nicht linear L1→L39**, sondern in **funktionalen Clustern**. Layer werden nur aktiviert wenn `_is_problem=True` (nicht bei `normal`/`idle`/`night_idle`). Lazy-Layer werden per **Adaptive Activation** selektiv uebersprungen.

> ℹ **Namenskonvention (Fluss-Diagramme):** Klassennamen in den folgenden `decide()`-Fluss-
> Diagrammen sind teils **abgekuerzt** (Kurzform ohne Suffix). Die realen Klassen tragen oft
> `…Engine`/`…System`/`…Manager`/`…Layer`/`…Tracker`/`…Analyzer`/`…Selector`. Beispiele:
> `QLearning`→`QLearningSystem`, `HealthPredictor`→`SystemHealthPredictor`,
> `HealthScorer`→`HealthScoreCalculator`, `RootCause`→`RootCauseAnalyzer`,
> `Fingerprinter`→`ProblemFingerprinter`, `Evidence`→`EvidenceAggregator`,
> `ProxyConnections`→`ProxyConnectionTracker`, `Scheduler`→`CognitiveScheduler`,
> `CognitiveControl`→`CognitiveControlLayer`, `AnomalyForecaster`→`PredictiveAnomalyForecaster`,
> `ContextualStrategy`→`ContextualStrategySelector`, `CircularDetector`→`CircularReasoningDetector`,
> `FallbackPlanner`→`FallbackChainPlanner`. Auf diese Kurzformen wird unten **nicht** einzeln
> hingewiesen; nur echte Fehler (falscher Methodenname / falsche Signatur) sind korrigiert.

### Phase 1: Wahrnehmung (L1 Perception)

```
snapshot → StateEncoder.coarse(snap) → state_c (Format: "load|temp|cpu", z.B. "idle|cool|low")
         → StateEncoder.fine(snap)   → state_f (detaillierter Zustand)
         → StateEncoder.features(snap)→ features (kategorische Features)
         → _numeric_features()   → num_feat (float-Vektor)
         → StateEncoder.temporal(snap)→ state_temporal (zeitbewusst)
         → StateEncoder.state_hash(snap)→ Dedup-Hash
         ↓
AnomalyClassifier.classify(snapshot) → anomalies (Liste von Anomalie-Dicts)
AnomalyClassifier.update_baseline(metric, value) — passt Schwellwerte an
TimeSeriesPredictor.observe(metric, value)  → trackt Volatilitaet
PatternMatcher.frequent_patterns(3) → bekannte Muster
```

> ℹ **Krisen-Marker im Q-Zustand (Faehigkeits-Audit):** `coarse()` haengt bei erhoehten
> Werten Marker an: `|ram:hi` (≥75%), `|ram:x` (≥90%), `|disk:x` (≥90%), `|svc:down`
> (failed Service). Vorher hatte der Zustand KEINE RAM/Disk/Service-Dimension — eine
> RAM-Krise (97%) und ein gesundes Idle-System waren derselbe Zustand `idle|cool|low`,
> problemspezifisches Q-Lernen war strukturell unmoeglich. Gesunde Zustaende bleiben
> byte-identisch (persistierte Q-Tabellen kompatibel). Tests: `TestCoarseCrisisMarkers`.

### Phase 2: Kontext + Vorhersage (L1+L4+L6)

```
ProxyConnections.record(conns, ips, usage)
CognitiveControl.update_attention(snapshot)
Context.get_context(snapshot) → ext_ctx
EnergyOptimizer.record(hour, cpu, conns)
LoadForecaster.record(hour, weekday, cpu, conns, ram)
LoadForecaster.predict_hour(next_hour) → forecast
BeliefSystem.update_from_snapshot(snapshot)  → Beliefs aktualisieren
BeliefSystem.decay()  → alte Evidenz vergessen
PredictiveMaintenance.record("cpu_temp"|"ram"|"cpu", value)
AttentionScheduler.get_boosts() → zeitbasierte Aufmerksamkeit
```

### Phase 3: Gesundheit + Problem-Erkennung (L1+L6+L8)

```
HealthScorer.compute(snapshot) → {score: 0-100, details: {...}}
Identity.personality → {caution, curiosity, ...}  — Persoenlichkeit
ThinkingStrategyMemory.classify_problem(snapshot, anomalies, health, state)
  → problem_type: "normal"|"idle"|"night_idle"|"cpu_high"|"ram_critical"|
                   "service_down"|"disk_critical"|"temp_high"|"multi_problem"|...
```

**EARLY-EXIT:** Wenn `problem_type ∈ {normal, idle, night_idle}` → Springe direkt zu Phase 9 (Voting). Alle erweiterten Layer werden uebersprungen.

### Phase 4: Erweiterte Analyse (L10-L12, nur bei Problemen)

```
┌─ Adaptive Layer Activation: Welche Lazy-Layers werden aktiviert?
│  → get_active_layers(problem_type) → active_layers, skipped_layers
│  → Background-erkannte Low-Value-Layers zusaetzlich skippen
│
├─ Pipeline.run(observations, actions, context) → pipeline_result
├─ Evidence.add_evidence("anomaly"|"health", problem_type, confidence)
├─ RootCause.analyze(problem_type, observations) → rca_result
├─ ModeSelector.select_mode(problem_type, confidence) → Denk-Modus
│
├─ TemporalPatterns.record_event(problem_type, details)
├─ AnomalyForecaster.update_metric("cpu"|"ram"|"temp", value)
├─ CrossCorrelation.update({cpu, ram, temp, conns})
├─ SituationalAwareness.update_from_snapshot(snapshot)
├─ Fingerprinter.record(snapshot) → {deja_vu, best_action}
├─ CausalChainTracker.record_event(problem_type, severity)
│
└─ FEEDBACK: Causal→Predictive
     Kausale Staerken → anomaly_forecaster.adjust_baseline()
```

### Phase 4b: Vorhersage-Kette (Cluster 2: L1→L11→L12→L22)

```
Perception (L1)
    ↓ Anomalien, Metriken
Temporal Patterns (L11)
    ↓ predict_next_hour() → erwartete Events
    ↓ fliessen als Precursors in L22
Predictive Engine (L12)
    ↓ forecast_all_metrics(30min) → metric_forecasts, imminent_breach
    ↓ precursor_warnings
Error Anticipation (L22)
    ├─ ErrorPatternRecognizer.observe_metrics({cpu, ram, temp})
    ├─ + learn_precursor(temporal_events)  ← L11 Feed
    ├─ + check_precursors(enriched_metrics) ← L12 Forecasts eingerechnet
    ├─ FailureProbabilityEstimator.estimate(service, metrics, precursors)
    ├─ PreventiveAdvisor.advise(precursors, actions)
    └─ CascadePredictor.predict_cascade(service, status) ← L11 Kausale Ketten
```

### Phase 4c: Verstehens-Pipeline (Cluster 1: L14→L20→L23→L24)

```
Semantic Understanding (L14)
    ├─ IntentClassifier.classify(snapshot, anomalies, health, recent_acts)
    │    → intent: "normal_operation"|"recovery"|"degradation"|"crisis"|...
    ├─ AbstractionLadder.abstract(snapshot, anomalies, health, problem, intent)
    │    → optimal_level: 1=Symptome, 2=Prozesse, 3=Architektur, 4=Prinzipien, 5=Systemisch
    ├─ ContextFusion.fuse(metrics, anomalies, intent, fingerprint, forecast, causal)
    │    → {severity, urgency, recommended_posture: "wait"|"monitor"|"act"|"escalate"}
    └─ AnalogicalReasoner.get_best_analogical_action(features, problem, actions)
        ↓
Rapid Comprehension (L20)
    ├─ SemanticPatternMatcher.match(metrics, context) ← Intent von L14
    ├─ AnalogyDB.find_analogies(metrics, top_k=3) ← Features von L14
    ├─ ComprehensionPredictor.observe(state) + predict_next(state, top_k=3)
    └─ RapidInsightGenerator.generate_rapid_insight(pattern_matches, analogies, predictions)
        ↓
Semantic Acceleration (L23)
    ├─ ConceptCache.get(key) — wenn gecacht: ueberspringen
    ├─ ConceptCache.put(key, understanding, ttl=300s)
    └─ SemanticIndex.index_entry(data, concepts, relevance)
        ↓
Deep Comprehension (L24) — nur bei aktiven Problemen
    ├─ DeepComprehension.comprehend(metrics, events, context)
    │    → 5-Schichten-Analyse: Oberflaechlich → Strukturell → Temporal →
    │      Kausal → Systemisch
    ├─ ComprehensionFuser.fuse(5_layers) → gewichtete Fusion
    ├─ InsightExtractor.extract(result, fusion) → handlungsfaehige Insights
    └─ ComprehensionMemory.store(result, tags, synthesis) → persistiert
```

### Phase 4d: Weitere Layer-Cluster (L16-L19)

```
Proactive Intelligence (L16)
    ├─ HealthPredictor.predict(6h) → Score-Vorhersage fuer naechste Stunden
    ├─ OpportunityDetector.detect_opportunities(snapshot, health_score, is_idle) → Gelegenheiten
    └─ PreemptiveActionPlanner.plan_preemptive_actions(forecasts, temporal_patterns, health_prediction, current_state, available_actions)

Hypothesis Engine (L17) — nur bei Symptomen (high_ram, high_cpu, etc.)
    ├─ HypothesisGenerator.generate(symptoms, context) → Hypothesen-Liste
    ├─ EvidenceWeigher.initialize_hypotheses(hypotheses)
    └─ DiagnosticReasoner.start_diagnosis(symptoms, metrics, hypotheses)

Wisdom Engine (L18)
    ├─ PatternAbstractor.observe_problem(type, hour, weekday) → langfristige Muster
    ├─ StrategicMemory.get_best_action(problem_type) → Prinzipien-basiert
    └─ CascadePredictor.predict_cascade(problem_type) → Ausbreitungs-Vorhersage

Communication Intelligence (L19) + Narrative (L13)
    ├─ AnomalyNarrator.start_story(type, metrics, headline) → Story-ID
    ├─ SystemNarrative.add_event(category, headline, details, severity)
    ├─ ContextualExplainer.explain(event, audience="dashboard")
    └─ FEEDBACK: Narrative → Entscheidung
         Vergangene Stories → action_hint (wenn aehnliches Problem geloest)
```

### Phase 4e: Forensik (L32-L34, nur bei health < adaptive_threshold)

```
Error Forensics (L32)
    ├─ LogForensics.get_active_clusters() → Fehler-Cluster
    ├─ StackTraceIntelligence.get_active_clusters()
    ├─ TemporalErrorCorrelation.record_error(type, problem, severity)
    └─ ErrorDNA.create_dna(incident) → DNA-Fingerprint + matches

Cognitive Search (L33)
    ├─ SemanticErrorSearch.index_error(source, description, metrics)
    ├─ MultiDimensionalTriage.triage(type, source, services, metrics, ttf)
    └─ ErrorPatternClusterer.add_error(error_dict)

Error Resolution (L34)
    └─ ResolutionKnowledgeBase.search(query, error_type, max_results) → kb_solutions
```

### Phase 5: GOAP Planung (L5)

```
GOAPPlanner:
    1. snapshot_to_goap_state(snapshot) → GOAP-Zustand
    2. _determine_goal(problem_type, state) → Ziel (z.B. "system_healthy")
    3. A*-Suche ueber Aktionen mit Preconditions + Effects
    4. → Plan (Aktions-Sequenz) oder None
        ↓
IntentionSystem.set_intention(goal, priority)
GoalGenerator.generate(snapshot, beliefs, intentions) → neue Ziele
```

> ✅ **STATUS (behoben):** GOAP war fuer die Problemloesung lange **wirkungslos** — sein
> Aktions-Vokabular (`restart_service`, `throttle_cpu`, `remount`, `start_backup`) ist
> DISJUNKT von `SystemAction.ALL_STATIC`, daher schlug das Gate `goap_plan[0] in viable`
> bei jedem echten Problem fehl (Beleg: 6h-Soak = 9/2153 Entscheidungen ueber GOAP).
> Seit dem Interaktions-Audit uebersetzt die **Vokabular-Bruecke** `_GOAP_TO_SYSTEM_ACTION`
> (+ `_goap_action_to_system()`, `_decide.py`) alle 7 GOAP-Namen in ausfuehrbare
> SystemActions (z.B. `throttle_cpu`→`temp_throttle`, `restart_service`→`service_check`);
> die GOAP-Stimme nimmt seither real am Konsens teil (Tests: `TestGoapVocabularyBridge`).

### Phase 6: Adaptive Strategy (L27)

```
StrategyEvolutionEngine.get_current_weights() → Voter-Gewichte
ContextualStrategy.select(problem_type, metrics) → Strategie + Confidence
MultiHorizonPlanner.plan(current_metrics, problem_type, health_score, forecasts, available_actions) → kurzfristig vs. langfristig
```

### Phase 7: Filter + Risiko (L5+L8+L26+L29)

```
CognitiveControl.filter_inhibited(actions) → viable actions
    ↓ Budget-Filter, Constraint-Filter
CircularDetector.is_stuck() → bei Erkennung: break_suggestion()
    ↓
RiskMatrixEngine.assess_risk(action, context) → pro Aktion
    ├─ likelihood (1-5) × impact (1-5) = Risiko-Score
    ├─ recommendation: "proceed"|"monitor"|"mitigate"|"block"
    └─ Hochrisiko → aus viable entfernen
        ↓
RiskMitigationPlanner.plan_mitigation(risk, viable) → Alternativen
ImpactSimulator.simulate(action, metrics) → vorhergesagte Auswirkungen
    ↓
FEEDBACK: Risk→Fallback
    Wenn max_risk > 0.7 → FallbackPlanner.start_chain(problem, viable)
```

### Phase 8: Consensus Intelligence (L28)

```
VoterReliabilityTracker.get_all_reliabilities() → pro Voter: 0-1
DynamicVoterWeighting.compute_weights(voters, reliabilities)
    → Gewichte basierend auf: EMA-Performance × Reliability × Diversity
UncertaintyQuantifier.quantify(state, voters, problem_type) → epistemisch + aleatorisch
```

### Phase 9: Voting (84 Voter aus L1-L45)

```
84 Voters geben je (action, confidence) ab:
    ├─ IMMER aktiv: thompson, qlearn (2 Voter)
    └─ KONDITIONAL: 82 Voter nur wenn Subsystem-Aufruf erfolgreich
         und Confidence ueber Schwelle

Voter-Quellen nach Layer:
    L1-L4 (Kern-RL+Planung):  thompson, qlearn, bandit, bayesian_empirical,
                                policy, goap, mcdm, causal, maintenance, ...
    L5-L8 (Executive+Meta):    world_model, intention, curiosity, skill,
                                habit, fingerprint, analogy, ...
    L9-L12 (Intelligence):     restart_scorer, restart_intel, effectiveness,
                                predictive, pipeline, ...
    L16-L24 (Verstehen):       rapid_insight, deep_insight, hypothesis,
                                narrative, opportunity, ...
    L25-L29 (Loesung+Risiko):  solution_eval, fast_path, risk_safety,
                                risk_mitigation, resilience, fatigue_guard, ...
    L30-L39 (Meta+Adaptive):   causal_predict, impact_sim, self_improvement,
                                dream_insight, energy_optimizer, ...
    L41-L45 (Higher Cognition): semantic_net, semantic_assoc, hypothetical,
                                 abstraction, active_explore
                                 → Mit Cross-Module Bridges vernetzt
                                 → Risk-Gate daempft riskante Alternativen
```

### Phase 10: Post-Consensus + Final

```
collect_votes(voters) → action + vote_details
    ↓
Schwacher Konsens (< 0.4)?
    → ConflictResolver.resolve_with_context(proposals, context)
    ↓
ConsensusQualityPredictor.predict(consensus_score, disagreement_level, avg_reliability, voter_count, problem_type) → cq_boost
    → Schlechte Qualitaet: Confidence senken
    → Hohe Qualitaet: Confidence boosten
    ↓
Post-Consensus Safety:
    1. AdversarialChallenger.challenge(action, context, voters)
       → Bei challenge_score > 0.7: Override zu NOOP/SERVICE_CHECK
    2. FatigueMonitor: Bei should_throttle → Override zu NOOP
    3. FallbackPlanner: Aktive Eskalationskette ueberschreibt Konsens
    4. ReversibilityScorer: Bei hoher Unsicherheit + niedriger Reversibilitaet
       → Wechsel zu reversiblerer Aktion
       → FEEDBACK: Kausale Staerke beeinflusst Reversibilitaets-Schwelle
    5. CoherenceChecker: System-Kohaerenz pruefen
    6. GovernanceGate: Mindest-Confidence pro Aktionstyp
    7. Calibrator: Historische Ueberconfidence korrigieren
    ↓
DecisionExplainer.explain(decision, context) → menschenlesbar
AnomalyNarrator.add_chapter(story_id, "action", ...)
_record_decision() → alle Systeme informieren
    ↓
FINAL: (action, reasoning_dict)
```

### Datenfluss-Diagramm der Cluster

```
Cluster 1: VERSTEHEN          Cluster 2: VORHERSAGE
L14 → L20 → L23 → L24         L1 → L11 → L12 → L22
Intent  Rapid  Cache  Deep     Perceive Temporal Predict Error
                                                        Anticipation

Cluster 3: PLANEN             Cluster 4: BEWERTEN
L5 → L10 → L17                L26 → L28 → L29
GOAP  Mode  Hypothesis         Risk  Consensus  Resilience

Cluster 5: LERNEN             Cluster 6: KOMMUNIZIEREN
L3 → L15 → L18 → L35          L13 → L19 → L24
RL   Meta  Wisdom  Adaptive    Narrative  AnomalyStory  Explain

Cluster 7: FORENSIK           Cluster 8: META
L32 → L33 → L34               L30 → L31 → L36 → L39
ErrorDNA Search Resolve        Causal Coherence Drift SelfAware
```

---

## 6. Learn Pipeline (`_learn.py`, 2892 Zeilen)

### Eingabe fuer learn()

```
learn(action, success, reward, next_snapshot) wird nach JEDER Aktion aufgerufen.

Eingabe:
  action       = "service_check"     ← ausgefuehrte Aktion
  success      = True/False          ← hat es funktioniert?
  reward       = 0.0 - 1.0          ← Qualitaets-Score (auto-berechnet wenn None)
  next_snapshot = {...}              ← System-Zustand NACH der Aktion
  vote_details = {...}               ← 28+ Felder aus decide() (wer hat was empfohlen)

Ablauf (vereinfacht):
  1. Kern-Algorithmen (L1-L8): Bayesian, Q-Learning, Bandit, Policy, World Model, etc.
  2. Higher Cognition (L41-L45): SemanticNet, ConceptFormation, ActiveLearner, etc.
  3. Cross-Module Bridges: 7 Bruecken zwischen L41-L45 und bestehenden Modulen
  4. Voter-Accuracy Tracking: Welcher Voter lag richtig? → Feedback an Module
  5. Advanced Learners (L9-L39): CausalReasoning, StrategyComposer, etc.
```

### Die 8 Kern-Lernalgorithmen (L3 Learning)

#### 1. Bayesian Learning (Beta-Binomial)

```
Fuer jedes (state, action) Paar existiert eine Beta-Verteilung Beta(α, β):

Schritt 1: Decay (Vergessen):
    α' = max(1.0, α × 0.995)
    β' = max(1.0, β × 0.995)

Schritt 2: Update:
    Erfolg  → α' += 1
    Misserfolg → β' += 1

Ergebnis:
    P(Erfolg | state, action) = α / (α + β)
    Varianz = α×β / ((α+β)² × (α+β+1))

Nutzung in decide():
    Thompson Sampling: Ziehe Zufallswert aus Beta(α, β) pro Aktion
    → Aktion mit hoechstem Sample gewinnt (natuerliche Exploration)
```

#### 2. Q-Learning (Double-Q mit Eligibility Traces)

```
Zwei Q-Tabellen: Q1[state][action], Q2[state][action]

Schritt 1: Aktion waehlen (ε-greedy):
    Mit Wahrscheinlichkeit ε: zufaellige Aktion (gewichtet 1/√visits)
    Sonst: argmax Q(state, action)

Schritt 2: Double-Q Update (50/50 Muenzwurf):
    Kopf → best_a = argmax_a Q1[next_state, a]
           target = reward + γ × Q2[next_state, best_a]
           td_error = target - Q1[state, action]
           Q1[state, action] += α × td_error
    Zahl → symmetrisch mit Q2

Schritt 3: Eligibility Traces (Temporal Credit Assignment):
    e[aktuell] = 1.0                    ← replacing trace
    Fuer alle anderen e[k]:
        e[k] *= γ × λ                  ← Trace zerfaellt
        wenn e[k] < 0.01: loeschen
        sonst: Q1[k] += α × td_error × e[k] × 0.5
               Q2[k] += α × td_error × e[k] × 0.5

Schritt 4: Epsilon-Decay:
    ε = max(0.02, ε × 0.9995)          ← ~6000 Schritte bis Minimum

Parameter: α=0.1, γ=0.9, λ=0.7, ε_start=0.3, ε_min=0.02
```

#### 3. Contextual Bandit (LinUCB)

```
Pro Aktion: A_inv (d×d Inverse), b (d-Vektor)
Features x = [cpu, temp, ram, conns, hour_sin, hour_cos, is_night, is_weekend]

Auswahl (UCB):
    θ = A_inv × b                       ← Parameter-Schaetzung
    score = θᵀ × x + α × √(xᵀ × A_inv × x)  ← Optimismus bei Unsicherheit
    → Aktion mit hoechstem UCB Score

Update (Sherman-Morrison):
    A_inv = A_inv - (A_inv×x×xᵀ×A_inv) / (1 + xᵀ×A_inv×x)
    b = b + reward × x
```

#### 4. Policy Gradient (REINFORCE mit Baseline)

```
Pro Aktion: Gewichtsvektor w[action] = [w_0 ... w_7]

Sammlung: episode_log = [(features, action, reward), ...]

Batch-Update (alle 10 Schritte):
    baseline = Mittelwert bisheriger Rewards

    Rueckwaerts diskontierte Returns:
        G = 0
        fuer jedes (x, a, r) rueckwaerts:
            G = r + 0.99 × G

    Softmax-Wahrscheinlichkeiten:
        π(a|x) = exp(wₐᵀ × x) / Σ exp(wₐ'ᵀ × x)

    Gewichts-Update:
        advantage = G - baseline
        fuer jede Aktion a':
            grad = (𝟙{a'==a} - π(a'|x)) × advantage
            w[a'][i] += lr × grad × x[i]
            w[a'][i] = clamp(-10, 10)
```

#### 5. Causal Learning (Kausale Analyse)

```
Pro Feature-Wert: Beta(s, f) — Erfolge/Misserfolge
Pro Feature×Aktion: Beta(s, f) — Interaktions-Effekte

Update:
    Fuer jedes Feature k=v im Zustand:
        s = max(1, s × decay) + (1 wenn Erfolg)
        f = max(1, f × decay) + (1 wenn Misserfolg)

Feature-Wichtigkeit: importance[k] = s / (s + f)

Interventions-Effekt:
    intervention_rate = P(Erfolg | Feature + Aktion)
    baseline_rate = P(Erfolg | Feature ohne Aktion)
    kausaler_effekt = intervention_rate - baseline_rate

Kontrafaktisch:
    lift = (beobachtetes_Ergebnis) - (NOOP-Vorhersage)
    → "Haette NOOP zum gleichen Ergebnis gefuehrt?"
```

#### 6. World Model (Transitions + Belohnungen)

```
Transitions: _trans[state][action][next_state] = count
Sequenzen:   _seq["s1>s2"][s3] = count  (3-Schritt-Muster)
Belohnungen: _reward[state||action] = deque(rewards, maxlen=50)

Update:
    _trans[s][a][s'] += 1
    _reward[s||a].append(reward)

Vorhersage:
    P(s'|s,a) = count(s,a,s') / Σ count(s,a,*)
    E[reward|s,a] = mean(_reward[s||a])

Anomalie-Erkennung:
    Wenn P(s'|s,a) < 0.05 → seltener Uebergang → ε erhoehen
```

#### 7. Curiosity (Intrinsische Motivation)

```
novelty(state, action) = 1 / √(visits[state||action])
surprise = 0 wenn vorhergesagt==tatsaechlich, sonst 1
intrinsic_reward = 0.5 × novelty + 0.5 × surprise
```

#### 8. Confidence Calibrator (ECE)

```
Expected Calibration Error:
    ECE = Σ (bin_size / total) × |bin_confidence - bin_accuracy|

Wenn ECE > 0.15:
    adjusted_confidence = 0.5 + (raw - 0.5) × 0.7
    → Komprimiert Richtung 0.5 (konservativer)
```

### Belief System (Beta-Binomial Netzwerk)

```
Pro Belief (z.B. "system_healthy"): Beta(α, β)

Update:
    Evidenz unterstuetzt  → α += 1
    Evidenz widerspricht → β += 1

    P(wahr) = α / (α + β)
    confidence = 1 - 2×√(α×β / ((α+β)² × (α+β+1)))

Conditional Update (Bayes-Regel):
    posterior = prior × likelihood_ratio / (prior × lr + (1-prior))

Belief-Netzwerk Propagation:
    parent_probs = [P(parent_i) fuer i in Eltern]
    combined = Produkt(parent_probs)
    child.α += combined
    child.β += (1 - combined)

Decay: Alle Beliefs zerfallen langsam → Vergessen alter Evidenz
```

### Emotionaler Zustand (beeinflusst Entscheidungen)

```
3 Dimensionen:
    valence = 0.8 × alt + 0.2 × (reward × 2 - 1)     ← [-1, 1]
    arousal += 0.1 × anomaly_count  (oder -0.05 in Ruhe) ← [0, 1]
    stress  += 0.15 bei Misserfolg (oder -0.03 in Ruhe)  ← [0, 1]

Stimmung:
    stress > 0.7           → "stressed"
    valence > 0.3, arousal < 0.5 → "content"
    valence > 0.3, arousal > 0.5 → "energized"
    valence < -0.3, stress > 0.4 → "frustrated"

Risiko-Toleranz (beeinflusst Entscheidungen):
    toleranz = 0.5 + (valence × 0.2) - (stress × 0.3)
    → gestresst = vorsichtiger, zufrieden = mutiger
```

### PID Feedback Controller

```
error = setpoint - current_value
integral = clamp(integral + error, -10, 10)  ← Anti-Windup
derivative = error - prev_error

output = kp × error + ki × integral + kd × derivative
```

### Regret Tracking (Bedauern)

```
regret = max(0, best_possible_reward - chosen_reward)
cumulative_regret += regret

Trend:
    Vergleiche Durchschnitt erste Haelfte vs. zweite Haelfte
    Wenn zweite Haelfte < 0.8 × erste → "improving"
    Wenn zweite Haelfte > 1.2 × erste → "worsening"

Wirkung: Steigende Regret → ε += 0.01 (mehr Exploration)
```

### Lernkurven-Erkennung

```
Plateau: |Durchschnitt(letzte 10) - Durchschnitt(vorherige 10)| < 0.02
→ Plateau erkannt → MetaLearner passt Lernrate an
```

### Learn-Stufen Uebersicht

| # | Stufe | Input | Was passiert | Output → wohin |
|---|-------|-------|-------------|----------------|
| 1 | **Bayesian** | state, action, success | Beta update + decay | → Thompson Voter |
| 2 | **Q-Learning** | state, action, reward, next_state | Double-Q + Traces + ε-decay | → qlearn Voter, td_error |
| 3 | **World Model** | state, action, next_state, reward | Transition + Reward Model | → world_model Voter, Anomalie-Erkennung |
| 4 | **Causal** | features, action, success | Kausaler Effekt + Kontrafaktisch | → causal Voter, Risk Assessment |
| 5 | **LinUCB** | num_features, action, reward | Sherman-Morrison Update | → bandit Voter |
| 6 | **Policy** | features, action, reward | REINFORCE + Baseline | → policy Voter |
| 7 | **Curiosity** | state, predicted, actual | Novelty + Surprise | → Exploration Bonus |
| 8 | **Calibrator** | predicted_conf, actual_reward | ECE Berechnung | → Confidence Governance |
| 9 | **Transfer** | features, action, success | Jaccard-gewichteter Transfer | → transfer Voter |
| 10 | **Replay** | (gespeicherte Transitions) | Sample → Q-Update | → Q-Learning Nachtraining |
| 11 | **Habits** | state, action, success | Haeufigkeits-Tracking | → habit Voter |
| 12 | **Regret** | chosen_r, best_r | Bedauern-Trend | → Exploration Boost |
| 13 | **Emotional** | reward, anomalies, success | Valence, Arousal, Stress | → Risiko-Toleranz |
| 14 | **SelfModel** | action, success, time | Zuverlaessigkeit | → Faehigkeits-Einschaetzung |
| 15 | **KnowledgeGraph** | state, action, outcome | Neue Kanten + Gewichte | → knowledge_graph Voter |
| 16 | **Beliefs** | evidence, likelihood | Beta-Update + Netzwerk-Propagation | → Exploration via Thompson |
| 17-18 | **Patterns, Thresholds** | Metriken, Anomalien | Schwellwert-Anpassung | → Monitor-Thresholds |
| 19-21 | **AdvancedCausal** | Hypothesen, Features | A/B-Testing, Deep Reflect, Incremental | → Hypothesen bestaetigen |
| 22 | **Forgetting-Check** | Performance-Drop? | Rollback wenn dramatischer Abfall | → Q-Tables wiederherstellen |
| 23 | **ThinkingStrategy** | Outcome | Blockade-Diagnose | → Paralysis/Circular Erkennung |
| 24-25 | **Intelligence** | Fingerprint, Effectiveness | Muster-Lernen | → fingerprint/effectiveness Voter |
| 26-27 | **Autonomous** | Meta-Lernen | Lernraten, PostMortem, Regeln | → α/ε Anpassung |
| 28-30 | **Hypothesis+Wisdom** | Hypothesen, Prinzipien | Bestaetigen/Widerlegen | → strategic_memory Voter |
| 31-33 | **Risk+Strategy+Consensus** | Risiko, Voter-Accuracy | Kalibrierung, Gewichtung | → Voter-Gewichte |
| 34 | **Resilience** | Adversarial, Fatigue | Herausforderungs-Kalibrierung | → Adversarial-Schwelle |
| 34b | **vote_details** | 28+ Felder aus decide() | Jedes Feld einzeln validieren | → 28+ Feedback-Pfade |
| 35-41 | **Extended** | Causal, Meta, Forensik | LayerPerformance, ErrorDNA | → Layer-Aktivierung |

---

## 6b. Background Tasks (`_background.py`)

Laeuft alle **150s** (Brain-Loop `_cycle % 30 == 0`):

### Phase 1: Schnelle Tasks (unter Lock)

```
Scheduler.due_tasks() → welche periodischen Tasks faellig?
    ├─ q_decay: Q-Learning Werte verfallen lassen (Vergessens-Rate)
    ├─ policy_update: Policy-Gradient Verfeinerung
    └─ health_check: Gesundheits-Score aus Snapshot berechnen
```

### Phase 2: Schwere Tasks (ohne Lock)

```
DreamConsolidator: Episodic Memory konsolidieren + Q-Learning Replay
MemoryCleanup: Replay Buffer samplen fuer Q-Updates
Save: Komplette Orchestrator-Persistenz → JSON + KnowledgeStore
```

### Phase 3: Kognitive Hintergrund-Analyse

```
Confidence Calibration: ECE (Expected Calibration Error) tracken
    → Wenn ECE > 0.15 und >20 Vorhersagen: confidence_governance anpassen
Pipeline Performance: Layer-Leistung tracken → low_value_layers erkennen
Restart Storm: Restart-Haeufigkeit pruefen
Causal Patterns: Kausale Muster erkennen + Hypothesen generieren
Learning Insights: Temporal + Fingerprint + Kausale Insights erzeugen
```

### Phase 3b: Higher Cognition Consolidation (Idle)

```
Nur in Idle-Phasen (is_idle=True):

ConceptFormation Taxonomie:
    get_taxonomy() → Uebersicht aller Konzepte + Hierarchie
    → Prueft ob Split/Merge noetig

ActiveLearner Wissensluecken:
    detect_knowledge_gaps(recent_states, recent_outcomes)
    → Neue Luecken in _bg_insights["knowledge_gaps"]

Abstraction Inventar:
    get_rules(min_confidence=0.4) → Alle geltenden Regeln
    get_principles() → Alle abstrahierten Prinzipien

Dream-Szenarien (SemanticNet → Hypothetical):
    Fuer jedes bekannte Problem-Konzept:
        what_if(problem, "noop") → Was passiert bei Nichtstun?
        → Entdeckte Risiken in _bg_insights["dream_risks"]
    → Proaktive Risiko-Erkennung ohne reale Probleme
```

### Phase 4: Autonomes Lernen (ab 50 Entscheidungen)

```
Alle 100 Entscheidungen: Knowledge Distillation → neue Regeln
Alle 200 Entscheidungen: Self-Improvement Parameter-Optimierung
MetaLearner: Domain-Analyse, Stagnations-Erkennung
Golden Rules: Bewaehrte Prinzipien aktualisieren
```

---

## 6c. Selbstverbesserung (Autonomous Learning, L15)

### MetaLearner — Lernraten-Anpassung

```
Fuer jede Problem-Domaene (z.B. "thermal", "service_down"):
    Vergleiche letzte 5 Rewards mit frueheren 5 Rewards:

    improvement = avg(letzte_5) - avg(fruehere_5)

    Wenn improvement > 0.1:  (es wird besser)
        neue_lr = min(0.5, alte_lr × 1.1)     ← schneller lernen

    Wenn improvement < -0.1:  (es wird schlechter)
        neue_lr = max(0.01, alte_lr × 0.8)    ← langsamer lernen

    Wenn |improvement| < 0.02 UND > 20 Datenpunkte:  (Stagnation!)
        neue_lr = min(0.3, alte_lr × 1.05)    ← leicht erhoehen

Trackt pro (Domaene, Strategie):
    strategy_effectiveness = succeeded / tried
    novelty_ratio = novel_count / seen_count
```

### FailurePostMortem — Aus Fehlern lernen

```
Bei JEDEM Misserfolg (success=False):

1. Kategorisieren:
   wrong_timing | wrong_action | insufficient | side_effect | external | unknown

2. Pattern-Hash:
   hash = md5(f"{action}:{problem_type}:{category}")[:10]

3. Wenn gleiches Pattern >= 3 mal:
   → Lesson generieren: "Diese Aktion funktioniert nicht bei diesem Problem"
   → In _lessons speichern → beeinflusst Voter-Confidence

4. Statistik pro Aktion:
   _failure_stats[action] = {total_failures, categories: {count pro Typ}}
```

### SuccessAmplifier — Goldene Regeln

```
Pro (problem_type, action) Paar wird Erfolgsrate getrackt:
    success_rate = successes / (successes + failures)

4 Verstaerkungs-Stufen:
    "initial":     min 1 Erfolg,  Rate >= 0.0 → Gewicht 1.0
    "confirmed":   min 3 Erfolge, Rate >= 0.6 → Gewicht 1.5
    "generalized": min 5 Erfolge, Rate >= 0.7 → Gewicht 2.0
    "golden_rule": min 10 Erfolge, Rate >= 0.9 → Gewicht 3.0

Goldene Regeln fliessen als Voter-Boost zurueck in decide()
```

### KnowledgeDistiller — Wissen destillieren

```
Alle 100 Entscheidungen: Regeln aus Erfahrungen extrahieren:

IF-THEN Regeln:
    Gruppiere nach (action, problem_type)
    Wenn success_rate >= 0.7 UND >= 3 Erfolge:
        → Erstelle Regel: "WENN problem=X DANN action=Y"
        → Finde gemeinsame Bedingungen aller Erfolge

AVOID Regeln:
    Wenn success_rate <= 0.3 UND >= 3 Misserfolge:
        → Erstelle Vermeidungsregel

TIMING Regeln:
    Extrahiere Uhrzeiten der Erfolge vs. Misserfolge
    → "Backup am besten zwischen 2:00-5:00"
```

### Strategie-Evolution (L27)

```
Population von Strategie-Gewichtungs-Vektoren:

Fitness pro Individuum:
    fitness = total_reward / decisions

Selektion:
    Top 25% ueberleben (Elitismus)

Crossover:
    Pro Strategie-Dimension: 50/50 von Elternteil A oder B

Mutation (30% Rate):
    delta = Gauss(0, 0.1)
    gewicht = clamp(gewicht + delta, 0.05, 2.0)
    → Renormalisierung

Zyklus: Nach 20 Evaluierungen × Populationsgroesse → neue Generation
```

---

## 6d. Voter-Zuverlaessigkeit und Dynamische Gewichtung (L28)

### VoterReliabilityTracker

```
Pro Voter wird nach JEDER Entscheidung getrackt:
    voter_correct += 1 wenn empfohlene_aktion == finale_aktion UND Erfolg

Zuverlaessigkeit (4 Faktoren):
    reliability = 0.40 × recent_accuracy      ← EMA ueber letzte 50
                + 0.25 × overall_accuracy      ← korrekt / total
                + 0.20 × (1 - calibration_error)  ← ECE pro Voter
                + 0.15 × consistency           ← 1 - 2×√(Varianz)

Consistency:
    mean = Mittelwert(letzte_ergebnisse)
    variance = Σ(ergebnis - mean)² / n
    consistency = max(0, 1 - √variance × 2)

ECE pro Voter (Kalibrierung):
    Gruppiere Vorhersagen in Confidence-Bins (0.1er Aufloesung)
    calibration_error = Σ (bin_size/total) × |bin_conf - bin_accuracy|
```

### DynamicVoterWeighting

```
Gewicht pro Voter:
    weight = (0.40 × base_reliability
            + 0.30 × performance_ema
            + context_bonus) × diversity_factor

    base_reliability = zuverlaessigkeit (siehe oben, cold-start: 0.3)
    performance_ema = 0.9 × alt + 0.1 × letzter_reward  (α=0.1)
    context_bonus = +0.15 wenn bester Voter in aktuellem Kontext

    diversity_factor:
        type_share = Anzahl_gleicher_Typ / Gesamtzahl
        diversity = 1 / max(type_share × Anzahl_Typen, 1)
        diversity = clamp(0.5, 1.5)

    Finales Gewicht: clamp(0.1, 2.0)
```

### DisagreementAnalyzer

```
Uneinigkeits-Level (Shannon Entropy):
    entropy = -Σ p_action × log2(p_action)
    normalized = entropy / max_entropy

3 Muster werden erkannt:
    1. "Reliable Disagreement": Zuverlaessige Voter (rel > 0.6) uneins
       → Hohe Unsicherheit → NOOP bevorzugen
    2. "Speed-Depth Conflict": Schnelle vs. tiefe Voter widersprechen
       → Tiefe Voter werden bevorzugt
    3. "Memory-Learning Conflict": Erfahrung vs. neues Lernen
       → Kontextabhaengig entschieden
```

### ConsensusQualityPredictor

```
quality = 0.30 × consensus_score
        + 0.25 × avg_reliability
        + 0.25 × (1 - disagreement_level)
        + 0.20 × min(1, voter_count / 8)

Wenn >= 5 historische Daten:
    predicted = 0.6 × historical + 0.4 × heuristic

Empfehlung:
    > 0.7: "trust_consensus"      (confidence × 1.1)
    > 0.5: "moderate_trust"       (confidence × 1.0)
    > 0.3: "low_trust"            (confidence × 0.8)
    sonst: "distrust_consensus"   (confidence × 0.6)
```

---

## 6e. Decision Resilience — Robustheit (L29)

### Adversarial Challenger

```
Nach JEDER Konsens-Entscheidung: 3 Gegen-Argumente pruefen:

1. Historische Fehlerrate (letzte 20 Versuche):
   recent_fail_rate = failures / total
   Wenn > 0.4: strength = min(0.9, fail_rate)

2. Transiente Probleme (loesen sich von selbst):
   transient_rate = resolved_alone / total_cases
   Wenn > 0.3: strength = min(0.8, transient_rate)

3. Schwache Voter-Einigkeit:
   agreement = winner_count / total_voters
   Wenn < 0.4: strength = 1 - agreement

Challenge Score:
   score = min(1.0, mean(scores) + max(scores) × 0.2)

   > 0.7: "reconsider"          → Override zu NOOP
   > 0.5: "proceed_with_caution"
   > 0.3: "minor_concerns"
   sonst: "proceed"
```

### Decision Fatigue Monitor

```
fatigue = 0.40 × rate_fatigue
        + 0.35 × quality_fatigue
        + 0.25 × oscillation_fatigue

Rate:
    decisions_per_minute = count(letzte 60s)
    rate_fatigue = min(1.0, rate / max_per_minute)  ← max 6/min

Quality:
    avg_erste_haelfte vs. avg_zweite_haelfte
    Wenn Qualitaets-Drop > 0.1:
        quality_fatigue = min(1.0, drop × 3)

Oscillation (Hin-und-Her):
    Wenn nur 2 verschiedene Aktionen UND > 60% Wechsel:
        oscillation_fatigue = 0.8

Reaktion:
    > 0.8: "force_pause" (30s Pause erzwungen)
    > 0.6: "throttle"    (10s Pause)
    > 0.4: "caution"
```

### Fallback-Ketten (Eskalation)

```
Vordefinierte Ketten pro Problem-Typ:
    "thermal":         [temp_throttle → housekeeping → service_check → suspend]
    "service_problem": [service_check → housekeeping → temp_throttle]
    "disk_full":       [housekeeping → log_rotate → backup_check]

Eskalations-Logik:
    1. Starte bei Schritt 0 der Kette
    2. Wenn Erfolg → Kette beenden, Outcome speichern
    3. Wenn 5 Minuten ohne Erfolg ODER Misserfolg → naechster Schritt
    4. Pro Schritt: Erfolgsrate tracken (successes / total)
    5. Gelernte Ketten ersetzen Defaults wenn bessere Erfolgsrate
```

### Reversibility Scorer

```
Basis-Scores (fest):
    noop: 1.0, service_check: 0.95, housekeeping: 0.8
    temp_throttle: 0.7, suspend: 0.3, reboot: 0.1

Gelernter Blend (nach >= 5 Beobachtungen):
    effective = 0.4 × basis + 0.6 × (reversible_count / total)

Kontext-Modifier:
    Aktive Verbindungen: -0.2 × min(1, conns / 5)
    Backup laeuft:       -0.5 fuer suspend/reboot
    Hauptzeiten (8-22):  -0.1 fuer aggressive Aktionen

Wirkung: Bei hoher Unsicherheit + niedriger Reversibilitaet
    → Automatischer Wechsel zu reversiblerer Aktion
```

---

## 6f. Alle Feedback-Loops im Ueberblick

### Loop-Diagramm

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DECIDE → EXECUTE → LEARN Zyklus                   │
│                         (alle 15 Sekunden)                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─ Snapshot ──→ decide() ──→ action ──→ execute() ──→ learn() ─┐  │
│  │                                                                │  │
│  │  ┌─────────── FEEDBACK-PFADE (learn → decide) ──────────────┐ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 1: BAYESIAN → Thompson Voter                        │ │  │
│  │  │    Beta(α,β) update → naechste decide() zieht Sample     │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 2: Q-LEARNING → qlearn Voter                        │ │  │
│  │  │    TD-Update → Q-Tabelle verbessert → bessere Q-Auswahl  │ │  │
│  │  │    Traces → vergangene Aktionen bekommen Credit/Blame     │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 3: BANDIT → bandit Voter                             │ │  │
│  │  │    Sherman-Morrison → UCB Score genauer → bessere Auswahl │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 4: VOTER RELIABILITY → Gewichtung                   │ │  │
│  │  │    Voter-Accuracy getrackt → Gewichte angepasst           │ │  │
│  │  │    Schlechte Voter verlieren Einfluss                     │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 5: EMOTION → Risiko-Toleranz                        │ │  │
│  │  │    Erfolg → valence↑ → mutiger                            │ │  │
│  │  │    Misserfolg → stress↑ → vorsichtiger                    │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 6: REGRET → Exploration                              │ │  │
│  │  │    Steigender Regret → ε += 0.01 → mehr Exploration      │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 7: CALIBRATOR → Confidence Governance                │ │  │
│  │  │    ECE > 0.15 → Confidence komprimiert → vorsichtiger     │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 8: META-LEARNER → Lernraten                         │ │  │
│  │  │    Stagnation → LR↑ / Verschlechterung → LR↓             │ │  │
│  │  │    → Q-Learning α und Bayesian Decay angepasst            │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 9: WORLD MODEL → Anomalie-Erkennung                 │ │  │
│  │  │    Seltener Uebergang (P<5%) → ε erhoehen                │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 10: GOLDEN RULES → Voter-Boost                      │ │  │
│  │  │    10+ Erfolge bei 90% Rate → 3× Gewicht fuer Voter      │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 11: FAILURE POSTMORTEM → Vermeidung                  │ │  │
│  │  │    3+ gleiche Fehler → Lesson → Aktion inhibiert          │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 12: CONCEPT DRIFT → Reset                            │ │  │
│  │  │    Drift erkannt → ε *= 1.5 + Schwellwerte halbiert       │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 13: CATASTROPHIC FORGETTING → Rollback               │ │  │
│  │  │    Drastischer Performance-Drop → Q-Tables restaurieren   │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 14: CAUSAL → Interventions-Effekte                   │ │  │
│  │  │    Kausaler Effekt gemessen → Predictive Baselines passt  │ │  │
│  │  │    sich an (cpu→temp Kette → temp-Vorhersage staerker)    │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 15: ADVERSARIAL → Schwellen-Kalibrierung             │ │  │
│  │  │    Challenge-Score validiert → Schwelle angepasst          │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 16: FALLBACK CHAINS → Gelernte Ketten                │ │  │
│  │  │    Eskalations-Schritte getrackt → Bessere Ketten gelernt │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 17: STRATEGY EVOLUTION → Voter-Gewichte               │ │  │
│  │  │    Fitness gemessen → Crossover + Mutation                 │ │  │
│  │  │    → Neue Strategie-Generation alle ~400 Entscheidungen    │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 18: LAYER PERFORMANCE → Adaptive Aktivierung         │ │  │
│  │  │    value = 0.5×impact + 0.3×latency + 0.2×error_rate     │ │  │
│  │  │    Low-Value Layer werden in decide() uebersprungen        │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 19: COHERENCE → Governance Gate                      │ │  │
│  │  │    coherence = 0.4×agreement + 0.3×temporal + 0.3×(1-con) │ │  │
│  │  │    Bei < 0.4 → NOOP erzwungen                             │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 20: KNOWLEDGE DISTILLER → Neue Regeln               │ │  │
│  │  │    Alle 100 Entscheidungen: IF-THEN + AVOID + TIMING      │ │  │
│  │  │    → Regeln fliessen in distilled_knowledge Voter          │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 21: DREAM CONSOLIDATION → Offline-Lernen             │ │  │
│  │  │    Bei Idle: Episoden wiederholen, Muster abstrahieren     │ │  │
│  │  │    → Konsolidiertes Wissen in dream_insight Voter          │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 22: MULTI-AGENT → Peer-Einfluss                     │ │  │
│  │  │    Peer-Stress → td_error *= (1 - peer_stress × 0.5)     │ │  │
│  │  │    → Gedaempftes Lernen bei Netzwerk-Stress               │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 23: BELIEF NETWORK → Exploration                     │ │  │
│  │  │    Belief-Update → Thompson-Sampling fuer Exploration      │ │  │
│  │  │    → Unsichere Beliefs → mehr Exploration                  │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 24: VOTE_DETAILS → 28+ Micro-Feedbacks              │ │  │
│  │  │    Override gerechtfertigt? → KnowledgeGraph update       │ │  │
│  │  │    MC-Simulation korrekt? → curves.record("mc_error")     │ │  │
│  │  │    Exploration erfolgreich? → ε anpassen                   │ │  │
│  │  │    Bias erkannt + Fehlschlag? → Aktion inhibieren          │ │  │
│  │  │    Degraded-Mode + Fehlschlag? → Exploration erhoehen     │ │  │
│  │  │                                                            │ │  │
│  │  │  ─── HIGHER COGNITION FEEDBACK (L41-L45) ───              │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 25: SEMANTIC KNOWLEDGE → Ontologie-Wachstum         │ │  │
│  │  │    Erfolg → add_relation(action, RESOLVES, problem)       │ │  │
│  │  │    Misserfolg → Relation-Strength verringern              │ │  │
│  │  │    Neue Konzepte aus ConceptFormation → IS_A Relationen   │ │  │
│  │  │    Induzierte Regeln aus Abstraction → RESOLVES eintragen │ │  │
│  │  │    Kausale Erkenntnisse → CAUSES Relationen               │ │  │
│  │  │    → Ontologie waechst organisch mit jeder Erfahrung      │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 26: CONCEPT FORMATION → Konzept-Evolution            │ │  │
│  │  │    observe(features) → Prototyp inkrementell updaten       │ │  │
│  │  │    Hohe Varianz → Split in Subkonzepte                     │ │  │
│  │  │    Aehnliche Konzepte → Merge                              │ │  │
│  │  │    Neue Konzepte → SemanticNet (Bridge 1)                 │ │  │
│  │  │    → Taxonomie passt sich an veraenderte Probleme an       │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 27: ACTIVE LEARNING → Exploration-Steuerung          │ │  │
│  │  │    Unsicherheit(state,action) hoch → exploration_rate↑     │ │  │
│  │  │    Unsicherheit niedrig → exploitation_rate↑               │ │  │
│  │  │    Wissensluecken → gap_boost auf active_explore Voter     │ │  │
│  │  │    → System exploriert gezielt wo Wissen fehlt             │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 28: HYPOTHETICAL → Kontrafaktisches Lernen           │ │  │
│  │  │    Nach jeder Entscheidung: Was waere mit Runner-Up?       │ │  │
│  │  │    Regret > 0.2 → Alternative als RESOLVES lernen         │ │  │
│  │  │    + Alternative als Erfahrung in Abstraction speichern    │ │  │
│  │  │    → Lernt auch aus Entscheidungen die es NICHT traf      │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 29: ABSTRACTION → Regel-Induktion + Validierung      │ │  │
│  │  │    Alle 10 Erfahrungen: Neue IF-THEN Regeln induzieren    │ │  │
│  │  │    Aehnliche Regeln → Prinzipien abstrahieren              │ │  │
│  │  │    Voter-Accuracy Feedback → Regel-Confidence anpassen     │ │  │
│  │  │    → Symbolisches Wissen emergiert aus Erfahrung           │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 30: RISK-GATE → Voter-Modulation                     │ │  │
│  │  │    Hypothetical erkennt hohes Risiko (>0.7)                │ │  │
│  │  │    → Riskante Alternativen anderer Voter abschwaechen      │ │  │
│  │  │    → Sicherere Aktionen werden bevorzugt                   │ │  │
│  │  │                                                            │ │  │
│  │  │  Loop 31: DREAM SCENARIOS → Proaktive Risiko-Erkennung    │ │  │
│  │  │    In Idle: SemanticNet bekannte Probleme → Hypothetical   │ │  │
│  │  │    "Was passiert wenn Problem X erneut auftritt?"           │ │  │
│  │  │    → Risiken in _bg_insights → beeinflusst naechste decide│ │  │
│  │  │                                                            │ │  │
│  │  └────────────────────────────────────────────────────────────┘ │  │
│  │                                                                │  │
│  └────────────── naechster Zyklus (15s spaeter) ──────────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Wie die Loops zusammenwirken (Beispiel: Service-Down)

```
Zyklus 1: Service faellt aus
    decide() → problem_type="service_down"
    84 Voter stimmen ab → Konsens: "service_check" (0.72)
      └─ semantic_net: find_resolution("service_down") → "service_check" (0.6)
      └─ hypothetical: compare_actions() → "service_check" (risk=0.2, reward=0.8)
      └─ abstraction: keine Regeln (noch zu wenig Erfahrung)
    execute() → systemctl restart smbd → Erfolg
    learn():
        Bayesian: α["service_down||service_check"] += 1
        Q-Learning: Q[service_down][service_check] += 0.1 × td_error
        Causal: service_check→service_down lift = +0.8
        VoterReliability: alle Voter die "service_check" empfahlen → correct++
        Emotional: valence += 0.2×(0.8×2-1) = +0.12 → "content"
        SuccessAmplifier: ("service_down","service_check") → 1 Erfolg
        ─── Higher Cognition ───
        SemanticNet: add_relation("service_check" RESOLVES "service_down", 0.86)
        ConceptFormation: observe({cpu:45, ram:60}) → Konzept "C0001" (service_issue)
        ActiveLearner: uncertainty("service_down","service_check") 0.25 → 0.15
        Hypothetical: observe_transition → Transitions-Modell waechst
        Abstraction: record_experience → Erfahrungs-Buffer fuellt sich

Zyklus 50: Gleiches Problem zum 10. Mal erfolgreich geloest
    SuccessAmplifier → "golden_rule" Level (Gewicht 3.0!)
    KnowledgeDistiller → IF service_down THEN service_check (Rate 0.95)
    MetaLearner → Domain "service_down": improvement +0.3 → LR × 1.1
    ─── Higher Cognition ───
    Abstraction → Regel induziert: IF service=down THEN service_check (conf=0.92)
    → Bridge 2: Regel → SemanticNet RESOLVES Relation (staerkt Ontologie)
    SemanticNet → service_check RESOLVES service_down (strength=0.95)
    ActiveLearner → uncertainty < 0.05 → exploration_rate -= 0.005

Zyklus 200: Service-Problem aendert sich (neuer Fehler-Typ)
    DriftDetector → Concept Drift erkannt!
    → ε *= 1.5 (mehr Exploration)
    → Schwellwerte halbiert
    → MetaLearner: novelty_ratio steigt → empfiehlt andere Strategie
    ─── Higher Cognition ───
    ConceptFormation → Neues Konzept "C0012" entdeckt (anderes Feature-Profil)
    → Bridge 1: C0012 → SemanticNet IS_A "service_down" (Subkategorie!)
    ActiveLearner → uncertainty("service_down","service_check") springt auf 0.8
    → Bridge 6: exploration_rate += 0.04 (gezielt explorieren)
    Hypothetical → counterfactual: Was waere mit "clear_caches" passiert?
    → Bridge 4: Regret=0.3 → "clear_caches" als Alternative in SemanticNet lernen
    Einige Zyklen spaeter: Neue Loesung gefunden → Alte Golden Rules zerfallen (Decay)

Idle-Phase (Dream):
    SemanticNet → "service_down" hat 3 Subtypen → Hypothetical simuliert:
    "Was wenn Subtyp C0012 erneut auftritt?" → Risiko erkannt → bg_insights
    ConceptFormation → Taxonomie konsolidiert: 2 aehnliche Konzepte gemerged
    ActiveLearner → 2 offene Wissensluecken erkannt → naechste Exploration gezielt
```

---

## 7. Aktions-Ausfuehrung

**Dateien:** `actions/executor.py`, `actions/backup.py`, `actions/updates.py`

### Verfuegbare Aktionen (`SystemAction` Enum, `perception.py:13-31`)

**44 statische Aktionen** in `ALL_STATIC` (repraesentative Auswahl unten). Die
Cognitive-Aktionsnamen werden im Executor per Alias-Map auf echte Handler abgebildet
(`actions/executor.py::_ACTION_ALIASES`, z.B. `cache_clear`→`clear_caches`).

**Stand nach dem Faehigkeits-Audit + Ausbau (40/44 mit echtem Handler):**
Echte Handler u.a.: `service_check` (systemd-Check + Restart failed units),
`temp_throttle` (Governor powersave, Fallback renice), `throttle_heavy_processes`
(renice +10 Top-CPU), `load_shed` (renice +19 aggressiv), `oom_protect`
(oom_score_adj +500 fuer RAM-Hogs), `tcp_tune` (sysctl-Haertung),
`emergency_cleanup` (Caches+Journal+Temp), `force_process_cleanup`
(Zombie-Reaping via SIGCHLD), `remount_check` (remount,rw NUR bei
fstab-rw-Evidenz), `backup_check` (Status/Alter aus StateManager, failed/
ueberfaellig ⇒ Fehlschlag), `kill_runaway_process` (Auto-Target ohne PID),
`restart_memory_leaking_service` (Top-RAM-Prozess → systemd-Unit → Restart,
Fallback Cache-Drop), `health_report`, `analyze_auth_logs`, `set_io_scheduler`.
**⚠ Verbleibende No-ops (nur Hardware, bewusst):** `bt_reconnect`,
`display_reconfigure`, `touch_recalibrate` (PASSIV) und `usb_reset`
(Registry-Fallback) — echte Umsetzung braucht Hardware-Schichten.
CI-Guard: `tests/test_capability_fixes.py`.

| Aktion | Konstante | Was passiert |
|--------|-----------|-------------|
| `noop` | `NOOP` | Nichts tun (System laeuft gut) |
| `suspend` | `SUSPEND` | System suspendieren (Energie sparen) — echter Handler |
| `backup_check` | `BACKUP_CHECK` | ✅ **ECHT** — Backup-Status/-Alter aus StateManager, ehrliche Bewertung |
| `service_check` | `SERVICE_CHECK` | ✅ **ECHT** — findet failed systemd-Units, startet bis zu 2 gezielt neu |
| `remount_check` | `REMOUNT_CHECK` | ✅ **ECHT** — erkennt ro-Mounts; remount,rw nur bei fstab-rw-Evidenz |
| `temp_throttle` | `TEMP_THROTTLE` | ✅ **ECHT** — CPU-Governor powersave; ohne cpufreq: renice Top-CPU |
| `housekeeping` | `HOUSEKEEPING` | Temp-Dateien, Caches aufraeumen — echter Handler |
| `log_rotate` | `LOG_ROTATE` | Log-Dateien rotieren — echter Handler |
| `health_report` | `HEALTH_REPORT` | ✅ **ECHT** — schreibt konkreten Report nach `state/health_report.json` |

**Weitere definierte Konstanten** (nicht in ALL_STATIC): `GOV_POWERSAVE`, `GOV_PERFORMANCE`
**Dynamische Aktionen:** `backup_manual_{path}`, `restart_{service}` (via Factory-Methoden)

### Ausfuehrungs-Pipeline (`actions/executor.py`)

```
ActionRequest Event
    ↓
1. Readiness-Check: readiness.is_safe_for_action(action)
2. Thread-Mutex: Nur eine Aktion gleichzeitig
3. Permission-Check: PrivilegeManager / Whitelist
4. Dispatch: ~40 Action-Handler (backup, suspend, restart, etc.)
5. Reward berechnen:
   │  Basis: success=0.8, fail=0.1
   │  Bonus: schnell(<1s)=+0.1, langsam(>30s)=-0.1
   │  Spezial: suspend=0.95, restart_verified=0.95, backup=0.9
   │  Clamp: [0.0, 1.0]
6. ActionCompleted Event publishen
```

### Weitere Action-Module

| Modul | Datei | Zweck |
|-------|-------|-------|
| HashBackupManager | `backup.py` | Hash-verifizierte Backups mit cp/rsync, Progress-Tracking |
| UpdateChecker | `updates.py` | apt/system Updates pruefen und installieren |
| SystemOptimizer | `optimizer.py` | CPU Governor, Cache-Tuning, Swappiness |
| ActionScheduler | `scheduler.py` | Zeitgesteuerte Aktionen (cron-aehnlich) |
| ActionRegistry | `action_registry.py` | Dynamische Aktions-Registrierung fuer Feedback |

---

## 8. Netzwerk und Externe Integration

### MQTT (Multi-Agent Kommunikation)

**Datei:** `network/mqtt.py` — Paho-MQTT v2

```
Nora ←──MQTT──→ Holo (Rule-based UI Mediator)
  │                │
  └──MQTT──→ Myuri (Independent KI)
```

**Topics:**

| Topic | Richtung | Zweck |
|-------|----------|-------|
| `ki/{agent}/inbox` | Empfang (QoS=1) | Direkte Nachrichten an diesen Agent |
| `ki/{agent}/response` | Senden | Antworten an andere KIs |
| `ki/{agent}/status` | Senden (retained) | Online/Offline Status |
| `ki/{agent}/heartbeat` | Senden | Device Discovery |
| `ki/group` | Bidirektional | Gruppen-Chat (alle KIs hoeren mit) |
| `ki/consensus/propose` | Senden | Konsens-Vorschlaege |
| `ki/consensus/vote` | Empfang | Abstimmungen |
| `home/presence/#` | Empfang | Smart-Home Praesenz |

**Fallback-Kette:** MQTT → HTTP Direct → UDP (wenn MQTT nicht erreichbar)
**Reconnect:** Exponential Backoff 1s → 120s, max 50 Warnungen

### Web Dashboard + REST API

**Datei:** `network/web.py` — Python `http.server` (kein Flask/FastAPI!)

**Authentifizierung:** Basic Auth + PBKDF2-SHA256 (260.000 Iterationen, OWASP 2024)
**HTTPS:** Optional, Self-Signed Cert Generation wenn aktiviert

**50+ API Endpoints** (Auswahl):

| Endpoint | Auth | Zweck |
|----------|------|-------|
| `GET /health` | Nein | Liveness-Check (Kubernetes/Docker) |
| `GET /metrics` | Nein | Prometheus OpenMetrics Export |
| `GET /api/state` | Ja | Kompletter System-Snapshot |
| `GET /api/cognitive/insights` | Ja | KI-Einsichten, Voter-Accuracy, Focus |
| `GET /api/agent/soul` | Ja | Noras Persoenlichkeit und Prinzipien |
| `GET /api/agent/trends` | Ja | Metriken-Trends + MetricsRing |
| `GET /api/infrastructure/health` | Ja | Watchdog, Threads, EventBus, CircuitBreaker |
| `GET /api/fleet` | Ja | Multi-Agent Fleet Status |
| `GET /api/ki/peers` | Ja | KI Peer-Netzwerk (Nora/Holo/Myuri) |
| `GET /api/alerts/active` | Ja | Aktive Eskalations-Alarme |
| `GET /api/audit` | Ja | Audit-Trail (100 letzte Eintraege) |
| `GET /api/audit/search?q=...` | Ja | Volltextsuche im Audit |
| `POST /api/action` | Ja | Aktion ausloesen |
| `POST /api/shell` | Ja | Terminal-Befehl ausfuehren |
| `POST /api/llm/chat` | Ja | Chat mit LLM (Ollama) |
| `POST /api/suspend` | Ja | System suspendieren |
| `POST /api/backup/now` | Ja | Sofort-Backup starten |
| `POST /api/device/wol` | Ja | Wake-on-LAN |

**Dashboard:** `dashboard.html` (443KB) mit Echtzeit-Updates via WebSocket

### Benachrichtigungen (11 Kanaele)

**Datei:** `notifications.py`

| Kanal | Protokoll | Config-Key |
|-------|-----------|------------|
| Discord | Webhook HTTP POST | `notifications.discord` |
| Telegram | Bot API | `notifications.telegram` |
| Pushover | REST API | `notifications.pushover` |
| Slack | Webhook | `notifications.slack` |
| Email | SMTP/TLS | `notifications.email` |
| ntfy | HTTP POST | `notifications.ntfy` |
| Gotify | HTTP POST | `notifications.gotify` |
| Signal | signal-cli REST | `notifications.signal` |
| Matrix | Homeserver API | `notifications.matrix` |
| Webhook | Generic HTTP | `notifications.webhook` |
| Apprise | Multi-Service Lib | `notifications.apprise` |

**Intelligentes Routing:**
- **Deduplizierung:** 5 Minuten Fenster fuer identische Nachrichten
- **Cooldown:** 30 Minuten pro Alert-Key (unterdrueckte werden zusammengefasst)
- **Retry:** 3 Versuche mit Backoff (2s → 5s → 10s)
- **Circuit Breaker:** Pro Kanal, deaktiviert nach wiederholtem Versagen
- **Lernen:** `_channel_reliability` trackt Erfolgsrate → bevorzugt zuverlaessige Kanaele
- **Worker-Thread:** Hintergrund-Queue (max 500), 10s Drain bei Shutdown

---

## 9. Stabilitaet und Sicherheit

**Verzeichnis:** `stability/`

| Modul | Datei | Zweck |
|-------|-------|-------|
| **SelfHealingEngine** | `self_healing.py` | Config-Snapshots, automatische Reparatur bei Drift |
| **FailoverCoordinator** | `failover.py` | Multi-Node Failover mit Leader-Election |
| **IntrusionDetector** | `intrusion_detect.py` | SSH-Login-Monitoring, Datei-Integritaet (SHA256), Rootkit-Indikatoren |
| **ThreatScorer** | `intrusion_detect.py` | Bedrohungs-Score (1-10) mit Decay (0.95/h), Severity-Level |
| **IntegrityGuardian** | `intrusion_detect.py` | Ueberwacht `/etc/ssh/`, `/etc/pam.d/`, `/etc/sudoers.d/`, Cronjobs |
| **AgentFleetManager** | `agent_fleet.py` | Fleet-Koordination, Multi-Agent Deployment (55KB!) |
| **PriorityMessageBus** | `message_bus.py` | Inter-Agent Message Bus mit Prioritaets-Queues |
| **DistributedTaskEngine** | `task_engine.py` | Verteilte Task-Ausfuehrung ueber mehrere Agents |
| **CanaryDeployer** | `canary_deploy.py` | Schrittweise Rollouts mit automatischem Rollback |
| **LocalAutonomyLayer** | `autonomy.py` | Lokale Entscheidungen auch ohne Fleet-Verbindung |
| **AgentRoleManager** | `agent_roles.py` | Rollen-Zuweisung: Leader, Follower, Specialist |
| **FaultInjector** | `fault_injection.py` | Chaos-Engineering: kontrollierte Fehler zum Testen |

### Sicherheits-Features

- **Passwort-Hashing:** PBKDF2-SHA256 (260.000 Iterationen, OWASP 2024)
- **SSL/TLS:** Selbst-signierte Zertifikate, automatische Generierung
- **Firewall-Monitoring:** UFW/nftables Regelzaehlung, geblockte Verbindungen
- **Zertifikat-Ueberwachung:** SSL-Ablauf 14 Tage vorher warnen
- **Datei-Integritaet:** SHA256-Hashes fuer kritische System-Dateien
- **Bedrohungs-Scoring:** Pro-Event Scoring (1-10), zeitbasierter Decay, Eskalations-Level

---

## 10. Persistenz

### State-Dateien

| Datei | Format | Inhalt |
|-------|--------|--------|
| `state/persistent_state.json` | JSON | System-Metriken, Service-Status, Backup-Status |
| `state/cognitive_state_v5.json` | JSON | Alle 208 Subsysteme serialisiert (Bayesian, Q-Tables, WorldModel, Beliefs, etc.) |
| `state/agent_memory.json` | JSON | Langzeit-Erinnerungen, System-Profil, Decision-Stats |
| `state/metrics_ring.mmap` | Binary mmap | 720-Eintrag Ring-Buffer (12h Metriken-Historie) |
| `data/mastermind.db` | SQLite | Episodische Metriken, historische Daten |

### Kognitive Persistenz (`_persistence.py`)

**`save()` serialisiert 65+ Eager-Subsysteme** via `to_dict()`:
```
semantic, episodic, knowledge, journal, bayesian, qlearn, causal,
bandit, policy, replay, transfer, curiosity, calibrator, world,
beliefs, forecaster, maintenance, goap, reasoning, optimizer,
conflict, intentions, control, meta, emotional, anomaly_clf,
pattern, timeseries, self_model, regret, profiler, curves,
narrator, energy, health, thresholds, habits, dream, identity,
audit, alerts, scheduler, goal_gen, coop, composer, budget,
adv_causal, strategy_composer, checkpoints, constraints, ...
```

**Lazy-Subsysteme:** Aktive Instanzen werden serialisiert, evicted State wird beibehalten.
**Metadaten:** Version, Zeitstempel, Decision-Count, Voter-Accuracy, BG-Insights, Focus-State, Stability-Tier.
**Atomar:** Schreibt in Temp-Datei, dann `rename()` → kein korrupter State bei Crash.
**Trigger:** Alle paar Minuten via Background-Task oder bei Shutdown.

### Audit Trail (`core/audit.py`)

| Eigenschaft | Wert |
|-------------|------|
| Format | JSONL (eine JSON-Zeile pro Event) |
| Datei | `data/audit/audit_YYYYMMDD.jsonl` (taegliche Rotation) |
| Max Groesse | 10MB pro Datei (behaelt neuere Haelfte) |
| Retention | 30 Tage automatische Bereinigung |
| In-Memory Buffer | 500 letzte Eintraege (Ring-Buffer) |
| Kategorien | `action`, `healing`, `threat`, `integrity`, `service`, `privilege`, `config`, `brain`, `notification`, `system` |

### Weitere Log-Dateien

| Datei | Format | Zweck |
|-------|--------|-------|
| `data/backup_log.jsonl` | JSONL | Backup-Historie mit Outcomes |
| `data/service_log.jsonl` | JSONL | Service Health/Restart Events |
| `data/access_log.jsonl` | JSONL | Netzwerk-Zugriffs-Tracking |
| `data/backup_hashes.json` | JSON | Hash-Datenbank fuer Backup-Verifizierung |

---

## 11. Abhaengigkeiten

**Minimal-Design:** Nur 2 optionale externe Dependencies!

```
psutil>=6.0.0        # Optional: System-Metriken (Fallback: /proc)
paho-mqtt>=2.0.0     # Optional: Nur wenn MQTT aktiviert
```

Alles andere: Python 3.11+ Standardbibliothek.

---

## 12. Konfiguration

**Datei:** `config.json` — gesucht in: `./config.json` → `~/config.json` → `/etc/mastermind/config.json`

### Haupt-Sektionen

| Sektion | Zweck | Beispiel |
|---------|-------|---------|
| `system_name` | Agent-Name | `"NAS Brain Agent"` |
| `system_role` | Rolle | `"nas"` / `"server"` / `"agent"` |
| `ai_brain` | KI-Engine | `enabled`, `confidence_threshold`, `llm_router` |
| `mqtt` | Multi-Agent | `enabled`, Broker-IP, Port, Agent-Name |
| `web_server` | Dashboard | `listen_ip`, `listen_port`, `auth_user`, `https_enabled` |
| `backup` | Backup-Paare | `backup_pairs: {"src": "dst"}`, `method`, `scan_interval` |
| `notifications` | 11 Kanaele | Pro Kanal: `enabled`, `webhook_url`/`token`/etc. |
| `services` | Dienste | Pro Service: `systemd_name`, `check_port`, `health_url` |
| `docker` | Container | `enabled`, `check_interval_s` |
| `watchdog` | Schwellwerte | `disk_percent`, `ram_percent`, `cpu_temp`, `cpu_temp_critical` |
| `system_limits` | Ressourcen | `cpu_busy_threshold`, `max_subprocesses`, `rss_warn_mb` |
| `autofix` | Auto-Repair | `enabled`, `max_retries`, `base_cooldown_minutes`, `backoff_multiplier` |
| `stability` | Fleet/HA | `enabled`, `failover`, `self_healing` |
| `ki_peers` | Peer-Liste | Array von `{name, ip, type}` |

**Validierung:** `core/config_validator.py` — Prueft Typen, Ranges, Pflichtfelder beim Start.
**Hot-Reload:** `SIGHUP` → Config neu laden ohne Neustart, alle Aenderungen geloggt.

---

## 13. Thread-Architektur

```
Main Thread          → run_health_monitor() (Watchdog-Loop)
├── SystemMonitor    → 5s Intervall
├── DiskMonitor      → 30s Intervall
├── AccessMonitor    → 4s Intervall
├── NetworkMonitor   → 10s Intervall
├── ServiceManager   → 30s Intervall
├── AIBrain          → 5s Intervall (decide() alle 15s)
├── Watchdog         → 10s Intervall (prueft alle anderen Threads)
├── BackupManager    → config Intervall
├── DockerMonitor    → 30s Intervall (optional)
├── GPUMonitor       → 120s Intervall (optional)
├── UPSMonitor       → 120s Intervall (optional)
├── ... (22+ weitere)
├── EventBus         → Consumer-Thread
├── UDP Listener     → Port 5005
├── MQTT Client      → Reconnect-Logik
└── WebServer        → Port 8009 (Threading)
```

Alle Threads sind `ResilientThread`: Auto-Restart nach Crash, max 5 Versuche, Exponential Backoff.

---

## 14. Core-Infrastruktur (`core/`, 34 Dateien)

| Modul | Zweck |
|-------|-------|
| `config.py` | Config-Laden, Hot-Reload, Defaults |
| `config_validator.py` | Typ-Pruefung, Pflichtfelder, Ranges |
| `events.py` | EventBus + 32 Event-Dataclasses |
| `state.py` | StateManager, MetricsRingBuffer, AgentMemory |
| `infrastructure.py` | GracefulShutdown, ReadinessGate, ThreadPool, ResilientThread |
| `sqlite_store.py` | SQLite-Persistenz fuer Metriken |
| `audit.py` | Audit-Logger mit taeglicher Rotation |
| `security.py` | PBKDF2 Hashing, SSL Context, Passwort-Migration |
| `circuit_breaker.py` | Circuit Breaker Pattern (Open/HalfOpen/Closed) |
| `health_aggregator.py` | Komponenten-Health zu Gesamt-Score |
| `event_bridge.py` | EventBus ↔ PriorityMessageBus Bridge |
| `event_reaktion_bridge.py` | Events → Automatische Reaktionen |
| `feedback_coupling.py` | 8 bidirektionale Feedback-Kopplungen |
| `integration_bridge.py` | Dependency-Checks, Wiring-Validierung |
| `prometheus.py` | Prometheus/OpenMetrics Export |
| `encryption.py` | Fernet-Verschluesselung fuer Secrets |
| `lazy_loader.py` | Lazy-Loading Infrastruktur |
| `log_rotation.py` | Log-Rotation (50MB max, 5 Backups, Kompression) |
| `utils.py` | PidFile, interruptible_sleep, Helper |

### Feedback-Kopplungen (`core/feedback_coupling.py`)

| Kopplung | Richtung | Was passiert |
|----------|----------|-------------|
| PersonaLearning | Knowledge → Persona | Neues Wissen passt Persoenlichkeits-Dimensionen an |
| MonitorKnowledge | ServiceFailure → Knowledge | Service-Fehler als Wissen speichern |
| ActionPersona | ActionCompleted → Persona | Aktions-Ergebnisse beeinflussen Stimmung |
| AlertFeedback | Alert → Monitor | Alarm-Outcomes kalibrieren Schwellwerte |
| CascadeFailure | Multi-Event → Alert | Erkennt Kaskaden-Ausfaelle |
| ThresholdLearning | ThresholdAdjusted → Learn | Schwellwert-Aenderungen in Lernsystem einspeisen |

---

## 15. Brain-Architektur (`brain/`, 13 Dateien)

| Modul | Zweck |
|-------|-------|
| `ai_brain.py` | Haupt-Loop: Snapshot → decide() → execute → learn() |
| `persona.py` | Noras Persoenlichkeit (Dimensionen: Vorsicht, Neugier, Empathie, etc.) |
| `soul.py` | Agent-Identitaet: Prinzipien, Werte, Ethik |
| `coordinator.py` | Brain-Koordination fuer Fleet-Betrieb |
| `llm_router.py` | Ollama-Integration: Chat, Dashboard-Summaries, Caching |
| `speech_module.py` | Sprach-/Text-Generierung |
| `semantic_log_analyzer.py` | Semantische Log-Analyse |

### Brain-Enhancements (`brain/enhancements/`, 8 Dateien)

| Enhancement | Zweck |
|-------------|-------|
| `confidence.py` | Adaptiver Confidence-Threshold pro Problem-Typ |
| `decision.py` | Decision Advisors: zusaetzliche Entscheidungs-Signale |
| `execution.py` | Aktions-Ausfuehrung mit Cooldown + Retry |
| `action_management.py` | Action-Workflows und Sequenzierung |
| `persistence.py` | Kognitive State-Persistenz fuer AIBrain |
| `holo_gate.py` | Holo-Kommunikations-Gateway |
| `soul_validator.py` | Prueft Aktionen gegen Soul-Prinzipien |

---

## 16. Agent Intelligence (`agent_intelligence/`, 6 Dateien)

Laeuft auf jedem Remote-Agent (nicht nur Mastermind):

| Modul | Zweck |
|-------|-------|
| `intelligence.py` | Haupt-Orchestrator fuer lokale Vor-Analyse |
| `collectors.py` | Metriken-Sammlung (CPU, RAM, Disk, Services) |
| `metrics.py` | Metriken-Aggregation und Normalisierung |
| `baseline.py` | Baseline-Berechnung fuer Anomalie-Erkennung |
| `problems.py` | Lokale Problem-Erkennung vor Eskalation an Mastermind |

---

## 17. Weitere Verzeichnisse

| Verzeichnis | Dateien | Zweck |
|-------------|---------|-------|
| `brain/enhancements/` | 8 | AIBrain-Erweiterungen: Confidence, Decision Advisors, Action Management, Persistence, HoloGate, SoulValidator |
| `agent_intelligence/` | 6 | Lokale Vor-Analyse: Metriken-Sammlung, Baseline, Problem-Erkennung (laeuft auf jedem Agent) |
| `state/` | 5 Dateien | Runtime: `persistent_state.json`, `cognitive_state_v5.json`, `metrics_ring.mmap`, `agent_memory.json`, `mastermind.log` |
| `data/` | variabel | Audit-Logs (`audit/`), Backup-Hashes, Service-Logs, SQLite DB (`mastermind.db`) |

---

## 18. Memory-Safety Architektur (Pi Zero 2W, 512 MB RAM)

**Stand:** 2026-03-27

Das gesamte kognitive System ist fuer 24/7-Dauerbetrieb auf einem Pi Zero 2W (512 MB RAM) optimiert.
Jede wachsende Datenstruktur hat explizite Groessenbegrenzungen mit LRU/LFU-Eviction.

### RAM-Budget Aufteilung

```
┌──────────────────────────────────────────────────────────┐
│                  Pi Zero 2W — 512 MB RAM                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────┐  ┌─────────────────────────────┐│
│  │    Linux OS + Libs  │  │   Mastermind-Nora Prozess   ││
│  │    ~80 MB           │  │                             ││
│  │  ┌───────────────┐  │  │  ┌───────────────────────┐  ││
│  │  │ Kernel + Deps │  │  │  │ Python Runtime  15 MB │  ││
│  │  │ systemd, libs │  │  │  ├───────────────────────┤  ││
│  │  └───────────────┘  │  │  │ Module Import   18 MB │  ││
│  └────────────────────┘  │  ├───────────────────────┤  ││
│                          │  │ 148 Lazy Module  41 MB │  ││
│  ┌────────────────────┐  │  │ (on-demand geladen)    │  ││
│  │   Andere Dienste   │  │  ├───────────────────────┤  ││
│  │   MQTT, Webserver  │  │  │ Arbeits-Speicher      │  ││
│  │   ~30 MB           │  │  │ decide+learn    8 MB  │  ││
│  └────────────────────┘  │  ├───────────────────────┤  ││
│                          │  │ deques+dicts   ~20 MB  │  ││
│  ┌────────────────────┐  │  │ (gekappt, stabil)     │  ││
│  │   Freier RAM       │  │  └───────────────────────┘  ││
│  │   ~280 MB Reserve  │  │        Gesamt: ~102 MB      ││
│  └────────────────────┘  └─────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
```

### Volllast-Profil (600s, 1500 Zyklen, alle 148 Module geladen)

```
RAM-Delta (MB)
  +90 ┤
  +80 ┤                                              ████████████
  +70 ┤                              ████████████████
  +60 ┤          ████████████████████
  +50 ┤  ████████
  +40 ┤
  +30 ┤
      └────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬──→ Zeit
          60s  120s 180s 240s 300s 360s 420s 480s 540s 600s

  Peak: +82 MB | Traced: 36 MB | Drift: +15 MB (stabil, kein Leak)
  Vorher (ohne Caps): +152 MB Peak, +90 MB Drift = LEAK!
```

### Memory-Cap Architektur

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MEMORY-CAP GUARD SYSTEM                           │
│                                                                     │
│  Jede Datenstruktur hat: [Max-Size] → [Trigger] → [Eviction]       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TIER 1: Hochfrequent (jeder decide/learn Zyklus)           │   │
│  │                                                             │   │
│  │  environment_model.py:                                      │   │
│  │    _transition_model ─┐                                     │   │
│  │    _reward_model     ─┤ 5 Dicts mit (state_hash, action)   │   │
│  │    _model_age        ─┤ keys — wachsen bei jedem Zyklus    │   │
│  │    _model_confidence ─┤                                     │   │
│  │    _fisher_info      ─┘ Cap: 2000 → trim 1000 (aelteste)  │   │
│  │                                                             │   │
│  │  self_supervised.py:                                        │   │
│  │    _transitions          Cap: 2000 → trim 1000 (LFU)      │   │
│  │    _mode_transitions     Cap:  200 → trim  100             │   │
│  │    _metric_correlations  Cap:  500 → trim  300 (wenigste n)│   │
│  │    _feature_importance   Cap:  200 → trim  100             │   │
│  │                                                             │   │
│  │  _learn.py:                                                 │   │
│  │    _voter_correlations   Cap:  500 → trim  300 (LFU)      │   │
│  │                                                             │   │
│  │  cognitive_core.py:                                         │   │
│  │    _access_counts        Cap:  300 → trim  200 (LFU)      │   │
│  │    _co_access            Cap:  300 → trim  200             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TIER 2: Mittelfrequent (bei Problemen / learn)             │   │
│  │                                                             │   │
│  │  autonomous_learning.py:                                    │   │
│  │    _lessons              Cap: 500 → trim 300 (LRU)        │   │
│  │    _failure_patterns     Cap: 500 → trim 300 (LRU)        │   │
│  │    _success_tracker      Cap: 500 → trim 300 (LFU)        │   │
│  │    contexts (per tracker) Cap: 100 → trim  50              │   │
│  │                                                             │   │
│  │  consensus_intelligence.py:                                 │   │
│  │    _performance_ema      Cap: 200 → trim 150               │   │
│  │    _feature_outcomes     Cap: 500 → trim 300 (LFU)        │   │
│  │                                                             │   │
│  │  decision_resilience.py:                                    │   │
│  │    _step_success         Cap: 500 → trim 300 (LFU)        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TIER 3: Niederfrequent (Background / Insights / User)      │   │
│  │                                                             │   │
│  │  collective_meta_intelligence.py:                           │   │
│  │    _layer_stats/_cumulative/_dependencies    Cap: 200       │   │
│  │    _relevance/_time_budgets                 Cap: 100       │   │
│  │    _step_stats/_pipeline_latency            Cap: 100-200   │   │
│  │    _last_recommendations/_stability_tracker Cap: 200       │   │
│  │                                                             │   │
│  │  causal_decision_theory.py:                                 │   │
│  │    _spontaneous_recovery/_layer_contributions Cap: 200     │   │
│  │    _action_attribution/_voter_attribution     Cap: 200     │   │
│  │                                                             │   │
│  │  error_anticipation.py:                                     │   │
│  │    _learned_precursors (outer)    Cap: 200                 │   │
│  │    _learned_precursors (per type) Cap:  50                 │   │
│  │    _verification/_adaptive_weights Cap: 200/500            │   │
│  │    _base_rates/_cascade_probs      Cap: 200/1000           │   │
│  │                                                             │   │
│  │  system_comprehension.py / user_cognitive_model.py /        │   │
│  │  wisdom_engine.py / thinking_methods.py:                    │   │
│  │    Diverse Dicts                   Cap: 200-1000           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  SONDERFALL: Reasoning-Kompaktierung                        │   │
│  │                                                             │   │
│  │  _record_decision() → Journal + Narrator:                   │   │
│  │                                                             │   │
│  │    VORHER:  reasoning dict (50+ Felder, 5-20 KB)           │   │
│  │    ┌─────────────────────────────────────┐                  │   │
│  │    │ strategy, confidence, decision_ms,  │                  │   │
│  │    │ risk_assessments, impact_sims,      │ × 2000 entries  │   │
│  │    │ evolved_weights, forecasts,         │ = 20-40 MB      │   │
│  │    │ fingerprint, explanations, ...      │                  │   │
│  │    └─────────────────────────────────────┘                  │   │
│  │                        ↓                                    │   │
│  │    NACHHER: _compact_reasoning (4 Felder, ~200 B)          │   │
│  │    ┌────────────────────────┐                               │   │
│  │    │ strategy, confidence,  │ × 2000 entries               │   │
│  │    │ decision_ms,           │ = 0.4 MB                     │   │
│  │    │ explanation_summary    │                               │   │
│  │    └────────────────────────┘                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Eviction-Strategien im Detail

```
LRU (Least Recently Used):          LFU (Least Frequently Used):
┌──────────────────────────┐        ┌──────────────────────────┐
│ Sortiere nach last_seen  │        │ Sortiere nach count/total│
│ oder model_age timestamp │        │ oder Beobachtungs-Anzahl │
│                          │        │                          │
│ Aelteste ────→ Loeschen  │        │ Seltenste ──→ Loeschen   │
│ Neueste  ────→ Behalten  │        │ Haeufigste ─→ Behalten   │
└──────────────────────────┘        └──────────────────────────┘

Trigger-Muster (ueberall gleich):
  if len(dict) > MAX:
      sorted_items = sorted(dict.items(), key=eviction_fn)
      for k, _ in sorted_items[:len(dict) - TARGET]:
          del dict[k]

  Beispiel: MAX=500, TARGET=300 → bei 501 Eintraegen werden 201 geloescht
```

---

## 19. Cross-Layer Integration Bridges (Neu)

**Stand:** 2026-03-27

Zusaetzlich zu den bestehenden 7 L41-L45 Cross-Module Bridges (Abschnitt 4) wurden
**35+ neue Bridges** eingefuegt, die bisher isolierte Subsysteme verbinden.

### Gesamtbild: Neue Bridge-Architektur

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   NEUE CROSS-LAYER BRIDGES                              │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                   _decide.py VOTER-MODULATION                   │   │
│  │                                                                 │   │
│  │  ┌──────────────────┐     ┌──────────────────────────────────┐ │   │
│  │  │  AdaptiveGoals   │────→│  Voter Boost/Suppress            │ │   │
│  │  │  (L36)           │     │                                  │ │   │
│  │  │  get_action_     │     │  stability-Goal aktiv?           │ │   │
│  │  │  priorities()    │     │    → restart_service:  +0.3      │ │   │
│  │  │                  │     │    → watchdog_restart: +0.3      │ │   │
│  │  │  Vorher: Ziele   │     │    → governor_perf:   ×0.6      │ │   │
│  │  │  existierten     │     │                                  │ │   │
│  │  │  aber trieben    │     │  efficiency-Goal aktiv?          │ │   │
│  │  │  keine Aktionen  │     │    → governor_powersave: +0.3   │ │   │
│  │  └──────────────────┘     │    → governor_perf:     ×0.6   │ │   │
│  │                           └──────────────────────────────────┘ │   │
│  │                                                                 │   │
│  │  ┌──────────────────┐     ┌──────────────────────────────────┐ │   │
│  │  │ AdaptiveStrategy │────→│  Kontext-Performance Boost       │ │   │
│  │  │ (L27)            │     │                                  │ │   │
│  │  │ get_context_     │     │  Tageszeit=Nacht, Strategy=qlearn│ │   │
│  │  │ insights()       │     │    success_rate=0.82, obs=15     │ │   │
│  │  │                  │     │    → qlearn Voter: +4.8% Boost   │ │   │
│  │  └──────────────────┘     └──────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              _bg_insights → _decide.py (6 neue Kanaele)        │   │
│  │                                                                 │   │
│  │  _background.py              _decide.py                        │   │
│  │  ┌─────────────────┐        ┌────────────────────────────────┐│   │
│  │  │ DreamConsolid.  │───────→│ dream_risks                    ││   │
│  │  │ simulate()      │  write │ → Risiko-Aktionen dampfen      ││   │
│  │  ├─────────────────┤        ├────────────────────────────────┤│   │
│  │  │ ActiveLearner   │───────→│ knowledge_gaps                 ││   │
│  │  │ get_gaps()      │        │ → Exploration-Boost bei Luecke ││   │
│  │  ├─────────────────┤        ├────────────────────────────────┤│   │
│  │  │ FatigueMonitor  │───────→│ fatigue_trend                  ││   │
│  │  │ get_trend()     │        │ → increasing: ×0.9 / high: ×0.85│   │
│  │  ├─────────────────┤        ├────────────────────────────────┤│   │
│  │  │ CoherenceCheck  │───────→│ coherence_issues               ││   │
│  │  │ check()         │        │ → Viele Widersprueche: ×0.98  ││   │
│  │  ├─────────────────┤        ├────────────────────────────────┤│   │
│  │  │ CurriculumLearn │───────→│ curriculum_focus               ││   │
│  │  │ get_focus()     │        │ → Domain-Match: Exploration +  ││   │
│  │  ├─────────────────┤        ├────────────────────────────────┤│   │
│  │  │ ConceptDrift    │───────→│ drift_recommendations          ││   │
│  │  │ get_recs()      │        │ → Monitoring-Aktionen boosten  ││   │
│  │  └─────────────────┘  read  └────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │            _insights.py TELEMETRIE (4 neue Kanaele)             │   │
│  │                                                                 │   │
│  │  Vorher verwaist          Jetzt in Insights-Pipeline            │   │
│  │  ┌──────────────────┐     ┌──────────────────────────────────┐ │   │
│  │  │ ArchitectureMap  │────→│ architecture_awareness.           │ │   │
│  │  │ .get_arch_map()  │     │   architecture_map               │ │   │
│  │  ├──────────────────┤     ├──────────────────────────────────┤ │   │
│  │  │ RemoteAgent      │────→│ architecture_awareness.           │ │   │
│  │  │ .get_distro_est()│     │   fleet_distro_estimates         │ │   │
│  │  ├──────────────────┤     ├──────────────────────────────────┤ │   │
│  │  │ AdaptiveLR       │────→│ adaptive_learning.               │ │   │
│  │  │ .get_all_rates() │     │   learning_rates                 │ │   │
│  │  ├──────────────────┤     ├──────────────────────────────────┤ │   │
│  │  │ AdaptiveEnsemble │────→│ adaptive_learning.               │ │   │
│  │  │ .get_top_subs()  │     │   top_subsystems                 │ │   │
│  │  └──────────────────┘     └──────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Datenfluss: Goal → Voter → Action

```
AdaptiveGoalSystem                           _decide.py Voter-Modulation
┌─────────────────────────┐
│ _active_goals:          │
│   stability:  prio=0.9  │──→ get_action_priorities()
│   efficiency: prio=0.6  │      │
│   resilience: prio=0.4  │      ▼
└─────────────────────────┘   ┌─────────────────────────────────────┐
                              │ boost:                               │
                              │   restart_service:    +0.27         │
                              │   watchdog_restart:   +0.27         │
                              │   service_check:      +0.27         │
                              │   governor_powersave: +0.18         │
                              │   housekeeping:       +0.18         │
                              │                                     │
                              │ suppress: [governor_performance]    │
                              │ urgency_override: 0.9               │
                              └──────────────┬──────────────────────┘
                                             │
                              ┌──────────────▼──────────────────────┐
                              │ Voter-Loop:                         │
                              │   for voter in voters:              │
                              │     if str(action) in boost:        │
                              │       confidence *= (1 + boost)     │
                              │     if str(action) in suppress:     │
                              │       confidence *= 0.6             │
                              └─────────────────────────────────────┘
```

### Datenfluss: bg_insights Pipeline

```
  _background.py (alle 150s)              _decide.py (alle 15s)
  ┌──────────────────────────┐            ┌────────────────────────┐
  │                          │            │                        │
  │ DreamConsolidator:       │   write    │  _bgi = _bg_insights   │
  │   simulate_scenarios()   │──────────→ │                        │
  │   risk_actions=[...]     │            │  dream_risks:          │
  │                          │            │    if action in risks: │
  │ ActiveLearner:           │            │      conf *= 0.85     │
  │   detect_knowledge_gaps()│──────────→ │                        │
  │   gaps=[{domain,prio}]   │            │  knowledge_gaps:       │
  │                          │   shared   │    if gap matches:     │
  │ FatigueMonitor:          │   dict     │      explore += 0.1   │
  │   get_trend()            │──────────→ │                        │
  │   trend="increasing"     │            │  fatigue_trend:        │
  │                          │            │    all conf *= 0.9    │
  │ CoherenceChecker:        │            │                        │
  │   get_chronic_issues()   │──────────→ │  coherence_issues:     │
  │   count=3                │   read     │    all conf *= 0.98   │
  │                          │            │                        │
  └──────────────────────────┘            └────────────────────────┘
```

---

## 20. Fehlende Handler (Systematisch ergaenzt)

**Stand:** 2026-03-27

### Problem: Caller ohne Handler

```
  _learn.py ruft nach jedem Zyklus ~25 Subsysteme auf:

  ┌──────────────────────────────────────────────────────────────┐
  │  learn(action, success, reward)                               │
  │    │                                                          │
  │    ├─→ self.pattern_abstractor.record_outcome(pt, action, …) │ ← FEHLTE
  │    ├─→ self.pipeline.record_outcome(action, success, reward)  │ ← FEHLTE
  │    ├─→ self.solution_evaluator.record_outcome(…)              │ ← FEHLTE
  │    ├─→ self.insight_gen.record_outcome(…)                     │ ← FEHLTE
  │    ├─→ self.preemptive_planner.record_outcome(…)              │ ← FEHLTE
  │    ├─→ self.rapid_insight.record_outcome(…)                   │ ← FEHLTE
  │    ├─→ self.failure_probability.record_outcome(…)             │ ← FEHLTE
  │    ├─→ self.multi_triage.record_outcome(…)                    │ ← FEHLTE
  │    ├─→ … (17 weitere)                                        │ ← FEHLTEN
  │    │                                                          │
  │    │   _safe() fing die AttributeErrors ab, aber:             │
  │    │   → 3150 Fehler / 100 Zyklen = CPU-Verschwendung        │
  │    │   → Kein Outcome-Feedback an Subsysteme                  │
  │    └──────────────────────────────────────────────────────────│
  └──────────────────────────────────────────────────────────────┘
```

### Loesung: 30 Handler hinzugefuegt

```
  Hinzugefuegte Methoden nach Kategorie:

  ┌─────────────────────────────────────────────────────────────────┐
  │  record_outcome(*args, **kwargs) → pass    (17 Klassen)         │
  │                                                                 │
  │  PatternAbstractor          ComprehensionFuser                  │
  │  StructuredThinkingPipeline ComprehensionMemory                 │
  │  ParallelSolutionEvaluator  ContextualRelevanceEngine           │
  │  LearningInsightGenerator   DisagreementAnalyzer                │
  │  PreemptiveActionPlanner    EthicsChecker                       │
  │  RapidInsightGenerator      DecisionExplainer                   │
  │  FailureProbabilityEstim.   InsightExtractor                    │
  │  MultiDimensionalTriage     AdaptiveLayerActivation             │
  │  SemanticPatternMatcher                                         │
  ├─────────────────────────────────────────────────────────────────┤
  │  Echte Implementierungen                    (13 Klassen)        │
  │                                                                 │
  │  ConceptDriftDetector    .is_drifting()         → bool          │
  │  RegretTracker           .regret_for_strategy() → float         │
  │  ConfidenceGovernance    .set_default_threshold()→ None         │
  │  ContinualLearningMgr   .get_protected_rules()  → list[dict]   │
  │  PatternMatcher          .stats()                → dict         │
  │  ActiveLearningEngine    .get_learning_focus_areas() → list     │
  │  LayerPerformanceTracker .record_layer_value()   → None         │
  │  QueryAccelerator        .put()                  → None         │
  │  OpportunityDetector     .record_success()       → None         │
  │  FederatedLearningHub    .add_to_replay_buffer() → None         │
  │  ActiveLearningEngine    .add_experience()       → None         │
  │  DreamConsolidator       .last_dream()           → dict         │
  │  SemanticIndex           .index_decision()       → None         │
  └─────────────────────────────────────────────────────────────────┘
```

### Korrigierte Aufruf-Signaturen

```
  VORHER (falsch)                              NACHHER (korrekt)
  ┌────────────────────────────────────┐      ┌────────────────────────────────────┐
  │ curriculum_learner.record_attempt( │      │ curriculum_learner.record_attempt( │
  │   pt, success, difficulty=1.0)     │  →   │   pt, success, difficulty_hint=1.0)│
  ├────────────────────────────────────┤      ├────────────────────────────────────┤
  │ meta_cognition.observe_outcome(    │      │ meta_cognition.observe_outcome(    │
  │   pt, action, success,             │  →   │   pt, action, confidence,          │
  │   confidence=0.5)  ← DOPPELT!     │      │   success, features={})            │
  ├────────────────────────────────────┤      ├────────────────────────────────────┤
  │ adaptive_goals.update_goal_progress│      │ adaptive_goals.update_goal_progress│
  │   (goal, reward, details={...})    │  →   │   (goal, reward, metrics={...})    │
  ├────────────────────────────────────┤      ├────────────────────────────────────┤
  │ temporal_patterns.record_event(    │      │ temporal_patterns.record_event(    │
  │   pt, action, success, hour)       │  →   │   f"{pt}_{action}",               │
  │   ← 4 Args, Methode nimmt 3!      │      │   details={"success":s,"hour":h})  │
  └────────────────────────────────────┘      └────────────────────────────────────┘
```

### Type-Coercion Architektur (unhashable dict Fix)

```
  Problem: SystemAction Enums/Dicts als Dict-Keys → TypeError

  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  decide() gibt zurueck: action = SystemAction.SERVICE_CHECK     │
  │                          (Enum, hashable)                        │
  │                                                                  │
  │  Aber manche Module speichern action als dict:                   │
  │    vote_details["_voter_snapshot"] = {voter: (action_dict, conf)}│
  │                                                                  │
  │  Wenn learn() diesen Snapshot weitergibt:                        │
  │    self.qlearn.update(state, action_dict, reward, next_state)   │
  │      → self._known.add(action_dict)  ← TypeError!               │
  │      → action_outcomes[action_dict]  ← TypeError!                │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘

  Loesung: 3-stufige str()-Coercion

  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │  Stufe 1: Am Eingang von learn()                                │
  │  ┌─────────────────────────────────────────────────────────┐   │
  │  │  def learn(self, action, success, reward, ...):          │   │
  │  │      action = str(action)  ← Zentrale Konvertierung     │   │
  │  └─────────────────────────────────────────────────────────┘   │
  │                                                                 │
  │  Stufe 2: In QLearning.update() (fuer Replay-Buffer Daten)     │
  │  ┌─────────────────────────────────────────────────────────┐   │
  │  │  def update(self, state, action, reward, next_state):    │   │
  │  │      action = str(action) if not isinstance(...) else …  │   │
  │  │      state  = str(state)  if not isinstance(...) else …  │   │
  │  └─────────────────────────────────────────────────────────┘   │
  │                                                                 │
  │  Stufe 3: An allen Dict-Key Stellen in Subsystemen              │
  │  ┌─────────────────────────────────────────────────────────┐   │
  │  │  advanced_intelligence.py:  key = str(a["action"])       │   │
  │  │  semantic_knowledge.py:     source = str(source)         │   │
  │  │  abstraction_engine.py:     key = (cond, str(action))    │   │
  │  │  layers/learning.py:        ba[str(cf["action"])]        │   │
  │  │  layers/memory.py:          from_e = str(from_e)         │   │
  │  └─────────────────────────────────────────────────────────┘   │
  │                                                                 │
  │  Ergebnis: 3150 Fehler/100 Zyklen → 0 Fehler/100 Zyklen       │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 21. Neue Features & Verbesserungen (April 2026)

> ~130 Verbesserungen in einer Session: Memory-Leaks, kognitive Qualitaet,
> Sicherheit, Stabilitaet, Performance, neue APIs, geschlossene Feedback-Loops.

### 21.1 REST-API Endpoints

| Methode | Pfad | Beschreibung |
|---------|------|-------------|
| GET | `/api/explain` | Letzte Entscheidung erklaert: Voter-Stimmen, Confidence, Problem-Typ, natuerlichsprachliche Zusammenfassung |
| POST | `/api/feedback` | User-Feedback: `{"type": "positive"\|"negative", "action": "...", "comment": "..."}` → Voter-Gewichte angepasst |
| POST | `/api/simulate` | Decision Dry-Run: `{"snapshot": {...}}` → Simulierte Entscheidung OHNE Ausfuehrung |
| GET | `/api/cognitive/export` | Kognitiver State als ZIP: cognitive_state_v5.json + WAL + edge_learner + peer_behavior |
| POST | `/api/cognitive/import` | State-Import: ZIP hochladen → Restore + Validierung (Pi-Migration) |
| GET/POST | `/api/maintenance-windows` | Wartungsfenster verwalten: `{"windows": [{"start_hour": 2, "end_hour": 6}]}` |

### 21.2 Sicherheits-Haertung

| Schicht | Vorher | Nachher |
|---------|--------|---------|
| Terminal-Exec | `Path(cmd).name` (umgehbar) | `shutil.which()` + `/usr/bin/` Whitelist |
| UDP Agent Auth | HMAC optional | HMAC mandatory, ohne Secret nur Heartbeat |
| LLM Prompts | `f"{user_query}"` direkt | `<user_input>` Tags + Max 4096 Chars |
| LLM Response | Unbegrenzt | Max 8KB + Content-Validierung |
| Ollama SSRF | Unkontrolliert | Nur private IPs (localhost, 10.x, 192.168.x) |
| Peer-Behavior | Reward unkontrolliert | Clamped [-1, 1] + Laengen-Limits |
| Encryption | XOR mit doppeltem Block | CTR-Mode + Warnung wenn kein AES |
| CertificateMonitor | `bash -c` + f-string | Direkter subprocess ohne Shell |
| Startup | SIGTERM crasht __init__ | `_init_complete` Flag + safe exit |

### 21.3 Memory-Management: Rolling-Caps

Alle zuvor unbegrenzten In-Memory-Dicts haben jetzt harte Obergrenzen:

| Dict | Modul | Cap | Eviction |
|------|-------|-----|----------|
| `_ki_reachability` | holo_comm.py | 50 | LRU nach last_ts |
| `_sessions` | totp_2fa.py | 200 | Expired + aelteste |
| `_attempts` | totp_2fa.py | 1000 | Expired + aelteste Haelfte |
| `_known_subnets` | security_ml.py | 500 | Seltenste 25% |
| `_counts/_means/_variances` | security_ml.py | 500 | Wenigste Samples |
| `_action_history` | autonomy.py | 200 | Seltenste 25% |
| `_source_reliability` | backup.py | 200 | Wenigste Nutzungen |
| `_command_outcomes` | devices.py | 200 | Aeltester Hostname |
| `_login_ips` | intrusion_detect.py | 1000 | (User-Dict) |
| `_context_knowledge` | learning.py | 500 | Heapq nsmallest |

**Stress-Test:** 10 Min, 24 Threads, 15M+ Ops → **+2.4 MB Delta, Trend +0.9 MB/h (stabil)**

### 21.4 Kognitive Entscheidungs-Pipeline: Neue Mechanismen

| Mechanismus | Beschreibung | Datei |
|-------------|-------------|-------|
| **Early-Exit** | Health>90 + keine Probleme → sofort NOOP (spart 143 Lazy-Loads) | `_decide.py` |
| **UrgencyScorer** | 5-Faktor Score (Typ, Service, Health, Peak, Wiederholung) | `urgency_scorer.py` |
| **Voter Warm-Up** | Erste 20 Entscheidungen: nur Heuristik, danach ML-Voter | `_decide.py` |
| **Temporal Q-Learning** | `state@h{hour}` + `state@d{weekday}` Voter | `_decide.py` + `_learn.py` |
| **Pareto Soft-Penalty** | Nicht-Pareto Voter: *0.4 statt Loeschung | `_decide.py` |
| **Health+Urgency Boost** | Low Health/High Urgency → Action-Voter *1.0-1.5x | `_decide.py` |
| **Subsystem Soft-Modifier** | Warnende Subsysteme reduzieren Consensus proportional | `_decide.py` |
| **Concept Drift Reset** | Drift → Voter-Weights Richtung Default + Exploration++ | `_decide.py` |
| **DecisionConfig** | 30+ Thresholds als Dataclass statt Magic Numbers | `decision_config.py` |
| **WAL** | Write-Ahead-Log: Outcomes auf Disk VOR learn(), Crash-Recovery | `_learn.py` |
| **Negativ-Feedback** | `record_noop_outcome()` bewertet Voter bei NOOP-Entscheidungen | `consensus_intelligence.py` |
| **Immutable Snapshots** | Shallow-Copy bei decide() → learn() sieht Entscheidungs-Zustand | `_decide.py` |
| **Voter-Recovery** | Reliability driftet zurueck Richtung 0.5 ueber Zeit | `consensus_intelligence.py` |
| **Semantic Symmetrie** | learn() schreibt RESOLVES + RESOLVED_BY bidirektional | `_learn.py` |
| **Beliefs Cap** | Alpha/Beta max 100 → Beliefs bleiben falsifizierbar | `world_model.py` |
| **Threshold Symmetrie** | +0.3/-0.3 statt +0.5/-0.2 → kein Aufwaerts-Drift | `world_model.py` |

### 21.5 Geschlossene Feedback-Loops

```
  decide() ──────────────────────────→ learn()
     │                                    │
     │  ┌─ Anomaly-Detector ←──── Outcome-Feedback (confirmed?)
     │  ├─ Forecaster ←──────────── Actual Values (cpu, ram)
     │  ├─ Voters ←────────────────── NOOP-Feedback (verpasste Chance?)
     │  ├─ Q-Learning ←────────────── Temporal: @h{hour} + @d{weekday}
     │  ├─ Causal ←────────────────── State-basiertes Update
     │  ├─ Health → Voter-Boost ────→ Aggressivitaet bei niedriger Health
     │  ├─ Subsystem-Recs → Consensus → Proportionale Confidence-Reduktion
     │  ├─ Voter-Reliability ←─────── Zeitbasierte Recovery
     │  └─ Outcomes ←──────────────── WAL (Crash-Recovery)
     │
     └──→ UrgencyScorer ──→ Voter-Boost Staerke
```

### 21.6 Stabilitaets-Mechanismen

| Mechanismus | Beschreibung |
|-------------|-------------|
| **Watchdog Isolation** | Jeder Check in eigenem try/except. 5 Failures → Check deaktiviert |
| **Healing-Loop Detection** | Max 3 Heals pro Service in 30 Min, danach Eskalation |
| **IP-Block Expiration** | Automatischer Unblock nach konfigurierbarer TTL (default 1h) |
| **Circuit Breaker Reset** | HALF_OPEN auto-reset nach 30s statt ewigem Stuck |
| **EventBus Recovery** | Graduelle Listener-Recovery (1 pro Zyklus statt alle gleichzeitig) |
| **Persistence Checksums** | SHA256 bei Save, Verifikation bei Load |
| **WAL Atomic Rename** | wal_mark_processed nutzt tmp+rename statt in-place Rewrite |
| **Fencing Tokens** | Failover-Epoch verhindert stale Actions nach Brain-Recovery |
| **State Type-Validation** | load_from_disk() prueft Feldtypen vor Merge |
| **Layer-Degradation** | Ab 10 Failures → Layer als degradiert geloggt, System laeuft weiter |

### 21.7 Performance-Architektur

| Optimierung | Vorher | Nachher | Impact |
|-------------|--------|---------|--------|
| Eager Module | 65 (~500 MB) | 43 (~300 MB) | ~200 MB gespart |
| Lock-Architektur | 1 Lock (4600 Zeilen) | `_lock` + `_state_lock` | API liest ohne decide()-Block |
| Snapshot-Kopie | `copy.deepcopy()` | Shallow dict copy | 5-10x schneller |
| Early-Exit | Nie | Health>90 → sofort NOOP | 70% der Aufrufe 6-8x schneller |
| Background-Lock | Unbegrenzt warten | `acquire(timeout=0.2)` | Kein Blocking bei decide() |
| Strategy-Liste | 3x `["thompson",...]` | `_STRATEGIES` Konstante | Weniger Allokationen |
| Membership-Test | `list` (O(n)) | `set` (O(1)) | 10-100x schneller |
| Metric-Map | Pro-Aufruf Dict | `_SEMANTIC_METRIC_MAP` Konstante | Kein GC-Druck |

### 21.8 Stress-Test Ergebnisse (20 Minuten)

```
  Phase       │ Dauer   │ RSS          │ RAM-Drift  │ decide() p50
  ────────────┼─────────┼──────────────┼────────────┼─────────────
  VOLLLAST    │ 6 min   │ 38 → 73 MB   │ +34 MB*    │ 15-21 ms
  IDLE        │ 6 min   │ 72.7 MB      │ 0.0 MB     │ —
  MIXED       │ 8 min   │ 72.7 → 73.4  │ +0.7 MB    │ 17-25 ms

  * Einmaliges Lazy-Loading beim ersten decide()/learn() Aufruf

  Peak RSS:           73.5 MB (14% von 512 MB Pi Zero)
  decide() Median:    17.4 ms
  decide() p95:       40.2 ms
  decide() p99:       56.6 ms
  learn() Median:     24.4 ms
  Throughput:         7,919 Entscheidungen in 20 Min
  Memory Leak:        Keiner nachweisbar
```

### 21.9 Prometheus Cognitive Metrics

```
  # Cognitive Decision Metrics (core/prometheus.py)
  nora_decisions_total     1523
  nora_degraded_layers     2
  nora_integrity_pct       98.1
  nora_wal_pending         0
```

### 21.10 Konfigurierbare Thresholds (DecisionConfig)

```python
  # cognitive/decision_config.py
  DecisionConfig:
    approval_threshold       = 0.6    # Min Score fuer Genehmigung
    weak_consensus_threshold = 0.4    # ConflictResolver-Trigger
    warmup_decisions         = 20     # ML-Voter Warmup
    pareto_soft_penalty      = 0.4    # Nicht-Pareto Confidence
    restart_policy_penalty   = 0.2    # Policy-Verbot Penalty

  VoterConfig:
    thompson_default_confidence    = 0.5
    qlearn_confidence_multiplier   = 2.33
    bandit_uncertainty_threshold   = 0.8
    goap_confidence_cap            = 0.6
    policy_base_confidence         = 0.45
```

### 21.11 UrgencyScorer (`cognitive/urgency_scorer.py`)

Neues Modul: Bewertet Dringlichkeit eines Problems auf einer 0-1 Skala.

```
  5-Faktor Berechnung:
  ┌──────────────────────────────────────────────┐
  │ 1. Problem-Typ Basis    (0.0 - 1.0)         │
  │    cascade_failure=1.0, service_down=0.9,    │
  │    security_threat=0.95, disk_full=0.8       │
  │                                              │
  │ 2. Service-Criticality  (aus Config)         │
  │    nginx=0.9, sshd=0.95, redis=0.7          │
  │    70% Problem-Typ + 30% Criticality        │
  │                                              │
  │ 3. Health-Score inverse (0.0 - 1.0)         │
  │    80% bisheriger Score + 20% Health-Faktor  │
  │                                              │
  │ 4. Peak-Hour Boost      (1.2x wenn Peak)    │
  │                                              │
  │ 5. Wiederholungs-Faktor (+0.1 pro Ausfall)  │
  │    > 3 Ausfaelle/h → Eskalation             │
  └──────────────────────────────────────────────┘

  Integration in decide():
    _urgency = urgency_scorer.score(problem_type, service, health, cpu, is_peak)
    → Beeinflusst Voter-Boost Staerke (1.0 + urgency * 0.5)
```

### 21.12 Fleet-Learning & Edge-Agent Feedback

```
  ┌───────────┐                    ┌───────────┐
  │  Agent 1  │──heartbeat────────→│           │
  │ (Pi Zero) │←─fleet_feedback───│   Brain   │
  ├───────────┤                    │  (Nora)   │
  │  Agent 2  │──heartbeat────────→│           │
  │ (Pi 4)    │←─fleet_feedback───│           │
  └───────────┘                    └───────────┘

  EdgeLearner (stability/autonomy.py):
    get_weekly_pattern()         → Wochentags-CPU-Profile (Mo-So)
    is_unusual_for_day(cpu, ram) → 2-Sigma Anomalie fuer diesen Wochentag
    receive_central_feedback()   → Fleet-Patterns: CPU by hour, bad procs
    get_aggregated_patterns()    → Sendet lokale Muster an Zentrale

  Orchestrator Fleet-Feedback (alle 10 Zyklen):
    Aggregiert CPU-Profile aller Agents → sendet an lokalen EdgeLearner
    Lokale Profile geblendet: 70% lokal + 30% Fleet-Durchschnitt
```

### 21.13 WAL (Write-Ahead-Log) & Crash-Recovery

```
  Datei: data/outcome_wal.jsonl (Rolling 500 Eintraege)

  Normaler Ablauf:
    1. learn() aufgerufen
    2. _wal_append() → Outcome atomar auf Disk geschrieben
    3. 40+ Learner verarbeiten Outcome
    4. wal_mark_processed() → Eintraege als verarbeitet markiert (atomic rename)

  Crash-Recovery:
    1. Orchestrator startet
    2. cognitive.replay_pending_outcomes() aufgerufen
    3. Liest alle unverarbeiteten WAL-Eintraege
    4. Replayed sie durch learn()
    5. Kein Outcome geht verloren

  WAL-Eintrag Format:
    {"ts": 1712345678, "action": "restart_nginx", "success": true,
     "reward": 0.8, "state": "high_cpu", "strategy": "thompson",
     "processed": false}
```

### 21.14 Layer-Degradation & Health-API

```python
  # CognitiveOrchestrator._safe() trackt degradierte Layer:
  # Ab 10 consecutive Failures → Layer als "degradiert" geloggt
  # System laeuft weiter ohne dieses Subsystem

  CognitiveOrchestrator.get_health_status() → {
      "total_failure_count": 42,
      "degraded_layer_count": 2,
      "degraded_layers": {
          "semantic_net.query": {"count": 15, "last_error": "...", "since": 1712345678},
      },
      "healthy": False,
      "integrity_pct": 98.1   # Prozent nicht-degradierter Layer
  }
```

### 21.15 Intelligente Retry-Logik (Fehlerklassifikation)

```
  record_outcome(action, success, error_type="transient|permanent|resource")

  ┌────────────────┬──────────────────────┬─────────────────────────┐
  │ Fehlertyp      │ Cooldown             │ Verhalten               │
  ├────────────────┼──────────────────────┼─────────────────────────┤
  │ transient      │ 10s * consecutive    │ Schneller Retry, max 3  │
  │ (Timeout, DNS) │ max 120s             │                         │
  ├────────────────┼──────────────────────┼─────────────────────────┤
  │ permanent      │ 60s * 2^consecutive  │ Exponential Backoff     │
  │ (not found)    │ max 3600s (1h)       │ bis manueller Fix       │
  ├────────────────┼──────────────────────┼─────────────────────────┤
  │ resource       │ 30s * 2^consecutive  │ Mittlerer Backoff       │
  │ (OOM, disk)    │ max 600s             │ bis Ressourcen frei     │
  ├────────────────┼──────────────────────┼─────────────────────────┤
  │ unknown (3+)   │ 60s * consecutive    │ Vorsichtiger Backoff    │
  │                │ max 600s             │ ab 3. Fehlschlag        │
  └────────────────┴──────────────────────┴─────────────────────────┘

  Automatische Klassifikation aus Error-Messages:
    "timeout|connection refused" → transient
    "not found|permission denied" → permanent
    "memory|oom|disk full" → resource
```

### 21.16 Gesamtbilanz der Session

```
  ┌────────────────────────────────────────────────────────┐
  │  VERBESSERUNGEN GESAMT: ~130                           │
  │  COMMITS: 15                                           │
  │  TESTS: 2536/2536 bestanden                            │
  │  DATEIEN GEAENDERT: 40+                                │
  │  ZEILEN GEAENDERT: ~2500+                              │
  ├────────────────────────────────────────────────────────┤
  │  Runde 1:  6 Memory-Leak Rolling-Caps                  │
  │  Runde 2:  7 Kognitive Schwaechen                      │
  │  Runde 3: 15 Architektur-Fixes                         │
  │  Runde 4: 37 Security + Stabilitaet                    │
  │  Runde 5: 11 Neue Features + APIs                      │
  │  Runde 6: 22 Tiefenanalyse-Fixes                       │
  │  Runde 7:  7 Feedback-Loops + Voter-Dynamik            │
  │  Runde 8:  8 Geschlossene Loops + Temporal-Learning    │
  │  Runde 9:  3 Final Polish                              │
  │  Refact.: Eager→Lazy + Lock-Split + Optimierungen      │
  ├────────────────────────────────────────────────────────┤
  │  ERGEBNIS:                                             │
  │    RAM:    73 MB Peak (14% von 512 MB Pi Zero)         │
  │    Latenz: 17 ms Median decide()                       │
  │    Leak:   0.0 MB ueber 6 Min Idle                     │
  │    Status: Produktionsreif                              │
  └────────────────────────────────────────────────────────┘
```
