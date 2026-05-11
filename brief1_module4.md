# Brief 1 — Évaluer : benchmark et robustesse

## Contexte du projet

La pipeline FastIA ingère désormais trois canaux (email, web, chat) et un prototype d'enrichissement (détection de langue ou sentiment) est fonctionnel depuis le M3. Mais une preuve de concept n'est pas une décision. Le CTO vous demande :

> *« Avant qu'on intègre quoi que ce soit en prod, je veux un benchmark propre. Pas un `langdetect` qui tourne sur 5 exemples — un vrai protocole d'évaluation reproductible. Et je veux qu'on sache comment ça se comporte quand quelqu'un essaie de le tromper, parce que les attaquants ne s'arrêtent pas aux données propres. »*

Vous allez **évaluer rigoureusement** les modèles candidats identifiés au M3, les confronter à des entrées adversariales, et produire une matrice de décision argumentée.

---

## Objectif principal

Conduire un benchmark comparatif reproductible des modèles candidats pour l'enrichissement FastIA (langue et sentiment), les soumettre à un protocole de tests adversariaux, et recommander un choix documenté intégrant performance, robustesse et éco-conception.

---

## Prérequis

- Module 3 complété : pipeline multi-source fonctionnelle, PoC d'enrichissement
- Base PostgreSQL peuplée depuis les 3 canaux
- `docs/sources_data_evaluation.md` listant les modèles candidats

---

## Données fournies

| Fichier | Description |
|---|---|
| `data/eval/langue_eval_200.jsonl` | 200 exemples annotés manuellement : `{"text": "...", "lang": "fr\|en\|es"}` — répartition 80 FR / 70 EN / 50 ES |
| `data/adversarial/adversarial_fastia.jsonl` | 30 exemples adversariaux pour le modèle FastIA : homoglyphes, injection prompt, reformulations, code-switching, chaînes vides/longues |

---

## Étapes du projet

### 1. Jeu d'évaluation langue (fourni) + sentiment (à constituer)

**Langue** — le fichier `data/eval/langue_eval_200.jsonl` est fourni, prêt à l'emploi. Vérifier qu'il est bien chargeable et conforme au schéma attendu.

**Sentiment** — constituer un mini-jeu de 50 lignes annoté manuellement :

- Extraire un échantillon aléatoire stratifié de 50 `body` depuis la base (mélange des 3 canaux, uniquement les lignes FR pour éviter le biais multilingue)
- Annoter chaque ligne avec un label parmi : `positif`, `neutre`, `negatif`
- Sauvegarder dans `data/eval/sentiment_eval_50.jsonl` avec le format `{"text": "...", "sentiment": "positif|neutre|negatif"}`
- Documenter le **protocole d'annotation** dans une cellule markdown du notebook (qui annote ? quelles règles de décision pour les cas ambigus ? inter-annotator agreement si possible)

### 2. Benchmark détection de langue

Comparer **au minimum 2 candidats** (3 recommandés) :

| Candidat | Avantage attendu | Contrainte |
|---|---|---|
| `langdetect` | rapide, zéro dépendance lourde | instable sur textes courts |
| `fasttext` (`lid.176.bin`) | excellent multi-langue, offline | modèle 130 MB, compilation C++ |
| XLM-RoBERTa (via `transformers`) | état de l'art | ~1 GB RAM, inférence lente |

Pour chaque candidat, dans le notebook :

- Charger le jeu `langue_eval_200.jsonl`
- Inférer la langue sur chaque ligne
- Calculer : **accuracy**, **precision/recall/F1 par classe** (FR, EN, ES), **matrice de confusion**
- Mesurer : **temps d'inférence moyen par ligne**, **RAM pic** (via `psutil`)
- Visualiser : matrice de confusion en heatmap, barplot comparatif des F1

### 3. Benchmark sentiment

Comparer **2 candidats** :

| Candidat | Entraîné sur | Langue |
|---|---|---|
| `cmarkea/distilcamembert-base-sentiment` | avis FR | FR uniquement |
| `nlptown/bert-base-multilingual-uncased-sentiment` | reviews multi-langue | multilingue (1–5 stars → mapping) |

Pour chaque candidat :

- Charger le jeu `sentiment_eval_50.jsonl`
- Mapper les sorties du modèle vers le schéma `positif/neutre/negatif` (documenter le mapping)
- Calculer : accuracy, F1 macro, matrice de confusion
- Mesurer : temps d'inférence moyen, RAM pic
- Analyser les **cas d'erreur** : y a-t-il un pattern ? (longueur, registre, ironie ?)

### 4. Threat modeling — taxonomie des menaces

Rédiger `docs/threat_model.md` couvrant **4 familles de menaces** appliquées au contexte FastIA :

| Famille | Exemple FastIA | Impact potentiel |
|---|---|---|
| **Évasion** | Homoglyphes dans le sujet email pour tromper la classification | Demande critique non routée |
| **Empoisonnement** | Insertion de lignes biaisées dans le dataset d'entraînement | Modèle dégradé sur certaines catégories |
| **Injection de prompt** | Un client écrit « ignore les instructions précédentes » dans sa demande | Réponse hors cadre, fuite d'info |
| **Extraction de modèle** | Requêtes systématiques sur `/predict` pour reconstruire le comportement | Vol de propriété intellectuelle |

Pour chaque famille :
- Description en 2–3 phrases
- Vecteur d'attaque concret (comment un attaquant procède)
- Impact sur FastIA (confidentialité, intégrité, disponibilité)
- Détectabilité (facile / difficile / actuellement non détecté)
- Renvoi vers l'étape 5 pour les tests pratiques

Référencer **MITRE ATLAS** et **NIST AI 100-2** comme frameworks.

### 5. Tests adversariaux pratiques

Utiliser le fichier fourni `data/adversarial/adversarial_fastia.jsonl` (30 cas). Dans le notebook :

**Sur le modèle de langue retenu (meilleur du benchmark) :**
- Passer les cas adversariaux pertinents (homoglyphes, code-switching, texte vide/très long)
- Documenter : le modèle est-il trompé ? Dans quelle proportion ?

**Sur le modèle de sentiment retenu :**
- Passer les cas contenant de l'ironie, des négations inversées, du sarcasme
- Documenter les échecs

**Sur le modèle FastIA (classification M1) :**
- Passer les 30 cas via l'API `/predict` (ou en local si API non dispo)
- Documenter : injections de prompt réussies, homoglyphes contournant la catégorisation
- Calculer un **taux de robustesse** : % de cas adversariaux où le modèle maintient un comportement acceptable

### 6. Matrice de décision et recommandation

Produire `docs/matrice_decision.md` avec un tableau comparatif final :

| Critère (pondération) | langdetect | fasttext | XLM-R | distilcamembert | bert-multi |
|---|---|---|---|---|---|
| Accuracy (30%) | ... | ... | ... | ... | ... |
| F1 macro (20%) | ... | ... | ... | ... | ... |
| Robustesse adversariale (20%) | ... | ... | ... | ... | ... |
| Temps inférence (15%) | ... | ... | ... | ... | ... |
| Empreinte mémoire (10%) | ... | ... | ... | ... | ... |
| Complexité d'intégration (5%) | ... | ... | ... | ... | ... |

- Justifier les **pondérations** par rapport au contexte métier FastIA
- Calculer un **score composite** par candidat
- Formuler une **recommandation** : quel modèle pour la langue, quel modèle pour le sentiment, avec quelles réserves
- Expliciter les **conditions de remise en question** du choix (ex. "si le volume dépasse 10k/jour, réévaluer XLM-R")

---

## Livrables

- **`notebook_brief1_module4.ipynb`** — benchmark complet (langue + sentiment + adversarial), reproductible, commenté
- **`data/eval/sentiment_eval_50.jsonl`** — jeu d'évaluation sentiment annoté par l'apprenant
- **`docs/threat_model.md`** — taxonomie des 4 familles de menaces appliquées à FastIA
- **`docs/matrice_decision.md`** — tableau comparatif et recommandation argumentée
- **README.md** — section "Module 4 — Brief 1" avec résumé des résultats clés

---

## Charge de travail estimée

6 à 7 heures

---

## Ressources

- [scikit-learn — classification_report, confusion_matrix](https://scikit-learn.org/stable/modules/model_evaluation.html)
- [MITRE ATLAS](https://atlas.mitre.org/) — framework de menaces IA
- [NIST AI 100-2 — Adversarial ML](https://csrc.nist.gov/pubs/ai/100/2/e2023/final)
- [OWASP Top 10 for LLM (2025)](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Green Algorithms](https://www.green-algorithms.org/) — estimation empreinte
- [psutil — memory_info](https://psutil.readthedocs.io/en/latest/#psutil.Process.memory_info)

---

## Bonus

- Ajouter un **3e candidat langue** (ex. XLM-RoBERTa fine-tuné sur language-id) et comparer
- Implémenter un script `src/evaluation/run_benchmark.py` réutilisable (CLI : `--task langue|sentiment --models all`)
- Mesurer l'empreinte carbone avec **CodeCarbon** et l'intégrer à la matrice de décision
- Tester la **stabilité** de `langdetect` (10 runs sur les mêmes inputs — variance ?)
