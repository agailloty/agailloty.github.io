---
title: "uv : moderniser son projet Python"
description: "Présenter uv, le gestionnaire de paquets Python révolutionnaire"
date: 2025-04-26T20:12:07+01:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [python, devops]
---

Cela fait maintenant un an que je travaille avec `uv`, ce gestionnaire de paquets qui révolutionne l'utilisation de Python.

# C'est quoi `uv` ?

`uv` est un gestionnaire de paquets Python open-source écrit en Rust par la société [Astral](https://astral.sh/).

`uv` se veut être un outil performant, facile à prendre en main et portable.

Dans cet article je vais partager trois cas fréquents de mon utilisation de `uv`. 

# Créer un projet avec une version spécifique de Python

`uv` est un exécutable que vous installez sur votre ordinateur et qui est indépendant de Python contrairement à pip. Ce côté portable de `uv` est très pratique et permet de créer un projet Python en spécifiant la version du langage.  

Voici un exemple :

```python
uv init --python 3.11 .
```

`uv` génère l'arborescence d'un projet qui ressemble à ceci : 

```sh
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        26/04/2025     21:45              5 .python-version
-a----        26/04/2025     21:45             84 hello.py
-a----        26/04/2025     21:45            152 pyproject.toml
-a----        26/04/2025     21:45              0 README.md
```

Il génère par défaut un fichier `hello.py` qui contient un script test Python à lancer. Vous pouvez bien entendu le supprimer.

## Lancer le programme avec `uv run`

```sh
uvdemo> uv run .\hello.py
Using Python 3.11.10
Creating virtual environment at: .venv
Hello from uvdemo!
```

Lorsque je lance le projet avec la commande `uv run .\hello.py`, automatiquement la version du langage Python que j'ai spécifiée est téléchargée pour lancer mon code. 

Je trouve ça assez extraordinaire : spécifier dans mon projet la version de Python que je veux sans avoir à l'installer manuellement est vraiment cool. 

Dans l'arborescence des fichiers générés par `uv init` on peut voir un fichier `.python-version` qui contient une chaîne de caractères correspondant à la version de Python. 

Il y a aussi un fichier `pyproject.toml` qui vous permet de spécifier les dépendances. Ce fichier n'est pas spécifique à `uv`. Le gestionnaire de paquet poetry l'utilise également. 

# Gérer correctement les dépendances de mon code 

Pour la démo, supposons que je souhaite créer un outil CLI en Python qui dépend de la version 0.7.0 de fire. 
Simplement avec cette ligne de commande j'ajoute fire à mon projet. 

```sh
uv add fire==0.7.0
```

En arrière-plan `uv` a fait des choses pour moi afin que la version de fire que j'utilise soit la même si je veux reconstruire le projet plus tard ou bien qu'un autre développeur veut lancer mon code. 

Le fichier `pyproject.toml` a été mis à jour pour inclure les dépendances : 

```toml
[project]
name = "uvdemo"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "fire==0.7.0",
]
```

Plus intéressant, un fichier `uv.lock` a été ajouté à l'arborescence du projet pour contenir des métadonnées nécessaires à l'installation de cette version spécifique de fire. Cela garantit que c'est exactement cette version de fire ainsi que les versions exactes des packages Python sur lesquels dépend fire==0.7.0 qui seront installés quand je lance mon projet. 

# Partager un script avec toutes les dépendances

Cette fonctionnalité rend à Python toute sa gloire d'un langage de script. Le nom de cette fonctionnalité : **inline script metadata**

Voici un exemple d'un script qui utilise le package textual pour afficher l'heure actuelle dans votre console.

```python
# /// script
# dependencies = [
#   "textual",
# ]
# ///
"""
An App to show the current time.
"""

from datetime import datetime

from textual.app import App, ComposeResult
from textual.widgets import Digits


class ClockApp(App):
    CSS = """
    Screen { align: center middle; }
    Digits { width: auto; }
    """

    def compose(self) -> ComposeResult:
        yield Digits("")

    def on_ready(self) -> None:
        self.update_clock()
        self.set_interval(1, self.update_clock)

    def update_clock(self) -> None:
        clock = datetime.now().time()
        self.query_one(Digits).update(f"{clock:%T}")


if __name__ == "__main__":
    app = ClockApp()
    app.run()
```

```sh
uvdemo> uv run .\textual_script.py
Reading inline script metadata from: .\textual_script.py
Installed 10 packages in 435ms
```

Cette fonctionnalité est vraiment pratique lorsque vous partagez des scripts. L'utilisateur de votre script n'a pas à installer toutes les dépendances avant de lancer votre projet. Tout est spécifié dans le script lui-même et `uv` se charge du reste.
