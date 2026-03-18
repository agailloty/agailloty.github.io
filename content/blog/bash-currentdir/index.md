---
authors: [Axel-Cleris Gailloty]
title: "N'afficher que le répertoire courant dans bash"
description: Guide complet pour personnaliser l'invite de commande bash
date: 2026-03-18T19:59:07+01:00
categories: [linux]

---

Quand je travaille, j'aime avoir le moins de distractions possibles. Cela peut surprendre plus d'un mais si voir le chemin complet du répertoire dans lequel je me trouve dans VS Code me perturbe. 
Heureusement qu'il est tout à fait possible de modifier ce qui s'affiche dans le terminal lorsqu'on travaille avec `bash`. 

Dans ce petit article je vais vous montrer quelques astuces pour personnaliser les informations qui s'affichent dans votre bash. 

## Comprendre la variable PS1

L'invite de commande bash est contrôlée par la variable d'environnement `PS1` (Prompt String 1). Cette variable définit ce qui s'affiche avant votre curseur dans le terminal.

Pour voir votre configuration actuelle, tapez :

```bash
echo $PS1
```

Vous verrez probablement quelque chose comme :
```
\u@\h:\w\$
```

## Les codes de personnalisation du prompt

Voici les codes les plus utiles pour personnaliser votre prompt :

- `\u` : Nom d'utilisateur
- `\h` : Nom de l'hôte (ordinateur)
- `\w` : Chemin complet du répertoire courant
- `\W` : **Nom du répertoire courant uniquement** (ce que nous voulons)
- `\$` : Affiche `$` pour un utilisateur normal, `#` pour root
- `\t` : Heure actuelle (HH:MM:SS)
- `\d` : Date actuelle
- `\n` : Nouvelle ligne

## Afficher uniquement le répertoire courant

Pour n'afficher que le répertoire courant au lieu du chemin complet, il suffit de remplacer `\w` par `\W` dans votre PS1.

### Modification temporaire

Pour tester immédiatement, tapez dans votre terminal :

```bash
PS1='\W\$ '
```

Votre prompt affichera maintenant uniquement le nom du répertoire courant. Par exemple :
- Au lieu de : `utilisateur@ordinateur:~/Documents/projets/monprojet$`
- Vous verrez : `monprojet$`

### Configuration minimaliste

Personnellement, j'utilise cette configuration encore plus simple :

```bash
PS1='\W > '
```

Ce qui donne un prompt très épuré :
```
monprojet > 
```

## Rendre les changements permanents

Les modifications ci-dessus sont temporaires et disparaissent quand vous fermez le terminal. Pour les rendre permanentes, vous devez modifier votre fichier de configuration bash.

### Étapes pour une configuration permanente

1. **Ouvrez votre fichier `.bashrc`** (situé dans votre répertoire personnel) :

```bash
nano ~/.bashrc
```

2. **Ajoutez votre configuration PS1** à la fin du fichier :

```bash
# Personnalisation du prompt
PS1='\W > '
```

3. **Sauvegardez et fermez** le fichier (Ctrl+O puis Ctrl+X avec nano)

4. **Rechargez la configuration** :

```bash
source ~/.bashrc
```

Maintenant, votre prompt personnalisé sera appliqué à chaque nouvelle session terminal.

## Autres personnalisations utiles

### Ajouter des couleurs et du formatage

Vous pouvez enrichir votre prompt avec des couleurs et différents styles de texte en utilisant des codes ANSI.

#### Codes de couleur

```bash
PS1='\[\033[01;32m\]\W\[\033[00m\] > '
```

Cela affichera le nom du répertoire en vert.

Codes de couleur utiles :
- `\[\033[00;31m\]` : Rouge
- `\[\033[00;32m\]` : Vert
- `\[\033[00;33m\]` : Jaune
- `\[\033[00;34m\]` : Bleu
- `\[\033[00;35m\]` : Magenta
- `\[\033[00;36m\]` : Cyan
- `\[\033[00;37m\]` : Blanc
- `\[\033[00m\]` : Réinitialiser tous les styles

#### Mettre le texte en gras

Pour mettre le nom du dossier en gras, utilisez le code `01` :

```bash
PS1='\[\033[01m\]\W\[\033[00m\] > '
```

Le `01` active le mode gras. Vous pouvez aussi combiner gras et couleur :

```bash
PS1='\[\033[01;34m\]\W\[\033[00m\] > '
```

Cela affiche le répertoire en **bleu et gras**.

#### Autres styles de formatage

Codes de style disponibles :
- `\[\033[01m\]` : **Gras** (ou intensité élevée)
- `\[\033[02m\]` : Atténué/Dim
- `\[\033[03m\]` : *Italique* (pas supporté sur tous les terminaux)
- `\[\033[04m\]` : Souligné
- `\[\033[05m\]` : Clignotant (lent)
- `\[\033[07m\]` : Inversé (inverse la couleur de fond et de texte)
- `\[\033[00m\]` : Réinitialiser tous les styles

#### Exemples de combinaisons populaires

**Répertoire en gras cyan :**
```bash
PS1='\[\033[01;36m\]\W\[\033[00m\] > '
```

**Répertoire en gras vert avec flèche colorée :**
```bash
PS1='\[\033[01;32m\]\W\[\033[00m\] \[\033[01;34m\]>\[\033[00m\] '
```

**Répertoire souligné et en jaune :**
```bash
PS1='\[\033[04;33m\]\W\[\033[00m\] > '
```

**Style moderne avec émojis :**
```bash
PS1='📁 \[\033[01;35m\]\W\[\033[00m\] ▶ '
```

**Couleur de fond personnalisée :**

Vous pouvez aussi changer la couleur de fond :
- `\[\033[40m\]` : Fond noir
- `\[\033[41m\]` : Fond rouge
- `\[\033[42m\]` : Fond vert
- `\[\033[43m\]` : Fond jaune
- `\[\033[44m\]` : Fond bleu
- `\[\033[45m\]` : Fond magenta
- `\[\033[46m\]` : Fond cyan
- `\[\033[47m\]` : Fond blanc

Exemple avec fond et texte en gras :
```bash
PS1='\[\033[01;37;44m\]\W\[\033[00m\] > '
```

Cela affiche le répertoire en blanc gras sur fond bleu.

### Prompt sur deux lignes

Pour plus de lisibilité avec des commandes longues :

```bash
PS1='\W\n> '
```

Cela place le curseur sur une nouvelle ligne :
```
monprojet
> 
```

### Afficher l'heure et le répertoire

Si vous voulez garder une trace de l'heure :

```bash
PS1='[\t] \W > '
```

Résultat : `[14:23:45] monprojet >`

## Conclusion

La personnalisation du prompt bash est simple mais puissante. En utilisant `\W` à la place de `\w`, vous obtenez un affichage minimaliste qui réduit les distractions tout en gardant l'information essentielle : où vous vous trouvez.

N'hésitez pas à expérimenter avec différentes combinaisons pour trouver le prompt qui vous convient le mieux ! 

