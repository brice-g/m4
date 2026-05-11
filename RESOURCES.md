# Ressources — Module 4

## Cadre du module

M4 est le point de bascule entre la construction (M1–M3) et la conception robuste. On évalue formellement les modèles candidats issus du M3, on les confronte à des menaces adversariales, et on produit un dossier de conception complet avant l'implémentation finale (M5+).

---

## Documentation technique

### Évaluation et benchmark

- [scikit-learn — metrics (precision, recall, F1, confusion matrix)](https://scikit-learn.org/stable/modules/model_evaluation.html)
- [HuggingFace `evaluate` library](https://huggingface.co/docs/evaluate/) — métriques standardisées
- [langdetect — PyPI](https://pypi.org/project/langdetect/) — détection de langue rapide
- [fasttext — language identification (lid.176.bin)](https://fasttext.cc/docs/en/language-identification.html) — modèle robuste offline
- [XLM-RoBERTa — HuggingFace](https://huggingface.co/xlm-roberta-base) — Transformer multilingue (baseline haute)
- [`cmarkea/distilcamembert-base-sentiment`](https://huggingface.co/cmarkea/distilcamembert-base-sentiment) — sentiment FR
- [`nlptown/bert-base-multilingual-uncased-sentiment`](https://huggingface.co/nlptown/bert-base-multilingual-uncased-sentiment) — sentiment multilingue

### Sécurité et robustesse

- [OWASP — Top 10 for LLM Applications (2025)](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — taxonomie des menaces
- [MITRE ATLAS — Adversarial Threat Landscape for AI Systems](https://atlas.mitre.org/) — framework d'attaques IA
- [NIST AI 100-2 — Adversarial Machine Learning](https://csrc.nist.gov/pubs/ai/100/2/e2023/final) — taxonomie officielle
- [Homoglyph attacks — Unicode confusables](https://util.unicode.org/UnicodeJsps/confusables.jsp) — substitutions visuelles
- [Prompt injection — Simon Willison](https://simonwillison.net/2023/Apr/14/worst-that-can-happen/) — vulgarisation des risques

### Éco-conception

- [Green Algorithms](https://www.green-algorithms.org/) — estimer l'empreinte carbone d'un calcul
- [CodeCarbon](https://codecarbon.io/) — mesure embarquée de la consommation
- [Luccioni et al. — Power Hungry Processing (2023)](https://arxiv.org/abs/2311.16863) — coût énergétique des modèles NLP

### Architecture et conception

- [Model Cards — Mitchell et al. (2019)](https://arxiv.org/abs/1810.03993) — template de documentation modèle
- [HuggingFace Model Card Guide](https://huggingface.co/docs/hub/model-cards)
- [Alembic — migrations SQLAlchemy](https://alembic.sqlalchemy.org/en/latest/)
- [Pydantic v2 — validators et schemas](https://docs.pydantic.dev/latest/)
- [FastAPI — dépendances et middleware](https://fastapi.tiangolo.com/tutorial/dependencies/)

---

## Cadre réglementaire et éthique

- **AI Act — classification des systèmes** : <https://eur-lex.europa.eu/eli/reg/2024/1689/oj> — annexes III et IV (évaluation de conformité, catégorisation du risque)
- **RGPD — profilage et décision automatisée** : <https://www.cnil.fr/fr/profilage-et-decision-entierement-automatisee> — pertinent dès qu'un score de sentiment influence une décision
- **CNIL — recommandation sur les systèmes d'IA** : <https://www.cnil.fr/fr/intelligence-artificielle>
- **ISO/IEC 42001:2023** — Management de l'IA (mentionné pour culture, pas exigé)

---

## Papiers — rappel et nouveautés

- Mitchell et al. — *Model Cards for Model Reporting* (2019) — structure adoptée au Brief 2
- Gebru et al. — *Datasheets for Datasets* — mise à jour continue depuis M2
- Goodfellow et al. — *Explaining and Harnessing Adversarial Examples* (2015) — fondation du threat modeling
- Carlini & Wagner — *Towards Evaluating the Robustness of Neural Networks* (2017) — attaques par optimisation

---

## Données fournies (`data/`)

| Fichier | Brief | Description |
|---|---|---|
| `eval/langue_eval_200.jsonl` | B1 | 200 exemples annotés (FR/EN/ES) pour benchmark de détection de langue |
| `adversarial/adversarial_fastia.jsonl` | B1 | 30 exemples adversariaux (homoglyphes, injection prompt, reformulations, edge cases) |

Les données d'évaluation sentiment (50 lignes) sont à annoter par l'apprenant au Brief 1 — cela fait partie de l'exercice.

---

## Outillage

- [Loguru](https://loguru.readthedocs.io) — logs structurés (déjà en place)
- [Pytest — fixtures, paramétrage, tmp_path](https://docs.pytest.org)
- [Mermaid — diagrammes d'architecture](https://mermaid.js.org/) — utilisé au Brief 2 pour le schéma cible
- [MLflow — model registry](https://mlflow.org/docs/latest/model-registry.html) — pour tracer les modèles candidats
