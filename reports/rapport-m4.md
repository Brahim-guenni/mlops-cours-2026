# Rapport — Lab 4 : packaging du modèle

Binôme : À compléter · Dataset : `german-credit` · Modèle : `scoring-model` v3

## 1. Comparatif des tailles

| Image | Taille | CVE HIGH + CRITICAL |
|---|---:|---:|
| `scoring-api:naive` | 2.16 GB | 195 |
| `scoring-api:optimized` | 535 MB | 53 |
| `scoring-api:bentoml` | 1.41 GB | 96 |

L'image optimisée est environ 75 % plus petite que l'image naive.
Elle présente également le nombre de vulnérabilités le plus faible :
53 vulnérabilités HIGH et aucune CRITICAL.

## 2. Comparatif des latences

Environnement : x86_64, Python 3.12.16.
Méthode : 1 000 appels par backend, après 10 appels de chauffe.

| Backend | P50 (ms) | P95 (ms) | P99 (ms) |
|---|---:|---:|---:|
| sklearn_pipeline | 4.531 | 8.085 | 12.421 |
| onnx_python | 0.079 | 0.147 | 0.248 |
| onnx_batch_100 | 0.291 | 0.427 | 0.592 |

Accélération ONNX par rapport à scikit-learn (P50, une ligne) : 57.2x.

## 3. Interprétation

**Pourquoi l'image optimisée est-elle plus petite ?**

L'image naive utilise une image Python complète et installe les dépendances directement dans
l'environnement final. Elle contient également des fichiers inutiles au fonctionnement de l'API.

L'image optimisée utilise une construction multi-stage avec deux étapes :
- Un stage builder pour installer les dépendances.
- Un stage runtime basé sur `python:3.12-slim`, qui ne conserve que les dépendances et les fichiers nécessaires à l'exécution.

Le `.dockerignore` réduit également le contexte de construction. Le résultat est une image
plus légère, de 535 MB contre 2.16 GB pour l'image naive.

**Pourquoi ONNX est-il plus rapide que scikit-learn ?**

ONNX Runtime exécute le modèle dans un format optimisé pour l'inférence. Il réduit les
surcoûts liés à l'exécution du pipeline Python et utilise un moteur d'exécution optimisé.

Le benchmark montre une accélération de 57.2x au P50 pour une prédiction individuelle.
Le traitement par lots permet également de traiter plusieurs observations ensemble.

**Que montrent les scans Trivy et ModelScan ?**

Trivy montre que l'image optimisée possède le meilleur résultat parmi les trois images :
53 vulnérabilités HIGH, contre 195 pour l'image naive et 96 pour BentoML.
Aucune vulnérabilité CRITICAL n'a été détectée dans les images optimisée et BentoML,
contre 3 dans l'image naive.

ModelScan a détecté un opérateur dangereux `system` dans le pickle malveillant de démonstration,
ce qui confirme le risque d'exécution de code lors du chargement de fichiers pickle non fiables.

Le scan ONNX n'a trouvé aucun problème, mais le fichier ONNX a été ignoré par ModelScan :
il n'a donc pas réellement été analysé.

Le scan du modèle enregistré au format skops est non concluant à cause de 11 erreurs de
compatibilité entre ModelScan et NumPy. L'analyse des types non fiables avec skops a identifié
uniquement `numpy.dtype`, type déjà autorisé dans le Lab 1.

## 4. Choix recommandé

Pour ce cas d'usage, nous recommandons `scoring-api:optimized` en production.

Cette image offre le meilleur compromis entre taille, sécurité et performances.
Elle utilise une base Python slim, un build multi-stage, des dépendances limitées,
un utilisateur non-root (UID 10001) et un healthcheck.

Elle embarque le modèle ONNX et expose une API FastAPI testée localement et avec Locust.
Les prédictions ont été vérifiées par comparaison avec le modèle scikit-learn.

Même si sa taille de 535 MB dépasse légèrement l'objectif de 500 MB,
elle reste beaucoup plus légère que l'image naive et présente le moins de vulnérabilités.

## 5. Pistes d'amélioration

- Appliquer la quantification INT8 du modèle ONNX pour réduire sa taille et accélérer
  l'inférence, tout en vérifiant la conservation de la qualité des prédictions.
- Étudier une image de base distroless pour réduire davantage la surface d'attaque.
- Réduire encore la taille de l'image en analysant les dépendances réellement nécessaires
  et en supprimant les composants inutilisés.
- Mettre en place un Init Container pour précharger le modèle depuis un registre externe
  lorsque le modèle n'est plus intégré directement dans l'image.
- Automatiser les scans de sécurité et les tests de performance dans la CI/CD.
