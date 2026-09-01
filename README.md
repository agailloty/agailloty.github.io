# website

Site personnel construit avec Hugo et le thème [Kyrie](https://github.com/agailloty/kyrie).

## Installer le thème

Kyrie est référencé comme un sous-module Git. Après avoir cloné ce dépôt, initialisez-le avec :

```bash
git submodule update --init --recursive
```

Le dépôt principal enregistre un commit précis du thème. Cette commande installe donc automatiquement la même version de Kyrie que celle utilisée par le site.

## Utiliser une version spécifique de Kyrie

Pour sélectionner une version publiée, récupérez les tags puis placez le sous-module sur le tag souhaité :

```bash
git -C themes/kyrie fetch --tags
git -C themes/kyrie checkout v0.3.0
```

Enregistrez ensuite la nouvelle référence du sous-module dans le dépôt du site :

```bash
git add themes/kyrie
git commit -m "Update Kyrie to v0.3.0"
```

Pour utiliser une autre version, remplacez simplement `v0.3.0` par le tag voulu. Après avoir récupéré des changements du dépôt principal, synchronisez le thème avec la version qui y est enregistrée :

```bash
git submodule update --init --recursive
```
