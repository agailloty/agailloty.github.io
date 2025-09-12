---
authors: [Axel-Cleris Gailloty]
title: Coder en C# avec Jupyter Notebook
description: Manipulation de documents HTML avec HtmlAgilityPack
date: 2024-04-15T18:05:00Z
categories: [Tutos]
tags: [dotnet, Csharp, docker]
resources:
  - src: featured.png
    name: featured
    params:
      alt: "Docker and Jupyter logos"
      attr: "towardsdatascience"
      attrlink: https://towardsdatascience.com/docker-jupyter-for-machine-learning-in-1-minute-30e1df969d09
---
Ayant beaucoup utilisé les notebooks Jupyter, dans l'environnement data science de Python et R j'ai réalisé que l'intégration de langages comme C# et F# dans ce même environnement serait incroyablement puissante.
{.p-first} <!--more-->

Pouvoir expérimenter rapidement avec différentes syntaxes et concepts de programmation est une un moyen rapide de prendre en main un langage de programmation et représente un gain de temps énorme. En tant que passionné de programmation, je me trouve souvent à explorer de nouveaux langages et à tester différentes approches pour résoudre des problèmes. Récemment, j'ai eu besoin d'un moyen simple et rapide de tester des syntaxes en C# et en F# sans avoir à créer un projet .NET complet à chaque fois. C'est là que Jupyter Lab et Docker ont fait leur entrée.

Ayant beaucoup utilisé les notebooks Jupyter, dans l'environnement data science de Python et R j'ai réalisé que l'intégration de langages comme C# et F# dans ce même environnement serait incroyablement puissante. J'ai donc entrepris de créer une image Docker contenant Jupyter Lab préconfiguré avec le support de .NET 8, me permettant ainsi de créer et d'exécuter facilement des blocs de code .NET dans un notebook.

Grâce à cette image Docker, je peux désormais lancer rapidement un environnement Jupyter Lab avec le support complet de .NET, sans avoir à créer un projet C# dans Visual Studio ou VS Code. Cela me permet d'expérimenter avec les syntaxes de C# et de F# en quelques secondes, ce qui accélère considérablement mon processus d'apprentissage et de test.

Le processus de création de cette image Docker a été relativement simple. J'ai commencé par installer le SDK .NET 8.0 et les outils .NET interactifs dans l'image Docker, en m'assurant de configurer correctement Jupyter Lab pour qu'il reconnaisse ces outils. Ensuite, j'ai publié cette image sur Docker Hub pour la rendre accessible à toute personne intéressée par l'expérimentation avec .NET dans un environnement de notebook.

# Le fichier Dockerfile

Voici le contenu que j'ai défini pour le fichier Dockerfile.

Il est bien évidemment disponible [sur ce référentiel GitHub dédié](https://github.com/agailloty/jupyter-dotnet).

```Dockerfile {.expand}
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

- **FROM jupyter/base-notebook:latest**: Cette ligne spécifie l'image de base à utiliser pour construire notre image Docker. Dans ce cas, nous utilisons une image de Jupyter Lab préexistante avec le support de .NET 8.

- **USER root**: Cette instruction permet de passer à l'utilisateur root pour exécuter les commandes suivantes en tant qu'administrateur.

- **RUN ...**: Cette série de commandes installe le SDK .NET 8.0 en téléchargeant les paquets nécessaires et en les installant sur l'image Docker.

- **RUN dotnet tool install --global Microsoft.dotnet-interactive**: Cette commande installe les outils .NET interactifs globalement sur l'image Docker, permettant ainsi d'utiliser les fonctionnalités interactives de .NET dans Jupyter Lab.

- **ENV PATH="${PATH}:~/.dotnet/tools"**: Cette instruction ajoute le répertoire contenant les outils .NET interactifs au PATH de l'utilisateur, ce qui permet de les exécuter facilement depuis n'importe quel emplacement.

- **RUN dotnet interactive jupyter install**: Cette commande installe les notebooks Jupyter pour .NET, permettant ainsi d'exécuter du code .NET dans des cellules de notebook Jupyter.

- **EXPOSE 8888**: Cette ligne expose le port 8888 sur lequel Jupyter Lab sera accessible depuis l'extérieur du conteneur Docker.

- **CMD ["jupyter", "lab", "--ip=0.0.0.0", "--allow-root"]**: Enfin, cette commande spécifie la commande à exécuter lorsque le conteneur Docker est lancé, démarrant ainsi Jupyter Lab.


Si vous êtes curieux de découvrir cette image Docker par vous-même, vous pouvez la trouver sur [mon référentiel GitHub dédié](https://github.com/agailloty/jupyter-dotnet). N'hésitez pas à la tester et à me faire part de vos commentaires ou suggestions d'amélioration. Je suis convaincu que cette approche simplifiée pour tester les syntaxes .NET sera utile à de nombreux développeurs cherchant à explorer de nouveaux langages et à améliorer leur efficacité dans leurs projets.

# Conclusion

Dans un monde où la vitesse et l'efficacité sont essentielles, avoir des outils qui permettent d'itérer rapidement et de tester de nouvelles idées est un atout précieux pour tout développeur. Avec cette image Docker, j'ai trouvé un moyen simple et efficace de faire exactement cela, et j'espère qu'elle sera également utile à d'autres passionnés de programmation dans leur parcours d'apprentissage et d'exploration.