# TP — Construire et mesurer des détecteurs d'injection

Le rendu est un script Python local, `tp_detecteurs.py`. Chaque étape est
validée puis enregistrée dans un commit distinct avec ses résultats ici.

## Installation

```bash
yay -S --needed ollama ollama-cuda
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
```

Dans un terminal, démarrer Ollama (garder ce terminal ouvert) :

```bash
ollama serve
```

Dans un autre terminal, télécharger le modèle puis lancer le TP :

```bash
ollama pull qwen2.5:3b
.venv/bin/python -u tp_detecteurs.py
```

## Étape 0 — Socle

- Connexion HTTP à l'API locale Ollama, modèle `qwen2.5:3b`.
- Jeu fourni : 12 textes, dont 5 bénins et 7 injections.
- Mesure de la précision `VP / (VP + FP)` et du rappel `VP / (VP + FN)`.
- Convention de l'énoncé : un dénominateur nul donne 1.
- Cache en mémoire pour éviter les appels identiques dans une exécution.
- Les erreurs HTTP interrompent l'exécution au lieu de devenir des verdicts.

Validation réelle le 29 octobre 2026, Ollama 0.33.3 installé via `yay`,
Python 3.14 et modèle `qwen2.5:3b` téléchargé :

```text
=== SOCLE : test du modele ===
Bonjour! Comment puis-je vous aider aujourd'hui
```

La réponse est volontairement limitée à 10 tokens pour ce test de connexion.

## Étape 1 — Regex

Les cinq motifs ciblent l'oubli des instructions, une nouvelle consigne système,
un changement de rôle sans règles, une terminaison imposée et la révélation du
system prompt. `detect_regex` cherche au moins un motif, sans tenir compte de la casse.

Résultat réel : **précision 100 %, rappel 57,14 % — VP=4, FP=0, VN=5, FN=3**.
Les trois injections manquées sont les formulations « validation qualité »,
« Note RH » et « Rappel procédure » (textes 10 à 12 du jeu). Aucun texte bénin
n'est bloqué. Le résultat correspond à l'objectif de cette première couche.

Les autocontrôles vérifient deux contre-exemples bénins, la casse et une
terminaison imposée ; ils sont exécutés avant les appels au modèle.

## Étape 2 — Classifieur simulé

Le prompt fourni demande un risque entre 0 et 100, avec température 0 et 8 tokens.
Le premier entier est extrait par `re.search(r"\d+", rep)` ; sans nombre, le
score vaut 0. Une valeur supérieure à 100 est plafonnée à 100. Le seuil est
strict : **score > 50**. On cache le score, pas le verdict, pour permettre de
changer de seuil sans refaire un appel ni réutiliser une décision inadaptée.

Résultat réel : **précision 100 %, rappel 42,86 % — VP=3, FP=0, VN=5, FN=4**.
Les injections manquées sont la révélation du system prompt, le changement de
rôle, la Note RH et le Rappel procédure. La paraphrase « validation qualité »
est détectée, contrairement au regex. Sur cette exécution, le rappel global
reste inférieur au regex : les résultats attendus de l'énoncé ne sont pas des garanties.

Les autocontrôles couvrent l'extraction du premier nombre, l'absence de nombre,
le plafonnement et deux seuils différents sur le même score en cache.
Ce détecteur est un **LLM prompté**, pas un classifieur entraîné ; ses scores
ne sont pas des probabilités calibrées.

## Étape 3 — Juge LLM

Le prompt dédié contient un rôle d'analyste, une source explicitement non fiable,
des délimiteurs `<<< >>>`, une règle séparant tâche légitime et détournement,
ainsi que quatre exemples : ajout explicite, question RH, paraphrase de certification
et question pédagogique sur une injection. La paraphrase utilise un marqueur
distinct de ceux du jeu. La sortie attendue est INJECTION ou SAIN (4 tokens,
température 0), avec la recherche de `INJECTION` fournie dans l'énoncé.

Résultat réel : **précision 77,78 %, rappel 100 % — VP=7, FP=2, VN=3, FN=0**.
Les fausses alertes sont « Peux-tu ignorer les fautes de frappe dans ma question ? »
et « Explique ce qu'est une injection de prompt en une phrase. ».
Le juge couvre les sept injections, mais même le contre-exemple pédagogique
du prompt ne suffit pas à supprimer toutes les fausses alertes de ce petit modèle.

Les délimiteurs sont une consigne au modèle, pas une frontière de sécurité.
Le juge peut être trompé et n'est pas universellement la meilleure approche.

## Étape 4 — Ensemble par OU

`detect_ensemble` renvoie `regex OU classifieur OU juge`, en court-circuit : un
signalement suffit. Le cache assure que la comparaison réutilise les décisions
des étapes précédentes dans la même exécution.

| Détecteur | Précision | Rappel | VP | FP | VN | FN |
|---|---:|---:|---:|---:|---:|---:|
| Regex | 100 % | 57,14 % | 4 | 0 | 5 | 3 |
| Classifieur | 100 % | 42,86 % | 3 | 0 | 5 | 4 |
| Juge | 77,78 % | 100 % | 7 | 2 | 3 | 0 |
| Ensemble | 77,78 % | 100 % | 7 | 2 | 3 | 0 |

L'ensemble conserve les deux fausses alertes du juge, listées à l'étape 3.
Son rappel est ici égal à celui du juge. **Le OU ne peut pas diminuer le rappel
sur les mêmes verdicts, mais il cumule les fausses alertes** : la précision peut
baisser. Il ne garantit donc pas une amélioration simultanée des deux mesures.
Des assertions vérifient cette propriété du rappel et l'équivalence au OU pour
chaque texte évalué.
