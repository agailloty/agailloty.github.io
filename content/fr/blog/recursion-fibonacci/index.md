---
title: "La récursion, expliquée en douceur (avec Fibonacci et ses amis)"
description: "Un tuto bienveillant pour débutants sur la récursion en C# : comment ça marche, pourquoi Fibonacci est l'exemple parfait, et comment corriger son coût caché."
date: 2026-09-24T10:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [algorithms, recursion, csharp, dotnet, programming]
---

Si vous avez déjà ouvert un livre d'algorithmique, vous avez rencontré la récursion. Et si le mot seul vous rend nerveux, c'est parfaitement normal : la récursion fait partie de ces idées qui semblent étrangères jusqu'au jour où elles cliquent d'un coup, après quoi vous les voyez partout.

Cet article est écrit pour ce parcours. Aucun prérequis au-delà des bases du C#. Prenez votre temps. Personne ne comprend la récursion du premier coup.

## La récursion, c'est quoi au juste ?

Une fonction récursive est une fonction qui s'appelle elle-même. C'est toute la définition, et aussi tout le problème : ça paraît circulaire. Une fonction qui s'appelle elle-même, comme un serpent qui se mord la queue — comment cela pourrait-il produire quoi que ce soit d'utile ?

L'astuce, c'est que chaque appel travaille sur une version un peu plus petite du même problème. Imaginez que vous êtes dans une longue file d'attente et que vous voulez connaître votre position. Vous demandez à la personne devant vous : « quelle est ta position ? » Elle demande à celle devant elle, et ainsi de suite. La personne en tête répond « je suis premier ». Puis les réponses remontent : deuxième, troisième... jusqu'à ce que votre voisin vous donne son numéro et que vous ajoutiez un.

C'est de la récursion à l'état sauvage :

- Quelqu'un doit arrêter la chaîne et donner une réponse simple. Cette personne, c'est le cas de base.
- Tous les autres résolvent le problème en s'appuyant sur une version plus petite. C'est le cas récursif.

## L'anatomie d'une fonction récursive

Presque toutes les fonctions récursives que vous écrirez suivent cette forme :

```csharp
int Resoudre(Probleme probleme)
{
    if (probleme.EstAssezPetit)          // cas de base
        return ReponseEvidente;
    var plusPetit = Reduire(probleme);   // progresser
    return Combiner(Resoudre(plusPetit)); // cas récursif
}
```

Trois règles, qui méritent d'être retenues :

1. Un cas de base qui renvoie une réponse sans s'appeler lui-même.
2. Un cas récursif qui s'appelle sur un problème strictement plus petit.
3. Le progrès : chaque appel doit se rapprocher du cas de base.

Enfreignez la règle 1 et vous obtenez une récursion infinie et une StackOverflowException. Enfreignez la règle 2 ou 3 et la fonction ne converge jamais. Respectez-les et, assez remarquablement, la logique tient toute seule.

## Un premier exemple tout doux : la factorielle

La factorielle de n (notée n!) est le produit de tous les entiers de 1 à n. Donc 5! = 5 x 4 x 3 x 2 x 1 = 120.

Il y a une version avec une boucle, et une version récursive :

```csharp
long FactorielleBoucle(int n)
{
    long resultat = 1;
    for (int i = 2; i <= n; i++)
        resultat *= i;
    return resultat;
}

long Factorielle(int n)
{
    if (n <= 1)                        // cas de base
        return 1;
    return n * Factorielle(n - 1);    // cas récursif
}
```

Lisez la version récursive à voix haute : « la factorielle de n, c'est n fois la factorielle de n moins 1, et la factorielle de 1, c'est 1. » C'est littéralement la définition mathématique. Voilà le super-pouvoir discret de la récursion : elle permet de traduire une définition directement en code, sans effort de traduction. La version boucle calcule ; la version récursive déclare.

Faites une fois la trace à la main pour Factorielle(3), sur papier :

```text
Factorielle(3)
= 3 * Factorielle(2)
= 3 * (2 * Factorielle(1))
= 3 * (2 * 1)
= 6
```

Si vous savez faire cette trace, vous comprenez la récursion mieux que vous ne le croyez.

## Fibonacci : le grand classique, et son piège caché

La suite de Fibonacci est la vedette de la récursion. Chaque terme est la somme des deux précédents :

```text
0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...
```

En définition :

- F(0) = 0
- F(1) = 1
- F(n) = F(n - 1) + F(n - 2) pour n >= 2

Traduit en C#, presque mot pour mot :

```csharp
long Fib(int n)
{
    if (n < 2)                          // cas de base
        return n;
    return Fib(n - 1) + Fib(n - 2);     // deux appels récursifs
}
```

Six lignes. Élégant. On dirait que la définition est tombée du manuel directement dans votre éditeur. Essayez Fib(10) : instantané. Fib(25) : très bien. Fib(40) ? Vous pouvez aller préparer un café. Sur un ordinateur portable classique, Fib(40) prend plusieurs secondes et Fib(50) prendra plus de temps que vous n'accepterez d'attendre. Quelque chose cloche profondément, et il vaut la peine de comprendre exactement quoi, car c'est l'une des leçons les plus précieuses de toute l'algorithmique.

### Pourquoi est-ce si lent ?

Le problème est que Fib s'appelle deux fois, et que ces appels refont sans cesse le même travail. Regardez ce qui se passe pour Fib(5) :

```text
                Fib(5)
              /        \
          Fib(4)        Fib(3)
         /      \       /    \
     Fib(3)   Fib(2)  Fib(2) Fib(1)
     /   \     /  \    /  \
 Fib(2) Fib(1) ...  ... ...
```

Fib(3) est calculé deux fois. Fib(2) est calculé trois fois. Et plus on descend, plus ça s'aggrave : le nombre d'appels croît de façon exponentielle, en se multipliant environ par 1,6 à chaque niveau. Calculer Fib(50) naïvement demande de l'ordre de vingt milliards d'appels de méthode. Pour produire un seul nombre.

Ce n'est ni un problème de C#, ni un problème de syntaxe. C'est un problème de conception, et il enseigne la leçon générale : une fonction récursive qui se divise en plusieurs appels avec des sous-problèmes qui se recouvrent explose, sauf si vous gérez ce recouvrement.

### Le remède : la mémoïsation

L'observation est simple : la réponse de Fib(2) ne change jamais. Une fois calculée, pourquoi la recalculer ? La mémoïsation consiste à tenir un petit carnet (un cache) des résultats déjà calculés. En C#, l'outil naturel est un Dictionary :

```csharp
long FibMemo(int n, Dictionary<int, long>? cache = null)
{
    cache ??= new Dictionary<int, long>();

    if (cache.TryGetValue(n, out var dejaCalcule))   // déjà calculé ?
        return dejaCalcule;

    long resultat;
    if (n < 2)
        resultat = n;
    else
        resultat = FibMemo(n - 1, cache) + FibMemo(n - 2, cache);

    cache[n] = resultat;                             // on l'écrit dans le carnet
    return resultat;
}
```

Désormais chaque valeur de 0 à n est calculée exactement une fois. FibMemo(50) répond instantanément. FibMemo(1000) aussi. Nous sommes passés de milliards d'appels à quelques milliers, avec trois lignes en plus.

Une mise en garde propre à ce motif : la version naïve « cache en champ statique » n'est pas thread-safe. Si plusieurs threads appellent la méthode en même temps, le Dictionary peut être corrompu. Pour des caches partagés, utilisez plutôt ConcurrentDictionary :

```csharp
private static readonly ConcurrentDictionary<int, long> _cache = new();

long FibMemoThreadSafe(int n)
{
    if (n < 2)
        return n;
    return _cache.GetOrAdd(n, k => FibMemoThreadSafe(k - 1) + FibMemoThreadSafe(k - 2));
}
```

### L'autre remède : partir du bas

La mémoïsation est descendante : on part de n et on fait confiance au cache en descendant. L'alternative est montante : on part des cas de base et on remonte. À ce moment-là, on réécrit une boucle, mais une boucle maligne qui ne garde que le nécessaire :

```csharp
long FibIter(int n)
{
    if (n < 2)
        return n;
    long precedent = 0, courant = 1;
    for (int i = 2; i <= n; i++)
        (precedent, courant) = (courant, precedent + courant);
    return courant;
}
```

Deux variables, une boucle, une mémoire constante, plus de récursion du tout. Cette technique s'appelle la programmation dynamique, et Fibonacci en est l'introduction la plus douce. L'idée clé est la même dans les deux versions : ne jamais calculer deux fois la même chose.

Alors, laquelle utiliser ? La version récursive naïve est merveilleuse pour apprendre et pour expliquer la définition. Les versions mémoïsée ou itérative sont celles qu'on utilise en vrai. Savoir dans quel contexte on se trouve fait partie du métier.

## Les autres endroits où la récursion brille

Fibonacci est, en un sens, un jouet : personne ne le calcule récursivement en production. Mais la forme du problème qu'il illustre — « une chose définie à partir de versions plus petites d'elle-même » — est partout. Voici trois exemples réalistes.

### La recherche dichotomique

Dans un tableau trié, trouver un élément. On compare au milieu : soit on l'a trouvé, soit on peut jeter la moitié du tableau et chercher dans la moitié restante. Une version plus petite du même problème, par définition :

```csharp
int RechercheDichotomie(int[] items, int cible, int bas, int haut)
{
    if (bas > haut)                                // cas de base : intervalle vide
        return -1;
    int milieu = (bas + haut) / 2;
    if (items[milieu] == cible)                    // cas de base : trouvé
        return milieu;
    if (items[milieu] < cible)                     // chercher dans la moitié droite
        return RechercheDichotomie(items, cible, milieu + 1, haut);
    return RechercheDichotomie(items, cible, bas, milieu - 1);  // moitié gauche
}

// Premier appel : RechercheDichotomie(tableau, valeur, 0, tableau.Length - 1)
```

Chaque appel divise l'espace de recherche par deux. Un million d'éléments signifie une vingtaine d'appels, pas un million de comparaisons. Remarquez comme la structure récursive rend la logique « jeter la moitié » explicite et lisible. (Oui, Array.BinarySearch existe ; l'écrire une fois soi-même, c'est comprendre ce qu'elle fait.)

### Parcourir un arbre (ou un dossier)

Les boucles sont naturelles pour les tableaux et les listes, qui sont plats. Mais certaines données sont naturellement imbriquées : des répertoires contenant des répertoires, des documents JSON contenant des objets contenant des tableaux, des commentaires contenant des réponses contenant des réponses. Pour les données imbriquées, la récursion n'est pas une astuce élégante — c'est la traduction honnête de la forme de la donnée :

```csharp
void ListerFichiers(string dossier)
{
    foreach (var entree in Directory.EnumerateFileSystemEntries(dossier))
    {
        if (File.Exists(entree))
            Console.WriteLine(entree);          // cas de base : un fichier
        else
            ListerFichiers(entree);              // cas récursif : un dossier
    }
}
```

Essayez d'écrire cela avec des boucles seulement, sans récursion, en gérant une profondeur arbitraire. Il vous faudra gérer une pile à la main. La version récursive laisse la pile d'appels faire cette comptabilité pour vous. C'est la vérité générale ici : la récursion et les données imbriquées sont faites l'une pour l'autre.

Si vous craignez des hiérarchies de dossiers si profondes qu'elles débordent la pile, .NET permet de convertir toute récursion en pile explicite, et System.IO.Enumeration offre un parcours itératif. Mais ne sortez ces outils que quand la profondeur est réellement non bornée.

### Les tours de Hanoï

Le casse-tête classique : déplacer une pile de disques d'une tige vers une autre, un disque à la fois, sans jamais poser un grand disque sur un plus petit. En itératif, la solution embrouille. En récursif, elle est presque embarrassante :

```csharp
void Hanoi(int n, char source, char cible, char auxiliaire)
{
    if (n == 0)                                // cas de base : rien à déplacer
        return;
    Hanoi(n - 1, source, auxiliaire, cible);   // déplacer la pile du dessus
    Console.WriteLine($"Déplacer le disque {n} de {source} vers {cible}");
    Hanoi(n - 1, auxiliaire, cible, source);   // la remettre par-dessus
}
```

Lisez-le comme une histoire : pour déplacer n disques, déplacez les n - 1 disques du-dessus à l'écart, déplacez le grand disque, puis remettez la pile par-dessus. Trois lignes de logique qui prendraient une page en boucles. Certains problèmes sont simplement récursifs par nature, et les combattre ne sert à rien.

## Les erreurs classiques (tout le monde les fait)

- Oublier le cas de base. La méthode s'appelle indéfiniment et le programme plante avec une StackOverflowException. Si vous voyez cette exception, vérifiez d'abord le cas de base ; neuf fois sur dix, il manque ou n'est jamais atteint.
- Un cas de base jamais atteint. Méfiez-vous des conditions qui sautent par-dessus le cas de base, comme soustraire 2 à chaque fois d'un nombre impair en testant l'égalité avec 0. Préférez les inégalités (n <= 1) aux égalités strictes (n == 1), par sécurité.
- Aucun progrès. Si l'appel récursif reçoit le même argument, ou un plus grand, le cas de base ne sera jamais atteint.
- Des appels qui se ramifient en vain. Si un appel engendre plusieurs copies qui résolvent des sous-problèmes qui se recouvrent (bonjour, le Fibonacci naïf), il faut de la mémoïsation ou une autre stratégie.

## Quand atteindre la récursion ?

Utilisez la récursion quand le problème lui-même est auto-similaire : arbres, structures imbriquées, diviser-pour-régner (recherche, tri, parcours), casse-têtes définis à partir de versions plus petites d'eux-mêmes. Utilisez les boucles quand la donnée est plate et séquentielle. Aucune n'est « meilleure » : ce sont des outils pour des formes différentes de problèmes. Et rappelez-vous qu'une solution récursive élégante mais trop lente peut presque toujours être transformée en solution rapide par mémoïsation ou réécriture montante, sans perdre la clarté de l'idée.

## Pour pratiquer, en douceur

Si vous voulez que ça s'ancre, voici quelques exercices par difficulté croissante :

1. Écrivez une méthode récursive qui décompte de n à 0 en affichant chaque nombre.
2. Écrivez une somme récursive d'un tableau de nombres (indice : la somme, c'est le premier élément plus la somme du reste).
3. Écrivez une méthode récursive qui inverse une chaîne de caractères.
4. Calculez la profondeur d'un arbre d'objets imbriqués (le nombre de niveaux qu'il contient).
5. Reprenez le Fibonacci naïf et ajoutez la mémoïsation vous-même, sans regarder cet article.

Faites-les d'abord sur papier, en traçant les appels comme nous l'avons fait pour la factorielle. C'est le traçage, pas la frappe, qui fait naître la compréhension.

## Pour finir

La récursion n'est pas une technique à mémoriser ; c'est un changement de point de vue. Cessez de demander « comment calculer cela pas à pas » et demandez plutôt « comment ce problème est-il construit à partir de versions plus petites de lui-même ». Fibonacci, les arbres, Hanoï — tous récompensent la même question. Posez-la assez souvent et elle devient un réflexe, et un jour vous surprendrez vous-même à voir un document JSON imbriqué comme un petit arbre amical, qui n'attend que d'être parcouru.
