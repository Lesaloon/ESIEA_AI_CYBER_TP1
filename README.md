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
