# Système IoT Industriel pour la Maintenance Prédictive et Détection d'Anomalies

Bienvenue sur le projet **IOT-predictive-maintenance**, une plateforme complète d'**Industrie 4.0** dédiée à la surveillance en temps réel, à la détection proactive d'anomalies par intelligence artificielle et à la gestion des interventions de maintenance sur des équipements industriels.

---

## Présentation et Contexte

Dans les environnements industriels modernes, les pannes imprévues sur les lignes de production entraînent des coûts opérationnels majeurs et des arrêts de chaîne critiques. Ce projet propose une solution de **maintenance prédictive de bout en bout** :

1. **Collecte et Ingestion continue** : Émulation et capture de télémétries multi-capteurs sur différentes machines industrielles.
2. **Transport résilient** : Acheminement temps réel à faible latence via le protocole IoT standard MQTT.
3. **Pipeline Data Science et Machine Learning** :
   * Importation de jeux de données industriels historiques (benchmark ou télémétrie réelle).
   * Entraînement hors-ligne (*offline training*), validation et sérialisation de modèles de détection d'anomalies.
4. **Traitement et Persistance** : Ingestion, validation et historisation des flux de données avec un cœur applicatif robuste (Spring Boot et PostgreSQL).
5. **Diagnostic et Inférence temps réel** : Détection précoce des dérives et scoring de risque en direct via une API d'inférence (FastAPI).
6. **Aide à la décision et IHM** : Interface utilisateur ergonomique (Angular) offrant un monitoring temps réel, une gestion d'alertes et le suivi des ordres d'intervention.

---

## Architecture Globale du Système

L'architecture est découpée de manière modulaire en microservices et composants spécialisés, garantissant haute disponibilité, passage à l'échelle et découplage fort entre composants :

![Architecture Globale du Système](mermaid-diagram.png)

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                               1. SIMULATION IOT (Temps Réel)                           │
│  Machines M001, M002, M003  ──(Température, Vibration, Pression, Vitesse)              │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Publish MQTT
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                               2. BROKER MQTT                                           │
│  Eclipse Mosquitto (Acheminement temps réel des messages)                             │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Subscribe MQTT
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         3. BACKEND MÉTIER ET PERSISTANCE                               │
│  Spring Boot (Ingestion, Validation, Règles métier) ◄──────────► PostgreSQL            │
└───────────────────────────────┬───────────────────────────────────────▲────────────────┘
           Demande d'inférence  │                                       │ HTTP / JSON
                                ▼                                       │
┌───────────────────────────────────────────────────────────────┐       │
│               4. INTELLIGENCE ARTIFICIELLE                    │       │
│                                                               │       │
│  ┌─────────────────────────────────────────────────────────┐  │       │
│  │ A. PIPELINE ML (Entraînement Hors-Ligne / Training)     │  │       │
│  │ Dataset Industriel ──► Preprocessing ──► Training ──►   │  │       │
│  │ Modèle Sérialisé (model.joblib, scaler.joblib)          │  │       │
│  └────────────────────────────┬────────────────────────────┘  │       │
│                               ▼ Chargement du modèle          │       │
│  ┌─────────────────────────────────────────────────────────┐  │       │
│  │ B. MICROSERVICE D'INFÉRENCE (FastAPI - Temps Réel)      │  │       │
│  │ Endpoints /predict et /health (Détection + Score Risque)│  │       │
│  └─────────────────────────────────────────────────────────┘  │       │
└───────────────────────────────────────────────────────────────┘       │
                                                                        ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              5. INTERFACE UTILISATEUR                                  │
│  Angular (Dashboard Machines, Alertes / Prédictions, Gestion des Interventions)       │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Description Détaillée du Projet en 5 Parties

Le projet s'articule autour de **5 parties interdépendantes et complémentaires** :

---

### Partie 1 : Simulation Industrielle et Génération de Télémétrie (Python)
* **Rôle** : Modéliser le comportement d'un parc de machines industrielles sans nécessiter d'équipements physiques au démarrage.
* **Équipements émulés** : Machine `M001`, Machine `M002`, Machine `M003`.
* **Grandeurs physiques mesurées** :
  * **Température** (°C) : Évolution thermique et détection d'échauffements anormaux.
  * **Vibration** (mm/s ou Hz) : Analyse des balourds, désalignements et jeux mécaniques.
  * **Pression** (bar) : Contrôle des circuits hydrauliques ou pneumatiques.
  * **Vitesse de rotation** (RPM) : Surveillance des variations de régime moteur.
* **Scénarios d'exploitation simulés** :
  * **Régime Normal** : Variations statistiques nominales autour des points de consigne.
  * **Dégradation Progressive** : Dérive lente des indicateurs (usure des roulements, encrassement thermique).
  * **Anomalie Franche** : Rupture brusque, pic de pression ou surchauffe critique nécessitant un arrêt d'urgence.
* **Émission des données** : Publication asynchrone au format JSON vers les topics MQTT dédiés via la bibliothèque `paho-mqtt`.

---

### Partie 2 : Communication et Broker IoT (Eclipse Mosquitto)
* **Rôle** : Assurer la couche de messagerie IoT haute performance et garantir un découplage total entre producteurs de données et consommateurs.
* **Technologie** : Broker **Eclipse Mosquitto** léger et conforme à la norme MQTT v3.1.1 / v5.0.
* **Organisation des Topics** :
  * Structure hiérarchique : `factory/machines/{machineId}/telemetry`.
  * Support de canaux de commande ou d'état : `factory/machines/{machineId}/status`.
* **Qualité de Service (QoS)** : Configuration optimisée (QoS 0 / 1) pour assurer la transmission fiable sans surcharger le réseau.

---

### Partie 3 : Backend Métier et Ingestion des Données (Spring Boot et PostgreSQL)
* **Rôle** : Servir de chef d'orchestre applicatif, responsable de la gouvernance des données, de l'application des règles métier et de la persistance.
* **Composants clés** :
  * **MQTT Subscriber (Spring Integration MQTT)** : Écoute en continu et consommation sans blocage des messages de télémétrie.
  * **Validation et Contrôle d'Intégrité** : Rejet des trames corrompues, normalisation des unités et détection des dépassements de seuils statiques.
  * **Persistance Relationnelle (PostgreSQL + Spring Data JPA)** :
    * Table des équipements (référentiel machines).
    * Table d'historique des mesures (séries temporelles de télémétrie horodatées).
    * Table des alertes générées.
    * Table des interventions de maintenance.
  * **Connecteur d'Inférence IA** : Requêtage HTTP REST synchrone/asynchrone vers le microservice FastAPI lors de la réception des trames de capteurs.
  * **API REST et WebSocket** : Exposition d'endpoints sécurisés pour alimenter l'interface graphique en données statiques et en flux temps réel.

---

### Partie 4 : Intelligence Artificielle — Pipeline Data, Entraînement et Inférence (FastAPI et ML)
* **Rôle** : Couvrir l'intégralité du cycle de vie Data Science / Machine Learning, depuis l'ingestion du jeu de données jusqu'à l'inférence temps réel pour la détection préventive d'anomalies.

Cette partie se subdivise en deux volets indissociables :

#### A. Phase Hors-Ligne (Offline ML Pipeline : Dataset et Training)
1. **Importation et Acquisition du Dataset** :
   * Exploitation d'un jeu de données de référence en maintenance prédictive (ex: *AI4I 2020 Predictive Maintenance Dataset*, *NASA C-MAPSS Turbofan*, ou historique de télémétrie usine).
   * Ingestion via scripts Python (`pandas`, `numpy`) dans le répertoire `ai-service/data/raw/`.
2. **Exploration et Préparation des Données (EDA et Preprocessing)** :
   * Analyse exploratoire des corrélations (Température vs Vitesse vs Pression vs Vibration).
   * Nettoyage, traitement des valeurs aberrantes ou manquantes.
   * Mise à l'échelle et normalisation (`StandardScaler`, `MinMaxScaler`).
   * Découpage rigoureux en ensembles d'entraînement, validation et test (`train_test_split`).
3. **Entraînement des Modèles (Model Training)** :
   * Entraînement d'algorithmes de détection d'anomalies et classification :
     * *Non supervisé* : **Isolation Forest**, **One-Class SVM**, ou **Autoencoder**.
     * *Supervisé* (si labels de défaillance disponibles) : **Random Forest Classifier**, **XGBoost**.
4. **Évaluation et Validation des Performances** :
   * Calcul des métriques d'évaluation clés : Matrice de confusion, Précision, Rappel (*Recall*), F1-Score, score ROC-AUC.
   * Optimisation des hyperparamètres (GridSearch / RandomSearch).
5. **Exportation et Sérialisation** :
   * Sauvegarde des artefacts entraînés (`model.joblib` et `scaler.joblib`) prêts pour le serving.

#### B. Phase En-Ligne (Online Serving : FastAPI)
1. **Microservice d'Inférence FastAPI** :
   * Chargement en mémoire des artefacts au démarrage (`@app.on_event("startup")` ou `lifespan`).
   * Endpoint d'inférence haute performance : `POST /predict`.
   * Endpoint de monitoring de santé : `GET /health`.
2. **Scoring de Risque et Détection** :
   * Réception de la mesure courante (Température, vibration, pression, vitesse).
   * Transformation via le scaler sérialisé.
   * Calcul du score d'anomalie et déduction du niveau de risque ($0\%$ à $100\%$, statut : *Normal*, *Dégradé*, *Critique*).
   * Renvoi immédiat de la réponse JSON au backend Spring Boot.

---

### Partie 5 : Interface Utilisateur et Dashboard de Maintenance (Angular)
* **Rôle** : Offrir aux opérateurs et responsables d'usine une vision synthétique, ergonomique et exploitable de la santé des équipements.
* **Modules et Écrans principaux** :
  * **Dashboard / Machines** :
    * Visualisation d'ensemble de l'atelier industriel.
    * Cartes individuelles par machine (`M001`, `M002`, `M003`) avec statut opérationnel (En marche, Dégradé, En panne).
    * Graphiques interactifs d'évolution temporelle (courbes de température, vibrations, etc.).
  * **Alertes et Prédictions** :
    * Liste chronologique des alertes déclenchées par l'IA ou les seuils physiques.
    * Filtrage par niveau de sévérité (Faible, Moyen, Élevé) et par machine.
  * **Gestion des Interventions** :
    * Création et assignation des ordres de maintenance préventive/corrective.
    * Suivi du cycle de vie des interventions (*À planifier*, *En cours*, *Terminée*).
    * Historique des opérations effectuées sur chaque machine.

---

## Stack Technologique Complète

| Composant | Technologie | Description / Rôle |
| :--- | :--- | :--- |
| **Simulateur IoT** | Python 3.10+, `paho-mqtt`, `numpy` | Génération de mesures physiques et scénarios industriels |
| **Broker IoT** | Eclipse Mosquitto | Broker de messages MQTT haute disponibilité |
| **Backend Métier** | Java 17+, Spring Boot 3, Spring Data JPA, Spring Integration | Ingestion, validation, logique métier et API REST |
| **Base de Données** | PostgreSQL 15+ | Stockage relationnel pérenne des machines, mesures, alertes |
| **Data et Entraînement ML** | Python, Pandas, Scikit-learn, Joblib | Importation du dataset, prétraitement, entraînement et évaluation du modèle |
| **Inférence IA** | FastAPI, Uvicorn, Pydantic | API REST d'inférence en temps réel |
| **Frontend** | Angular 17+, TypeScript, TailwindCSS / Bootstrap, Chart.js | IHM opérateur, graphiques temps réel et gestion des interventions |
| **Orchestration** | Docker et Docker Compose | Conteneurisation et déploiement unifié des services |

---

## Arborescence du Projet

```plaintext
IOT-predictive-maintenance/
├── docker/
│   ├── mosquitto/
│   │   └── config/mosquitto.conf
│   └── docker-compose.yml
│
├── simulator/
│   ├── src/
│   │   ├── machine_simulator.py
│   │   └── scenarios.py
│   └── requirements.txt
│
├── ai-service/
│   ├── data/
│   │   ├── raw/                      # Jeu de données brut (ex: predictive_maintenance.csv)
│   │   └── processed/                # Données nettoyées et normalisées
│   ├── notebooks/
│   │   └── 01_eda_and_training.ipynb # Exploration, analyse et tests de modèles
│   ├── models/
│   │   ├── model.joblib              # Modèle entraîné et sérialisé
│   │   └── scaler.joblib             # Scaler de normalisation
│   ├── src/
│   │   ├── data_loader.py            # Téléchargement / chargement du dataset
│   │   ├── train.py                  # Script d'entraînement et d'export du modèle
│   │   ├── evaluate.py               # Calcul des métriques de performance
│   │   ├── main.py                   # Serveur d'inférence FastAPI
│   │   └── schemas.py                # Schémas de validation Pydantic (entrées / sorties)
│   └── requirements.txt
│
├── backend-spring/
│   ├── src/main/java/com/iot/maintenance/...
│   └── pom.xml
│
├── frontend-angular/
│   ├── src/app/
│   │   ├── dashboard/
│   │   ├── alerts/
│   │   └── interventions/
│   └── package.json
│
├── mermaid-diagram.png
└── README.md
```

---

## Workflow de Démarrage (Guide Opérationnel)

1. **Démarrer les services d'infrastructure** :
   ```bash
   cd docker
   docker-compose up -d
   ```
2. **Entraîner le modèle de Machine Learning** :
   ```bash
   cd ai-service
   pip install -r requirements.txt
   python src/train.py   # Génère models/model.joblib et models/scaler.joblib
   ```
3. **Lancer le microservice d'IA (FastAPI)** :
   ```bash
   uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload
   ```
4. **Démarrer le backend Spring Boot** :
   ```bash
   cd backend-spring
   ./mvnw spring-boot:run
   ```
5. **Lancer l'application Frontend Angular** :
   ```bash
   cd frontend-angular
   npm install
   ng serve --open
   ```
6. **Démarrer le simulateur de machines** :
   ```bash
   cd simulator
   python src/machine_simulator.py
   ```
