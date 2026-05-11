# Brief 2 — Concevoir : architecture et dossier de conception

## Contexte du projet

Le benchmark du Brief 1 a produit une recommandation claire : vous savez quel modèle de langue et quel modèle de sentiment retenir pour FastIA. Maintenant, le CTO veut un **dossier de conception** complet avant de lancer le développement :

> *« OK, les résultats sont convaincants. Mais je ne veux pas qu'on code dans le vide. Je veux un diagramme d'architecture cible, une spec API, un plan de sécurité qui intègre ce qu'on a trouvé sur les attaques adversariales, et une roadmap réaliste. Si quelqu'un reprend le projet dans 6 mois, il doit pouvoir implémenter la suite sans nous appeler. »*

Vous allez **concevoir l'architecture cible** de la pipeline enrichie et produire un dossier technique complet, prêt à transmettre à une équipe de développement.

---

## Objectif principal

Produire un dossier de conception technique complet pour l'intégration des enrichissements (langue, sentiment, routage prioritaire) dans la pipeline FastIA, incluant architecture, sécurité, spécification API, model card, et plan de mise en œuvre phasé.

---

## Prérequis

- Brief 1 du Module 4 complété : benchmark, threat model, matrice de décision
- Recommandation de modèles validée (langue + sentiment)
- `docs/threat_model.md` à jour

---

## Étapes du projet

### 1. Architecture cible — diagramme enrichi

Produire `docs/architecture_cible.md` contenant un diagramme **Mermaid** de la pipeline cible :

```
Sources (email/web/chat) → Loaders → Validation Pydantic → Cleaning
→ Enrichissement (langue → sentiment) → Stockage PostgreSQL
→ Routage prioritaire → API /predict enrichie
```

Le diagramme doit montrer :
- Les composants existants (grisés) vs les nouveaux (colorés)
- Les flux de données avec les formats à chaque étape
- Les points d'entrée/sortie de l'API
- Le cache d'enrichissement (hash body → résultat stocké)
- Le fallback si un modèle d'enrichissement est indisponible

Accompagner le diagramme d'un tableau des composants :

| Composant | Statut | Responsabilité | Entrée | Sortie |
|---|---|---|---|---|
| `email_loader` | existant (M3) | Ingestion mbox | fichier .mbox | `List[RawDemande]` |
| `enrich_language` | à implémenter | Détection langue | `RawDemande.body` | `lang, confidence` |
| ... | ... | ... | ... | ... |

### 2. Input sanitizer — implémentation défensive

Implémenter `src/security/input_sanitizer.py` — un module de nettoyage défensif en amont de tout modèle :

```python
def sanitize(text: str) -> SanitizedInput:
    """Nettoie une entrée utilisateur avant passage au modèle."""
```

Fonctionnalités requises :
- **Détection et remplacement d'homoglyphes** Unicode (confusables → ASCII le plus proche)
- **Troncature** à une longueur maximale configurable (défaut : 2000 caractères)
- **Détection de patterns d'injection** (regex sur "ignore", "oublie", "system prompt", etc.) → flag `injection_suspected: bool` sans bloquer
- **Suppression de caractères de contrôle** (U+0000–U+001F sauf whitespace standard)
- Retour d'un objet `SanitizedInput` (Pydantic) avec le texte nettoyé + métadonnées (flags, longueur originale, homoglyphes détectés)

**Tests Pytest** (`tests/test_input_sanitizer.py`) — au minimum :
- Texte propre → pas de modification
- Texte avec homoglyphes cyrilliques (а, е, о) → remplacement correct
- Texte avec tentative d'injection → flag levé, texte conservé
- Texte > 2000 chars → troncature + flag
- Texte vide → gestion sans erreur
- Caractères de contrôle → suppression

### 3. Évolution du schéma SQL — migration proposée

Rédiger la migration Alembic **sans l'appliquer** (fichier `alembic/versions/xxx_add_enrichment_columns.py` ou, si Alembic n'est pas initialisé, un fichier `docs/migration_enrichment.sql` avec le DDL) :

Nouvelles colonnes sur la table `demandes` :

| Colonne | Type | Nullable | Défaut | Description |
|---|---|---|---|---|
| `langue` | `VARCHAR(5)` | oui | `NULL` | Code ISO 639-1 détecté |
| `langue_confidence` | `FLOAT` | oui | `NULL` | Score de confiance [0, 1] |
| `sentiment` | `VARCHAR(10)` | oui | `NULL` | `positif`, `neutre`, `negatif` |
| `sentiment_score` | `FLOAT` | oui | `NULL` | Score de confiance |
| `enriched_at` | `TIMESTAMPTZ` | oui | `NULL` | Date d'enrichissement |
| `routed_priority` | `VARCHAR(20)` | oui | `NULL` | Priorité post-routage (`high_intl`, `high_negative`, `normal`) |

Documenter :
- Pourquoi toutes les colonnes sont **nullable** (enrichissement progressif, backward-compatible)
- L'**index** recommandé (`CREATE INDEX idx_demandes_langue ON demandes(langue)`)
- La stratégie de **backfill** (enrichir l'historique en batch, pas au fil de l'eau)

### 4. Spécification API enrichie

Rédiger `docs/spec_api_enrichie.md` décrivant l'évolution de l'API FastIA :

**Endpoints existants mis à jour :**

`POST /predict` — ajouter dans la réponse :
```json
{
  "categorie": "...",
  "priorite": "...",
  "reponse_suggeree": "...",
  "langue": "fr",
  "langue_confidence": 0.97,
  "sentiment": "negatif",
  "sentiment_score": 0.82,
  "routed_priority": "high_negative"
}
```

**Nouveaux endpoints :**

| Endpoint | Méthode | Description |
|---|---|---|
| `POST /enrich` | POST | Enrichit un texte (langue + sentiment) sans classification |
| `GET /models` | GET | Liste les modèles actifs et leurs versions |
| `GET /models/{task}/metrics` | GET | Métriques du dernier benchmark (accuracy, F1, date) |

Pour chaque endpoint, fournir :
- Schéma Pydantic d'entrée et de sortie (écrire les classes dans `src/api/schemas.py`)
- Codes HTTP (200, 422, 503)
- Exemple de requête/réponse

### 5. Analyse éthique consolidée

Mettre à jour `docs/risques_ethiques.md` en ajoutant une section **"Risques liés à l'enrichissement automatique"** :

- **Détection de langue → profilage indirect** : la langue corrélée à l'origine géographique peut induire un traitement différencié (AI Act, article 5 — pratiques interdites)
- **Sentiment → décision défavorable** : un score négatif automatique ne doit pas entraîner un refus de service sans intervention humaine (RGPD, article 22)
- **Routage prioritaire** : prioriser les demandes non-FR revient à déprioritiser les FR — est-ce discriminatoire ? documenter la justification métier
- **Biais des modèles de sentiment** : entraînés sur des reviews ≠ support client — le registre de langue diffère, les erreurs sont systématiques sur certains profils

Pour chaque risque : probabilité, gravité, mesure d'atténuation proposée.

### 6. Plan d'atténuation des menaces

Rédiger `docs/plan_attenuation.md` — pour chaque menace du threat model (B1), proposer des défenses concrètes :

| Menace | Défense | Implémentée ? | Priorité |
|---|---|---|---|
| Évasion par homoglyphes | `input_sanitizer.py` — normalisation Unicode | oui (étape 2) | haute |
| Injection de prompt | Détection regex + flag, pas de blocage | oui (étape 2) | haute |
| Empoisonnement dataset | Validation Pydantic + anomaly detection sur `clean.py` | partiel (M2-M3) | moyenne |
| Extraction modèle | Rate limiting sur `/predict` + logging des patterns | à implémenter (M5) | basse |

Pour chaque défense :
- Mécanisme technique (1–2 phrases)
- Limites connues (ce que ça ne couvre PAS)
- Effort d'implémentation (déjà fait / 1h / 1 jour / hors scope)

### 7. Plan de mise en œuvre phasé

Rédiger `docs/plan_mise_en_oeuvre.md` — roadmap d'intégration en 3 phases :

**Phase 1 — Langue (2 sprints)** :
- Appliquer la migration SQL
- Intégrer `enrich_language()` dans la pipeline après cleaning
- Backfill sur l'historique en base
- Ajouter `langue` et `langue_confidence` à la réponse `/predict`
- Tests d'intégration

**Phase 2 — Sentiment (2 sprints)** :
- Intégrer `enrich_sentiment()` (uniquement sur les lignes `langue == 'fr'` pour le modèle distilcamembert)
- Backfill
- Ajouter `sentiment` à la réponse `/predict`
- Monitoring des distributions en production

**Phase 3 — Routage prioritaire (1 sprint)** :
- Implémenter la logique de routage : `if langue != 'fr' → high_intl` ; `if sentiment == 'negatif' and score > 0.8 → high_negative`
- Exposer `routed_priority` dans la réponse
- Endpoint `/enrich` standalone
- Dashboard de suivi (volumes routés par catégorie)

Pour chaque phase : pré-requis, livrables, critères d'acceptation, risques.

### 8. Model card — modèle langue retenu

Produire `docs/model_card_langue.md` au format Mitchell et al. (2019) :

- **Model Details** : nom, version, type, auteurs, date, licence
- **Intended Use** : cas d'usage FastIA (détection de langue sur demandes clients support)
- **Out-of-Scope Uses** : ce pour quoi le modèle ne doit PAS être utilisé
- **Training Data** : résumé (si modèle pré-entraîné : source d'entraînement originale)
- **Evaluation Data** : `langue_eval_200.jsonl` — composition, source, limites
- **Metrics** : résultats du benchmark B1 (accuracy, F1 par classe, temps d'inférence)
- **Ethical Considerations** : risques identifiés à l'étape 5
- **Caveats and Recommendations** : conditions de revalidation, seuil de confiance minimal recommandé

---

## Livrables

- **`src/security/input_sanitizer.py`** — module de nettoyage défensif + objet `SanitizedInput`
- **`tests/test_input_sanitizer.py`** — tests Pytest (≥6 cas)
- **`src/api/schemas.py`** — schémas Pydantic de l'API enrichie
- **`docs/architecture_cible.md`** — diagramme Mermaid + tableau des composants
- **`docs/migration_enrichment.sql`** (ou migration Alembic) — DDL des nouvelles colonnes
- **`docs/spec_api_enrichie.md`** — spécification des endpoints enrichis
- **`docs/risques_ethiques.md`** — mise à jour avec section enrichissement
- **`docs/plan_attenuation.md`** — défenses pour chaque menace du threat model
- **`docs/plan_mise_en_oeuvre.md`** — roadmap en 3 phases
- **`docs/model_card_langue.md`** — model card du modèle retenu
- **README.md** — section "Module 4 — Brief 2" avec vue d'ensemble du dossier

---

## Charge de travail estimée

6 à 7 heures

---

## Ressources

- [Mitchell et al. — Model Cards for Model Reporting (2019)](https://arxiv.org/abs/1810.03993)
- [HuggingFace — Model Card Guide](https://huggingface.co/docs/hub/model-cards)
- [Mermaid — diagrammes](https://mermaid.js.org/)
- [MITRE ATLAS — mitigations](https://atlas.mitre.org/mitigations/)
- [OWASP Top 10 LLM — mitigations](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [AI Act — article 5, pratiques interdites](https://eur-lex.europa.eu/eli/reg/2024/1689/oj)
- [Alembic — autogenerate](https://alembic.sqlalchemy.org/en/latest/autogenerate.html)

---

## Bonus

- Implémenter un **prototype de rate limiter** sur `/predict` (défense extraction) avec un middleware FastAPI + compteur Redis ou in-memory
- Produire une **model card sentiment** en plus de celle langue
- Générer le diagramme Mermaid en **SVG exportable** via la CLI mermaid
- Rédiger une **note de transfert** d'une page simulant le handoff à une équipe de développement
