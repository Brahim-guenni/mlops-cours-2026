# Rapport — Lab 4 : packaging du modèle

Binôme : À compléter · Dataset : `german-credit` · Modèle : `scoring-model` v3

## 1. Comparatif des tailles et de la sécurité

| Image | Taille disque | CVE HIGH | CVE CRITICAL |
|---|---:|---:|---:|
| `scoring-api:naive` | 2.11 GB | 192 | 3 |
| `scoring-api:optimized` | 535 MB | 53 | 0 |
| `scoring-api:bentoml` | 1.41 GB | 96 | 0 |

L'image optimisée est environ 75 % plus petite que l'image naive.

### Résultats détaillés de Trivy

Les résultats sont présentés séparément pour le système d'exploitation et les
packages Python.

| Image | Scanner | Total | HIGH | CRITICAL |
|---|---|---:|---:|---:|
| Naive | OS | 2 878 | 192 | 3 |
| Naive | Python | 6 | 0 | 0 |
| Optimized | OS | 222 | 53 | 0 |
| Optimized | Python | 6 | 0 | 0 |

Les résultats détaillés sont disponibles dans `reports/trivy_naive.txt` et
`reports/trivy_optimized.txt`.

Pour BentoML, les chiffres du tableau proviennent du scan précédent :
96 vulnérabilités HIGH et aucune CRITICAL. Cette image n'a pas été rescannée
dans cette comparaison.

L'image optimisée réduit fortement le nombre de vulnérabilités détectées et
ne présente aucune vulnérabilité CRITICAL dans le scan actuel.

## 2. Comparatif des latences

Environnement : x86_64, Python 3.12.16.
Méthode : 1 000 appels par backend, après 10 appels de chauffe.

| Backend | P50 (ms) | P95 (ms) | P99 (ms) |
|---|---:|---:|---:|
| sklearn_pipeline | 4.531 | 8.085 | 12.421 |
| onnx_python | 0.079 | 0.147 | 0.248 |
| onnx_batch_100 | 0.291 | 0.427 | 0.592 |

Accélération ONNX par rapport à scikit-learn (P50, une ligne) : 57.2x.

## 3. Test de charge avec Locust

Le test de charge a produit les résultats suivants :

- Requêtes totales : 19 271
- Requêtes échouées : 0
- Débit moyen : 687.05 requêtes/seconde
- Temps de réponse moyen : 95 ms
- P50 : 100 ms
- P95 : 150 ms
- Temps maximal : 285 ms

Aucune requête n'a échoué pendant ce test.

## 4. Interprétation

**Pourquoi l'image optimisée est-elle plus petite ?**

L'image naive utilise une image Python complète et installe les dépendances
directement dans l'environnement final.

L'image optimisée utilise une construction multi-stage avec deux étapes :
- Un stage builder pour installer les dépendances.
- Un stage runtime basé sur `python:3.12-slim`, qui ne conserve que les
  dépendances et les fichiers nécessaires à l'exécution.

Le `.dockerignore` réduit également le contexte de construction. Les fichiers
sont copiés explicitement afin d'éviter d'intégrer des éléments sensibles ou
inutiles comme `.env`.

Le résultat est une image de 535 MB contre 2.11 GB pour l'image naive.

**Pourquoi ONNX est-il plus rapide que scikit-learn ?**

ONNX Runtime exécute le modèle dans un format optimisé pour l'inférence. Il
réduit les surcoûts liés à l'exécution du pipeline Python et utilise un moteur
d'exécution optimisé.

Le benchmark montre une accélération de 57.2x au P50 pour une prédiction
individuelle. Le traitement par lots permet également de traiter plusieurs
observations ensemble.

**Que montrent les scans Trivy et ModelScan ?**

Trivy montre que l'image optimisée possède le meilleur résultat parmi les
images naive et optimisée : 53 vulnérabilités HIGH et aucune CRITICAL,
contre 192 HIGH et 3 CRITICAL pour l'image naive.

Les scans Python détectent 6 vulnérabilités dans chaque image, dont aucune
HIGH ou CRITICAL. Les vulnérabilités les plus importantes proviennent donc
principalement de la couche système dans ces résultats.

ModelScan a détecté un opérateur dangereux `system` dans le pickle malveillant
de démonstration, ce qui confirme le risque d'exécution de code lors du
chargement de fichiers pickle non fiables.

Le scan ONNX n'a trouvé aucun problème, mais le fichier ONNX a été ignoré par
ModelScan : il n'a donc pas réellement été analysé.

Le scan du modèle enregistré au format skops est non concluant à cause de
11 erreurs de compatibilité entre ModelScan et NumPy. L'analyse des types
non fiables avec skops a identifié uniquement `numpy.dtype`, type déjà
autorisé dans le Lab 1.

## 5. Vérification de l'API et des images Docker



Les deux images naive et optimisée ont été reconstruites et testées.

- L'endpoint `/health` retourne un statut HTTP 200.
- L'endpoint `/predict` retourne une prédiction valide.
- Les deux images produisent exactement la même prédiction pour le même échantillon.
- Le fichier `.env` est absent des deux images.
- Les deux images fonctionnent avec un utilisateur non-root (UID 10001).
- Le healthcheck Docker de l'image optimisée fonctionne correctement.

L'image optimisée a également été publiée dans le registre local puis téléchargée
à nouveau avec succès.
## 6. Choix recommandé

Pour ce cas d'usage, nous recommandons `scoring-api:optimized` en production.

Cette image offre le meilleur compromis entre taille, sécurité et performances.
Elle utilise une base Python slim, un build multi-stage, des dépendances
verrouillées, un utilisateur non-root (UID 10001) et un healthcheck.
Contrairement à l'image naive, elle réduit également la surface d'attaque
grâce à une image de base plus légère.
Les dépendances sont installées à partir d'un fichier `requirements.lock`
avec des versions fixées et des hashes vérifiés.

Elle embarque le modèle ONNX et expose une API FastAPI testée localement et
avec Locust. Les prédictions ont été vérifiées par comparaison avec le modèle
scikit-learn.

Même si sa taille de 535 MB dépasse légèrement l'objectif de 500 MB, elle reste
beaucoup plus légère que l'image naive et présente moins de vulnérabilités
dans les scans effectués.

## 7. Pistes d'amélioration

- Appliquer la quantification INT8 du modèle ONNX pour réduire sa taille et
  accélérer l'inférence, tout en vérifiant la conservation de la qualité
  des prédictions.
- Étudier une image de base distroless pour réduire davantage la surface
  d'attaque.
- Réduire encore la taille de l'image en analysant les dépendances réellement
  nécessaires et en supprimant les composants inutilisés.
- Mettre en place un Init Container pour précharger le modèle depuis un
  registre externe lorsque le modèle n'est plus intégré directement dans
  l'image.
- Automatiser les scans de sécurité et les tests de performance dans la CI/CD.
