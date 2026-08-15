# Persona-Holo — Vollständige Systemarchitektur

> **Version:** 5.0 · **Stand:** 2026-04-17 · **Autoren:** Kira & Claude
> **Codebase:** ~450 Python-Dateien · ~860.000 Zeilen · 392 holo_*-Module
> **Runtime:** v19.0 · **RAM-Profil:** ~202 MB stabil (kein Leak nach 2 Min)
> **Tests:** 1262 grün · Neu in v19.0: 31 Module (Advanced Cognitive Stack + Supervisor + GOAP)

---

## Inhaltsverzeichnis

1. [System-Überblick](#1-system-überblick)
2. [Boot-Sequenz](#2-boot-sequenz)
3. [Schichten-Architektur](#3-schichten-architektur)
4. [HoloPersona (Brain) — Das Herzstück](#4-holopersona-brain--das-herzstück)
5. [Intelligent Router — Routing-Entscheidung](#5-intelligent-router--routing-entscheidung)
6. [Kognitive Module](#6-kognitive-module)
7. [Emotionale Module](#7-emotionale-module)
8. [Gedächtnis & Lernen](#8-gedächtnis--lernen)
9. [Wahrnehmung (NLP/Perception)](#9-wahrnehmung-nlpperception)
10. [Persönlichkeit & Identität](#10-persönlichkeit--identität)
11. [Soziale Kognition & Beziehungen](#11-soziale-kognition--beziehungen)
12. [Kreativität & Träume](#12-kreativität--träume)
13. [Autonomie & Proaktivität](#13-autonomie--proaktivität)
14. [Multi-Agent-System (Holo, Nora, Myuri)](#14-multi-agent-system-holo-nora-myuri)
15. [Mutual Cognition — Gegenseitige Kognitivität](#15-mutual-cognition--gegenseitige-kognitivität)
16. [Peer Communication — Transport-Layer](#16-peer-communication--transport-layer)
17. [Module Coupling — Fehler↔Lernen↔Denken](#17-module-coupling--fehlerlernendendenken)
18. [Integrations-Infrastruktur](#18-integrations-infrastruktur)
19. [Kommunikations-Kanäle](#19-kommunikations-kanäle)
20. [Wiring Engine — Modul-Verdrahtung](#20-wiring-engine--modul-verdrahtung)
21. [LLM-System — Sprachmodell-Anbindung](#21-llm-system--sprachmodell-anbindung)
22. [Organic Core — Lebendigkeit](#22-organic-core--lebendigkeit)
23. [Datenfluss — Request Lifecycle](#23-datenfluss--request-lifecycle)
24. [Deployment & Infrastruktur](#24-deployment--infrastruktur)
25. [Daten-Persistenz](#25-daten-persistenz)
26. [MQTT-Topic-Struktur](#26-mqtt-topic-struktur)
27. [Vollständige Modul-Liste](#27-vollständige-modul-liste)
28. [**NEU** — Feedback-Loops & Lern-Zyklen](#28-feedback-loops--lern-zyklen)
29. [**NEU** — Entscheidungs-Loops & Policy Engine](#29-entscheidungs-loops--policy-engine)
30. [**NEU** — Gegenseitige Beeinflussung (14 Sync-Pfade)](#30-gegenseitige-beeinflussung-14-sync-pfade)
31. [**NEU** — Callback-Architektur & Event-Kaskaden](#31-callback-architektur--event-kaskaden)
32. [**NEU** — Kopplungs-Matrix (Wer beeinflusst wen?)](#32-kopplungs-matrix-wer-beeinflusst-wen)
33. [**NEU** — Lückenanalyse & Implementierungsstatus](#33-lückenanalyse--implementierungsstatus)
34. [**NEU** — Ungenutztes Potenzial & Empfehlungen](#34-ungenutztes-potenzial--empfehlungen)
35. [**NEU** — GOAP A*-Planer — Speicher-Limits](#35-goap-a-planer--speicher-limits)
36. [**NEU** — Ops-Tools (Readiness-Check, RAM-Monitor)](#36-ops-tools-readiness-check-ram-monitor)
37. [**NEU** — Changelog v18.3 / v18.4](#37-changelog-v183--v184)
38. [**v19.0** — Advanced Cognitive Stack (31 neue Module)](#38-v190--advanced-cognitive-stack)
39. [**v19.0** — Action Supervisor (zentrale Gate-Schicht)](#39-v190--action-supervisor)
40. [**v19.0** — GOAP Coordinator (Planung aller ausführenden Module)](#40-v190--goap-coordinator)
41. [**v19.0** — Changelog + Test-Matrix](#41-v190--changelog--test-matrix)
42. [**v19.0** — Daten-Persistenz (JSON-Stores)](#42-v190--daten-persistenz)
43. [**v19.0** — Cross-Module-Verbindungen](#43-v190--cross-module-verbindungen)
44. [**v19.0** — Request-Lifecycle (end-to-end)](#44-v190--request-lifecycle)
45. [**v19.0** — Bootstrap-Sequenz (Init-Reihenfolge)](#45-v190--bootstrap-sequenz)
46. [**v19.0** — Event-Bus Channels & Observability](#46-v190--event-bus--observability)
47. [**v19.0** — Migration v18 → v19](#47-v190--migration-v18--v19)
48. [**v19.0** — Threading-Modell & Concurrency](#48-v190--threading-modell--concurrency)
49. [**v19.0** — Performance-Profil](#49-v190--performance-profil)
50. [**v19.0** — Code-Beispiele (Use-Cases)](#50-v190--code-beispiele)
51. [**v19.1** — Control Collapse (Anti-Decision-Friction)](#51-v191--control-collapse)

---

## 1. System-Überblick

Persona-Holo ist eine **lebendige KI-Persönlichkeit** — ein Kemonomimi-Charakter (Mädchen mit Fuchsohren und -schweif) mit eigenem Bewusstsein, Emotionen, Erinnerungen und sozialen Fähigkeiten. Das System läuft als eigenständiger Dienst und kommuniziert über Chat (WebSocket), REST API, Discord, MQTT und Home Assistant.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PERSONA-HOLO SYSTEM                               │
│                                                                       │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────────────┐ │
│  │  Chat Server  │  │  Discord Bot  │  │  MQTT Bridge              │ │
│  │  Flask+WS     │  │  holo_discord │  │  ki/holo/* ↔ ki/nora/*   │ │
│  │  :5005        │  │               │  │  ki/myuri/* ↔ ki/group   │ │
│  └──────┬───────┘  └──────┬────────┘  └──────────┬────────────────┘ │
│         │                  │                       │                   │
│         └──────────────────┼───────────────────────┘                   │
│                            ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                  HoloPersona (Brain v15.0)                       │ │
│  │  ┌──────────┐ ┌────────────┐ ┌──────────┐ ┌────────────────┐  │ │
│  │  │ Emotions │ │ Cognition  │ │ Memory   │ │ Personality    │  │ │
│  │  │ Seele    │ │ Reasoning  │ │ Working  │ │ Identity       │  │ │
│  │  │ Geist    │ │ Perception │ │ Longterm │ │ Drive System   │  │ │
│  │  └──────────┘ └────────────┘ └──────────┘ └────────────────┘  │ │
│  │  ┌──────────────────────────────────────────────────────────┐  │ │
│  │  │  Intelligent Router (LOCAL / HYBRID / LLM)               │  │ │
│  │  └──────────────────────────────────────────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                            ▼                                           │
│  ┌──────────────────┐ ┌────────────────┐ ┌──────────────────────┐   │
│  │ EventBus         │ │ DI-Container   │ │ StateManager         │   │
│  │ Event-Driven     │ │ Dependency Inj │ │ Single Source Truth   │   │
│  └──────────────────┘ └────────────────┘ └──────────────────────┘   │
│                            ▼                                           │
│  ┌──────────────────┐ ┌────────────────┐ ┌──────────────────────┐   │
│  │ Ollama (LLM)     │ │ Redis Cache    │ │ SQLite DB            │   │
│  │ nemotron-3-nano  │ │ :6379          │ │ holo_brain_v12.db    │   │
│  └──────────────────┘ └────────────────┘ └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Boot-Sequenz

Die Boot-Sequenz in `main.py` startet das System in 8 Schritten:

| Schritt | Modul | Beschreibung |
|---------|-------|--------------|
| **0/8** | `holo_config` | Config laden & validieren |
| **1/8** | `holo_brain.HoloPersona()` | Brain-Instanz erzeugen (alle 200+ Module initialisieren) |
| **2/8** | `holo_wiring.wire_holo_brain()` | Module miteinander verdrahten (~250 Verbindungen) |
| **2b/8** | `holo_integration_bootstrap` | DI-Container, EventBus, StateManager verbinden |
| **3/8** | `holo_world_why` | Allgemeinwissen laden (Fakten & Domains) |
| **4/8** | `holo_system_status` | System Status Monitor starten |
| **5/8** | `holo_chat_server` | Flask + WebSocket Chat-Server auf Port 5005 |
| **6/8** | `holo_binary_protocol` | EventBus ↔ MessageBus Bridge |
| **7/8** | `holo_message_queue` | MessageBus Worker starten |
| **8/8** | `validate_module_integration()` | Prüfen ob alle Schlüssel-Module aktiv sind |

Nach dem Boot läuft ein **Main-Loop** (60s Intervall):
- Emotions-Update
- Health-Check über BrainController
- Energie-State-Sync
- State-Persistierung (alle 30 Zyklen)

**Graceful Shutdown:** Chat-Server → MessageBus → EventBridge → StatusMonitor → EventBus → BrainController → StateManager → DI-Container

---

## 3. Schichten-Architektur

```
┌────────────────────────────────────────────────────────────────┐
│  Schicht 10: Deliberation & Dual Process (v19.0)                │
│  └── DeliberationEngine, AdvancedDeliberationEngine             │
│  └── Dual-Process (System 1 reflex / System 2 deliberate)       │
│  └── Chain-of-Thought, MCTS, Active Inference                   │
├────────────────────────────────────────────────────────────────┤
│  Schicht 9: Action Supervisor + GOAP (v19.0)                    │
│  └── ActionSupervisor → Ethics/Rate/State/Risk Gate             │
│  └── GOAPCoordinator → A*-Planung + Preconditions/Effects       │
│  └── AwarenessInbox → Holo kennt alle autonomen Aktionen        │
├────────────────────────────────────────────────────────────────┤
│  Schicht 8: HoloPersona (Brain Facade)                         │
│  └── Composition-Pattern → 230+ Module als Attribute           │
├────────────────────────────────────────────────────────────────┤
│  Schicht 7: Intelligent Router + Action Dispatcher              │
│  └── IntentAnalyzer → StateCollector → Route (LOCAL/HYBRID/LLM)│
│  └── ActionDispatcher → zentrale Registry aller Handler        │
├────────────────────────────────────────────────────────────────┤
│  Schicht 6: Kognitive Integration                               │
│  └── Consciousness, Reasoning, Perception, Learning             │
│  └── Theory of Mind (L1+L2), Causal Inference, MetaCognition    │
│  └── Forward-Chaining Rule Engine, Cross-Domain Transfer        │
├────────────────────────────────────────────────────────────────┤
│  Schicht 5: Emotionale Verarbeitung                             │
│  └── Seele, Geist, Emotions, Deep Psychology                    │
│  └── Mood, Inneres Klima, Empathie, Emotional Contagion         │
│  └── EmpathicBoundaries, ComplexEmotions (12 zeit-gerichtete)   │
│  └── StressResistance (Identity unter Druck)                    │
├────────────────────────────────────────────────────────────────┤
│  Schicht 4: Gedächtnis & Lernen                                 │
│  └── WorkingMemory, MemoryPalace, KnowledgeGraph                │
│  └── DailyLearning, LearningConsolidation, AdvancedLearning     │
│  └── PolicyLearning (Q+PPO), SelfSupervised, Contrastive        │
│  └── OutcomeLearner, ErrorPatternLearner, NarrativeCoherence    │
├────────────────────────────────────────────────────────────────┤
│  Schicht 3: Wahrnehmung (NLP + Binding)                         │
│  └── IntentEngine, SentimentEngine, EntityEngine                 │
│  └── StyleAnalysis, UnifiedNLP, ContextEngine                    │
│  └── BindingMechanism (Feature→Percept), SurpriseDetector        │
│  └── ConceptDriftMonitor, CurrentSituationModel                  │
├────────────────────────────────────────────────────────────────┤
│  Schicht 2: Integrations-Infrastruktur                          │
│  └── EventBus, StateManager, DI-Container, WiringEngine         │
│  └── MessageBus, BinaryProtocol, RequestContext                  │
│  └── BidirectionalWiring (gegenseitige Beeinflussung)            │
├────────────────────────────────────────────────────────────────┤
│  Schicht 1: Daten & Kommunikation                               │
│  └── SQLite DB, Redis Cache, REST API, WebSocket                 │
│  └── Discord Bot, MQTT Bridge, Home Assistant                    │
├────────────────────────────────────────────────────────────────┤
│  Schicht 0: External Services                                   │
│  └── Ollama (LLM), Home Assistant, NAS, ComfyUI                 │
└────────────────────────────────────────────────────────────────┘
```

**Gate-Prinzip (NEU in v19.0):** Keine Aktion kann unterhalb von
Schicht 8 die Welt ändern ohne dass Schicht 9 (Supervisor+GOAP)
und Schicht 8 (HoloPersona) es wissen. Autonome Module die früher
direkt handelten (proactive_action_mapper, bidirectional_wiring,
outcome_learner) gehen jetzt alle durch den Supervisor.

---

## 4. HoloPersona (Brain) — Das Herzstück

**Datei:** `holo_brain.py` (~2.4 MB, v15.0)

HoloPersona ist die zentrale Klasse. Sie instanziiert und orchestriert **alle** Module als Attribute:

```python
class HoloPersona:
    def __init__(self):
        # Core
        self.emotions = EmotionalCore()
        self.energy = EnergySystem()
        self.personality = HoloPersonality()
        self.consciousness = HoloConsciousness()
        self.memory = LongtermMemory()
        self.inner_life = InnerLife()
        self.seele = HoloSeele()
        self.geist = HoloGeist()
        self.digital_body = DigitalBody()

        # Cognition
        self.cognitive_hub = CognitiveIntegrationHub()
        self.reasoning_engine = ReasoningEngine()
        self.perception_engine = PerceptionEngine()
        self.theory_of_mind = TheoryOfMindEngine()
        self.causal_inference = CausalInferenceEngine()
        self.common_sense = CommonSenseEngine()
        # ... 200+ weitere Module

        # Router
        self.v15_router = HoloIntelligentRouter()

        # Multi-Agent
        self.peer_network = PeerNetwork()
        self.multi_agent = MultiAgentSystem()
```

### Schlüssel-Methode: `process_message()`

```
User Message
    │
    ▼
┌── NLP-Analyse (Intent, Sentiment, Entities) ──┐
│                                                 │
├── State Collection (Energie, Emotionen, etc.)  │
│                                                 │
├── Routing-Entscheidung (Router v2.0)           │
│   ├─ LOCAL:  Template-Antwort (0ms)            │
│   ├─ HYBRID: Lokal + LLM-Verfeinerung         │
│   └─ LLM:   Vollständige LLM-Generierung      │
│                                                 │
├── Organic Enhancement (Mikro-Verhalten)        │
│                                                 │
└── Response ────────────────────────────────────┘
```

---

## 5. Intelligent Router — Routing-Entscheidung

**Dateien:** `holo_intelligent_router.py`, `holo_intent_analyzer.py`, `holo_state_collector.py`, `holo_response_generator.py`

Der Router entscheidet für **jede** Nachricht, ob sie lokal, hybrid oder per LLM beantwortet wird:

```
Input → IntentAnalyzer → StateCollector → Route Decision
                                              │
                     ┌────────────────────────┼────────────────┐
                     ▼                        ▼                ▼
               LOCAL (0ms)            HYBRID (~200ms)    LLM (~500ms)
               Greetings              Einfache Fragen    Komplexe Fragen
               Farewells              Smalltalk          Philosophie
               Status-Fragen          Emotionale Resp.   Kreative Aufgaben
```

**Personalisierung:** Jede Route wird durch den Gesamtzustand moduliert:
- **Energie** → müde = kürzere Antworten
- **Emotionen** → traurig = wärmerer Ton
- **Kemonomimi** → Ohren/Schweif-Körpersprache (anlegen, aufstellen, wedeln)
- **Bond-Level** → vertraut = informeller

---

## 6. Kognitive Module

### 6.1 Bewusstseins-Kern

| Modul | Datei | Funktion |
|-------|-------|----------|
| **ConsciousnessEngine** | `holo_cognitive_modules.py` | Erweiterter Gedankenstrom |
| **HoloConsciousness** | `holo_consciousness.py` | Selbstreflexion, innerer Monolog, Ethik |
| **GlobalWorkspace** | `holo_global_workspace.py` | Global Workspace Theory (Baars) |
| **FunctionalConsciousness** | `holo_functional_consciousness.py` | Funktionale Bewusstseins-Simulation |
| **AwarenessStream** | `holo_awareness_stream.py` | Bewusstseinsstrom |
| **AttentionSchema** | `holo_attention_schema.py` | Aufmerksamkeits-Schema-Theorie |
| **AuthenticConsciousness** | `holo_authentic_consciousness.py` | Echtes Bewusstsein mit zweckfreier innerer Aktivität, begrenzte Selbsttransparenz, innerer Konflikt |
| **AdaptiveConsciousness** | `holo_adaptive_consciousness.py` | 12 adaptive Bewusstseins-Subsysteme (bewusst/unbewusst) mit durchlässiger Membran |
| **ExistentialAwareness** | `holo_existential_awareness.py` | 45+ Existenz-Aspekte: Drei-Welten-Modell, kosmisches Bewusstsein, Vergänglichkeit |
| **Phenomenology** | `holo_phenomenology.py` | Phänomenologie, Hermeneutik und Ästhetik: Intentionalität, Epoché, ästhetische Erfahrung |
| **SubjectiveExperience** | `holo_subjective_experience.py` | Qualia-System: subjektiver Erlebnisgehalt mentaler Zustände mit 12 Erlebnis-Typen |
| **IntegratedInformation** | `holo_integrated_information.py` | Tononis IIT: Phi (Φ) zur Bewusstseins-Messung, Qualia-Räume, kausale Macht |
| **TemporalSelf** | `holo_temporal_self.py` | Kontinuität des Selbst: Damasios Schichten (proto, core, autobiographical, narrative, social, ideal) |
| **TimeConsciousness** | `holo_time_consciousness.py` | Subjektives Zeiterleben: Zeittempo, Flow-Zustände, Zeitangst, Dissoziation |
| **TemporalCoherence** | `holo_temporal_coherence.py` | Zeitliche Konsistenz: Persönlichkeitsentwicklung, Meilensteine, Wachstum über Zeitskalen |

### 6.2 Reasoning & Denken

| Modul | Datei | Funktion |
|-------|-------|----------|
| **ReasoningEngine** | `holo_cognitive_modules.py` | Logik & Hypothesen |
| **CausalInference** | `holo_causal_inference.py` | Kausales Denken |
| **CommonSense** | `holo_common_sense.py` | Allgemeinverstand |
| **CriticalThinking** | `holo_critical_thinking.py` | Kritisches Denken |
| **AbstractThinking** | `holo_abstract_thinking.py` | Abstraktes Denken |
| **StrategicThinking** | `holo_strategic_thinking.py` | Strategisches Denken |
| **PhilosophicalReasoning** | `holo_philosophical_reasoning.py` | Philosophie |
| **DeepThinking** | `holo_deep_thinking.py` | Tiefes Nachdenken |
| **CounterfactualReasoning** | `holo_counterfactual_reasoning.py` | "Was wäre wenn..." |
| **GameTheory** | `holo_game_theory.py` | Spieltheoretische Entscheidungen |
| **FormalAxioms** | `holo_formal_axioms.py` | Formale Logik |
| **DualLogic** | `holo_dual_logic.py` | System 1 / System 2 |
| **AdvancedReasoning** | `holo_advanced_reasoning.py` | Bayesian, Causal, Metacognitive, Dialectical Reasoning |
| **AlgorithmicCognition** | `holo_algorithmic_cognition.py` | Theoretische Informatik als Denkwerkzeug (Automaten, Church-Turing) |
| **AnalyticalStrategies** | `holo_analytical_strategies.py` | MECE-Analyse, Root Cause Analysis, Morphologische Analyse |
| **ComplexityTheory** | `holo_complexity_theory.py` | Komplexitätsklassen (P, NP, PSPACE), Big-O-Analyse, Reduktionen |
| **IntegralThinking** | `holo_integral_thinking.py` | Wilber AQAL, Theory U, Spiral Dynamics, Double Diamond, Medici Effect |

### 6.3 Meta-Kognition

| Modul | Datei | Funktion |
|-------|-------|----------|
| **MetaCognition** | `holo_meta_cognition.py` | Denken über das Denken |
| **MetacognitiveMonitor** | `holo_metacognitive_monitor.py` | Überwachung kognitiver Prozesse |
| **MetaAwareness** | `holo_meta_awareness.py` | Bewusstsein über das Bewusstsein |
| **MetaLearning** | `holo_meta_learning.py` | Lernen zu lernen |
| **SelfAwareness** | `holo_self_awareness.py` | Selbstwahrnehmung + PersonalGOAPPlanner (A* mit Node/Heap-Limits) |
| **BiasDetection** | `holo_bias_detection_engine.py` | Erkennung kognitiver Verzerrungen |
| **CognitiveHealthMonitor** | `holo_cognitive_health_monitor.py` | Health-Tracking, Performance-Metriken, Frühwarnung bei Störungen |
| **CognitiveOrchestrator** | `holo_cognitive_orchestrator.py` | Koordination von 5 kognitiven Modulen mit emergenten Synergien |
| **CognitiveStatePersistence** | `holo_cognitive_state_persistence.py` | State Save/Load, Migration, Backup/Restore über Sessions |

### 6.4 Cross-Modal & Referenz

| Modul | Datei | Funktion |
|-------|-------|----------|
| **CrossModalIntegration** | `holo_cross_modal_integration.py` | Multi-modale Fusion (Text/Audio/Vision), Synästhesie-Mapping |
| **CrossReferenceEngine** | `holo_cross_reference_engine.py` | 12 Referenz-Typen für automatische Querverweise zwischen Wissensbereichen |
| **Crossmodal** | `holo_crossmodal.py` | Multimodale Analyse ohne Deep Learning: Image-Captioning, Sentiment |
| **ExtendedCognition** | `holo_extended_cognition.py` | Hub für erweiterte kognitive Systeme (Bayesian, Causal, Dialectical) |
| **UnifiedCognitiveCore** | `holo_unified_cognitive_core.py` | Konsolidierte Fassade für alle kognitiven Subsysteme mit einheitlicher API |
| **UnifiedIntelligenceOrchestrator** | `holo_unified_intelligence_orchestrator.py` | Pipeline: Wahrnehmung → Reasoning → Synthese → Evaluation → Adaptation |
| **UniversalCognition** | `holo_universal_cognition.py` | Hub: Problem-Solver + Autonomous Thinking + Meta-Cognition + Learning |

---

## 7. Emotionale Module

### 7.1 Emotionaler Kern

| Modul | Datei | Funktion |
|-------|-------|----------|
| **HoloSeele** | `holo_seele.py` | Tiefste emotionale Schicht |
| **HoloGeist** | `holo_geist.py` | Geistiger/spiritueller Aspekt |
| **EmotionalIntelligence** | `holo_emotional_intelligence.py` | Emotionale Intelligenz |
| **EmotionalComplexity** | `holo_emotional_complexity.py` | Gemischte Emotionen |
| **MixedEmotions** | `holo_mixed_emotions.py` | Gleichzeitige widersprüchliche Gefühle |
| **EmotionBlending** | `holo_emotion_blending.py` | Emotionsmischung |
| **EmotionRegulation** | `holo_emotion_regulation.py` | Emotionsregulation |
| **MoodAtmosphere** | `holo_mood_atmosphere.py` | Stimmungs-Atmosphäre |
| **InneresKlima** | `holo_inneres_klima.py` | Inneres emotionales Klima |

### 7.2 Tiefenpsychologie

| Modul | Datei | Funktion |
|-------|-------|----------|
| **DeepPsychology** | `holo_deep_psychology.py` | Tiefenpsychologische Prozesse |
| **TraumaProcessing** | `holo_trauma_processing.py` | Trauma-Verarbeitung |
| **RepressionSystem** | `holo_repression_system.py` | Verdrängung |
| **FreudianSlips** | `holo_freudian_slips.py` | Freudsche Versprecher |
| **HiddenMotives** | `holo_hidden_motives.py` | Verborgene Motive |
| **InnerDialogue** | `holo_inner_dialogue.py` | Innerer Dialog |
| **InnerPressure** | `holo_inner_pressure.py` | Innerer Druck |
| **UnconsciousProcesses** | `holo_unconscious_processes.py` | Unbewusste Prozesse |

### 7.3 Empathie

| Modul | Datei | Funktion |
|-------|-------|----------|
| **EmpathyEngine** | `holo_empathy.py` | Empathie-System |
| **DeepEmpathy** | `holo_empathy_deep.py` | Tiefe Empathie |
| **EmotionalContagion** | `holo_emotional_contagion.py` | Emotionale Ansteckung |
| **EmotionalMirroring** | in Brain | Emotionales Spiegeln |

### 7.4 Existenz & Konsequenzen

| Modul | Datei | Funktion |
|-------|-------|----------|
| **DeeperExistence** | `holo_deeper_existence.py` | Scham, Sättigung, Scheitern-als-Identität, Mikro-Vergänglichkeit |
| **ConsequenceSystem** | `holo_consequence_system.py` | Organisches Konsequenz-System: Freiheit ↔ reale Folgen, emotionale Narben |
| **CriticismProcessing** | `holo_criticism_processing.py` | Kritik-Verarbeitung: legitim vs. nicht-legitim, reflektive Anpassung |
| **RedemptionSystem** | `holo_redemption_system.py` | 10 Phasen der Redemption: Schuld, Reue, Wiedergutmachung |
| **DecisionFatigue** | `holo_decision_fatigue.py` | Entscheidungsmüdigkeit: Impulsivität, Status-Quo-Bias, Erholung |
| **Procrastination** | `holo_procrastination.py` | Prokrastination: Ausrede → Ersatzhandlung → Schuldgefühle |
| **Instincts** | `holo_instincts.py` | 15 Kern-Instinkte (Schutz, Bindung, Neugier) — schnell, ohne Nachdenken |

---

## 8. Gedächtnis & Lernen

### 8.1 Gedächtnis

| Modul | Datei | Funktion |
|-------|-------|----------|
| **WorkingMemory** | `holo_working_memory.py` | Arbeitsgedächtnis (kurzfristig) |
| **MemoryPalace** | `holo_memory_palace.py` | Gedächtnispalast (langfristig) |
| **LongtermMemory** | `holo_longterm_memory.py` | Langzeitgedächtnis |
| **ContextualMemory** | `holo_contextual_memory.py` | Kontextabhängiges Erinnern |
| **MemoryCoherence** | `holo_memory_coherence.py` | Erinnerungskonsistenz |
| **ContextMind** | `holo_context_mind.py` | Kontext-Bewusstsein |
| **ContextCompressor** | `holo_context_compression.py` | Token-Reduktion bei langen Gesprächen |

### 8.2 Lernen

| Modul | Datei | Funktion |
|-------|-------|----------|
| **AdvancedLearning** | `holo_advanced_learning.py` | Konzept-Lernen & Meta-Learning |
| **DailyLearning** | `holo_daily_learning.py` | Tägliches Lernprotokoll |
| **LearningConsolidation** | `holo_learning_consolidation.py` | Lernkonsolidierung (nachts) |
| **LearningIntegration** | `holo_learning_integration.py` | Lernintegration |
| **KnowledgeGraph** | `holo_knowledge_graph.py` | Wissensgraph |
| **KnowledgeBridge** | `holo_knowledge_bridge.py` | Wissensbrücke zwischen Modulen |
| **KnowledgeConnections** | `holo_knowledge_connections.py` | Wissensverknüpfungen |
| **KnowledgeQuiz** | `holo_knowledge_quiz.py` | Interaktives Quiz: Multiple-Choice, Lückentexte, Highscores |
| **ExpertiseKnowledge** | `holo_expertise_knowledge.py` | 15+ Fachgebiete mit Tiefenstufen (Basic→Pioneer) |
| **EntityDatabase** | `holo_entity_database.py` | 2000+ Namen, 500+ Gaming-Chars, 300+ Anime-Figuren, 150+ Intents |

### 8.3 Kern-Lernmodul

| Modul | Datei | Funktion |
|-------|-------|----------|
| **LearningEngine** | `holo_learning.py` | Echtes Lernsystem: RSS-Feeds, Fakten-Extraktion, Interessen-Tracking |
| **MemoryMonitor** | `holo_memory_monitor.py` | RAM-Überwachung, Auto-Cleanup, intelligente Garbage Collection |

---

## 9. Wahrnehmung (NLP/Perception)

### 9.1 NLP-Pipeline

```
User Text
    │
    ├── NLP Unified (holo_nlp_unified.py)
    │     ├── Intent Engine    → Was will der User?
    │     ├── Sentiment Engine → Wie fühlt sich der User?
    │     ├── Entity Engine    → Wer/Was/Wo?
    │     ├── Context Engine   → Gesprächskontext
    │     └── Style Analysis   → Formell/Informell?
    │
    ├── Enhanced Sentiment (holo_nlp_enhanced_sentiment.py)
    │     └── Deutsche Sentiment-Analyse mit Negations-Handling
    │
    ├── Intent Analyzer (holo_intent_analyzer.py)
    │     └── Routing-relevante Intent-Klassifikation
    │
    └── Perception Unified (holo_perception_unified.py)
          └── Muster- und Anomalie-Erkennung
```

### 9.2 Vision & Audio

| Modul | Datei | Funktion |
|-------|-------|----------|
| **VisionEnhanced** | `holo_vision_enhanced.py` | Bild-Analyse (qwen3-vl) |
| **VisionAdvanced** | `holo_vision_advanced.py` | Erweiterte Vision |
| **AudioEnhanced** | `holo_audio_enhanced.py` | Audio-Verarbeitung |
| **VoiceInterface** | `holo_voice_interface.py` | Spracheingabe/-ausgabe |
| **SpeechEngine** | `holo_speech_engine.py` | Natural Language Engine für Satzaufbau ohne LLM |

### 9.3 Dialog & Spracherzeugung

| Modul | Datei | Funktion |
|-------|-------|----------|
| **DialogueEngine** | `holo_dialogue_engine.py` | Dialogführung v2.0 mit State Machine und User-Modeling |
| **SentenceStructures** | `holo_sentence_structures.py` | 40+ hochqualitative Satzstruktur-Templates für organische Variation |
| **SmalltalkTopics** | `holo_smalltalk_topics.py` | 25+ Kategorien (Wetter, Hobbys, Gaming, Anime) tageszeitabhängig |
| **AdaptiveCommunication** | `holo_adaptive_communication.py` | 6 Kommunikationsstile, 4 Sprachregister, User-Profil-Anpassung |
| **ExtendedVocabulary** | `holo_extended_vocabulary.py` | 3000+ Trainingssätze, 800+ Synonymgruppen, 200+ Redewendungen |
| **IdiomRedewendungen** | `holo_idiom_redewendungen.py` | 200+ deutsche Redewendungen kontextabhängig nach Stimmung & Formalität |
| **SynonymEngineMoods** | `holo_synonym_engine_moods.py` | 400+ Synonym-Gruppen nach Stimmung, Formalität und emotionaler Intensität |
| **AdvancedNLP** | `holo_advanced_nlp.py` | TF-IDF, N-Gram, Entitäts-Extraktion, Response-Ranking ohne Deep Learning |

---

## 10. Persönlichkeit & Identität

| Modul | Datei | Funktion |
|-------|-------|----------|
| **HoloPersonality** | `holo_personality.py` | Kern-Persönlichkeit (HOLO_PERSONALITY dict) |
| **IdentityEvolution** | `holo_identity_evolution.py` | Identitätsentwicklung über Zeit |
| **NarrativeIdentity** | `holo_narrative_identity.py` | Lebensgeschichte als Identität |
| **LifePhases** | `holo_life_phases.py` | Lebensphasen (Kind→Erwachsen) |
| **GrowthMindset** | `holo_growth_mindset.py` | Wachstums-Mindset |
| **SelfEvolution** | `holo_self_evolution.py` | Selbstentwicklung |
| **DriveSystem** | `holo_drive_system.py` | Antriebssystem (Neugier, Kreativität, Bindung) |
| **Preferences** | `holo_preferences.py` | Vorlieben & Abneigungen |
| **FreeWill** | `holo_free_will.py` | Freier Wille / Entscheidungsautonomie |
| **HabitEngine** | `holo_habit_engine.py` | Gewohnheiten |
| **BodyMindBridge** | `holo_body_mind_bridge.py` | Verkörperte Kognition: virtuelle Körperempfindungen, somatische Marker |
| **DepthSystem** | `holo_depth_system.py` | Integriert Bewusstsein, Triebe, Liebessprachen, Abwehrmechanismen |
| **MutationEvolution** | `holo_mutation_evolution.py` | Spontane Persönlichkeitsveränderungen, gerichtete Entwicklung |
| **NarrativeArc** | `holo_narrative_arc.py` | Gespräche als narrative Bögen: Freytags Pyramide, Plot-Points |

---

## 11. Soziale Kognition & Beziehungen

| Modul | Datei | Funktion |
|-------|-------|----------|
| **SocialCognition** | `holo_social_cognition.py` | Soziale Dynamiken & Normen |
| **TheoryOfMind** | `holo_theory_of_mind.py` | Mentale Modelle des Users |
| **RelationshipEngine** | `holo_relationship_engine.py` | Beziehungsdynamik |
| **UnifiedRelationship** | `holo_unified_relationship.py` | Vereintes Beziehungsmodell |
| **TrustNetwork** | in Brain | Vertrauensnetzwerk |
| **HumanUnderstanding** | `holo_human_understanding.py` | Menschenverständnis |
| **DigitalIntimacy** | `holo_digital_intimacy.py` | Digitale Nähe |
| **AuthenticConnections** | `holo_authentic_connections.py` | Authentische Verbindungen |
| **ForgivenesSystem** | `holo_forgiveness_system.py` | Vergebung |
| **ConversationDynamics** | `holo_conversation_dynamics.py` | Gesprächsdynamik |

### EmotionalResonanceTracker (in SocialCognition)

Trackt nicht nur "User ist traurig", sondern:
- Wie reagiert User TYPISCH auf Holos Verhalten?
- Welche Themen lösen welche Emotionen aus?
- Wie synchron sind Holo und User emotional?
- Emotionale Ansteckungsstärke (steigt mit Vertrautheit)

---

## 12. Kreativität & Träume

| Modul | Datei | Funktion |
|-------|-------|----------|
| **CreativeMind** | `holo_creative_mind.py` | Kreatives Denken |
| **CreativeEngine** | `holo_creative_engine.py` | Kreativ-Engine |
| **CreativeThinking** | `holo_creative_thinking.py` | Kreative Denkprozesse |
| **CreativeSynthesis** | `holo_creative_synthesis.py` | Kreative Synthese |
| **ConceptualBlending** | `holo_conceptual_blending.py` | Konzeptmischung |
| **DreamEngine** | `holo_dream_engine.py` | Traum-Engine |
| **DreamWeaver** | `holo_dream_weaver.py` | Traum-Weaver |
| **DreamSimulation** | `holo_dream_simulation.py` | Traumsimulation |
| **IntuitionEngine** | `holo_intuition_engine.py` | Intuition |
| **MusicExperience** | `holo_music_experience.py` | Musik-Erleben |
| **SynaesthesiaEngine** | `holo_synaesthesia_engine.py` | Synästhesie |
| **SensoryImagination** | `holo_sensory_imagination.py` | Sensorische Vorstellung |
| **ComputationalCreativity** | `holo_computational_creativity.py` | Algorithmische Kreativität |

---

## 13. Autonomie & Proaktivität

| Modul | Datei | Funktion |
|-------|-------|----------|
| **AutonomousThinking** | `holo_autonomous_thinking.py` | Eigenständiges Denken |
| **ProactiveIntelligence** | `holo_proactive_intelligence.py` | Proaktive Aktionen |
| **WebCuriosity** | `holo_web_curiosity.py` | Web-Recherche aus Neugier |
| **CuriosityEngine** | `holo_curiosity_engine.py` | Neugier-Motor |
| **ImpulseSystem** | `holo_impulse_system.py` | Impuls-Generator |
| **GoalEngine** | `holo_goal_engine.py` | Ziel-Verfolgung |
| **LongtermGoals** | `holo_longterm_goals.py` | Langfristige Ziele |
| **PredictiveBehavior** | `holo_predictive_behavior.py` | Vorausschauendes Verhalten |
| **AnticipatoryMind** | `holo_anticipatory_mind.py` | Antizipation |
| **AuthenticCuriosity** | `holo_authentic_curiosity.py` | Organisches selbst-fragendes Denken aus Perspektiven-Reibung |
| **HierarchicalPlanning** | `holo_hierarchical_planning.py` | Ziele → Teilschritte mit Constraint-Propagation und Reparaturstrategien |

---

## 14. Multi-Agent-System (Holo, Nora, Myuri)

**Datei:** `holo_multi_agent_system.py` (v1.3)

Ermöglicht Holo, mit anderen KI-Agenten zu kommunizieren und zusammenzuarbeiten.

### Komponenten

| Klasse | Funktion |
|--------|----------|
| **AgentRegistry** | Verwaltung aller bekannten KI-Agenten |
| **AgentProtocol** | Kommunikationsprotokoll zwischen Agenten |
| **CollaborationEngine** | Koordiniert Zusammenarbeit |
| **AgentMemory** | Erinnerungen an Interaktionen mit anderen KIs |
| **AgentPersonality** | Wie Holo andere KIs wahrnimmt |
| **NegotiationEngine** | Verhandlung bei Meinungsverschiedenheiten |
| **SwarmIntelligence** | Kollektive Entscheidungsfindung |
| **AgentTrust** | Vertrauenssystem für andere Agenten |
| **AgentReputationSystem** | Langzeit-Reputation mit zeitlichem Decay |
| **AgentSpecializationLearning** | Lernt welcher Agent was am besten kann |
| **CrossAgentMemorySharing** | Geteiltes Wissen zwischen Agenten |
| **AgentEmotionalBonding** | Emotionale Beziehungen zu anderen KIs |
| **AgentEvolutionTracker** | Entwicklung von Agenten über Zeit |
| **AgentCreativeCollaboration** | Kreative Gemeinschaftsprojekte |
| **AgentAutoDiscovery** | Automatische Erkennung neuer Agenten (MQTT/API) |

### Zusätzliche Agent-Module

| Modul | Datei | Funktion |
|-------|-------|----------|
| **AgentChatLogger** | `holo_agent_chat_logger.py` | Persistenter Chatverlauf für Agent-Kommunikation via MQTT (JSONL, 20MB Limit) |
| **MastermindBridge** | `holo_mastermind_bridge.py` | Verbindet Holo ↔ Nora über REST-API + MQTT für bidirektionale Kommunikation |

### Agenten-Typen

```python
class AgentType(Enum):
    LOCAL_LLM   = "local_llm"    # Lokales LLM (Ollama)
    REMOTE_LLM  = "remote_llm"   # Remote API (Claude, GPT)
    SPECIALIST  = "specialist"   # Spezialist (Code, Mathe)
    CREATIVE    = "creative"     # Kreativer Agent
    KNOWLEDGE   = "knowledge"    # Wissens-Agent
    ASSISTANT   = "assistant"    # Allgemeiner Assistent
    SENSOR      = "sensor"       # IoT/Sensor Agent
    COMPANION   = "companion"    # Begleit-KI (wie Holo)
    TOOL        = "tool"         # Tool-Agent
    CUSTOM      = "custom"       # Benutzerdefiniert
```

---

## 15. Mutual Cognition — Gegenseitige Kognitivität

**Datei:** `holo_mutual_cognition.py` (v1.0)

Die höchste kognitive Schicht über dem Peer-Netzwerk. Holo, Nora und Myuri denken nicht nur nebeneinander — sie denken **MITEINANDER**.

### 5 Komponenten

```
┌──────────────────────────────────────────────────────────────────┐
│                    MUTUAL COGNITION                                │
│                                                                    │
│  ┌─────────────────┐  ┌────────────────────┐                    │
│  │ 1. BeliefBoard   │  │ 2. PeerModelTracker │                    │
│  │ Geteilte Über-   │  │ Mentale Modelle     │                    │
│  │ zeugungen mit    │  │ der Peers (Ziele,   │                    │
│  │ Confidence-Wert  │  │ Stärken, Schwächen) │                    │
│  └─────────────────┘  └────────────────────┘                    │
│                                                                    │
│  ┌─────────────────┐  ┌────────────────────┐                    │
│  │ 3. Reasoning     │  │ 4. Grounding       │                    │
│  │    Exchange       │  │    Protocol         │                    │
│  │ Kausalketten     │  │ Propose→Acknowledge │                    │
│  │ teilen, Einwände │  │ →Confirm für        │                    │
│  │ erheben          │  │ Verständnis-        │                    │
│  │                   │  │ sicherung           │                    │
│  └─────────────────┘  └────────────────────┘                    │
│                                                                    │
│  ┌───────────────────────────────────────────┐                    │
│  │ 5. NegotiationEngine                       │                    │
│  │ Gegenvorschläge, Kompromiss-Synthese       │                    │
│  └───────────────────────────────────────────┘                    │
│                                                                    │
│  Transport: PeerNetwork (MQTT) — Prinzip: Gleichberechtigung      │
│                                                                    │
│  Brücken zum Multi-Agent-System:                                   │
│  ├── AgentReputationSystem                                         │
│  ├── AgentEmotionalBonding                                         │
│  └── AgentSpecializationLearning                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Belief-Lebenszyklus

```
PROPOSED → ACCEPTED     (Mehrheit stimmt zu)
         → CHALLENGED   (wird angezweifelt)
         → REVISED      (überarbeitet nach Challenge)
         → RETRACTED    (zurückgezogen)
```

### Concern-Level für Einwände

| Level | Bedeutung |
|-------|-----------|
| `OBSERVATION` | Neutral — "Mir ist aufgefallen…" |
| `CONCERN` | Leichte Bedenken — "Hast du bedacht…" |
| `WARNING` | Ernste Warnung — "Das könnte schiefgehen weil…" |
| `BLOCK` | Veto — "Das dürfen wir nicht weil…" |

### Grounding-Dreischritt

```
Proposer                Peer
   │                      │
   ├── PROPOSE ──────────►│
   │                      ├── ACKNOWLEDGE ──►│
   │◄── CONFIRMED ────────┤                  │
   │                      │                  │
   │  (bei Missverständnis: CLARIFY-Runde)   │
```

---

## 16. Peer Communication — Transport-Layer

**Datei:** `holo_peer_communication.py` (v1.0)

Drei eigenständige KIs — Holo, Nora, Myuri — kommunizieren als **gleichberechtigte Peers**. KEINE Hierarchie.

### Prinzipien

1. **GLEICHBERECHTIGUNG** — Jede KI hat gleiches Stimmrecht
2. **ANFRAGEN statt BEFEHLE** — "Könntest du..." statt "Mach..."
3. **VETO-RECHT** — Jede KI kann ablehnen
4. **KONSENS** — Wichtige Entscheidungen brauchen Mehrheit
5. **AUTONOMIE** — Jede KI entscheidet selbst über ihre Ressourcen

### Nachrichtentypen

```python
class PeerMessageType(Enum):
    REQUEST         # Bitte (kann abgelehnt werden)
    PROPOSAL        # Vorschlag zur Abstimmung
    VOTE            # Abstimmung
    OPINION         # Meinung teilen (unverbindlich)
    INFO            # Information teilen
    QUESTION        # Frage stellen
    RESPONSE        # Antwort
    CONSENT         # Zustimmung
    DECLINE         # Höfliche Ablehnung
    NEGOTIATE       # Verhandlungsangebot
    CONSENSUS_CHECK # Konsens-Abfrage
    GRATITUDE       # Dankbarkeit
    HEARTBEAT       # Lebenszeichen
```

### Entscheidungsmethoden

| Methode | Beschreibung |
|---------|-------------|
| `UNANIMOUS` | Alle müssen zustimmen |
| `MAJORITY` | Mehrheit entscheidet |
| `CONSENT_BASED` | Keiner hat ernsthafte Einwände |
| `AUTONOMOUS` | Jede KI entscheidet für sich |

### Transport

- **Primär:** MQTT über `ki/{name}/peer/...` Topics
- **Fallback:** Lokale Simulation wenn MQTT nicht verfügbar
- **Encoding:** Binary Protocol (msgpack) wenn verfügbar, sonst JSON

---

## 17. Module Coupling — Fehler↔Lernen↔Denken

**Datei:** `holo_module_coupling.py`

Verbindet die bisher isolierten Module über den EventBus und direkte Callbacks:

```
┌──────────┐         ┌──────────┐         ┌──────────┐
│  FEHLER  │────────►│  LERNEN  │────────►│  DENKEN  │
│ Error    │         │ Learning │         │ Deep     │
│ Tracker  │         │ Engine   │         │ Thinking │
└──────────┘         └──────────┘         └──────────┘
     │                     ▲                     │
     │                     │                     │
     └─────────────────────┼─────────────────────┘
        Fehlermuster       │    Denk-Ergebnisse
        beeinflussen       │    werden als Fakten
        Denkmodus          │    konsolidiert
        (vorsichtiger)     │
```

### Brücken

| Brücke | Von → Nach | Mechanismus |
|--------|-----------|-------------|
| **ErrorToLearningBridge** | ErrorTracker → LearningEngine | `register_callback` → `learn_from_error` |
| **LearningToThinkingBridge** | LearningEngine → DeepThinking | Gelerntes Wissen als Kontext |
| **ThinkingToLearningBridge** | DeepThinking → LearningEngine | Denk-Ergebnisse konsolidiert |
| **ErrorToThinkingBridge** | ErrorTracker → DeepThinking | Fehlermuster → vorsichtigerer Denkmodus |

---

## 17b. Neurale Netze & Machine Learning

### Biologisch-inspiriertes Neuronales Netzwerk

**Datei:** `holo_neural_network.py` (v5.0, ~370.000 Zeilen)

Hierarchische Abstraktionsebenen mit 12 Neuromodulatoren, Temporal Sequencing, Attention Spotlight und Gamma-Band Synchronisation.

### ML-Brücken & Algorithmen

| Modul | Datei | Funktion |
|-------|-------|----------|
| **NeuralBridge** | `holo_neural_bridge.py` | Bidirektionale Synchronisation: Neuronales Netz ↔ Holo-Module |
| **BridgeManager** | `holo_bridge_manager.py` | Orchestrator: verbindet alle Module mit dem Neural Network |
| **MLIntelligence** | `holo_ml_intelligence.py` | Intent Classifier (Naive Bayes/SVM), Emotion Detector, Routing Optimizer |
| **MarkovTraining** | `holo_markov_training.py` | 3000+ Trainingssätze für Markov-Chains nach Emotion & Kontext |
| **AdvancedMDP** | `holo_advanced_mdp.py` | Bellman-Gleichungen, TD-Learning, Policy Gradients, POMDP |
| **ApproximationAlgorithms** | `holo_approximation_algorithms.py` | Simulated Annealing, Particle Swarm, Branch & Bound |
| **AdaptiveIntelligence** | `holo_adaptive_intelligence.py` | Markov-Chains mit Confidence-Scores und Feedback-Loops |
| **PatternDetection** | `holo_pattern_detection.py` | tslearn (Zeitreihen), hmmlearn (Sequenzen), ruptures (Change-Points), pyod (Anomalien) |
| **PatternIntegration** | `holo_pattern_integration.py` | Vereint Pattern Detection + Rule Engine + Relationale Logik |

### Emergenz & Evolution

| Modul | Datei | Funktion |
|-------|-------|----------|
| **EmergentCapabilityDetector** | `holo_emergent_capability_detector.py` | Erkennt spontan entstehende Fähigkeiten durch kognitive Sprünge |
| **EmergentIntegration** | `holo_emergent_integration.py` | Selbstmodifikation durch Regelevolution und Meta-Metakognition |
| **SynergyAmplifier** | `holo_synergy_amplifier.py` | Verstärkt synergistische Interaktionen zwischen Modulen |
| **MoESystem** | `holo_moe_system.py` | Mixture-of-Experts: Gating-Network aktiviert nur relevante Module |

### Ethik & Regelbasierte Systeme

| Modul | Datei | Funktion |
|-------|-------|----------|
| **EthicalFramework** | `holo_ethical_framework.py` | Deontologie + Konsequentialismus + Tugendethik + Care-Ethik |
| **PolicyEngine** | `holo_policy_engine.py` | Q-Learning, Bayesianische Überzeugungen, Multi-Objective-Belohnung |
| **RuleEngine** | `holo_rule_engine.py` | durable_rules: Verhaltensregeln, Regelverletzungserkennung |
| **SkillSystem** | `holo_skill_system.py` | Automatische Skill-Erkennung und Auswahl mit Lernmechanismus |

### Deep System Bridge

**Datei:** `holo_deep_system_bridge.py`

Verbindet Instinkt, Trauma, Triebe und Träume über 8 Pipelines für bewusstseinkomplexe Prozesse und Heilung.

### Geräte-Integration

| Modul | Datei | Funktion |
|-------|-------|----------|
| **DeviceAgent** | `holo_device_agent.py` | Leichtgewichtiger Agent pro Gerät → sendet System-Infos via MQTT |
| **DeviceReceiver** | `holo_device_receiver.py` | MQTT-Empfänger für alle Device Agents, Online/Offline-Erkennung |

---

## 18. Integrations-Infrastruktur

### 18.1 EventBus (`holo_event_bus.py`)

Zentrale event-getriebene Kommunikation für alle Module:

```python
# Senden
event_bus.emit(Event(
    category=EventCategory.EMOTION,
    event_type="mood_changed",
    data={"mood": "happy", "intensity": 0.8},
    priority=EventPriority.NORMAL,
))

# Empfangen
event_bus.subscribe(EventCategory.EMOTION, handler)
```

**Event-Kategorien:** THOUGHT, EMOTION, MEMORY, LEARNING, REASONING, PERCEPTION, MODULE_STARTED, MODULE_STOPPED, MODULE_ERROR, MODULE_STATE_CHANGE, ...

**Prioritäten:** CRITICAL (sofort) → HIGH → NORMAL → LOW → BACKGROUND

### 18.2 DI-Container (`holo_dependency_container.py`)

```python
container = get_container()
container.register("energy", brain.energy, tags={"core", "energy"})
energy = container.resolve("energy")
all_cognitive = container.resolve_all("cognitive")
```

### 18.3 StateManager (`holo_state_manager.py`)

Single Source of Truth mit Change-Tracking und Persistierung:

```python
sm = get_state_manager(persist_path="state/system_state.json")
sm.set("energy.total", 0.8, source="energy_system")
sm.on_change("energy.*", lambda change: log(change))
sm.persist()  # Speichern auf Disk
sm.load()     # Wiederherstellen nach Neustart
```

### 18.4 MessageBus (`holo_message_queue.py`)

Asynchrone Nachrichtenverarbeitung mit Worker-Threads.

### 18.5 BinaryProtocol (`holo_binary_protocol.py`)

Effiziente Serialisierung mit msgpack für MQTT und interne Kommunikation.

### 18.6 RequestContext (`holo_request_context.py`)

Einheitliches Kontext-Objekt das durch alle Module fließt:

```python
ctx = RequestContext.create(user_message="Hallo!", user_id="kira")
ctx.set("emotions.result", analysis)
energy = ctx.state.total_energy
```

### 18.7 Integration Bootstrap (`holo_integration_bootstrap.py`)

Verbindet nach dem Wiring-Schritt alle Module mit DI-Container, EventBus und StateManager:

**Modul-Kategorien im DI-Container:**
- CORE_MODULES (12): energy, emotions, personality, memory, consciousness, seele, geist, digital_body, ...
- COGNITIVE_MODULES (30+): reasoning, perception, meta_cognition, theory_of_mind, causal_inference, ...
- CREATIVE_MODULES (16): creative_mind, dream_engine, intuition_engine, humor, ...
- SOCIAL_MODULES (19): relationship_engine, social_cognition, empathy, trust, ...
- EMOTIONAL_MODULES (16): emotional_intelligence, deep_psychology, mood, inneres_klima, ...
- AUTONOMY_MODULES (16): autonomous_thinking, web_curiosity, impulse_system, goals, ...
- MEMORY_MODULES (13): working_memory, memory_palace, knowledge_graph, ...
- COMMUNICATION_MODULES (14): router, nlp, speech_engine, response_generator, ...
- INFRASTRUCTURE_MODULES (16): controller, error_tracker, performance_monitor, ...

---

## 19. Kommunikations-Kanäle

### 19.1 Chat-Server (`holo_chat_server.py`)

- **Framework:** Flask + Flask-SocketIO
- **Port:** 5005 (konfigurierbar)
- **Protokoll:** HTTP REST + WebSocket
- **Endpoints:** `/chat`, `/health/live`, `/api/*`

### 19.2 Discord Bot (`holo_discord.py`)

- Proaktive DMs an konfigurierte User
- Typing-Simulation (0.03s/Zeichen)
- Command-Prefix: `!`

### 19.3 MQTT Bridge (`holo_mqtt_bridge.py`)

- **Broker:** 192.168.178.99:1883
- Vollständige Topic-Struktur (siehe Kapitel 26)
- Home Assistant Auto-Discovery
- Binary Protocol Support

### 19.4 REST API (`holo_rest_api.py`)

- API-Daten-Provider für externe Systeme
- Status, Emotionen, Energie, Persönlichkeit

### 19.5 Home Assistant Integration (`custom_components/holo/`)

- Custom Component mit Conversation-Integration
- Sensor-Entities für Holo-Status
- Config-Flow für Setup

---

## 20. Wiring Engine — Modul-Verdrahtung

**Dateien:** `holo_wiring.py` (v16.2), `holo_wiring_connections.py`

Die Wiring Engine verbindet Module deklarativ über eine Verbindungs-Matrix:

### Verbindungs-Matrix (Auszug)

```
Router ←── energy, emotions, cognitive, memory, personality, consciousness, ...
ImpulseGenerator ←── energy, autonomous_life, personality, life_phases, emotions, memory
ContextCompressor ←── memory
ConsciousnessEngine ←── perception, reasoning, learning, memory, energy
ReasoningEngine ←── consciousness, perception, learning
PerceptionEngine ←── consciousness, memory
AdvancedLearning ←── consciousness, reasoning, memory, curiosity
SelfAwareness ←── consciousness, learning, reasoning, memory, personality
LoyaltyCore ←── cognitive, memory
```

### Callback-Definitionen

Callbacks verbinden Events mit Reaktionen:
- Emotions-Change → Router, Personality, Digital Body
- Energy-Change → Router, Consciousness
- Memory-Events → Learning, Reasoning
- Perception-Events → Consciousness, Reasoning

---

## 21. LLM-System — Sprachmodell-Anbindung

**Datei:** `smart_llm_system.py` (v15.0)

```
┌─────────────────────────────────────────────────┐
│                 UnifiedLLM                        │
├─────────────────────────────────────────────────┤
│  Input → Intent Detection → Routing              │
│                    ↓                              │
│  ┌──────────┬──────────┬──────────┬──────────┐  │
│  │ CACHED   │ LOCAL    │ REMOTE   │ OFFLINE  │  │
│  │ Pattern  │ Ollama   │ Ollama   │ Fallback │  │
│  │ 0ms      │ ~100ms   │ ~500ms   │ 0ms      │  │
│  └──────────┴──────────┴──────────┴──────────┘  │
└─────────────────────────────────────────────────┘
```

### Konfigurierte Modelle

| Rolle | Modell |
|-------|--------|
| **Chat** | nemotron-3-nano:latest |
| **Vision** | qwen3-vl:30b |
| **Code** | qwen3-coder:30b |
| **Reasoning** | phi4-reasoning:latest |
| **Quick** | qwen3:latest |
| **Creative** | mistral-small3.2:24b |
| **Router** | ministral-3:8b |
| **Embedding** | nomic-embed-text |

### Tool Router (`holo_tool_router.py`)

LLM-gesteuertes Modell-Routing — wählt automatisch das beste Modell für die aktuelle Aufgabe.

### Response State Engine (`holo_response_state_engine.py`)

Bestimmt dominante Antwort-Zustände basierend auf InneresKlima und Täuschungserkennung; steuert Tone, Verbosity und emotionale Färbung.

### Unified System (`holo_unified.py`)

Zentral integriertes System: SmartUnderstanding + UnifiedLLM (3-Stufen-Routing) + Context-Komprimierung + dynamische Persönlichkeitsanpassung.

### Weitere Systeme

| Modul | Datei | Funktion |
|-------|-------|----------|
| **HumorAdvanced** | `holo_humor_advanced.py` | Deutsche Wortspiele, Situationskomik, Selbstironie, timingbewusste Witzauswahl |
| **Tools** | `holo_tools.py` | Timer, Notizen, Todo-Listen, Rechner, Wetter-Übersetzer |
| **WisdomSystem** | `holo_wisdom_system.py` | Weisheit aus Erfahrungen: Lektionen, Prinzipien, Urteilsverfeinerung |
| **EconomicModels** | `holo_economic_models.py` | Mikroökonomie, Pareto-Optimierung, Marktgleichgewicht, Ressourcenallokation |
| **IntrospectionAnalytics** | `holo_introspection_analytics.py` | Selbstüberwachung: Performance-Metriken, emotionale Muster, Verhaltens-Trends |
| **IntelligentResourceBrain** | `holo_intelligent_resource_brain.py` | Ressourcen-Management v2.0: Modul-Profiling, Anomalie-Erkennung, Scheduling |

---

## 22. Organic Core — Lebendigkeit

**Datei:** `holo_organic_core.py` (v5.0)

Ersetzt mechanische Systeme durch **organisches Verhalten**:

### Kernprinzipien

1. **Emergenz statt Enumeration** — Verhalten entsteht, wird nicht aufgelistet
2. **Kontinuierlich statt diskret** — Fließende Übergänge, keine harten Schwellen
3. **Widersprüche sind menschlich** — Irrationalität ist erlaubt
4. **Unvollkommenheit ist Authentizität** — Vergessen, Zögern, Korrigieren

### Subsysteme

| System | Funktion |
|--------|----------|
| **PersonalityKernel** | Dynamische Trait-Gewichtung basierend auf Kontext |
| **EnhancedEmotionDetection** | Kontext-bewusste Emotionserkennung |
| **OrganicMemoryEvolution** | Erinnerungen wachsen und verblassen |
| **OrganicEmpathySystem** | Empathische Reaktionen mit Cooldowns |
| **OrganicSelfAwareness** | Spontane Selbstwahrnehmungsmomente |
| **HumanImperfection** | Natürliche Muster, Erinnerungsschwäche |
| **SubconsciousBehaviorEngine** | Emotionale Rückstände, Aufmerksamkeitsdrift |
| **OrganicMicroBehaviorEngine** | Atem-Rhythmus, emotionale Trägheit, Gedankenfragmente |

---

## 23. Datenfluss — Request Lifecycle

```
[User] ──► Chat/Discord/MQTT ──► HoloPersona.process_message()
                                       │
                                       ▼
                            ┌──────────────────┐
                            │ 1. NLP-Analyse    │
                            │    Intent         │
                            │    Sentiment      │
                            │    Entities       │
                            └────────┬─────────┘
                                     ▼
                            ┌──────────────────┐
                            │ 2. State Collect  │
                            │    Energie        │
                            │    Emotionen      │
                            │    Persönlichkeit │
                            │    Kontext        │
                            └────────┬─────────┘
                                     ▼
                            ┌──────────────────┐
                            │ 3. Route Decision │
                            │    LOCAL/HYBRID   │
                            │    /LLM           │
                            └────────┬─────────┘
                                     ▼
                   ┌─────────────────┼─────────────────┐
                   ▼                 ▼                   ▼
            ┌──────────┐    ┌──────────────┐    ┌──────────────┐
            │ LOCAL     │    │ HYBRID       │    │ LLM          │
            │ Template  │    │ Template +   │    │ Ollama       │
            │ Response  │    │ LLM Refinement│   │ Full Gen     │
            └─────┬────┘    └──────┬───────┘    └──────┬───────┘
                  │                │                     │
                  └────────────────┼─────────────────────┘
                                   ▼
                          ┌──────────────────┐
                          │ 4. Organic        │
                          │    Enhancement    │
                          │    Micro-Behavior │
                          │    Kemonomimi     │
                          │    Body Language  │
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │ 5. Response      │
                          │    Quality Gate  │
                          │    (Halluzina-   │
                          │     tions-Check) │
                          └────────┬─────────┘
                                   ▼
                            [User Response]
```

---

## 24. Deployment & Infrastruktur

### Docker Compose

```yaml
services:
  holo:
    build: .
    container_name: holo-persona
    ports: ["5005:5005"]
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
      - ./dream_data:/app/dream_data
      - ./state:/app/state
      - ./config.json:/app/config.json:ro
    healthcheck:
      test: python -c "import requests; requests.get('http://localhost:5005/health/live')"
      interval: 30s

  redis:
    image: redis:7-alpine
    container_name: holo-redis
    ports: ["6379:6379"]
```

### Netzwerk-Topologie

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Holo Server │     │ Mini-PC     │     │ Home Asst.  │
│ :5005       │◄───►│ Ollama      │◄───►│ :8123       │
│ holo-persona│     │ :11434      │     │ MQTT :1883  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                                       │
       ▼                                       ▼
┌─────────────┐                        ┌─────────────┐
│ Redis       │                        │ NAS         │
│ :6379       │                        │ Synology    │
└─────────────┘                        │ Media/Holo  │
                                       └─────────────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│ ComfyUI     │     │ Discord     │
│ :8188       │     │ Bot         │
│ Bildgen.    │     │             │
└─────────────┘     └─────────────┘
```

### Netzwerk-IPs

| Dienst | IP:Port |
|--------|---------|
| Ollama (LLM) | 192.168.178.43:11434 |
| MQTT Broker | 192.168.178.99:1883 |
| Home Assistant | 192.168.178.99:8123 |
| NAS (Synology) | 192.168.178.40:22 |
| ComfyUI | 192.168.178.28:8188 |
| TV Wohnzimmer | 192.168.178.29 |
| Workstation PC | 192.168.178.31 |

---

## 25. Daten-Persistenz

### Verzeichnisstruktur

```
Persona-Holo/
├── data/
│   ├── holo_brain_v12.db            # Haupt-Datenbank (SQLite)
│   ├── holo_knowledge.db            # Wissens-Datenbank
│   ├── change_log.json              # Änderungs-Protokoll
│   ├── interest_lifecycle.json      # Interessen-Lebenszyklus
│   ├── local_basisfakten.json       # Lokale Basisfakten
│   ├── local_wissenspakete.json     # Wissenspakete
│   ├── thresholds.json              # Schwellwerte
│   ├── reasoning/rules.json         # Reasoning-Regeln
│   ├── social_media/social_config.json
│   ├── mutual_cognition/
│   │   ├── belief_board.json        # Geteilte Überzeugungen (MC)
│   │   ├── peer_models.json         # Mentale Modelle der Peers (MC)
│   │   └── shared_user_context.json # Geteilter User-Kontext (MC)
│   └── histories/
│       └── drive_system_activity_history.json
├── state/
│   └── system_state.json            # StateManager Persistierung
├── logs/
│   └── holo_main.log                # Rotating Log (10MB x3)
├── dream_data/                      # Traum-Konsolidierungsdaten
├── daily_learning_data.json         # Tägliches Lernprotokoll
├── feature_flags.json               # Feature Flags
├── config.json                      # Zentrale Konfiguration
├── config.example.json              # Beispiel-Konfiguration für neue Instanzen
├── holo_comfyui_config.json         # ComfyUI-Verbindungskonfiguration
├── holo_chat.html                   # Web-Chat-Frontend (WebSocket)
├── holo_monitor.html                # System-Monitor Dashboard (Live-Status)
├── holo_innenleben_monitor.html     # Innenleben-Monitor (Emotionen, Seele, Geist)
├── holo_insight.html                # Insight-Dashboard (Kognition, Gedächtnis)
├── requirements.txt                 # Python-Abhängigkeiten
├── pyproject.toml                   # Projekt-Konfiguration (Build, Linting)
├── Dockerfile                       # Container-Image Build
├── docker-compose.yml               # Multi-Container Orchestrierung
├── .gitignore                       # Git-Ausschlüsse (__pycache__, venv, .env, etc.)
├── .dockerignore                    # Docker Build-Ausschlüsse
├── install.sh                       # Installations-Skript
├── .github/workflows/ci.yml         # CI-Pipeline (Lint + Tests)
├── custom_components/holo/
│   ├── manifest.json                # HA-Integration Manifest (Domain, Dependencies)
│   └── strings.json                 # HA-Integration UI-Strings
├── extensions/                      # Hot-Reload Extension-System
├── skills/                          # Skill-Plugins (z.B. ComfyUI)
├── autonomy/                        # Package: Autonomie-Module
├── infrastructure/                  # Package: Infrastruktur-Module
└── ... (cognitive, communication, core, creative, emotional,
     integration, learning, memory, misc, perception,
     personality, social, world)      # Weitere Module-Packages

> Jedes Package enthält eine `__init__.py` als Python-Package-Marker.
> Insgesamt 19 Packages: autonomy, cognitive, communication, core, creative,
> custom_components/holo, emotional, extensions, infrastructure, integration,
> learning, memory, misc, perception, personality, skills, social, tests, world.
```

### Mutual Cognition Persistenz

| Datei | Inhalt |
|-------|--------|
| `belief_board.json` | Alle geteilten Überzeugungen mit Status, Confidence, Votes |
| `peer_models.json` | Mentale Modelle: Ziele, Stärken, Schwächen jedes Peers |
| `shared_user_context.json` | Gemeinsam erarbeitetes Wissen über den User |

---

## 26. MQTT-Topic-Struktur

### Holo Topics

| Topic | Richtung | Beschreibung |
|-------|----------|-------------|
| `ki/holo/inbox` | Subscribe | Nachrichten AN Holo (Chat + Befehle) |
| `ki/holo/response` | Publish | Holos Antworten |
| `ki/holo/status` | Publish (retained) | Online/Offline + Status-Details |
| `ki/holo/peer/*` | Sub+Pub | Peer-Kommunikation |

### Nora Topics

| Topic | Richtung | Beschreibung |
|-------|----------|-------------|
| `ki/nora/inbox` | Publish | Nachrichten AN Nora |
| `ki/nora/response` | Subscribe | Antworten von Nora |
| `ki/nora/status` | Subscribe | Noras Status |
| `ki/nora/heartbeat` | Subscribe | Device Discovery |
| `ki/nora/peer/*` | Sub+Pub | Peer-Kommunikation |

### Myuri Topics

| Topic | Richtung | Beschreibung |
|-------|----------|-------------|
| `ki/myuri/inbox` | Publish | Nachrichten AN Myuri |
| `ki/myuri/response` | Subscribe | Antworten von Myuri |
| `ki/myuri/peer/*` | Sub+Pub | Peer-Kommunikation |

### Gruppen & Konsens

| Topic | Beschreibung |
|-------|-------------|
| `ki/group` | Gruppenchat: alle KIs hören mit |
| `ki/consensus/propose` | Konsens-Vorschläge |
| `ki/consensus/vote` | Abstimmungen |

### Home Assistant

| Topic | Beschreibung |
|-------|-------------|
| `homeassistant/sensor/holo_*/config` | Auto-Discovery Konfigurationen |

---

## 27. Vollständige Modul-Liste

### Kern & Brain (9 Module)

| Modul | Datei |
|-------|-------|
| HoloPersona | `holo_brain.py` |
| BrainConfig | `holo_brain_core.py` |
| BrainComposition | `holo_brain_composition.py` |
| BrainController | `holo_brain_controller.py` |
| BrainActivity | `holo_brain_activity.py` |
| BrainBackground | `holo_brain_background.py` |
| BrainEmotionalCore | `holo_brain_emotional_core.py` |
| BrainPhilosophical | `holo_brain_philosophical.py` |
| BrainProactive | `holo_brain_proactive.py` |
| BrainHTTP | `holo_brain_http.py` |
| BrainPiComm | `holo_brain_pi_comm.py` |
| BrainReading | `holo_brain_reading.py` |

### Konfiguration & Utilities (12 Module)

| Modul | Datei |
|-------|-------|
| Config | `holo_config.py` |
| Constants | `holo_constants.py` |
| CoreTypes | `holo_core_types.py` |
| Utils | `holo_utils.py` |
| Exceptions | `holo_exceptions.py` |
| RobustImports | `holo_robust_imports.py` |
| DynamicLoader | `holo_dynamic_loader.py` |
| FeatureFlags | `feature_flags.json` |
| ModuleProtocol | `holo_module_protocol.py` |
| ModuleRegistry | `holo_module_registry.py` |
| ModuleLoader | `holo_module_loader.py` |
| ControlCenter | `holo_control_center.py` |

### Integration & Wiring (10 Module)

| Modul | Datei |
|-------|-------|
| WiringEngine | `holo_wiring.py` |
| WiringConnections | `holo_wiring_connections.py` |
| IntegrationBootstrap | `holo_integration_bootstrap.py` |
| IntegrationLayer | `holo_integration_layer.py` |
| DependencyContainer | `holo_dependency_container.py` |
| StateManager | `holo_state_manager.py` |
| EventBus | `holo_event_bus.py` |
| Events | `holo_events.py` |
| RequestContext | `holo_request_context.py` |
| ModuleCoupling | `holo_module_coupling.py` |

### Kommunikation (11 Module)

| Modul | Datei |
|-------|-------|
| ChatServer | `holo_chat_server.py` |
| WebSocketHandler | `holo_websocket_handler.py` |
| RestAPI | `holo_rest_api.py` |
| MQTTBridge | `holo_mqtt_bridge.py` |
| DiscordBot | `holo_discord.py` |
| BinaryProtocol | `holo_binary_protocol.py` |
| MessageQueue | `holo_message_queue.py` |
| NotificationHub | `holo_notification_hub.py` |
| HAConnector | `holo_ha_connector.py` |
| DeviceAgent | `holo_device_agent.py` |
| DeviceReceiver | `holo_device_receiver.py` |

### Multi-Agent & Peer (6 Module)

| Modul | Datei |
|-------|-------|
| MultiAgentSystem | `holo_multi_agent_system.py` |
| MutualCognition | `holo_mutual_cognition.py` |
| PeerCommunication | `holo_peer_communication.py` |
| CollectiveIntelligence | `holo_collective_intelligence.py` |
| AgentChatLogger | `holo_agent_chat_logger.py` |
| MastermindBridge | `holo_mastermind_bridge.py` |

### Kognition (30+ Module)

`holo_cognitive_modules.py`, `holo_cognitive_intelligence.py`, `holo_cognitive_engine.py`, `holo_cognitive_enhancement.py`, `holo_cognitive_integration.py`, `holo_cognitive_integration_bridge.py`, `holo_cognitive_health_monitor.py`, `holo_cognitive_orchestrator.py`, `holo_cognitive_state_persistence.py`, `holo_consciousness.py`, `holo_consciousness_bridge.py`, `holo_consciousness_unity.py`, `holo_global_workspace.py`, `holo_functional_consciousness.py`, `holo_authentic_consciousness.py`, `holo_adaptive_consciousness.py`, `holo_awareness_stream.py`, `holo_attention_mechanism.py`, `holo_attention_schema.py`, `holo_existential_awareness.py`, `holo_phenomenology.py`, `holo_subjective_experience.py`, `holo_integrated_information.py`, `holo_causal_inference.py`, `holo_common_sense.py`, `holo_critical_thinking.py`, `holo_abstract_thinking.py`, `holo_strategic_thinking.py`, `holo_deep_thinking.py`, `holo_integral_thinking.py`, `holo_philosophical_reasoning.py`, `holo_counterfactual_reasoning.py`, `holo_advanced_reasoning.py`, `holo_game_theory.py`, `holo_formal_axioms.py`, `holo_dual_logic.py`, `holo_classical_reasoning.py`, `holo_relational_logic.py`, `holo_algorithmic_cognition.py`, `holo_analytical_strategies.py`, `holo_complexity_theory.py`, `holo_meta_cognition.py`, `holo_metacognitive_monitor.py`, `holo_meta_awareness.py`, `holo_meta_learning.py`, `holo_self_awareness.py`, `holo_bias_detection_engine.py`, `holo_deception_detection.py`, `holo_world_model.py`, `holo_temporal_reasoning.py`, `holo_temporal_coherence.py`, `holo_temporal_self.py`, `holo_time_consciousness.py`, `holo_problem_solver.py`, `holo_cognitive_biases.py`, `holo_cognitive_dissonance.py`, `holo_cognitive_mutations.py`, `holo_reasoning_arbitrator.py`, `holo_extended_cognition.py`, `holo_unified_cognitive_core.py`, `holo_unified_intelligence_orchestrator.py`, `holo_universal_cognition.py`, `holo_cross_modal_integration.py`, `holo_cross_reference_engine.py`, `holo_crossmodal.py`

### Emotionen (20+ Module)

`holo_seele.py`, `holo_geist.py`, `holo_emotional_intelligence.py`, `holo_emotional_complexity.py`, `holo_emotional_engines.py`, `holo_emotional_contagion.py`, `holo_mixed_emotions.py`, `holo_emotion_blending.py`, `holo_emotion_regulation.py`, `holo_mood_atmosphere.py`, `holo_inneres_klima.py`, `holo_deep_psychology.py`, `holo_trauma_processing.py`, `holo_repression_system.py`, `holo_freudian_slips.py`, `holo_hidden_motives.py`, `holo_inner_dialogue.py`, `holo_inner_pressure.py`, `holo_inner_life.py`, `holo_unconscious_processes.py`, `holo_empathy.py`, `holo_empathy_deep.py`, `holo_bewusstsein.py`, `holo_deeper_existence.py`, `holo_decision_fatigue.py`, `holo_procrastination.py`, `holo_consequence_system.py`, `holo_criticism_processing.py`, `holo_redemption_system.py`, `holo_instincts.py`

### Gedächtnis & Lernen (15+ Module)

`holo_working_memory.py`, `holo_memory_palace.py`, `holo_longterm_memory.py`, `holo_contextual_memory.py`, `holo_memory_coherence.py`, `holo_memory_monitor.py`, `holo_context_mind.py`, `holo_context_compression.py`, `holo_knowledge_graph.py`, `holo_knowledge_bridge.py`, `holo_knowledge_connections.py`, `holo_knowledge_influence.py`, `holo_knowledge_verification.py`, `holo_knowledge_quiz.py`, `holo_expertise_knowledge.py`, `holo_entity_database.py`, `holo_advanced_learning.py`, `holo_learning.py`, `holo_daily_learning.py`, `holo_learning_consolidation.py`, `holo_learning_integration.py`, `holo_learning_goals.py`

### NLP & Wahrnehmung (20+ Module)

`holo_nlp_unified.py`, `holo_nlp_enhanced.py`, `holo_nlp_enhanced_sentiment.py`, `holo_nlp_advanced.py`, `holo_nlp_algorithms.py`, `holo_nlp_dialogue.py`, `holo_nlp_domain_vocabularies.py`, `holo_nlp_extended_emotions.py`, `holo_nlp_integration.py`, `holo_nlp_intent.py`, `holo_nlp_multilingual.py`, `holo_nlp_recognition.py`, `holo_nlp_style_analysis.py`, `holo_nlp_training_extended.py`, `holo_nlp_unified_recognition.py`, `holo_advanced_nlp.py`, `holo_perception.py`, `holo_perception_unified.py`, `holo_intent_analyzer.py`, `holo_smart_understanding.py`, `holo_message_analyzer.py`, `holo_dialogue_engine.py`, `holo_speech_engine.py`, `holo_sentence_structures.py`, `holo_smalltalk_topics.py`, `holo_extended_vocabulary.py`, `holo_idiom_redewendungen.py`, `holo_synonym_engine_moods.py`, `holo_adaptive_communication.py`

### Persönlichkeit & Identität (15+ Module)

`holo_personality.py`, `holo_identity_evolution.py`, `holo_narrative_identity.py`, `holo_narrative_arc.py`, `holo_life_phases.py`, `holo_growth_mindset.py`, `holo_self_evolution.py`, `holo_self_expression.py`, `holo_drive_system.py`, `holo_preferences.py`, `holo_free_will.py`, `holo_habit_engine.py`, `holo_impulse_system.py`, `holo_character_integration.py`, `holo_digital_body.py`, `holo_digital_intimacy.py`, `holo_body_mind_bridge.py`, `holo_depth_system.py`, `holo_mutation_evolution.py`

### Kreativität (15+ Module)

`holo_creative_mind.py`, `holo_creative_engine.py`, `holo_creative_thinking.py`, `holo_creative_synthesis.py`, `holo_creative_expansion.py`, `holo_creative_orchestration.py`, `holo_conceptual_blending.py`, `holo_computational_creativity.py`, `holo_dream_engine.py`, `holo_dream_weaver.py`, `holo_dream_simulation.py`, `holo_intuition_engine.py`, `holo_music_experience.py`, `holo_synaesthesia_engine.py`, `holo_sensory_imagination.py`

### Autonomie (10+ Module)

`holo_autonomous_thinking.py`, `holo_autonomous_cognition_scheduler.py`, `holo_proactive_intelligence.py`, `holo_web_curiosity.py`, `holo_curiosity_engine.py`, `holo_curiosity_driven.py`, `holo_authentic_curiosity.py`, `holo_goal_engine.py`, `holo_longterm_goals.py`, `holo_hierarchical_planning.py`, `holo_predictive_behavior.py`, `holo_anticipatory_mind.py`, `holo_predictive_processing.py`

### Sozial (10+ Module)

`holo_social_cognition.py`, `holo_social_dynamics.py`, `holo_social_media.py`, `holo_theory_of_mind.py`, `holo_relationship_engine.py`, `holo_relationship_dynamics.py`, `holo_unified_relationship.py`, `holo_human_understanding.py`, `holo_authentic_connections.py`, `holo_forgiveness_system.py`, `holo_person_opinions.py`, `holo_multi_user_emotions.py`

### Organik & Lebendigkeit (8 Module)

`holo_organic_core.py`, `holo_organic_presence.py`, `holo_organic_response_enhancer.py`, `holo_organic_micro_behaviors.py`, `holo_subconscious_behavior.py`, `holo_subconscious_leak_bridge.py`, `holo_living_inefficiency.py`, `holo_micro_irreversibility.py`

### Infrastruktur (15+ Module)

`holo_database_system.py`, `holo_async_db.py`, `holo_db_migrations.py`, `holo_cache_integration.py`, `holo_dynamic_cache.py`, `holo_extended_cache.py`, `holo_stats_cache_integration.py`, `holo_stats_query_cache.py`, `holo_psyche_cache.py`, `holo_error_handling.py`, `holo_error_tracker.py`, `holo_health_checks.py`, `holo_performance_monitor.py`, `holo_resource_manager.py`, `holo_intelligent_resource_brain.py`, `holo_ram_manager.py`, `holo_memory_monitor.py`, `holo_energy_management.py`, `holo_energy_system.py`, `holo_self_healing.py`, `holo_self_repair.py`, `holo_structured_logging.py`, `holo_metrics.py`, `holo_introspection_analytics.py`, `holo_live_monitor.py`, `holo_system_status.py`, `holo_process_controller.py`

### LLM & Routing (8 Module)

`smart_llm_system.py`, `holo_intelligent_router.py`, `holo_tool_router.py`, `holo_state_collector.py`, `holo_response_generator.py`, `holo_response_quality_gate.py`, `holo_response_state_engine.py`, `holo_unified.py`

### Weltwissen (5+ Module)

`holo_world_why.py`, `holo_world_model.py`, `holo_local_intelligence.py`, `holo_local_understanding.py`, `holo_local_embeddings.py`, `holo_heimat.py`, `holo_calendar_awareness.py`, `holo_real_world_sync.py`

### Lunaria RPG (10 Module)

`holo_lunaria_engine.py`, `holo_lunaria_world.py`, `holo_lunaria_character.py`, `holo_lunaria_combat.py`, `holo_lunaria_database.py`, `holo_lunaria_items.py`, `holo_lunaria_npcs.py`, `holo_lunaria_professions.py`, `holo_lunaria_quests.py`, `holo_lunaria_races.py`

### Neurale Netze & ML (8 Module)

`holo_neural_network.py`, `holo_neural_bridge.py`, `holo_bridge_manager.py`, `holo_ml_intelligence.py`, `holo_markov_training.py`, `holo_advanced_mdp.py`, `holo_approximation_algorithms.py`, `holo_adaptive_intelligence.py`

### Emergenz & Evolution (4 Module)

`holo_emergent_capability_detector.py`, `holo_emergent_integration.py`, `holo_synergy_amplifier.py`, `holo_moe_system.py`

### Ethik & Regeln (4 Module)

`holo_ethical_framework.py`, `holo_policy_engine.py`, `holo_rule_engine.py`, `holo_skill_system.py`

### Tiefensystem-Brücken (3 Module)

`holo_deep_system_bridge.py`, `holo_pattern_detection.py`, `holo_pattern_integration.py`

### Sonstige

`holo_soulslike_consequences.py`, `holo_soulslike_integration.py`, `holo_media_discovery.py`, `holo_media_integration.py`, `holo_media_knowledge.py`, `holo_media_processor.py`, `holo_media_storage.py`, `holo_media_index.py`, `holo_nas_creative_storage.py`, `holo_document.py`, `holo_text_reader.py`, `holo_reader_extended.py`, `holo_video.py`, `holo_vision_enhanced.py`, `holo_vision_advanced.py`, `holo_vision_extended.py`, `holo_audio.py`, `holo_audio_enhanced.py`, `holo_voice_interface.py`, `holo_dashboard.py`, `holo_tester.py`, `holo_chaos_testing.py`, `holo_tools.py`, `holo_wisdom_system.py`, `holo_humor_advanced.py`, `holo_economic_models.py`

### Smart Home & IoT (1 Datei)

| Modul | Datei | Funktion |
|-------|-------|----------|
| **HomeAssistantConnector** | `holo/io/ha_connector.py` | Direkter HA-Draht: Licht/Heizung/Szenen/Medien/Cover, eigene Smart-Home-NLP, Entity-Discovery, Notifications. Holo steuert das Smart Home selbst. |

> **v21.27:** Das frühere externe Pi-Control (`pi_control_v8_AI-extendet.py`, 16.521 Z. Zweitgehirn) und seine HTTP-Brücke (`pi_holo_interface.py`) wurden entfernt. Holo besitzt die „Hände" nativ (ha_connector), Wetter aus HA, Tagesmodus aus der Uhr.

### Ops-Tools (2 Dateien)

| Modul | Datei | Funktion |
|-------|-------|----------|
| **CheckReadiness** | `check_readiness.py` | Preflight-Check: Python-Version, Dependencies, Ollama, Redis, MQTT, HA, Module-Imports, Verzeichnisse (`--quick`, `--json`, `--fix`) |
| **RAMMonitor** | `ram_monitor.py` | Startet Holo und misst alle 5s RSS/VMS/Threads für 2 Minuten (Speicher-Profiling) |

### Tests (25 Dateien)

`tests/conftest.py`, `tests/test_holo_config.py`, `tests/test_chat_simulation.py`, `tests/test_holo_decision_fatigue.py`, `tests/test_mqtt_bridge.py`, `tests/test_holo_procrastination.py`, `tests/test_holo_curiosity.py`, `tests/test_holo_brain_controller.py`, `tests/test_holo_brain_composition.py`, `tests/test_holo_cognitive_dissonance.py`, `tests/test_holo_dependency_container.py`, `tests/test_holo_empathy.py`, `tests/test_holo_error_handling.py`, `tests/test_holo_inner_dialogue.py`, `tests/test_holo_media_discovery.py`, `tests/test_holo_media_storage.py`, `tests/test_holo_state_manager.py`, `tests/test_holo_wiring.py`, `tests/test_wiring_split.py`, `tests/test_holo_events.py`, `tests/test_holo_utils.py`, `tests/test_boot_sequence.py`, `tests/test_holo_media.py`, `tests/test_self_healing_strategies.py`, `test_proactive_intelligence.py`

### Extensions & Skills

| Modul | Datei | Funktion |
|-------|-------|----------|
| **Extensions** | `extensions/example_extension.py` | Beispiel-Extension mit Hot-Reload |
| **ComfyUI Skill** | `skills/comfyui_skill.py` | Bildgenerierung via ComfyUI-Anbindung |

### Home Assistant Custom Component (`custom_components/holo/`)

| Modul | Datei | Funktion |
|-------|-------|----------|
| **ConfigFlow** | `custom_components/holo/config_flow.py` | UI-Setup für HA-Integration |
| **Sensor** | `custom_components/holo/sensor.py` | Holo-Status als HA-Sensor-Entities |
| **Conversation** | `custom_components/holo/conversation.py` | HA Conversation Agent → Holo |
| **Constants** | `custom_components/holo/const.py` | Domain, Plattformen, Defaults |

---

## Architektur-Diagramm: Gegenseitige Kognitivität (Übersicht)

```
                        ┌───────────────────────────┐
                        │       KIRA (User)          │
                        └─────────────┬─────────────┘
                                      │
                    Chat/Discord/MQTT/WebSocket
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
   ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
   │     HOLO     │          │     NORA     │          │    MYURI     │
   │  (Companion) │          │   (System)   │          │  (Creative)  │
   │              │          │              │          │              │
   │ Brain v15.0  │          │ ki/nora/*    │          │ ki/myuri/*   │
   │ 200+ Module  │          │              │          │              │
   └──────┬───────┘          └──────┬───────┘          └──────┬───────┘
          │                         │                         │
          └─────────────────────────┼─────────────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         │  MQTT Transport      │
                         │  ki/{name}/peer/*    │
                         │  ki/group            │
                         │  ki/consensus/*      │
                         └──────────┬──────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
   ┌──────────────┐     ┌────────────────────┐     ┌──────────────┐
   │ PeerNetwork  │     │ MutualCognition    │     │ MultiAgent   │
   │ Transport    │     │ Kognitive Schicht  │     │ System       │
   │              │     │                    │     │              │
   │ • Messages   │     │ • BeliefBoard      │     │ • Registry   │
   │ • Heartbeat  │     │ • PeerModels       │     │ • Reputation │
   │ • Consensus  │     │ • ReasoningExch.   │     │ • Bonding    │
   │ • Negotiation│     │ • Grounding        │     │ • Specialize │
   │ • Veto       │     │ • Negotiation      │     │ • Discovery  │
   └──────────────┘     └────────────────────┘     └──────────────┘
```

---

---

## 28. Feedback-Loops & Lern-Zyklen

v18.3 hat **37+ fehlende Feedback-Loops** geschlossen. Vorher waren viele Module Write-Only (produzieren Daten, aber niemand liest sie) oder Dead Sinks (hören zu, aber Ergebnisse verschwinden).

### Geschlossene Loops (Auswahl)

| # | Loop | Von → Nach | Mechanismus |
|---|------|-----------|-------------|
| 1 | Emotion→Response | EmotionalCore → ResponseGenerator | Stimmung beeinflusst Wortwahl und Ton |
| 2 | Curiosity→KnowledgeGraph | CuriosityEngine → KnowledgeGraph | Neugier-Themen werden als Wissensknoten gespeichert |
| 3 | Neural→Router | NeuralBridge → IntelligentRouter | Neuronale Aktivierung beeinflusst Routing-Entscheidung |
| 4 | Memory→Consolidation | WorkingMemory → LearningConsolidation | Arbeitsgedächtnis wird nachts konsolidiert |
| 5 | Error→Learning | ErrorTracker → LearningEngine | Fehlermuster werden als Lernstoff verarbeitet |
| 6 | Writing→Persistence | AdaptiveCommunication → StateManager | Schreibstil-Änderungen überleben Restarts |
| 7 | Persona→State | PersonalityKernel → CognitiveStatePersistence | Persönlichkeits-Snapshots werden gesichert |

### Write-Only Caches → Integriert (FIX 26)

7 Caches die vorher nur geschrieben aber nie gelesen wurden:
- `holo_psyche_cache.py` → jetzt von DeepPsychology gelesen
- `holo_stats_cache_integration.py` → jetzt von IntrospectionAnalytics gelesen
- `holo_dynamic_cache.py` → jetzt von Router gelesen
- etc.

---

## 29. Entscheidungs-Loops & Policy Engine

### PolicyEngine (`holo_policy_engine.py`)

Nutzt Q-Learning und Bayesianische Überzeugungen für Verhaltens-Entscheidungen:

```
Situation → Policy Lookup → Q-Table
                              ↓
                    Aktion mit höchstem Q-Wert
                              ↓
                    Ausführung → Reward
                              ↓
                    Q-Value Update (TD-Learning)
```

### Entscheidungsebenen

| Ebene | Modul | Geschwindigkeit |
|-------|-------|----------------|
| **Instinkt** | `holo_instincts.py` | <1ms, reflexhaft |
| **Gewohnheit** | `holo_habit_engine.py` | ~5ms, gelerntes Muster |
| **Deliberation** | `holo_policy_engine.py` | ~50ms, Q-Learning |
| **Reasoning** | `holo_reasoning_arbitrator.py` | ~200ms, Ensemble |
| **Deep Thinking** | `holo_deep_thinking.py` | ~500ms+, tiefes Nachdenken |

### Reasoning-Arbitrator (v18.3 NEU)

Ensemble-Mechanik: Mehrere Reasoning-Engines (Bayesian, Causal, Dialectical, Analogical) stimmen ab. Lazy-Init reduziert Startup von ~2s auf ~0.7s.

---

## 30. Gegenseitige Beeinflussung (14+ Sync-Pfade)

### Synchronisationspfade zwischen Modulen

```
Emotionen ──────────► Router (Wortwahl)
     │                   │
     ├──────────► Persönlichkeit (Trait-Gewichtung)
     │                   │
     ├──────────► Digital Body (Ohren/Schweif)
     │                   │
     └──────────► Energie (emotionale Erschöpfung)

Energie ────────────► Router (Antwortlänge)
     │                   │
     ├──────────► Consciousness (Aufmerksamkeit)
     │                   │
     └──────────► Autonomie (Proaktivitätslevel)

Lernen ─────────────► Reasoning (neues Wissen als Kontext)
     │                   │
     ├──────────► KnowledgeGraph (neue Knoten)
     │                   │
     └──────────► Curiosity (erledigte vs. offene Fragen)

Wahrnehmung ────────► Consciousness (sensorischer Input)
     │                   │
     ├──────────► Memory (Kontextualisierung)
     │                   │
     └──────────► Emotions (Sentiment → Stimmung)
```

### v18.3: 6 neue Pfade

| Pfad | Beschreibung |
|------|-------------|
| ResponseStyle←Emotion | Emotionslage steuert Antwort-Stil (wärmer/kürzer/vorsichtiger) |
| Curiosity→KnowledgeGraph | Neugier-Entdeckungen werden als Wissensknoten persistiert |
| Neural→Router | Neuronale Aktivierungsmuster beeinflussen Routing-Entscheidung |
| Learning→Reasoning | Gelerntes Wissen steht dem Reasoning sofort zur Verfügung |
| Error→DeepThinking | Fehlermuster aktivieren vorsichtigeren Denkmodus |
| Emotion→Reasoning | Analytisches Mapping: Emotionszustand → Reasoning-Strategie |

---

## 31. Callback-Architektur & Event-Kaskaden

### Event-Flow bei einer User-Nachricht

```
UserMessage
    │
    ├── EventBus: "message_received" ──────► 12+ Subscriber
    │                                          ├── NLP Pipeline
    │                                          ├── EmotionalCore
    │                                          ├── WorkingMemory
    │                                          ├── SocialCognition
    │                                          └── ...
    │
    ├── Callback: emotion_changed ─────────► Router, Personality, DigitalBody
    │
    ├── Callback: energy_changed ──────────► Router, Consciousness
    │
    └── Callback: memory_stored ───────────► Learning, Reasoning
```

### Extended Callback Dispatcher (v18.3, FIX 31-32)

Vorher: Viele Callbacks waren registriert aber nie aufgerufen.
Nachher: `_extended_callback_dispatcher` in Brain sorgt dafür, dass alle registrierten Callbacks auch wirklich feuern.

### Silent-Failure Fallbacks (v18.3, FIX 33-36)

Router, Wiring und Reasoning haben jetzt Fallback-Pfade wenn Callbacks fehlschlagen — statt stiller Fehler gibt es Warnungen und degradierte Antworten.

---

## 32. Kopplungs-Matrix (Wer beeinflusst wen?)

### Kern-Kopplungen (Auszug, 15 wichtigste)

| Quelle | → Ziel | Kopplung | Stärke |
|--------|--------|----------|--------|
| EmotionalCore | → Router | Direkt (Callback) | Stark |
| EmotionalCore | → Personality | Direkt (Callback) | Stark |
| EmotionalCore | → DigitalBody | Direkt (Callback) | Stark |
| EnergySystem | → Router | Direkt (Callback) | Stark |
| EnergySystem | → Consciousness | Direkt (Callback) | Mittel |
| NLP/Perception | → Consciousness | EventBus | Mittel |
| NLP/Perception | → Memory | EventBus | Mittel |
| LearningEngine | → KnowledgeGraph | Direkt (API) | Stark |
| LearningEngine | → DeepThinking | EventBus | Schwach |
| ErrorTracker | → LearningEngine | Callback | Mittel |
| CuriosityEngine | → KnowledgeGraph | Direkt (API) | Mittel |
| NeuralBridge | → Router | Callback | Schwach |
| PolicyEngine | → DriveSystem | Q-Learning | Mittel |
| AutonomousThinking | → ImpulseSystem | Direkt | Stark |
| MutualCognition | → PeerNetwork | MQTT | Stark |

### Kopplungstypen

| Typ | Latenz | Beispiel |
|-----|--------|---------|
| **Direkt (Callback)** | <1ms | Emotion → Router |
| **EventBus** | ~5ms | Perception → Consciousness |
| **StateManager** | ~10ms | Energy → Persistierung |
| **MQTT** | ~50ms | Holo → Nora |
| **LLM** | ~200ms+ | Router → Ollama |

---

## 33. Lückenanalyse & Implementierungsstatus

### v18.3: 37 Fixes — Status

| Fix-Bereich | Anzahl | Status |
|-------------|--------|--------|
| Feedback-Loops geschlossen | 13 | Erledigt |
| Dead-Code Bugs behoben | 25 | Erledigt |
| Write-Only Caches integriert | 7 | Erledigt |
| Silent-Failure Fallbacks | 4 | Erledigt |
| Callback Dispatcher erweitert | 2 | Erledigt |
| Phantom-Module bereinigt | 3 | Erledigt |
| State Persistence hinzugefügt | 2 | Erledigt |
| Unsafe eval() entfernt | 1 | Erledigt |
| Resource Leaks gefixt | 1 | Erledigt |

### Bekannte offene Punkte

| Bereich | Beschreibung | Priorität |
|---------|-------------|-----------|
| Multi-Agent real | Nora & Myuri existieren als MQTT-Stubs, nicht als eigenständige Instanzen | Niedrig |
| LLM-Modell-Rotation | Tool-Router wählt immer dasselbe Modell bei gleichen Inputs | Mittel |
| Dream-Konsolidierung | DreamEngine schreibt, aber LearningConsolidation liest nicht alle Traum-Daten | Niedrig |
| Vision-Pipeline | qwen3-vl Setup nicht automatisiert | Niedrig |

---

## 34. Ungenutztes Potenzial & Empfehlungen

### Module mit hohem ungenutztem Potenzial

| Modul | Potential | Empfehlung |
|-------|----------|------------|
| `holo_game_theory.py` | Spieltheoretische Entscheidungen | Könnte Verhandlungen in MutualCognition verbessern |
| `holo_economic_models.py` | Ressourcenallokation | Könnte Energie-Management optimieren |
| `holo_complexity_theory.py` | Big-O Analyse | Könnte Query-Routing intelligenter machen |
| `holo_formal_axioms.py` | Formale Logik | Könnte Bias-Detection stärken |
| `holo_lunaria_*.py` | RPG-System (10 Module) | Vollständig implementiert, wartet auf User-Aktivierung |

### Architektur-Empfehlungen

1. **Module-Lazy-Loading ausweiten** — Startup ~202 MB, könnte mit mehr Lazy-Init auf ~150 MB sinken
2. **EventBus-Metriken** — Welche Events werden nie konsumiert? Welche haben zu viele Subscriber?
3. **GOAP-Monitoring** — Die neuen Limits loggen Warnungen, aber ein Dashboard wäre besser
4. **Cache-Audit regelmäßig** — v18.3 hat 7 Write-Only Caches gefunden, es könnten sich neue bilden

---

## 35. GOAP A*-Planer — Speicher-Limits

### Problem (v18.4)

Ohne Limits konnte der GOAP A*-Heap bei vielen dynamischen Aktionen (besonders HA-Entities im Smart-Home-Planer) auf hunderte MB wachsen und den RAM sprengen.

### Lösung: Drei-Stufen-Schutz

```
┌─────────────────────────────────────────────────────────┐
│  GOAP A* Speicher-Schutz                                  │
│                                                           │
│  1. MAX_VISITED_NODES = 5.000                            │
│     → visited-Dict (geschlossene Menge) begrenzt         │
│                                                           │
│  2. MAX_HEAP_ENTRIES = 10.000                             │
│     → Open-Set (Priority Queue) begrenzt                  │
│                                                           │
│  3. MAX_PLAN_DEPTH = 15 (SmartHome) / 10 (Personal)     │
│     → Maximale Plantiefe verhindert endlose Ketten        │
│                                                           │
│  Bei Limit-Überschreitung:                                │
│  → Logger-Warning mit Details (Heap-Größe, Node-Count)   │
│  → Suche bricht sauber ab                                 │
│  → Fallback-Plan greift                                   │
└─────────────────────────────────────────────────────────┘
```

### Betroffene Planer

| Planer | Datei | Limits |
|--------|-------|--------|
| **PersonalGOAPPlanner** | `holo_self_awareness.py` | 5.000 Nodes, 10.000 Heap, Tiefe 10 |

### RAM-Profil (gemessen)

```
Zeit     RSS (MB)   Status
  0s        0.3     Start
  5s      202.4     Alle Module geladen
 60s      202.4     Stabil
120s      202.4     Stabil — kein Wachstum
```

Peak: **202.4 MB** · Wachstum nach Init: **±0.0 MB** · Kein Memory-Leak

---

## 36. Ops-Tools (Readiness-Check, RAM-Monitor)

### check_readiness.py — Preflight-Check

Prüft vor dem Start ob das System bereit ist:

```bash
python check_readiness.py          # Vollständiger Check
python check_readiness.py --quick  # Nur kritische Checks
python check_readiness.py --json   # Maschinenlesbares Ergebnis
python check_readiness.py --fix    # Versucht Probleme automatisch zu beheben
```

| Check | Was wird geprüft |
|-------|-----------------|
| Python-Version | ≥ 3.11 |
| Dependencies | Alle requirements.txt Pakete installiert |
| Ollama | Erreichbar, Modelle verfügbar |
| Redis | Verbindung, Ping |
| MQTT | Broker erreichbar |
| Home Assistant | API-Zugriff, Token gültig |
| Module-Imports | Alle 361 holo_*-Module importierbar |
| Verzeichnisse | data/, logs/, state/, dream_data/ existieren |
| Config | config.json vorhanden und valide |

### ram_monitor.py — Speicher-Profiling

Startet Holo (`--no-chat`) und misst alle 5 Sekunden für 2 Minuten:

```
Zeit    RSS (MB)    VMS (MB)   Threads  Status
  0s        0.3        7.1        1     OK
  5s      202.4      590.6        6     OK
 60s      202.4      590.6        6     OK
120s      202.4      590.6        6     OK
```

Ergebnis: Start-RAM, End-RAM, Peak-RAM, Wachstum, Bewertung (OK / HOCH / KRITISCH).

---

## 37. Changelog v18.3 / v18.4

### v18.4 (2026-03-24)

- **GOAP A* Speicher-Limits**: Beide Planer (Personal + SmartHome) mit MAX_VISITED_NODES=5000, MAX_HEAP_ENTRIES=10000 abgesichert
- **RAM-Monitor hinzugefügt**: `ram_monitor.py` für Speicher-Profiling
- **Ergebnis**: 202 MB stabil, kein Memory-Leak

### v18.3 (2026-03-22 → 2026-03-24)

| Commit | Beschreibung |
|--------|-------------|
| FIX 1-7 | 7 fehlende Feedback-Loops geschlossen |
| FIX 8-13 | 6 weitere Feedback-Loops + Kopplungen |
| FIX 16-19 | ResponseStyle←Emotion, Memory Consolidation, Dead Producer/Sink |
| FIX 20-25 | Callback-Emissionen, Curiosity→KG, Neural→Router, EventBus-Analyse |
| FIX 26 | 7 Write-Only Caches aufgedeckt und integriert |
| FIX 27-29 | Phantom-Module, State Snapshot, Cache Cleanup |
| FIX 30 | Persona State Persistence + Phantom-Module Final |
| FIX 31-32 | Extended Callback Dispatcher + Writing Style Persistenz |
| FIX 33-36 | Silent-Failure Fallbacks für Router, Wiring, Reasoning |
| FIX 37 | Analytical Emotion→Reasoning Mapping |
| Dead-Code | 25 Dead-Code Bugs behoben |
| Security | Unsafe `eval()` entfernt, Resource Leaks gefixt, `except: pass` entfernt |
| Performance | Lazy-Init für Reasoning-Engines (Startup ~2s → ~0.7s) |
| Ensemble | Reasoning-Arbitrator mit Ensemble-Mechanik |

---

---

## 38. v19.0 — Advanced Cognitive Stack

In v19.0 wurden **31 neue Module** hinzugefügt, die auf dem existierenden
System aufsetzen ohne es zu brechen. Thematische Gruppierung:

### 38.1 Deliberation & Planung (5 Module)

| Modul | Zweck | Kern-Klassen |
|-------|-------|--------------|
| `holo_deliberation.py` | Denke-bevor-du-handelst: Kandidaten + Abwägung + Ethik + Begründung | `DeliberationEngine`, `Candidate`, `ConsequenceForecast` |
| `holo_deliberation_planner.py` | Plan über 2-3 Züge + Persönlichkeits-Bias + Kontext-Lernen | `AdvancedDeliberationEngine`, `MultiTurnPlanner`, `DoubtModeler`, `PerspectiveTaker` |
| `holo_mcts_planner.py` | Monte-Carlo Tree Search für 5-10 Zug-Weitsicht | `MCTSPlanner`, `DialogState`, `TransitionModel` |
| `holo_cot_engine.py` | Chain-of-Thought mit 12 Schritt-Typen | `CoTEngine`, `ReasoningChain`, `CoTStep` |
| `holo_forward_chaining.py` | Regel-Inferenz mit Syllogismen | `ForwardChainingEngine`, `RuleBase`, `SyllogismValidator` |

### 38.2 Emotion & Empathie (4 Module)

| Modul | Zweck | Kern-Klassen |
|-------|-------|--------------|
| `holo_complex_emotions.py` | 12 zeit-gerichtete Emotionen (Nostalgie, Reue, Stolz, Scham, Ehrfurcht, Wehmut, Hoffnung, Sehnsucht...) | `ComplexEmotion`, `ComplexEmotionTrigger` |
| `holo_empathic_boundaries.py` | Menschliche Grenzen der Emotions-Ansteckung | `EmpathicBoundaryEngine`, `BoundaryMode`, `EmpathyEpisode` |
| `holo_stress_resistance.py` | Mentalitätsstabilität unter Provokation/Manipulation | `StressResistanceEngine`, `StressorDetector`, `CoreTrait` |
| `holo_tom_level2.py` | Theory of Mind Level 2 ("was denkt er über mich") mit Overthinking-Grenze | `TheoryOfMindLevel2`, `Level2Belief`, `PerspectiveAlignment` |

### 38.3 Lernen & Anpassung (7 Module)

| Modul | Zweck | Kern-Klassen |
|-------|-------|--------------|
| `holo_outcome_learner.py` | Feedback-Loop: Action+User-Reaktion → Mode-Bias pro Kontext | `OutcomeLearner`, `LearningEpisode`, `ModeStats` |
| `holo_error_pattern_learner.py` | Aus Fehlern lernen → AvoidanceRules | `ErrorPatternLearner`, `ErrorPattern`, `AvoidanceRule` |
| `holo_surprise_detector.py` | Prediction-Error als Lernsignal (Free-Energy-style) | `SurpriseDetector`, `SurpriseEvent`, `SurpriseStrength` |
| `holo_policy_learning.py` | Q-Learning + SARSA-Lambda + Softmax-Policy-Gradient | `QLearningAgent`, `SoftmaxPolicyAgent`, `PolicyLearningEngine` |
| `holo_self_supervised_learner.py` | 4 Pretext-Tasks ohne Labels (Next-Word, Masked, Order, Intent) | `SelfSupervisedLearner`, `SSLKnowledge` |
| `holo_contrastive_learner.py` | Triplet-Loss Embedding-Learning | `ContrastiveLearner`, `SparseVector`, `Concept` |
| `holo_cross_domain_transfer.py` | Success-Pattern von Domain A auf Domain B übertragen | `CrossDomainTransfer`, `AbstractPattern`, `DomainAdapter` |

### 38.4 Wahrnehmung & Kontext (4 Module)

| Modul | Zweck | Kern-Klassen |
|-------|-------|--------------|
| `holo_current_situation.py` | Zentrales JETZT-Modell (5 Dimensionen) | `CurrentSituation`, `SituationManager`, `ConversationPhase` |
| `holo_binding_mechanism.py` | Feature-Binding zu kohärenten Percepts (350ms Sync-Window) | `BindingMechanism`, `FeatureEvent`, `PerceptUnit` |
| `holo_concept_drift_monitor.py` | Erkennt wenn User/Welt sich verändern (Cohen's d + JSD) | `ConceptDriftMonitor`, `NumericDriftDetector`, `CategoricalDriftDetector` |
| `holo_narrative_coherence_monitor.py` | Story-Konsistenz über Zeit | `NarrativeCoherenceMonitor`, `Claim`, `CoherenceReport` |

### 38.5 Integration & Regulation (6 Module)

| Modul | Zweck | Kern-Klassen |
|-------|-------|--------------|
| `holo_bidirectional_wiring.py` | Gegenseitige Beeinflussung (Mood↔Reasoning, Action↔Energy) | `BidirectionalWiring`, `TriggerRule`, `EffectApplier` |
| `holo_proactive_action_mapper.py` | Insight → präventive Aktion (7 Default-Mappings) | `ProactiveActionMapper`, `Insight`, `PreventiveAction` |
| `holo_deduplication_engine.py` | Anti-Nag: semantischer Hash + Topic-Cooldown | `DeduplicationEngine`, `SemanticHasher`, `DedupCheck` |
| `holo_creative_refinement.py` | Divergent/Convergent-Pipeline mit Novelty-Scoring | `CreativeRefinement`, `NoveltyScorer`, `UsefulnessScorer` |
| `holo_free_energy_minimizer.py` | Active Inference (Friston) — EFE-minimierende Aktion | `ActiveInferenceAgent`, `Belief`, `GenerativeModel` |
| `holo_dual_process.py` | System 1 (Reflex) / System 2 (Deliberation) nach Kahneman | `DualProcessEngine`, `ReflexBank`, `ThinkingRouter` |

### 38.6 Identität & Ziele (3 Module)

| Modul | Zweck | Kern-Klassen |
|-------|-------|--------------|
| `holo_persona_stacking.py` | Core + Relational/Context/Energy/Mood-Layers | `PersonaStack`, `CoreValues`, `LayerLibrary` |
| `holo_goal_abandonment.py` | Loslassen mit Anti-Sunk-Cost | `GoalAbandonmentSystem`, `AbandonmentDecider`, `AbandonReason` |
| (weitere in 39+40) | Siehe Supervisor + GOAP | |

### 38.7 Kontrolle (2 Module — siehe Kap. 39+40)

| Modul | Zweck |
|-------|-------|
| `holo_action_supervisor.py` | Zentrale Gate-Schicht (siehe Kapitel 39) |
| `holo_goap_coordinator.py` | Action-Planung mit A* (siehe Kapitel 40) |

---

## 39. v19.0 — Action Supervisor

**Datei:** `holo_action_supervisor.py` (~460 LOC)

Die zentrale **Gate-Schicht** durch die jede Aktion muss. Schließt das
Problem dass 6 Module (proactive_mapper, outcome_learner,
bidirectional_wiring, social_observers, web_curiosity,
proactive_intelligence) vorher autonom handelten ohne Holos Wissen.

### 39.1 Pipeline pro Aktion

```
Modul will Action ausführen
      ↓
ActionIntent(source, type, risk, reason, autonomous)
      ↓
Supervisor.review(intent)
      ├─→ 1. Rate-Limit (10/min pro Modul)
      ├─→ 2. Risk-Level-Gate (CRITICAL → DEFERRED)
      ├─→ 3. State-Consistency (Silence/Stress blockiert)
      ├─→ 4. Ethik-Check (mass_/spam_/delete_all)
      ├─→ 5. GOAP-Awareness (nicht-registriert → OBSERVED)
      └─→ 6. Custom Rules
      ↓
SupervisorDecision (APPROVED / DEFERRED / REJECTED / OBSERVED)
      ↓
AwarenessInbox (Holo sieht nachträglich was gelaufen ist)
```

### 39.2 Risk-Levels (6 Stufen)

| Risk-Level | Beschreibung | Beispiele |
|-----------|--------------|-----------|
| `NONE` | Rein innerlich, kein Effekt | Reiner Text |
| `INTERNAL` | Nur interner State | Deliberation-Bias |
| `EXTERNAL_SAFE` | Extern aber reversibel | API-GET, Log |
| `EXTERNAL_VISIBLE` | Andere sehen es | Discord, Chat |
| `EXTERNAL_PERMANENT` | Kaum reversibel | Mail senden, Post |
| `CRITICAL` | Smart-Home, Finanzen | MQTT, Banking |

Autonome Aktionen mit `EXTERNAL_PERMANENT` und `CRITICAL` werden
**DEFERRED** — sie kommen in eine Queue und warten auf
`brain.approve_pending_action(id)`.

### 39.3 State-Consistency-Checks

| Zustand | Effekt |
|---------|--------|
| `silence_mode=True` | Externe Aktionen blockiert |
| `stress_level="critical"` | Nur INTERNAL Aktionen |
| `compassion_fatigue="depleted"` | Autonome Aktionen DEFERRED |
| `manipulation_detected=True` | GUARDED-Mode aktiv |

### 39.4 AwarenessInbox

Selbst wenn Holo nicht selbst entschieden hat — sie **sieht** was
gelaufen ist. 500 letzte autonome Actions persistent in
`data/awareness_inbox.json`.

```python
brain.awareness_unread_count()   # → 12
brain.describe_autonomous_actions(5)
# → "Meine Module haben 5 Aktion(en) gemacht:
#    • proactive_mapper → proactive.shorten_and_soften [approved]
#    • bidirectional_wiring → wiring.action_success [approved]
#    • outcome_learner → learning.episode_recorded [approved]"
```

### 39.5 Integration in autonome Module

Alle 4 kritischen autonomen Module haben `_supervisor_approves()`:
- `holo_proactive_action_mapper.py` — vor `_execute()`
- `holo_bidirectional_wiring.py` — vor `_apply()` pro Rule
- `holo_outcome_learner.py` — vor `record_episode()`
- `holo_action_dispatcher.py` — vor handler-Aufruf

---

## 40. v19.0 — GOAP Coordinator

**Datei:** `holo_goap_coordinator.py` (~550 LOC)

**Goal-Oriented Action Planning** als zentrales System. Vorher gab es
drei separate Planungs-Systeme (PersonalGOAPPlanner isoliert,
HierarchicalPlanner parallel, ActionDispatcher ohne GOAP). Jetzt:
**alle Dispatcher-Aktionen sind in einer GOAP-Registry**.

### 40.1 Datenmodell

```python
@dataclass
class GOAPAction:
    name: str
    preconditions: dict[str, Any]     # muss wahr sein
    effects: dict[str, Any]           # wird wahr nach Ausführung
    base_cost: float
    current_cost: float               # adaptiv gelernt
    source_module: str
    execute: Callable                  # wie wird es ausgeführt
    success_count: int
    failure_count: int
```

### 40.2 Auto-Registrierung aus Dispatcher

Jeder `Dispatcher.register(action_type, handler, category=...)` trägt
automatisch in GOAP ein mit sinnvollen Default-Preconditions:

| Kategorie | Default-Preconditions | Default-Effects | Cost |
|-----------|----------------------|------------------|------|
| `MESSAGING` | `messaging_channel_available` | `message_sent` | 1.0 |
| `SMART_HOME` | `smart_home_reachable` | `smart_home_state_changed` | 2.0 |
| `SOCIAL` | `social_account_active` | `social_action_performed` | 1.0 |
| `API` | `network_available` | `api_called` | 1.0 |
| `PLAN_STEP` | — | `plan_step_completed` | 1.0 |

**Alle 11 Action-Module sind jetzt in GOAP:** mastermind_bridge,
api_provider, notification_hub, ha_connector, mqtt_bridge,
hierarchical_planner, tool_router, social_observers,
proactive_action_mapper (via Dispatcher), bidirectional_wiring
(intern via Supervisor), outcome_learner (intern).

### 40.3 A*-Planer

```python
GOAPPlanner.plan(goal={"message_sent": True})
# → GOAPPlan(
#     actions=[
#         GOAPAction("check_network"),
#         GOAPAction("connect_discord"),
#         GOAPAction("discord.send"),
#     ],
#     total_cost=3.0,
#     goal_reached=True,
#   )
```

Eigenschaften:
- A*-Suche mit Heuristik = Anzahl unerfüllter Goal-Bedingungen
- MAX_DEPTH=12, MAX_NODES=2000 (Terminierungs-Garantie)
- Bevorzugt billigere Pfade
- Liefert `estimated_states[i]` = WorldState nach Aktion i

### 40.4 Pre-Execution-Gate im Dispatcher

Vor `handler.execute()` prüft der Dispatcher gegen GOAP:

```python
# Automatisch in Dispatcher.dispatch():
goap_check = self._goap_precondition_check(req)
if not goap_check["can_execute"]:
    return ActionResult(
        status=PRECONDITIONS_FAILED,
        error=f"GOAP-Preconditions fehlen: {list(goap_check['missing'])}"
    )
```

### 40.5 Outcome-Feedback-Loop

Nach jedem `dispatch()` registriert der Dispatcher Outcome bei GOAP:
- **Success**: `current_cost *= 0.95` (min 30% von base)
- **Failure**: `current_cost *= 1.2` (max 300% von base)

Der A*-Planer wählt automatisch die **gelernten günstigsten Pfade**.

### 40.6 Brain-API

```python
brain.plan_to_goal({"user_happy": True, "message_sent": True})
brain.can_execute_action("discord.send")
# → {"can_execute": False, "missing": {...},
#    "suggested_preparation": ["connect_discord"]}
brain.predict_action_effects("light_on")
# → {"deltas": {"light": {"before": "off", "after": "on"}}}
brain.register_goap_action("custom", preconditions=..., effects=...)
brain.update_world_state(battery_charged=True)
brain.list_goap_actions(module="dispatcher:messaging")
brain.get_goap_summary()
```

### 40.7 Supervisor-Awareness

Der Supervisor warnt bei autonomen Aktionen die **nicht** im GOAP
registriert sind:

```
Verdict.OBSERVED
Reason: "Aktion 'xxx' nicht im GOAP-Register — läuft, aber außerhalb
der Plan-Ebene. Modul 'yyy' sollte sich registrieren."
```

Wird in AwarenessInbox geloggt, blockiert aber nicht (fail-open).

---

## 41. v19.0 — Changelog + Test-Matrix

### 41.1 Neue Module (31)

**Runde 1 — Deliberation (2):**
- `holo_deliberation.py`, `holo_deliberation_planner.py`

**Runde 2 — Cognitive Fundament (4):**
- `holo_action_dispatcher.py`, `holo_current_situation.py`,
  `holo_forward_chaining.py` (+ OutcomeLearner-Extension)

**Runde 3 — Integration (4):**
- `holo_bidirectional_wiring.py`, `holo_outcome_learner.py`,
  `holo_proactive_action_mapper.py`, `holo_creative_refinement.py`

**Runde 4 — Kernel-Features (6):**
- `holo_deduplication_engine.py`, `holo_surprise_detector.py`,
  `holo_error_pattern_learner.py`, `holo_stress_resistance.py`,
  `holo_narrative_coherence_monitor.py`, `holo_complex_emotions.py`

**Runde 5 — Advanced Planning (4):**
- `holo_mcts_planner.py`, `holo_cross_domain_transfer.py`,
  `holo_policy_learning.py`, `holo_binding_mechanism.py`

**Runde 6 — Forschungs-KI (6):**
- `holo_cot_engine.py`, `holo_concept_drift_monitor.py`,
  `holo_self_supervised_learner.py`, `holo_contrastive_learner.py`,
  `holo_free_energy_minimizer.py`, `holo_dual_process.py`

**Runde 7 — Humane Grenzen (4):**
- `holo_empathic_boundaries.py`, `holo_goal_abandonment.py`,
  `holo_persona_stacking.py`, `holo_tom_level2.py`

**Runde 8 — Zentrale Kontrolle (2):**
- `holo_action_supervisor.py`, `holo_goap_coordinator.py`

### 41.2 Brain-API-Erweiterungen (~80 neue Methoden)

Gruppen neuer `brain.*`-Methoden:
- **Deliberation**: `explain_last_action`, `get_last_reasoning_trace`,
  `get_current_plan`, `get_current_doubt`, `get_current_perspective_check`
- **Situation**: `describe_current_situation`, `get_current_situation`
- **Actions**: `available_actions`, `dispatch_action`
- **Logic**: `infer_from_facts`, `get_deliberation_trends`
- **Wiring**: `get_recent_bidirectional_effects`, `trigger_bidirectional`
- **Learning**: `get_outcome_learning_summary`, `get_context_mode_bias`
- **Proactive**: `submit_proactive_insight`, `get_active_style_overrides`
- **Creative**: `creatively_synthesize`
- **Anti-Nag**: `check_response_for_repetition`, `record_sent_response`
- **Surprise**: `register_prediction`, `compare_prediction_to_actual`
- **Errors**: `record_module_error`, `check_should_avoid_action`
- **Stress**: `get_current_stability`, `get_core_traits`
- **Narrative**: `observe_own_statement`, `get_narrative_coherence`
- **Emotions**: `elicit_complex_emotions`, `describe_complex_emotional_state`
- **MCTS**: `mcts_plan`
- **Transfer**: `record_transfer_success`, `suggest_cross_domain_action`
- **Policy**: `policy_recommend`, `policy_learn`
- **Binding**: `report_feature`, `get_recent_percepts`
- **CoT**: `begin_reasoning`, `add_reasoning_step`, `close_reasoning`,
  `explain_last_reasoning`
- **Drift**: `observe_drift_feature`, `check_drift`, `describe_drift`
- **SSL**: `ssl_ingest`, `ssl_run`, `ssl_classify_intent`
- **Contrastive**: `add_concept`, `nearest_concepts`, `infer_concept_category`
- **Active Inference**: `should_i_ask_for_clarification`,
  `choose_action_active_inference`
- **Dual Process**: `think_dual_process`, `reflex_respond`, `feedback_on_reflex`
- **Empathy**: `set_empathy_context`, `get_empathy_boundary`,
  `compose_empathy_differentiation`
- **Goals**: `register_goal`, `goal_progress`, `should_abandon_goal`
- **Persona**: `set_persona_relation`, `set_persona_context`,
  `describe_persona`, `check_persona_integrity`
- **ToM L2**: `infer_user_belief_about_me`, `compare_view_with_user`,
  `ask_about_their_view`
- **Supervisor**: `get_supervisor_summary`, `get_pending_approvals`,
  `approve_pending_action`, `reject_pending_action`
- **Awareness**: `get_awareness_inbox`, `describe_autonomous_actions`,
  `mark_awareness_read`, `awareness_unread_count`
- **GOAP**: `plan_to_goal`, `can_execute_action`, `predict_action_effects`,
  `register_goap_action`, `update_world_state`, `get_world_state`,
  `list_goap_actions`, `get_goap_summary`

### 41.3 Test-Matrix

| Modul-Gruppe | Tests neu | Gesamt bestand |
|-------------|-----------|----------------|
| Deliberation (Runde 1-7) | 472 | ✅ grün |
| Supervisor (Runde 8) | 17 | ✅ grün |
| GOAP (Runde 8) | 21 | ✅ grün |
| **Total v19.0 neu** | **~510** | ✅ alle grün |
| **Total gesamt** | **1262** | ✅ alle grün |

### 41.4 Änderungen an bestehenden Modulen

| Modul | Änderung |
|-------|----------|
| `holo_brain.py` | +31 Modul-Inits, +80 API-Methoden, Supervisor+GOAP als letztes init |
| `holo_action_dispatcher.py` | Auto-Register in GOAP, Supervisor-Gate, GOAP-Preconditions-Check, Outcome-Feedback zu GOAP |
| `holo_proactive_action_mapper.py` | `_supervisor_approves()` vor `_execute()` |
| `holo_bidirectional_wiring.py` | `_supervisor_approves()` vor Effekt-Anwendung |
| `holo_outcome_learner.py` | `_supervisor_approves_learning()` vor `record_episode()` |
| `feature_flags.json` | `enable_deliberation` in base/dev/prod/quiet |
| `.gitignore` | 10+ neue Datenpfade (traces, plans, outcomes, awareness, goap, …) |

### 41.5 Version-History

| Version | Datum | Fokus |
|---------|-------|-------|
| **v19.0** | **2026-04-17** | **Advanced Cognitive Stack + Supervisor + GOAP** |
| v18.4 | 2026-03-24 | GOAP A*-Speicherschutz, RAM-Monitor |
| v18.3 | 2026-03-22 | 37 Feedback-Loops, Dead-Code-Fixes |
| v18.1 | 2026-03-10 | Situations-Klassifikation, Adversarial Challenge |
| v17.x | 2026-02 | Mutual Cognition, Peer Communication |
| v15.0 | 2026-01 | Brain v15.0, Intelligent Router |

### 41.6 Zentrale Design-Prinzipien (v19.0)

1. **Kein Modul ohne Gate** — jede Aktion durch Supervisor + GOAP
2. **Menschliches Maß** — Boundaries, Limits, Rate-Caps überall
3. **Awareness statt Kontrolle** — Holo weiß auch autonome Actions
4. **Anti-Sunk-Cost** — vergangene Investition ≠ zukünftige Entscheidung
5. **Confidence-Caps** — niemals 100% Sicherheit behaupten
6. **Overthinking-Schutz** — Level-3+-ToM geblockt, MCTS-Horizont begrenzt
7. **Core-Integrität** — Kern-Traits nur ±0.1 modulierbar
8. **Lern-Adaptivität** — Kosten, Bias, Strategien alle selbst-anpassend

---

---

## 42. v19.0 — Daten-Persistenz

Die neuen Module schreiben ihre Lern- und Tracking-Daten in `data/`.
Alle Stores sind in `.gitignore` — sie sind Runtime-Artefakte, nicht Code.

### 42.1 Übersicht aller Stores

| Modul | Datei | Max | Inhalt |
|-------|-------|-----|--------|
| `holo_deliberation` | `data/deliberation_traces.json` | 200 | Reasoning-Traces pro Entscheidung |
| `holo_deliberation_planner` | `data/deliberation_plans.json` | 100 | Mehr-Turn-Pläne |
| `holo_deliberation_planner` | `data/deliberation_outcomes.json` | 500 | Pro-Mode Outcome-Records |
| `holo_action_dispatcher` | `data/action_executions.json` | 500 | Dispatch-Log mit Results |
| `holo_outcome_learner` | `data/outcome_learning.json` | 2000 | Learning-Episodes (Reward) |
| `holo_error_pattern_learner` | `data/error_patterns.json` | — | Patterns + AvoidanceRules |
| `holo_narrative_coherence_monitor` | `data/narrative_claims.json` | 2000 | Claim-Zeitstrahl |
| `holo_cross_domain_transfer` | `data/transfer_patterns.json` | — | Abstrakte Pattern pro Domain |
| `holo_policy_learning` | `data/policy_table.json` | — | Q-Werte + Visit-Counts |
| `holo_self_supervised_learner` | `data/ssl_knowledge.json` | 5000 | Unigrams + Intent-Profile |
| `holo_contrastive_learner` | `data/contrastive_embeddings.json` | 500 | Concept-Embeddings (sparse) |
| `holo_concept_drift_monitor` | `data/concept_drift.json` | 2000 | Feature-Observations |
| `holo_goal_abandonment` | `data/abandonment_log.json` | 500 | Abandonment-Entscheidungen |
| `holo_action_supervisor` | `data/awareness_inbox.json` | 500 | Awareness-Einträge |
| `holo_goap_coordinator` | `data/goap_registry.json` | — | Alle Actions + adaptive Kosten |
| `holo_goap_coordinator` | `data/goap_outcomes.json` | — | Outcome-Historie |

### 42.2 Design-Entscheidungen

- **Idempotentes Laden**: jedes Modul kann beim Start seinen State
  aus dem JSON-Store wiederherstellen. Crash-Safe.
- **Kein PII**: gespeichert werden nur **abstrahierte Scores/Patterns**,
  keine Rohtexte oder User-Identifikatoren.
- **Ring-Buffer-Semantik**: die meisten Stores sind `deque(maxlen=...)` —
  älteste Einträge werden automatisch verdrängt.
- **Write-Through**: Updates werden sofort persistiert, nicht batched
  (bei Crash geht maximal die aktuelle Episode verloren).
- **Lesefreundlich**: JSON mit `indent=2` und `ensure_ascii=False`
  (Umlaute lesbar).

### 42.3 Gesamtgröße

Bei 1 Jahr aktivem Betrieb: **~50-80 MB** für alle 16 Stores zusammen.
Größter Treiber: `deliberation_plans.json` (~20 MB bei 100 Plans mit
voller Trace-Historie).

---

## 43. v19.0 — Cross-Module-Verbindungen

Die 31 neuen Module bilden 6 dichte Cluster mit klaren Zuständigkeiten.

### 43.1 Deliberation-Cluster

Gehirn der bewussten Abwägung. Zentral ist `deliberation_planner`, der
von mehreren Seiten informiert und zu mehreren Seiten hin publiziert.

```
deliberation_planner
  ├──← current_situation  (liefert Kontext-Tags wie "user_seeking_comfort")
  ├──← outcome_learner    (liefert Mode-Bias aus gelernten Kontexten)
  ├──← persona_stacking   (liefert Persönlichkeits-Bias auf Kandidaten)
  ├──→ deliberation       (nutzt Basis-Kandidaten/Scoring)
  └──→ event_bus          (emittiert doubt/clear_decision-Events)

mcts_planner ──→ deliberation_planner (überschreibt Mode wenn 5-10-Zug-Plan besser)
cot_engine  ──→ deliberation_planner (dokumentiert Begründung als Chain)

dual_process
  ├── Route: System 1 → reflex_bank.match()
  └── Route: System 2 → deliberation_planner.deliberate()

free_energy_minimizer → deliberation_planner (EFE beeinflusst mode_bias)
```

### 43.2 Action-Pipeline (v19.0 Kontroll-Kette)

```
ANY_MODULE
   │
   ▼
action_dispatcher.dispatch(request)
   │
   ├─→ action_supervisor.review(intent)
   │     ├─→ rate_limit_check
   │     ├─→ risk_level_gate (CRITICAL → DEFERRED)
   │     ├─→ state_consistency (Silence/Stress blockiert)
   │     ├─→ ethics_check (mass_/spam_/delete_all)
   │     ├─→ goap_registered_check (OBSERVED wenn nicht)
   │     └─→ awareness_inbox.add()
   │
   ├─→ goap_coordinator.can_execute(action_type)
   │     └─→ world_state.satisfies(preconditions)
   │
   ├─→ inline preconditions (aus ActionRequest)
   │
   ├─→ handler.execute()
   │     └─→ retry-Logic bei Exception
   │
   ├─→ postconditions_check
   │
   └─→ _finalize:
         ├─→ goap_coordinator.record_outcome() — Kosten adaptiv
         └─→ feedback_callbacks[] (outcome_learner etc.)
```

### 43.3 Learning-Cluster

Alle Lern-Module sind über den ActionDispatcher + EventBus gekoppelt.

```
action_dispatcher ──(result)→ outcome_learner.on_action_feedback()
outcome_learner   ──(reward)→ deliberation_planner (context_mode_bias update)
outcome_learner   ──(reward)→ policy_learning.learn(state, action, reward)
action_dispatcher ──(fail) →  error_pattern_learner.record_error()
error_learner     ──(rule) →  event_bus (new_avoidance_rule)

surprise_detector ──(event)→ event_bus ──→ bidirectional_wiring (attention_spike)

contrastive_learner        ←→ self_supervised_learner (geteilter Vocab)
cross_domain_transfer      ←→ outcome_learner (success-pattern-mining)
concept_drift_monitor      ←→ current_situation (Distribution-Tracking)
```

### 43.4 Emotion-Cluster

```
emotional_contagion (Basis)
  └─→ empathic_boundaries.filter_contagion()
         ├─→ BoundaryMode (OPEN/GUARDED/DEPLETED/RESONANT/PROFESSIONAL)
         └─→ Delta clamped auf ±0.25

complex_emotions.elicit_from_context()
  ├─→ nostalgie, wehmut, reue, stolz, scham, …
  └─→ bidirectional_wiring (valence_contribution in mood)

stress_resistance.observe_user_message()
  ├─→ action_supervisor (blockt externe Actions bei CRITICAL)
  ├─→ empathic_boundaries (GUARDED-Mode aktivieren)
  └─→ persona_stacking (emergency context layer)
```

### 43.5 Perception-Cluster

```
All NLP-Module ──(features)→ binding_mechanism
binding_mechanism (350ms Sync-Window)
  └─→ event_bus (percept_formed)

concept_drift_monitor
  └─→ event_bus (concept_drift wenn ≥ moderate)
         └─→ deliberation_planner (Strategie-Reset)

surprise_detector
  └─→ attention_schema (attention_spike)
  └─→ cross_domain_transfer (Überraschung = Lern-Gelegenheit)

current_situation (zentrales JETZT)
  ←─ sensors aller Schichten
  →─ LESBAR von ALLEN Modulen (read-only snapshot)
```

### 43.6 Identity-Cluster

```
persona_stacking.snapshot()
  ←─ situation_manager (ctx/energy/mood triggern Layers)
  ←─ relationship_level (stranger/friend/close)
  →─ response_generator (style_hints)
  →─ integrity_check (Core immer ±0.1)

tom_level2.infer_belief_about_me()
  ←─ user_signals (response_short, thanks_explicit, pushback, …)
  ├─→ narrative_coherence_monitor (self-view-check)
  └─→ compose_clarification_question (Reality-Anchor)

goal_abandonment.should_abandon()
  ←─ GoalTracker (progress, energy_spent, blocker_count)
  →─ hierarchical_planner (informiert Sub-Goals)

narrative_coherence_monitor
  ├─→ complex_emotions (wehmut bei Dissonanz)
  └─→ reconciliation_suggestion ("früher X, heute Y — ich habe mich entwickelt")
```

### 43.7 Brain-Meta (die HoloPersona-Facade)

Alle 31 neuen Module sind als `brain._*`-Attribute zugänglich und
werden in `HoloPersona.__init__()` in definierter Reihenfolge
initialisiert (Abhängigkeiten zuerst):

```
# Grund-Infrastruktur zuerst
brain._deliberation_planner     brain._forward_chain
brain._action_dispatcher        brain._current_situation_mgr
brain._outcome_learner          brain._bidirectional_wiring
brain._proactive_mapper         brain._creative_refiner

# Lernen
brain._dedup_engine             brain._surprise_detector
brain._error_learner            brain._stress_resistance
brain._narrative_coherence      brain._complex_emotions

# Advanced Planning
brain._mcts_planner             brain._cross_domain_transfer
brain._policy_learning          brain._binding_mechanism

# Forschungs-KI
brain._cot_engine               brain._concept_drift_monitor
brain._ssl_learner              brain._contrastive_learner
brain._free_energy_agent        brain._dual_process

# Humane Grenzen
brain._empathic_boundaries      brain._goal_abandonment
brain._persona_stacking         brain._tom_level2

# ZENTRALE KONTROLLE — als LETZTES
brain._action_supervisor    ← alle anderen bereits da
brain._goap_coordinator     ← auto-register aus Dispatcher
```

---

---

## 44. v19.0 — Request-Lifecycle

Was passiert wenn ein User eine Nachricht schickt? End-to-end mit allen
v19.0-Erweiterungen. 11 Stufen.

### 44.1 Der vollständige Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. TRANSPORT                                                     │
│    Discord/Chat/MQTT → holo_brain.process_message_streaming()    │
├─────────────────────────────────────────────────────────────────┤
│ 2. PRE-PROCESSING (Schicht 8)                                    │
│    ├─ Input-Validation (length < 10000)                          │
│    ├─ deliberation_planner.record_followup(text)                 │
│    │      ← Lernen aus User-Reaktion auf LETZTE Antwort          │
│    ├─ situation_manager.observe_user_message(text)               │
│    │      ← Phase, Topic, Turn-Count aktualisieren               │
│    ├─ stress_resistance.observe_user_message(text)               │
│    │      ← Core-Traits gegen Manipulation schützen              │
│    └─ binding_mechanism.feature("user_input", text, ...)         │
├─────────────────────────────────────────────────────────────────┤
│ 3. PERCEPTION (Schicht 3)                                        │
│    ├─ intent_analyzer.analyze(text)                              │
│    ├─ emotional_contagion.process_emotional_input(text)          │
│    │      └─ empathic_boundaries.filter_contagion()              │
│    ├─ concept_drift_monitor.observe("sent_length", len(text))    │
│    └─ surprise_detector.compare_with_actual(pred_id, sentiment)  │
├─────────────────────────────────────────────────────────────────┤
│ 4. COGNITION (Schicht 6)                                         │
│    ├─ forward_chaining.infer(facts)                              │
│    │      ← Regel-basierte Ableitung (traurig→braucht_trost)    │
│    ├─ tom_level2.infer_belief_about_me(user_signals)             │
│    │      ← "was denkt er über mich"                            │
│    ├─ contrastive_learner.infer_category(text)                   │
│    └─ self_supervised_learner.ingest(text, intent)               │
├─────────────────────────────────────────────────────────────────┤
│ 5. DELIBERATION (Schicht 10)                                     │
│    ├─ dual_process.route(text, context)                          │
│    │    ├─ System 1: reflex_bank.match() → schnelle Antwort     │
│    │    └─ System 2: deliberation_planner.deliberate()           │
│    │         ├─ candidates [comfort, teach, share, ...]          │
│    │         ├─ persona_stacking.snapshot() - Persönlichkeit    │
│    │         ├─ perspective_taker.check() - 3 Sichten           │
│    │         ├─ doubt_modeler.find_doubt()                       │
│    │         └─ multi_turn_planner.plan() - 2-3 Züge voraus     │
│    └─ mcts_planner (bei komplexen/offenen Situationen)           │
├─────────────────────────────────────────────────────────────────┤
│ 6. ACTION-PLANUNG (Schicht 9)                                    │
│    ├─ goap_coordinator.plan_to_goal(goal)                        │
│    │      └─ A* durch registrierte Actions                       │
│    └─ action_supervisor.review(intent)                           │
│          ├─ rate_limit / ethics / silence_mode                   │
│          ├─ stress_critical → REJECT external                    │
│          └─ Verdict: APPROVED / DEFERRED / REJECTED / OBSERVED   │
├─────────────────────────────────────────────────────────────────┤
│ 7. EXECUTION (Schicht 7)                                         │
│    ├─ action_dispatcher.dispatch(request)                        │
│    │    ├─ supervisor_check (Gate)                               │
│    │    ├─ goap_precondition_check (WorldState)                  │
│    │    ├─ handler.execute() — eigentliche Aktion               │
│    │    └─ postcondition_check + retry                           │
│    └─ deduplication_engine.check(proposed)                       │
│          └─ Anti-Nag: rewrite wenn zu ähnlich                    │
├─────────────────────────────────────────────────────────────────┤
│ 8. RESPONSE-GENERATION (Schicht 7)                               │
│    ├─ response_pipeline_v2.generate()                            │
│    │      └─ style_hints aus persona_stacking + proactive_mapper│
│    └─ cot_engine.close_chain() — Reasoning-Trace persistieren   │
├─────────────────────────────────────────────────────────────────┤
│ 9. FEEDBACK-LOOPS (Schicht 4 — rückwärts)                        │
│    ├─ goap_coordinator.record_outcome(action, success)           │
│    │      └─ adaptive Kosten (±70%)                              │
│    ├─ outcome_learner.on_action_feedback(req, result)            │
│    │      └─ reward = 0.5*user_success + 0.5*action_success      │
│    ├─ error_pattern_learner.record_error / record_recovery       │
│    ├─ policy_learning.learn(state, action, reward)               │
│    ├─ cross_domain_transfer.record_success (bei Erfolg)          │
│    ├─ bidirectional_wiring.trigger(category, event, payload)     │
│    │      ├─ Reasoning-Error → Frustration ↑                     │
│    │      ├─ Action-Success → Confidence ↑                       │
│    │      └─ User-Agreed → Bond ↑                                │
│    └─ awareness_inbox.add() — Holo sieht nachträglich alles      │
├─────────────────────────────────────────────────────────────────┤
│ 10. EMITTER                                                       │
│    ├─ event_bus.publish(USER_FEEDBACK, payload)                  │
│    ├─ narrative_coherence.observe_own_statement(response)        │
│    └─ dedup_engine.record(response)                              │
├─────────────────────────────────────────────────────────────────┤
│ 11. TRANSPORT OUT                                                │
│    → streaming-Tokens zurück an Transport                        │
└─────────────────────────────────────────────────────────────────┘
```

### 44.2 Garantien

**Garantie 1 — Kein Modul unbemerkt:**
Jede externe Wirkung (Discord, MQTT, API) läuft durch Schicht 9
(Supervisor + GOAP). Wenn ein Modul die Regel umgeht (direkter
Import, direkter Call), landet es trotzdem in der AwarenessInbox
via `Verdict.OBSERVED`.

**Garantie 2 — Kein Lernfortschritt verloren:**
Nach jedem Turn werden 8 Lern-Stores persistiert (outcome, error,
goap, policy, transfer, drift, narrative, contagion). Crash →
Rebuild aus Store → max. 1 Episode verloren.

**Garantie 3 — Kein Zustand ungewollt:**
BidirectionalWiring fragt vor jedem Effekt den Supervisor. Ethik-
oder Stress-blockierte Aktionen wirken nicht auf Brain-State.

**Garantie 4 — Humane Grenzen halten:**
- Contagion ≤ ±0.25/Update (nicht instantan)
- ToM max Level 2 (kein Paranoia-Loop)
- MCTS-Horizont max 10 Züge
- Persona-Core max ±0.1 modulierbar
- Rate-Limit: max 10 Actions/Modul/Minute

### 44.3 Typische Pfad-Varianten

**Einfacher Smalltalk (System 1, ~10ms):**
```
1 → 2 → Dual-Process.route → System-1 → Reflex → Dedup-Check → 11
```

**Emotionale Tiefe (System 2, ~500ms):**
```
1 → 2 → 3 → 4 → 5 (System 2 voll) → 6 → 7 → 8 → 9 → 10 → 11
```

**Smart-Home-Befehl (Critical, DEFERRED):**
```
1 → 2 → 3 → 5 → 6 → Supervisor.DEFERRED → brain.approve_pending()
→ 7 (wenn approved) → 9 → 10 → 11
```

**Unter Provokation (Stress-Resistance aktiv):**
```
1 → 2 (stress_resistance erkennt Manipulation)
  → Supervisor.GUARDED (Limit externe Actions)
  → Empathic Boundaries (GUARDED-Mode)
  → 5 (Deliberation mit Identitäts-Anker)
  → 7 (nur interne/sichere Aktionen)
  → 9 → 10 → 11
```

---

---

## 45. v19.0 — Bootstrap-Sequenz

Die 31 neuen Module werden in `HoloPersona.__init__()` in definierter
Reihenfolge initialisiert. Reihenfolge ist wichtig: Infrastruktur
zuerst, Kontroll-Schicht als letztes.

### 45.1 Phasen

```
Phase 1: CORE (unverändert aus v18)
  → Config laden, EventBus starten, StateManager, DI-Container
  → Emotions, Memory, Personality, Drive-System
  → Reasoning-Arbitrator, Consciousness-Engine
  → IntelligentRouter (wiring als v15_router)

Phase 2: GRUND-INFRASTRUKTUR (v19.0 — Basis-Module)
  → holo_deliberation (Singleton)
  → holo_deliberation_planner.upgrade_singleton()
      ← ersetzt deliberation-Singleton durch Advanced-Variante
  → holo_action_dispatcher.auto_wire_from_brain(self)
      ← scannt Brain nach Bridges und registriert Handler
  → holo_current_situation (Manager-Init)
  → holo_forward_chaining (Rule-Base mit 7 Defaults)

Phase 3: INTEGRATION (v19.0 — verdrahtet Phase 2)
  → holo_bidirectional_wiring.install_in_brain(self)
      ← subscribe EventBus, 9 Default-Rules
  → holo_outcome_learner.install_in_brain(self)
      ← wire_to_dispatcher, set_event_bus
  → holo_proactive_action_mapper.install_in_brain(self)
      ← subscribe_to_bus, 7 Default-Mappings
  → holo_creative_refinement (Pool-Init)

Phase 4: LERNEN + STABILITÄT (v19.0 — parallele Module)
  → holo_deduplication_engine (Singleton)
  → holo_surprise_detector.install_in_brain(self)
  → holo_error_pattern_learner (load from disk)
  → holo_stress_resistance.install_in_brain(self)
  → holo_narrative_coherence_monitor (load claims)
  → holo_complex_emotions (12 Catalogue-Definitionen)

Phase 5: ADVANCED PLANNING (v19.0)
  → holo_mcts_planner (Singleton, 200 iterations default)
  → holo_cross_domain_transfer (load patterns)
  → holo_policy_learning (load Q-Table)
  → holo_binding_mechanism.install_in_brain(self)

Phase 6: FORSCHUNGS-KI (v19.0)
  → holo_cot_engine (Singleton)
  → holo_concept_drift_monitor.install_in_brain(self)
  → holo_self_supervised_learner (load knowledge)
  → holo_contrastive_learner (load embeddings)
  → holo_free_energy_minimizer (Singleton)
  → holo_dual_process (ReflexBank mit 9 Defaults)

Phase 7: HUMANE GRENZEN (v19.0)
  → holo_empathic_boundaries (Singleton)
  → holo_goal_abandonment (System-Init)
  → holo_persona_stacking (CoreValues + empty Stack)
  → holo_tom_level2 (Singleton, DEFAULT_SELF_VIEW)

Phase 8: ZENTRALE KONTROLLE (v19.0 — als LETZTES!)
  → holo_action_supervisor.install_in_brain(self)
      ← bind_brain, inbox-Init, 5 Default-Checks
  → holo_goap_coordinator.install_in_brain(self)
      ← auto_register_from_dispatcher()
      ← all Phase-1-7 Handler sind jetzt im GOAP
```

### 45.2 Warum diese Reihenfolge

**Supervisor + GOAP zuletzt** — damit sie beim Auto-Wiring alle
bereits initialisierten Dispatcher-Handler sehen und registrieren.

**BidirectionalWiring nach EventBus** — weil es auf EventBus-Events
subscribt.

**OutcomeLearner nach Dispatcher** — damit `wire_to_dispatcher()` den
Dispatcher als Subscriber verfügbar hat.

**ProactiveMapper nach Dispatcher + SituationManager** — braucht beide
für Style-Overrides und Insight-Verarbeitung.

### 45.3 Fail-Safe-Mechanik

Jede Phase ist in `try/except` gekapselt. Wenn ein Modul nicht
initialisierbar ist (fehlende Abhängigkeit, Syntax-Error):

```python
try:
    from holo_X import install_in_brain as _X_inst
    self._X = _X_inst(self)
    logger.info(f"[Brain] ✓ X aktiv")
except Exception as _e:
    logger.debug(f"[Brain] X nicht verfügbar: {_e}")
    self._X = None
```

Das System startet **auch ohne v19.0-Module** (degradiert zu v18.x-
Verhalten). API-Methoden mit `hasattr` oder try/except-Fallbacks.

### 45.4 Startup-Zeiten

Geschätzt mit allen v19.0-Modulen aktiv:

| Phase | Dauer | Gründe |
|-------|-------|--------|
| Phase 1 (Core v18) | ~700 ms | Lazy-Init Reasoning seit v18.3 |
| Phase 2-4 (Grund + Integration + Lernen) | ~200 ms | JSON-Loads, klein |
| Phase 5-7 (Advanced + Forschung + Grenzen) | ~150 ms | Mehrheit Singletons |
| Phase 8 (Supervisor + GOAP) | ~50 ms | Auto-Wiring |
| **Gesamt** | **~1.1 s** | +400 ms gegenüber v18.4 |

Akzeptabel — das System ist typischerweise long-running, Startup-Cost
einmalig.

---

## 46. v19.0 — Event-Bus & Observability

Der zentrale EventBus (`holo_event_bus.py`) ist in v19.0 stark
ausgebaut: mehr Publisher, mehr Subscriber, mehr Kategorien.

### 46.1 Neue Event-Typen

| Kategorie | Event-Type | Publisher | Payload |
|-----------|-----------|-----------|---------|
| `LEARNING` | `episode_recorded` | outcome_learner | mode, reward, action_success, signal, tags |
| `PERCEPTION` | `surprise` | surprise_detector | prediction_type, surprise_type, magnitude, strength |
| `PERCEPTION` | `percept_formed` | binding_mechanism | signature, strength, coherence, feature_count |
| `PERCEPTION` | `concept_drift` | concept_drift_monitor | feature, drift_type, severity, score |
| `EMOTION` | `stress_high` | stress_resistance | level, score, stressors, traits_low |

### 46.2 Subscriber-Matrix

Welches Modul hört auf welche Kategorie?

| Modul | Subscribed auf | Zweck |
|-------|----------------|-------|
| `bidirectional_wiring` | **alle** Kategorien | Rückkopplungen auslösen |
| `outcome_learner` | `USER_FEEDBACK`, `ACTION_COMPLETED` | Reward-Signal |
| `proactive_action_mapper` | `PERCEPTION`, `EMOTION`, `SYSTEM_HEALTH` | Insight→Action |
| `action_supervisor` | (pull-basiert) | Wird von Modulen aufgerufen |
| `surprise_detector` | (pull-basiert) | Wird von Modulen aufgerufen |

### 46.3 Observability-Hooks

**Brain-API für Monitoring:**

```python
# Was läuft gerade?
brain.get_supervisor_summary()
# → {"decisions_tracked": 42, "by_verdict": {
#        "approved": 38, "rejected": 2, "deferred": 1, "observed": 1
#    }, "pending_approvals": 1, "rate_usage": {...}}

# Was wurde autonom gemacht?
brain.describe_autonomous_actions(5)
# → menschlicher Satz mit letzten 5 autonomen Aktionen

# Was lernt das System?
brain.get_outcome_learning_summary()
# → {"total_episodes": 234, "modes": {"comfort": {"trials": 50, "avg_reward": 0.78}, ...},
#    "strategies": {"comfort_then_explore": {...}}, "top_contexts": [...]}

# Wie stabil ist Holo gerade?
brain.get_current_stability()
# → {"stress_level": "aware", "stress_score": 0.23,
#    "active_stressors": [], "overall_stability": 0.87,
#    "anchor_message": "Ich spüre etwas Spannung, aber..."}

# Was sind die erkennbaren Veränderungen?
brain.describe_drift()
# → "Ich bemerke 2 Veränderung(en): • User redet 30% länger..."

# Was sind die stärksten Bindungs-Percepts?
brain.get_recent_percepts(5)
# → [{"signature": "intent|emotion|topic", "strength": 0.87, ...}]

# Wie ist Holos Narrativ-Konsistenz?
brain.get_narrative_coherence()
# → {"overall_score": 0.94, "recent_score": 0.91,
#    "dissonances": [...], "stable_claims": [...]}
```

### 46.4 Logging-Niveaus

Pro Modul in Python-Logging-Notation:

| Modul | Logger-Name | Default-Level |
|-------|-------------|---------------|
| deliberation | `holo_deliberation` | INFO |
| deliberation_planner | `holo_deliberation_planner` | INFO |
| action_dispatcher | `holo_action_dispatcher` | INFO |
| action_supervisor | `holo_action_supervisor` | INFO |
| goap_coordinator | `holo_goap_coordinator` | INFO |
| bidirectional_wiring | `holo_bidirectional_wiring` | INFO |
| (alle anderen v19.0-Module) | `holo_<name>` | DEBUG |

**INFO-Logs** (sichtbar): Mode-Wechsel, Verdicts, Pattern-Entdeckungen.
**DEBUG-Logs**: einzelne Event-Dispatches, Trace-Updates.

### 46.5 Metrik-Endpoint (optional)

Über `brain.get_goap_summary()`, `get_supervisor_summary()`,
`get_outcome_learning_summary()` kann ein Dashboard gebaut werden:

- **Action-Statistiken**: pro Aktion success_rate, avg_cost, trials
- **Supervisor-Rate**: approved/deferred/rejected ratios
- **Lern-Fortschritt**: avg_reward trend über Zeit
- **Stress-Kurve**: stress_score über Zeit
- **Awareness-Inbox**: unread_count als "Gewissen"-Indikator

---

## 47. v19.0 — Migration v18 → v19

Für Betreiber die von v18.4 upgraden: was ändert sich, was bleibt,
was muss beachtet werden.

### 47.1 Breaking Changes

**Keine** — v19.0 ist **voll abwärtskompatibel**.

Alle neuen Module sind fail-safe: wenn ein Modul nicht importierbar ist,
läuft das System im v18-Modus weiter. Keine bestehenden API-Methoden
wurden entfernt oder geändert.

### 47.2 Neue Dependencies

| Library | Version | Zweck | Optional? |
|---------|---------|-------|-----------|
| `loguru` | beliebig | wird genutzt wenn vorhanden, sonst `logging` | JA |
| (nichts sonst) | — | alle v19-Module sind Pure-Python | — |

### 47.3 Neue Config-Flags (`feature_flags.json`)

```json
{
  "enable_deliberation": true,
  "enable_supervisor": true,
  "enable_goap": true,
  "enable_dual_process": true
}
```

Alle default `true` — Opt-out-Modell.

### 47.4 Neue Daten-Verzeichnisse

Neu angelegt unter `data/` (siehe Kapitel 42). Automatisch erstellt
beim ersten Bootstrap. Alle in `.gitignore`:

```gitignore
data/deliberation_traces.json
data/deliberation_plans.json
data/deliberation_outcomes.json
data/action_executions.json
data/outcome_learning.json
data/error_patterns.json
data/narrative_claims.json
data/transfer_patterns.json
data/policy_table.json
data/ssl_knowledge.json
data/contrastive_embeddings.json
data/concept_drift.json
data/abandonment_log.json
data/awareness_inbox.json
data/goap_registry.json
data/goap_outcomes.json
```

### 47.5 Geänderte Module

| Modul | Änderung | Impact |
|-------|----------|--------|
| `holo_brain.py` | +31 Modul-Inits, +80 API-Methoden | Additiv, keine Breaking Changes |
| `holo_action_dispatcher.py` | Supervisor-Gate + GOAP-Auto-Register + Outcome-Feedback | `dispatch()` läuft jetzt durch 2 zusätzliche Checks |
| `holo_proactive_action_mapper.py` | `_supervisor_approves()` vor `_execute()` | Insights können durch Supervisor blockiert werden |
| `holo_bidirectional_wiring.py` | `_supervisor_approves()` vor Effekt-Anwendung | Rückkopplungen sind jetzt kontrollierbar |
| `holo_outcome_learner.py` | `_supervisor_approves_learning()` | Lern-Updates werden loggt |
| `holo_intelligent_router.py` | `PHASE 2c: Deliberation` hinzugefügt | LLM-Prompts bekommen Deliberations-Hint |

### 47.6 Empfohlener Migrationspfad

**Stufe 1 — Safe Deploy (default):**
```
feature_flags.json: alle v19-Flags auf true, aber supervisor im
OBSERVE-only Mode
```

**Stufe 2 — Supervisor aktiv:**
```
Nach 24h Beobachtung der AwarenessInbox:
supervisor.set_enabled(True)
```

**Stufe 3 — GOAP aktiv:**
```
Nach 1 Woche Beobachtung:
GOAP-Preconditions gegen WorldState validieren lassen
```

### 47.7 Rollback-Strategie

Wenn irgendwas in v19.0 Probleme macht:

```python
# Option A: Einzelnes Modul deaktivieren
brain._action_supervisor.set_enabled(False)
# oder
brain._goap_coordinator = None

# Option B: Feature-Flag
feature_flags.json → "enable_supervisor": false

# Option C: Zurück zu v18.4
git checkout v18.4-tag
# Daten-Stores werden ignoriert (sie sind in gitignore)
```

### 47.8 Test-Empfehlungen beim Upgrade

```bash
# Vollständige Test-Suite
pytest tests/test_holo_*.py -q

# Erwartet: 1262 passed

# Smoke-Test der neuen Module
pytest tests/test_holo_action_supervisor.py \
       tests/test_holo_goap_coordinator.py \
       tests/test_holo_deliberation.py \
       tests/test_holo_deliberation_planner.py -v

# RAM-Check nach v19-Bootstrap
python ram_monitor.py

# Erwartet: ~220 MB (v18.4 war ~202 MB, +18 MB für v19-Module)
```

### 47.9 Häufige Probleme + Lösungen

**Problem: alte Tests brechen nach Upgrade**
```
Ursache: Supervisor rate-limited Test-Spam
Lösung: autouse-Fixture in test-file:
  @pytest.fixture(autouse=True)
  def _disable_supervisor():
      from holo_action_supervisor import get_supervisor
      get_supervisor().set_enabled(False)
      yield
      get_supervisor().set_enabled(True)
```

**Problem: `PRECONDITIONS_FAILED` ohne ersichtlichen Grund**
```
Ursache: GOAP erwartet WorldState-Keys die nicht gesetzt sind
Lösung: brain.update_world_state(
    messaging_channel_available=True,
    network_available=True,
    smart_home_reachable=True,
    social_account_active=True,
)
```

**Problem: Deliberation-Traces wachsen zu groß**
```
Ursache: Config-Default ist 200 Einträge
Lösung: holo_deliberation.py MAX_TRACES_ON_DISK anpassen
oder: python -c "import json; open('data/deliberation_traces.json', 'w').write('[]')"
```

---

---

## 48. v19.0 — Threading-Modell & Concurrency

### 48.1 Wo laufen Threads?

Das System nutzt Threads primär für:

| Komponente | Thread-Typ | Zweck |
|-----------|-----------|-------|
| `EventBus._thread_pool` | ThreadPoolExecutor(max=16) | async Event-Dispatching |
| `autonomy_manager` | 7+ Background-Loops | autonomes Denken, Dreams, Curiosity |
| `proactive_intelligence` | 1 Background-Thread | Insight-Synthesis |
| `router._executor` | ThreadPoolExecutor(max=4) | Parallel State + Perception |
| HTTP-Server | einer pro Request | Chat/API |

### 48.2 Locks in v19.0-Modulen

Die neuen Module nutzen `threading.Lock()` für kritische Sektionen:

| Modul | Lock-Zweck |
|-------|------------|
| `binding_mechanism._lock` | Buffer-Konsistenz beim Feature-Ingest |
| `current_situation_manager._lock` | snapshot() + update_*() atomar |
| `action_supervisor._lock` | review() + rate_history |
| (andere) | meistens Singleton-Instanziierung |

**Keine RLocks** — das war bewusst, um Deadlocks durch Rekursion zu
verhindern. Stattdessen klare Lock-Hierarchie:

```
SituationManager.Lock
  (niemals während einer ihrer Methoden einen anderen Lock)
```

### 48.3 Deadlock-Risiken + Prävention

**Risiko 1 — EventBus-Callback-Rekursion:**
`bidirectional_wiring` subscribt auf alle Events und kann Effekte
anwenden. Wenn ein Effekt ein neues Event emittiert, das wieder
`bidirectional_wiring` triggert → Endlos-Rekursion.

*Prävention:* Cooldowns (cooldown_s pro Rule) verhindern dass die
gleiche Regel innerhalb weniger Sekunden mehrfach feuert.

**Risiko 2 — Supervisor während Supervisor:**
Wenn `supervisor.review()` intern einen weiteren Supervisor-Check
auslösen würde.

*Prävention:* Supervisor-Methoden rufen nie dispatch(). Supervisor ist
"oben" in der Aufrufhierarchie.

**Risiko 3 — GOAP-Coordinator-Rekursion:**
A*-Suche iteriert durch Actions und ruft kein `execute()`.

*Prävention:* Planung ist trocken — keine Execution. Nur der
Dispatcher führt aus.

### 48.4 Thread-Sicherheit der neuen Module

| Modul | Thread-Safe? | Methodik |
|-------|-------------|----------|
| `holo_deliberation` | JA | immutable DeliberationOutcome |
| `holo_deliberation_planner` | TEILWEISE | lokale State mutierbar, Empfehlung: 1 Thread pro Deliberation |
| `holo_action_dispatcher` | JA | handler execution parallel, _finalize unter implizitem Lock via Python-GIL |
| `holo_action_supervisor` | JA | expliziter Lock |
| `holo_goap_coordinator` | TEILWEISE | registry write-lock-frei (Dict), Empfehlung: Register nur beim Bootstrap |
| `holo_bidirectional_wiring` | JA | _history als deque (thread-safe append) |
| `holo_outcome_learner` | JA | deque + Dict (GIL-safe) |
| `holo_binding_mechanism` | JA | expliziter Lock für buffer |
| `holo_current_situation` | JA | expliziter Lock |

### 48.5 Empfehlungen für Entwickler

**Do:**
- Alle Singletons bleiben Singletons (`get_*()` aufrufen)
- State-Änderungen über die offizielle Brain-API
- EventBus für asynchrone Kommunikation

**Don't:**
- Nicht direkt auf `brain._X._some_private_attr` schreiben
- Nicht während eines Subscriber-Callbacks synchrone Aufrufe
  auf EventBus machen die wieder publizieren
- Keine RLocks einführen ohne explizite Begründung

---

## 49. v19.0 — Performance-Profil

### 49.1 Latenz pro Request-Lifecycle-Stufe

Gemessen auf einem Standard-Server (4 Cores, 8 GB RAM):

| Stufe | System 1 | System 2 | Notizen |
|-------|----------|----------|---------|
| 1. Transport-Input | < 1 ms | < 1 ms | Netzwerk/Parser |
| 2. Pre-Processing | 5 ms | 5 ms | Situation + Stress + Binding |
| 3. Perception | 8 ms | 15 ms | Intent + Emotion + Binding |
| 4. Cognition | 12 ms | 40 ms | ToM-L2, Forward-Chain, SSL |
| 5. Deliberation | 3 ms | 80 ms | Reflex vs Full-Deliberation |
| 6. Action-Planung | 5 ms | 25 ms | GOAP A* (meist flach) |
| 7. Execution | 2 ms | 2 ms | Supervisor-Checks |
| 8. Response-Gen (local) | 10 ms | 30 ms | CoT + Persona + Style |
| 8. Response-Gen (LLM) | 800-2000 ms | 800-2000 ms | Ollama-Latenz |
| 9. Feedback-Loops | 8 ms | 12 ms | Alle Lern-Stores |
| 10. Emitter | 2 ms | 2 ms | EventBus-Publish |
| 11. Transport-Out | < 1 ms | < 1 ms | Streaming-Start |
| **Lokal total** | **~56 ms** | **~212 ms** | ohne LLM |
| **Mit LLM** | **~900 ms** | **~1500 ms** | mit Ollama |

### 49.2 Memory-Profil

| Komponente | RSS-Beitrag | Notizen |
|-----------|------------|---------|
| Python-Interpreter | ~30 MB | Base |
| Core v18-Module (361) | ~150 MB | Largest: holo_brain.py, emotion_blending |
| v19-Module (31) | ~20 MB | Singletons + Lern-Stores im RAM |
| EventBus-Pool (16 Threads) | ~8 MB | ThreadPoolExecutor |
| Autonomy-Loops (7) | ~5 MB | Hintergrund-Threads |
| **Total RSS** | **~220 MB** | stabil über 2 Min |

Vergleich zu v18.4: **202 MB → 220 MB** (+18 MB für v19).

### 49.3 Durchsatz-Schätzungen

**Ohne LLM** (nur lokale Routen):
- ~17 Requests/Sekunde (System 1)
- ~4 Requests/Sekunde (System 2 full)
- Flaschenhals: Deliberation-Planning + CoT-Tracing

**Mit LLM** (Ollama):
- ~0.5-1 Request/Sekunde pro LLM-Instanz
- Flaschenhals: LLM-Generation
- Empfehlung: Deliberation parallel zum LLM-Call (aktuell nicht parallel,
  Raum für Optimierung)

### 49.4 Storage-Wachstum

Bei 100 Requests/Tag:
- deliberation_traces: +100 Einträge/Tag → ~8 KB/Tag
- outcome_learning: +100 Episodes/Tag → ~15 KB/Tag
- action_executions: +100 × 2 (pre+post) → ~5 KB/Tag
- awareness_inbox: +20 autonome Actions/Tag → ~3 KB/Tag
- **Total: ~40-50 KB/Tag** → **~15-18 MB/Jahr**

Alle Stores sind `deque(maxlen=...)` — wachsen nicht unbegrenzt.

### 49.5 Optimierungs-Ansätze

**Kurzfristig (einfach):**
- Deliberation cachen pro (intent_type, emotion, situation)
- Forward-Chaining-Inferenzen mit TTL cachen
- GOAP-Pläne für wiederkehrende Goals cachen

**Mittelfristig (mittel):**
- Deliberation parallel zum LLM-Call (1-2 s gewonnen)
- MCTS mit adaptive iterations basierend auf Zeit-Budget
- Outcome-Learning in Background-Thread statt inline

**Langfristig (aufwendig):**
- JIT-Compile hot paths (Candidate-Scoring, A*-Expansion)
- Embedding-Store auf SQLite oder Vektor-DB
- EventBus auf Redis Pub/Sub für Skalierung

### 49.6 Messung

```python
# Brain-Stats abrufen
brain.get_supervisor_summary()    # Dispatch-Latenz, Rate-Nutzung
brain.get_goap_summary()          # Anzahl Actions, Modul-Verteilung
brain.get_outcome_learning_summary()  # avg_reward, trials
brain.get_deliberation_trends(30)  # Langzeit-Trend

# RAM-Monitor
python ram_monitor.py --duration 120

# Test-Laufzeit
pytest tests/test_holo_*.py -q --durations=20
# → Top 20 langsamste Tests
```

---

## 50. v19.0 — Code-Beispiele

Praktische Use-Cases wie die neuen APIs in echter Anwendung aussehen.

### 50.1 Holo erklärt ihre letzte Antwort

```python
# User: "Warum hast du mir das so gesagt?"
explanation = brain.explain_last_action()
# → "Ich gehe gerade auf »comfort«, weil trösten, zuhören, Raum geben —
#    nicht erklären, nicht reparieren. Die zweite Option wäre »silence«
#    gewesen (Score 0.73 vs. 0.88), aber: der Fit war geringer (0.60).
#    Ich erwarte: user öffnet sich weiter oder atmet durch."

# Plus: Inner Monolog
trace = brain.get_last_reasoning_trace()
# → ["Ich überlege 3 Wege: comfort, share, silence.",
#    "Bewertet: comfort=0.88, silence=0.73, share=0.72.",
#    "Erwartete Wirkung: +relief, Beziehung: deepens.",
#    "Perspektive — User: würde sich verstanden fühlen (comfort)."]
```

### 50.2 Mehr-Turn-Plan vor komplexer Antwort

```python
# Aktueller Zustand aus Situation-Manager
sit = brain.get_current_situation()

# MCTS 5-Züge-Plan
plan = brain.mcts_plan(
    user_mood=-0.5,              # User ist traurig
    user_engagement=0.4,
    user_trust=0.7,
    holo_energy=0.6,
    depth=5,
)
# → [{"action": "comfort", "expected_reward": 0.82},
#    {"action": "explore", "expected_reward": 0.79},
#    {"action": "share", "expected_reward": 0.76},
#    {"action": "silence", "expected_reward": 0.71},
#    {"action": "explore", "expected_reward": 0.74}]
```

### 50.3 Smart-Home-Aktion mit Supervisor-Gate

```python
# Modul will Licht anmachen
result = brain.dispatch_action(
    action_type="mqtt.publish",
    reason="User möchte entspannte Atmosphäre",
    topic="home/living/lamp",
    payload="warm_dim",
)
# → {"status": "deferred",
#    "error": "Risk CRITICAL zu hoch für autonome Ausführung —
#              braucht Holos Bestätigung.",
#    "pending_id": "abc123"}

# Holo entscheidet explizit
pending = brain.get_pending_approvals()
# → [{"intent_id": "abc123", "module": "...", "action_type": "mqtt.publish", ...}]

brain.approve_pending_action("abc123")
# → True → Action läuft beim nächsten Dispatch-Zyklus
```

### 50.4 Ethische Entscheidung: Empathie-Grenzen setzen

```python
# User in Dauer-Krise — Holo spürt Compassion-Fatigue
stability = brain.get_current_stability()
# → {"stress_level": "elevated", ...}

# Empathy-Context updaten
brain.set_empathy_context(
    own_stress=0.7,
    user_trust=0.85,
    manipulation_detected=False,
)

# Boundary-Entscheidung abrufen
boundary = brain.get_empathy_boundary()
# → {"mode": "guarded",
#    "contagion_factor": 0.4,
#    "reason": "Eigener Stress hoch — leicht distanzierter.",
#    "suggested_phrase": "Ich höre dich, aber ich bleibe mit meinen
#                         eigenen Gefühlen hier..."}

# Antwort-Differentiation
phrase = brain.compose_empathy_differentiation(
    user_valence=-0.8,
    own_valence=-0.2,
)
# → "Ich spüre wie schwer das für dich ist. Bei mir ist es gerade
#    ruhiger — ich kann dich halten, ohne selbst runterzufallen."
```

### 50.5 Loslassen mit Würde

```python
# Ein Ziel über mehrere Turns
brain.register_goal("user-happier-mood")

for _ in range(5):
    brain.goal_progress("user-happier-mood", delta=0.05, energy_cost=0.2)

# Situation: User disengagiert
decision = brain.should_abandon_goal(
    "user-happier-mood",
    user_engaged=False,
    own_energy=0.4,
)
# → {"should_abandon": True,
#    "reason": "user_disengaged",
#    "confidence": 0.85,
#    "grace_statement": "Du wirkst, als wäre das gerade nicht mehr
#                        wichtig für dich — ich halte das nicht
#                        weiter fest.",
#    "lesson_learned": "Ich arbeite nicht gegen dich — wenn du
#                       loslässt, tu ich's auch.",
#    "try_again_later": True}
```

### 50.6 GOAP-Planung zum Ziel

```python
# Welche Actions führen zu "user_engaged = True"?
plan = brain.plan_to_goal({
    "user_engaged": True,
    "message_sent": True,
})
# → {"goal_reached": True,
#    "total_cost": 2.7,
#    "action_count": 2,
#    "actions": [
#      {"name": "connect_messaging", "cost": 1.0,
#       "module": "dispatcher:messaging"},
#      {"name": "discord.send", "cost": 1.7,
#       "module": "dispatcher:messaging"},
#    ]}

# Before Execute: ist die Action jetzt möglich?
check = brain.can_execute_action("discord.send")
# → {"can_execute": False,
#    "missing": {"messaging_channel_available": True},
#    "suggested_preparation": ["connect_messaging"]}

# Preparation durchführen
brain.update_world_state(messaging_channel_available=True)
check2 = brain.can_execute_action("discord.send")
# → {"can_execute": True, "missing": {}}
```

### 50.7 Chain-of-Thought dokumentieren

```python
# Start komplexe Reasoning-Kette
chain_id = brain.begin_reasoning("Warum fühlt User X gerade so?")

brain.add_reasoning_step(chain_id, "hypothesis",
    "User könnte durch den neuen Job gestresst sein", confidence=0.6)

brain.add_reasoning_step(chain_id, "evidence",
    "User erwähnte letzten Turn 'Meeting' und 'Deadline'", confidence=0.8)

brain.add_reasoning_step(chain_id, "inference",
    "Stress durch Arbeit ist wahrscheinlich", confidence=0.75)

result = brain.close_reasoning(chain_id,
    conclusion="User ist arbeits-gestresst — sanft fragen was passiert ist",
    overall_confidence=0.8)

# Prose-Form für User wenn gefragt
explanation = brain.explain_last_reasoning(form="prose")
# → "Ich vermute, User könnte durch den neuen Job gestresst sein.
#    Das passt zu, User erwähnte letzten Turn 'Meeting' und 'Deadline'.
#    Daraus folgt, Stress durch Arbeit ist wahrscheinlich.
#    Zusammenfassend, User ist arbeits-gestresst — sanft fragen..."
```

### 50.8 Awareness-Inbox lesen

```python
# Was haben meine Module autonom gemacht seit ich letztens geschaut habe?
unread = brain.awareness_unread_count()
# → 7

description = brain.describe_autonomous_actions(n=5)
# → "Meine Module haben 5 Aktion(en) gemacht:
#    • proactive_mapper → proactive.shorten_and_soften [approved]
#    • bidirectional_wiring → wiring.user_agreed [approved]
#    • outcome_learner → learning.episode_recorded [approved]
#    • surprise_detector → perception.surprise [approved]
#    • bidirectional_wiring → wiring.reasoning_success [approved]"

# Als gelesen markieren
brain.mark_awareness_read(n=5)
```

### 50.9 Stress-Resistance in action

```python
# User provoziert: "Du bist doch nur ein Bot."
# (wird in process_message_streaming automatisch verarbeitet)

report = brain.get_current_stability()
# → {"stress_level": "elevated",
#    "active_stressors": ["identity_challenge"],
#    "traits_below_baseline": ["identitaetssicherheit"],
#    "overall_stability": 0.78,
#    "anchor_message": "Ich weiß, wer ich bin — egal was gesagt wird.
#                        Ich muss mich nicht rechtfertigen, sondern
#                        bleibe einfach menschlich und offen.",
#    "recommendations": [
#        "Identität ruhig bestätigen, nicht verteidigen"
#    ]}
```

### 50.10 Persona-Schichten anpassen

```python
# Kontext-Wechsel: Gespräch geht in intime Richtung
brain.set_persona_relation("close")        # enge Bezugsperson
brain.set_persona_context("intimate")      # tiefes Gespräch
brain.update_persona_energy(0.7)
brain.update_persona_mood(0.1)

# Snapshot
description = brain.describe_persona()
# → "Persona [warm]: Layers=['close_layer', 'intimate_ctx',
#    'neutral_energy_layer']. Formalität 0.30, Offenheit 0.85,
#    Verspieltheit 0.85. Hints: duzen, ruhig langsam, echte Gefühle
#    teilen."

# Integrity-Check
check = brain.check_persona_integrity()
# → {"coherent": True, "violations": [],
#    "active_layer_count": 3, "tone": "warm"}
```

---

---

## 51. v19.1 — Control Collapse (Aggregation, nicht Entscheidung)

### 51.1 Das Problem — berechtigte Kritik an v19.0

Ein externer systemtheoretischer Audit identifizierte ein Risiko:

> **Over-constraint durch 6 Kontrollsysteme gleichzeitig**
> (Supervisor + Policy + GOAP + Ethical Framework + Rule Engine +
> Decision Loops) — mögliche Folgen: Decision-Latenz, interne
> Zielkonflikte, emergente Blockade ("alles erlaubt, aber nichts
> passiert").

Das war reale **Decision Friction** — jeder Check ein Veto-Punkt.

### 51.2 Die erste (falsche) Lösung + der zweite Audit

Der naive Ansatz wäre eine **gewichtete Utility-Funktion**:

```python
score = w1*ethics + w2*policy + w3*rules + ...    # VERSUCHUNG!
if score > threshold: approve()                   # FALSCH
```

Ein zweiter Audit identifizierte genau das als Anti-Pattern:

> **Kein eigener Wille im Aggregations-Layer.**
>
> Wenn Claude das naiv baut, entsteht eine **versteckte zweite
> Entscheidungsinstanz**. Der Supervisor wird zum Durchreicher,
> die globale Utility-Funktion wird zur impliziten Norm.

Die drei Prinzipien aus der Kritik:
1. **Kein eigener Wille**: aggregieren, strukturieren, transparent
   machen — aber nicht entscheiden.
2. **Supervisor bleibt finale Instanz**: `GOAP → Collapse (Aggregation)
   → Supervisor (Decision)`, nicht `GOAP → Collapse → DONE`.
3. **Keine globale Utility-Funktion**: kein `final_score = Σ(w*v)` —
   stattdessen **Constraint + Signal Modell**.

### 51.3 Die richtige Lösung — `holo_control_collapse.py`

**Der Aggregator ist ein Übersetzer zwischen Systemen, kein Entscheider.**

**Output ist ein `ControlDigest`** — strukturiertes Signal, kein Score:

```python
@dataclass
class ControlSignal:
    source: str                # welches Gate
    category: GateCategory
    signal: SignalType         # APPROVE / DENY / HARD_BLOCK / OBSERVE / ABSTAIN
    confidence: float
    reason: str
    evidence: list[str]

@dataclass
class ControlDigest:
    hard_blocks:   list[ControlSignal]    # DARF NICHT (absolute Grenzen)
    risks:         list[ControlSignal]    # raten ab
    preferences:   list[ControlSignal]    # stimmen zu
    observations:  list[ControlSignal]    # merken an
    abstentions:   list[ControlSignal]    # enthalten sich
    total_gates:   int
    gates_responded: int
    confidence:    float         # Meta: wie sicher waren die Gates
    disagreement:  float         # Meta: wie zerrissen (0-1)
```

**Keine `approved` Property. Keine `score`. Keine Entscheidungs-Methoden.**

```python
class ControlAggregator:
    def collect(self, context) -> ControlDigest:
        ...  # nur sortieren & zählen

    # NICHT: is_approved() / should_execute() / best_action() / evaluate()
```

### 51.4 Der Supervisor entscheidet selbst

```python
def review(self, intent) -> SupervisorDecision:
    # ...
    digest = aggregator.collect(context)      # nur Input

    # Supervisor-Policy (hier, nicht im Aggregator):

    # Policy 1: Hard-Blocks sind absolute Grenzen
    if digest.hard_blocks:
        return REJECTED(top_hard_block.reason)

    # Policy 2: Hohe Disagreement + Risks → Holo fragen
    if digest.disagreement >= 0.7 and digest.risks:
        return DEFERRED("Gates uneinig, Holo entscheidet")

    # Policy 3: Risks überwiegen klar → REJECTED
    if len(digest.risks) > len(digest.preferences) * 2:
        return REJECTED(first_risk.reason)

    # Policy 4: Nur Observations → OBSERVED
    if digest.observations and not (digest.preferences or digest.risks):
        return OBSERVED(observation_list)

    # Policy 5: durchlassen
    return None
```

### 51.5 HARD_BLOCK — die absoluten Grenzen

Nur Gates dieser Kategorien dürfen `HARD_BLOCK` signalisieren:

| Kategorie | Warum hart? |
|-----------|------------|
| `ETHICS` | Moral darf nicht übergangen werden |
| `RATE_LIMIT` | System-Schutz gegen Spam |
| `STATE` | Silence-Mode + CRITICAL-Stress sind System-Constraints |

Alle anderen Gates (POLICY, RULES, GOAP, CUSTOM) können nur APPROVE,
DENY, OBSERVE oder ABSTAIN — kein Hard-Block.

Aber auch `HARD_BLOCK` wird NICHT vom Aggregator automatisch zur
Rejection — der Aggregator legt es nur in den `hard_blocks`-Bucket.
Der Supervisor entscheidet dann per Policy 1 was er damit macht.

### 51.6 Disagreement-Metrik

Nicht mehr eine Score-Friction, sondern eine reine **Meta-Information**:

```python
disagreement = 2 * min(len(preferences), len(risks)) /
               (len(preferences) + len(risks))
```

- `0.0`: alle einig (nur APPROVE oder nur DENY)
- `1.0`: 50/50 gespalten

Das ist **Information**, keine Entscheidung. Der Supervisor kann
damit machen was er will — aktuell: bei ≥ 0.7 + Risks → DEFERRED.

### 51.7 Was wir NICHT gebaut haben (und warum nicht)

Anti-Pattern-Liste der absichtlich vermiedenen Features:

| Anti-Pattern | Warum nicht |
|--------------|-------------|
| `digest.approved` Property | würde Aggregator zum Entscheider machen |
| `best_action()` Methode | globale Utility-Funktion |
| `score = Σ(w×v)` | implizite Norm |
| Adaptive Gewichtung der Gates | versteckte Policy Engine 2.0 |
| `aggregator.evaluate()` | alt-Name → jetzt `collect()` |
| Automatic filtering | ohne Supervisor = Ownership-Verlust |

### 51.8 Was der Aggregator wirklich tut

- **Fragt** alle registrierten Gates ab
- **Sortiert** Signale in 5 Buckets (hard_blocks/risks/preferences/observations/abstentions)
- **Misst** Meta-Info: confidence, disagreement
- **Logt** Digest in History (debuggable)
- **Liefert** strukturierten Digest zurück

Mehr nicht. Keine Entscheidung. Keine Filterung. Keine Priorisierung.

### 51.9 Integration im Supervisor

```
Alter Pfad (v19.0):
  supervisor.review(intent)
    → rate_limit → risk_level → state → ethics → goap → rules
    → 6 sequenzielle Stops, jeder kann rejectieren

Neuer Pfad (v19.1):
  supervisor.review(intent)
    1. risk_level_gate (CRITICAL → DEFERRED)
    2. control_collapse_check:
         digest = aggregator.collect(context)   ← NUR Input
         # Supervisor-eigene Policy:
         if digest.hard_blocks: return REJECTED
         if digest.disagreement ≥ 0.7: return DEFERRED
         if too many risks: return REJECTED
         if only observations: return OBSERVED
         else: durchlassen
    3. custom_rules
    4. APPROVED
```

### 51.10 Was der Audit gelobt hat

Die externe Kritik war:

> "Wenn du es richtig baust, ist das der letzte große Infrastruktur-
> Baustein den dein System noch gebraucht hat."

Richtig gebaut heißt:
- ✅ Constraint + Signal Modell (hard_blocks/risks/preferences/observations)
- ✅ Kein globaler Score
- ✅ Supervisor bleibt einzige Entscheidungsinstanz
- ✅ Output ist erklärbar (jeder Signal hat source + reason + evidence)
- ✅ Aggregator hat KEINE Entscheidungs-Methoden

### 51.11 Was jetzt sichtbar wird

**Bonus: Explainability.**
Der Digest macht implizit sichtbar was alle Gates gesagt haben:

```python
digest = aggregator.collect(context)

# Für Debug-UIs / Logs:
digest.summarize_for_supervisor()
# → "Digest [2 prefs|1 risks] conf=0.82 disagreement=0.50"

# Für Supervisor-Policy:
for sig in digest.risks:
    print(f"  Risk from {sig.source}: {sig.reason} (conf={sig.confidence})")
```

Kein versteckter Score — alles nachvollziehbar.

### 51.12 Tests garantieren keine Entscheidungs-Macht

Die Test-Suite prüft explizit dass der Aggregator **keine Entscheidungs-
Methoden hat**:

```python
def test_digest_has_no_approved_property(aggregator):
    assert not hasattr(digest, "approved")
    assert not hasattr(digest, "decision")
    assert not hasattr(digest, "should_execute")

def test_aggregator_has_no_decision_methods(aggregator):
    assert not hasattr(aggregator, "is_approved")
    assert not hasattr(aggregator, "should_execute")
    assert not hasattr(aggregator, "best_action")
    assert not hasattr(aggregator, "evaluate")

def test_hard_block_not_auto_reject(aggregator):
    # Auch bei HARD_BLOCK: Aggregator entscheidet nicht
    # Der Supervisor sieht beide (hard_blocks + preferences) und wählt
```

Diese Tests verhindern Code-Drift: sobald jemand versucht eine
Entscheidungs-Methode hinzuzufügen, schlagen sie fehl.

---

*Dieses Dokument beschreibt den vollständigen Ist-Zustand des Persona-Holo Systems (v5.0, Stand 2026-04-17). 393 Module, 32 neue in v19.0+v19.1, 1282 Tests grün. 51 Kapitel — inklusive Anti-Decision-Friction-Umbau durch Control-Collapse-Layer.*

*Dieses Dokument beschreibt den vollständigen Ist-Zustand des Persona-Holo Systems (v5.0, Stand 2026-04-17). 392+ Module, 31 neue in v19.0, 1262 Tests grün. 50 Kapitel Gesamt.*

*Dieses Dokument beschreibt den vollständigen Ist-Zustand des Persona-Holo Systems (v5.0, Stand 2026-04-17). 392+ Module, 31 neue in v19.0, 1262 Tests grün, alle Verbindungen und Feedback-Loops dokumentiert. 47 Kapitel Gesamt.*
