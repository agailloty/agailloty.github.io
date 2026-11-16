---
title: "La reconnaissance d'images Immich ne démarre plus ? Votre VM Proxmox est peut-être en cause"
description: "Le conteneur immich_machine_learning crashe en boucle avec « NumPy was built with baseline optimizations (X86_V2) ». Explication de la cause et du correctif en cinq minutes."
date: 2026-11-16T10:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: ["immich", "proxmox", "self-hosting", "docker", "dépannage"]
---

J'adore Immich. Auto-hébergé, gratuit et open source, il vous rend la souveraineté totale sur vos photos et vos vidéos. Vos données restent chez vous, sur votre serveur, sans intermédiaire ni plateforme qui revend vos souvenirs. C'est un projet d'une qualité rare, et c'est pour ça que voir la reconnaissance d'images tomber en panne est si frustrant : le logiciel fonctionne parfaitement, mais une brique de votre infrastructure le bloque en silence.

Un matin, vous ouvrez Immich. La recherche sémantique ne renvoie plus rien. Les visages ne sont plus détectés. Vous vérifiez les conteneurs, et `immich_machine_learning` redémarre en boucle. Dans les logs, toujours le même message :


RuntimeError: NumPy was built with baseline optimizations:
(X86_V2) but your machine doesn't support:
(X86_V2).
```

Ce message a de quoi décourager. Il parle de Python, de NumPy, d'optimisations de compilation. La bonne nouvelle : la cause n'est pas dans Immich, ni dans votre configuration Docker. Elle est dans votre VM Proxmox. Et elle se corrige en cinq minutes.

## Traduire l'erreur

NumPy dit une chose simple : « j'ai été compilé pour un niveau de CPU x86-64-v2, et la machine qui m'exécute ne le possède pas ».

Le niveau x86-64-v2 regroupe des instructions apparues vers 2009 : SSE4.2, POPCNT, CMPXCHG16B. Pratiquement tous les processeurs x86 depuis 2010 les possèdent. C'est pour ça que le message semble absurde. Votre serveur n'a sûrement pas quinze ans.

Sauf que le CPU que NumPy voit n'est pas votre CPU physique. C'est le CPU que votre hyperviseur présente à la VM. Et là, tout devient possible.

## La cause : le type de CPU virtuel

Depuis sa version 3, Immich embarque NumPy 2.x. L'équipe l'annonce dans son [guide de migration v3](https://immich.app/blog/v3-migration) : les CPU x86 doivent être de niveau x86-64-v2 ou supérieur. Ce n'est pas un bug, c'est une exigence assumée.

Le problème arrive côté hyperviseur. Quand vous créez une VM, Proxmox vous demande un type de CPU. Si vous avez laissé `kvm64`, ou un vieux défaut du genre, QEMU présente à l'invité un processeur générique du début des années 2010. Ce processeur virtuel masque des instructions comme SSE4.2, même si votre CPU physique les possède.

Le chemin complet de l'erreur :

1. Votre CPU physique possède SSE4.2. Aucun problème.
2. QEMU type `kvm64` masque cette instruction à la VM.
3. Docker dans la VM exécute NumPy.
4. NumPy teste le CPU visible, ne trouve pas SSE4.2, refuse de démarrer.

Votre matériel est capable. Votre VM, elle, fait semblant de ne pas l'être.

## Le diagnostic en trente secondes

Connectez-vous à la VM qui héberge Immich, et lancez :

```bash
grep -o -E 'sse4_2|popcnt' /proc/cpuinfo | sort -u
```

Deux résultats possibles :

- La commande renvoie `popcnt` et `sse4_2` : le problème est ailleurs. Regardez du côté de l'image ou de la configuration.
- La commande ne renvoie rien : votre CPU virtuel masque les instructions. C'est le cas de ce tutoriel.

Pour confirmer que le matériel est capable, exécutez la même commande sur l'hôte Proxmox, hors de la VM :

```bash
grep -o 'sse4_2' /proc/cpuinfo | sort -u
```

Si l'hôte affiche `sse4_2` et pas la VM, le diagnostic est certain.

## Le correctif

Cinq étapes, et une précaution. Le changement de type de CPU ne prend effet qu'au démarrage complet de la VM. Un redémarrage depuis l'OS invité ne suffit pas toujours. Arrêtez la VM proprement, ou depuis Proxmox.

### 1. Arrêter la VM

Dans l'interface Proxmox : clic droit sur la VM, Shutdown. Ou en ligne de commande :

```bash
qm shutdown <vmid>
```

### 2. Changer le type de CPU

Dans l'interface : VM → Hardware → Processors → Edit → Type, choisissez `host`. Ou en ligne de commande :

```bash
qm set <vmid> -cpu host
```

Le type `host` expose au invité toutes les instructions du CPU physique, sans masquage. La [documentation Proxmox](https://pve.proxmox.com/pve-docs/chapter-qm.html) le recommande pour les charges qui profitent des instructions du processeur. Le machine learning en est un bon exemple.

### 3. Redémarrer et vérifier la stack

Démarrez la VM, puis relancez Immich :

```bash
docker compose up -d
```

Vérifiez les logs du conteneur :

```bash
docker logs immich_machine_learning --tail 50
```

Vous devez voir :

```
INFO     Initializing Immich ML [v3.4.1]
INFO     Starting gunicorn 26.2.0
INFO     Listening at: http://[::]:3003 (12)
INFO     Application startup complete.
```

Plus aucune erreur NumPy. Le conteneur reste up.

### 4. Reprendre la commande du CPU

Après un passage à `host`, le CPU vu par la VM change. Si vous aviez fait des réglages fins dans l'invité (gouverneur de fréquence, placement de processus), c'est le bon moment pour les vérifier. Dans la plupart des cas, il n'y a rien à faire.

### 5. Relancer les jobs de reconnaissance

Immich n'a pas traité vos photos pendant la panne. Il faut rattraper le retard. Dans l'interface d'administration :

- Administration → Jobs → Encode CLIP : relancez pour la recherche sémantique et les descriptions.
- Face Detection : relancez la détection de visages.
- Face Recognition : relancez la reconnaissance des personnes.

Pour Face Detection et Face Recognition, une option reprocess permet de forcer le retraitement de tous les médias. Utile si des détections ont échoué silencieusement pendant plusieurs versions.

## Si `host` ne suffit pas

Il reste un cas limite. Si le CPU de l'hôte Proxmox lui-même ne supporte pas x86-64-v2, le type `host` ne changera rien. Vérifiez avec la commande `grep` directement sur l'hôte. Un processeur antérieur à 2010 ne passera pas le test.

Dans ce cas, la seule option est d'épingler l'image `immich-machine-learning` sur une version antérieure à la bascule NumPy 2.x, une 2.7.x, en attendant un remplacement du matériel. C'est un palliatif, pas une solution. Les images Docker modernes durcissent leurs exigences CPU les unes après les autres.

## La leçon plus large

Ce problème ne concerne pas qu'Immich. NumPy 2.x exige x86-64-v2. RHEL 10 et ses dérivés exigent x86-64-v3. La tendance est claire : les logiciels compilés récemment partent du principe que votre CPU a au moins quinze ans, pas vingt.

Un type de CPU générique dans une VM fonctionne parfaitement pendant des années. Puis un beau jour, une mise à jour d'image Docker crashe au démarrage, sans que vous ayez touché à rien. La config VM, elle, n'a pas changé depuis sa création.

Le réflexe à retenir : dès qu'un conteneur crashe au démarrage avec une erreur de bibliothèque compilée, vérifiez ce que la VM voit comme CPU. La commande `grep -o -E 'sse4_2|popcnt|avx2' /proc/cpuinfo | sort -u` prend dix secondes. Elle vous dira si votre matériel est coupable, ou seulement votre hyperviseur.

## Un dernier mot pour Immich

Immich est gratuit, open source, et développé par une petite équipe qui livre à un rythme impressionnant. Si le projet vous rend le même service qu'il me rend, pensez à le soutenir via leur [page Purchase](https://immich.app/pricing). C'est leur programme de soutien, et les contributeurs y sont reconnus avec un badge de supporter dans l'application. Une belle façon de dire merci, et de financer la souveraineté qu'on nous offre gratuitement.
```

---
