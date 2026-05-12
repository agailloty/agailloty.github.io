---
title: "Coder en C# avec Jupyter Notebook"
description: "Coder en C# de manière interactive grâce à Jupyter notebook"
date: 2024-04-15T18:05:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, Csharp, docker]
---

Coder en C# de manière interactive grâce à Jupyter notebook.

Pouvoir expérimenter rapidement avec différentes syntaxes et concepts de programmation est un moyen rapide de prendre en main un langage de programmation et représente un gain de temps énorme. En tant que passionné de programmation, je me trouve souvent à explorer de nouveaux langages et à tester différentes approches pour résoudre des problèmes. Récemment, j'ai eu besoin d'un moyen simple et rapide de tester des syntaxes en C# et en F# sans avoir à créer un projet .NET complet à chaque fois. C'est là que Jupyter Lab et Docker ont fait leur entrée.

Ayant beaucoup utilisé les notebooks Jupyter dans l'environnement data science de Python et R, j'ai réalisé que l'intégration de langages comme C# et F# dans ce même environnement serait incroyablement puissante. J'ai donc entrepris de créer une image Docker contenant Jupyter Lab préconfiguré avec le support de .NET 8, me permettant ainsi de créer et d'exécuter facilement des blocs de code .NET dans un notebook.

Grâce à cette image Docker, je peux désormais lancer rapidement un environnement Jupyter Lab avec le support complet de .NET, sans avoir à créer un projet C# dans Visual Studio ou VS Code.

# Le fichier Dockerfile

Voici le contenu que j'ai défini pour le fichier Dockerfile.

Il est bien évidemment disponible [sur ce référentiel GitHub dédié](https://github.com/agailloty/jupyter-dotnet).

```Dockerfile
# Use a base image with Jupyter Lab and .NET 8 support
FROM jupyter/base-notebook:latest

# Passer à l'utilisateur root pour installer des packages sans sudo
USER root

# Installation du SDK .NET 8.0
RUN wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb && \
    dpkg -i packages-microsoft-prod.deb && \
    apt-get update && \
    apt-get install -y apt-transport-https && \
    apt-get update && \
    apt-get install -y dotnet-sdk-8.0

# Installation des outils .NET interactifs
RUN dotnet tool install --global Microsoft.dotnet-interactive

# Ajout des outils .NET interactifs au PATH
ENV PATH="${PATH}:~/.dotnet/tools"

# Installation des notebooks Jupyter pour .NET
RUN dotnet interactive jupyter install

# Exposer le port de Jupyter Lab
EXPOSE 8888

# Démarrer Jupyter Lab
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--allow-root"]
```

Explications des différentes parties :

- **FROM jupyter/base-notebook:latest** : Image de base Jupyter Lab avec support .NET 8.
- **USER root** : Passage à l'utilisateur root pour les commandes d'administration.
- **RUN ...** : Installation du SDK .NET 8.0 et des outils .NET interactifs.
- **ENV PATH** : Ajout des outils .NET interactifs au PATH.
- **RUN dotnet interactive jupyter install** : Installation des notebooks Jupyter pour .NET.
- **EXPOSE 8888** : Exposition du port 8888 pour Jupyter Lab.
- **CMD** : Démarrage de Jupyter Lab.

Si vous êtes curieux de découvrir cette image Docker par vous-même, vous pouvez la trouver sur [mon référentiel GitHub dédié](https://github.com/agailloty/jupyter-dotnet). N'hésitez pas à la tester et à me faire part de vos commentaires ou suggestions d'amélioration.

# Conclusion

Dans un monde où la vitesse et l'efficacité sont essentielles, avoir des outils qui permettent d'itérer rapidement et de tester de nouvelles idées est un atout précieux pour tout développeur. Avec cette image Docker, j'ai trouvé un moyen simple et efficace de faire exactement cela, et j'espère qu'elle sera également utile à d'autres passionnés de programmation dans leur parcours d'apprentissage et d'exploration.
