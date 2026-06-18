# Audit MLOps du projet churn

## Défaut 1 — Secret en dur dans le code source

**Localisation** : `app.py`, ligne 11  
**Description** : `API_TOKEN = "churn-demo-token"` est écrit en dur dans le code source versionné dans Git. N'importe qui ayant accès au repo peut lire ce token et l'utiliser pour appeler l'API. De plus, ce token n'est jamais vérifié dans aucune route — il est déclaré mais inutilisé, ce qui signifie que l'API est totalement ouverte malgré l'apparence d'une protection.  
**Criticité** : HAUTE  
**Justification** : Un secret versionné dans Git est compromis définitivement — même supprimé, il reste dans l'historique. De plus, le token inutilisé crée une fausse impression de sécurité. En production, n'importe quel acteur malveillant peut appeler `/predict` sans restriction.

---

## Défaut 2 — Absence de vérification du dataset vide dans `prepare_data`

**Localisation** : `src/prepare.py`, fonction `prepare_data`  
**Description** : La fonction vérifie que le fichier existe et que la colonne cible est présente, mais ne vérifie pas que le dataset n'est pas vide. Si le CSV est vide (0 lignes), `train_test_split` plantera avec une erreur cryptique sklearn sans message clair sur la cause réelle.  
**Criticité** : MOYENNE  
**Justification** : Un pipeline qui plante avec une erreur non explicite est difficile à déboguer en production. Un contrôle `if df.empty: raise ValueError(...)` coûte une ligne et évite une investigation inutile.

---

## Défaut 3 — Métrique F1 absente de `evaluate_model`

**Localisation** : `src/evaluate.py`, fonction `evaluate_model`  
**Description** : La fonction retourne `accuracy`, `precision` et `recall` mais pas le `f1_score`. Le F1 est la métrique de référence pour les problèmes de classification déséquilibrés (ce qui est typiquement le cas du churn). Son absence empêche la comparaison complète des runs dans MLflow et le test de non-régression ne peut pas s'appuyer dessus.  
**Criticité** : MOYENNE  
**Justification** : Sans F1, on ne peut pas détecter qu'un modèle qui améliore la precision en dégradant le recall (ou l'inverse) est en réalité moins bon. C'est une métrique métier critique pour le churn.

---

## Défaut 4 — Test de non-régression qui ne teste pas la non-régression

**Localisation** : `tests/test_non_regression.py`, fonction `test_model_can_be_serialized`  
**Description** : `test_model_can_be_serialized` vérifie uniquement que `joblib.load` retourne un objet non-nul après `joblib.dump`. Ce test ne peut pas échouer — `loaded_model is not None` sera toujours vrai tant que joblib fonctionne. Il ne teste aucun comportement métier du modèle et ne constitue pas un vrai test de non-régression.  
**Criticité** : MOYENNE  
**Justification** : Un test qui ne peut pas échouer ne sert à rien — il donne une fausse impression de couverture. Un vrai test de non-régression vérifierait que le modèle rechargé produit les mêmes prédictions que le modèle original sur un jeu fixe.

---

## Défaut 5 — Workflow CI trop permissif et incomplet

**Localisation** : `.github/workflows/ci.yml`  
**Description** : Deux problèmes distincts. D'abord, le trigger `on: push` sans filtre de branche déclenche la CI sur toutes les branches y compris les branches de travail temporaires — ce qui consomme inutilement des minutes CI. Ensuite, le workflow ne vérifie pas que les artefacts ont bien été produits après `python main.py` (`model.pkl`, `feature_columns.json`) — si `save_artifacts` échoue silencieusement, les tests qui dépendent des artefacts passeront quand même car ils réentraînent eux-mêmes le modèle.  
**Criticité** : BASSE  
**Justification** : Le CI doit être un filet de sécurité strict. Sans vérification des artefacts et sans filtre de branche, des régressions peuvent passer inaperçues et des ressources sont gaspillées.

---

## Défaut 6 — Logs insuffisants dans `/predict`

**Localisation** : `app.py`, ligne 68  
**Description** : Le log de prédiction est `logger.info("prediction=%s", prediction)` — il ne contient pas les valeurs d'entrée (tenure_months, monthly_charges...) ni la confiance. En cas de problème en production, il est impossible de savoir quelle entrée a produit quelle prédiction et avec quelle confiance. De plus, `/metrics` n'expose que 2 indicateurs (`n_predictions`, `n_errors`) — l'énoncé des bonnes pratiques demande au minimum 4 indicateurs.  
**Criticité** : BASSE  
**Justification** : Un service ML non observable n'est pas exploitable en production. Sans logs d'entrée et sans indicateurs suffisants, il est impossible de détecter une dérive ou de déboguer un problème.