[← Retour au README](../../../README.md)

#### 🌐 Langue
[[EN]](../../advanced.md) **[FR]**


# Work in progress
### Ce fichier peut contenir des erreurs ou être incomplet.

### Table des matières
- [Références et sources](#références-et-sources)
- [Introduction](#introduction)
- [Equilibrage de charge](#équilibrage-de-la-charge)


# Introduction

Ce document est un recueil de méthodes et de techniques plus avancées qui peuvent être utilisées pour améliorer les performances de vos datapacks. Il ne s'agit pas d'un guide complet, mais plutôt d'une collection de conseils et d'astuces qui peuvent être utilisés pour améliorer vos datapacks.

Contrairement au [document sur les bases de l'optimisation](./basics.md), ce document n'est pas destiné à être lu de haut en bas. Chaque section est indépendante des autres, et vous pouvez les lire dans l'ordre que vous souhaitez. C'est pourquoi il n'y a pas d'index pour chaque section.

L'objectif est de rassembler des méthodes plus avancées, en donnant des liens et des références à des sources externes. Les méthodes peuvent être spécifiques à une fonction de mcf, ou des méthodes plus générales d'optimisation de la programmation.

# Équilibrage de la charge
L'équilibrage de la charge est une optimisation extrêmement importante pour éviter le décalage, généralement appliquée aux opérations de grande envergure.

Tout d'abord, soulignons une chose : dans minecraft, le jeu est mis à jour 20 fois par seconde. Chaque mise à jour est appelée « tick ». Pour que le jeu maintienne ce taux de mise à jour, chaque tick doit être exécuté en un maximum de 50 millisecondes.

Chaque commande que nous ajoutons à notre datapack prend un certain temps pour s'exécuter, et nous devons nous assurer qu'aucun tick ne dépasse cette marge de 50 millisecondes. Bien sûr, la quantité de données que chaque ordinateur peut traiter en un tick dépend de l'ordinateur, mais l'idée est la même.

Pour démontrer le problème, j'ai créé un datapack qui brûle toutes les entités exposées au ciel. A chaque tick, il vérifie pour chaque entité du jeu si elle est exposée au ciel, et si c'est le cas, il change le NBT `Fire` pour enflammer l'entité. Dans notre exemple, la fonction qui consiste à vérifier si l'entité est exposée au ciel est extrêmement lourde.

### Le problème

Regardons le graphique de la durée par TPS (que vous pouvez ouvrir en appuyant sur Alt+F3) dans un monde local

![TPS Normal](../../../assets/load_balancing/graph1.png)

Ça a l'air bien ! Maintenant, allumons notre datapack. Vérifions à nouveau le graphique

![Très mauvais TPS](../../../assets/load_balancing/graph2.png)

Le graphique vert est avant la mise en route du datapack, et le jaune après. C'est un mauvais signe. Sur certains ordinateurs ou serveurs, cette ligne peut passer au rouge et entraîner un ralentissement du fonctionnement du serveur. Cette ligne peut également passer au rouge en fonction de la croissance du datapack...

Nous devons faire en sorte que le graphique soit aussi proche que possible du datapack désactivé.

### Trouver le problème

Nous devons comprendre s'il est possible de l'optimiser. Dans cet exemple, nous savons déjà que la fonction consistant à détecter si l'entité est exposée au ciel est le problème, et qu'elle ne peut pas être optimisée directement car il n'y a pas d'autres optimisations à faire.

Cependant, il existe encore une solution. Lorsque vous ne disposez plus d'optimisations directes, posez-vous toujours la question suivante : cette fonction doit-elle vraiment être exécutée à chaque tic-tac ?

Dans notre cas, il n'est pas nécessaire qu'elle soit si instantanée. Si vous avez quelques ticks de retard, ce sera parfait.

**Et bien... commençons par les optimisations.**

### Essayer de résoudre le problème

Commençons par faire ce que la plupart des gens font pour essayer de résoudre ce problème : Utiliser `/schedule `. Au lieu de vérifier tous les ticks, je vais le faire tous les 5 ticks.

Commençons par changer notre commande de la fonction tick à une fonction spécifique, et déclenchons le programme à partir de la fonction load

```mcfunction
#> schedule _5t.mcfunction
execute as @e[type=!player] run function example:try_sky_burn

schedule function example:schedule _5t 5t replace
```

```mcfunction
#> on_load.mcfunction
function example:schedule _5t
```

Ceci est déjà suffisant pour exécuter la fonction tous les 5 ticks

Rechargeons et regardons à nouveau le graphique

![Lag Spikes](../../../assets/load_balancing/graph3.png)

Diriez-vous que c'est acceptable ? Pas du tout ! C'est peut-être même mieux, mais regardez les pics ! La différence entre avant et maintenant est qu'avant tous les ticks étaient lents, maintenant seulement certains. Cela peut encore causer du lag !

Nous devons atténuer ces pics, mais... comment ?

### La vraie solution à ce problème

Nous devons répartir le poids de chaque pic entre tous les ticks, afin de rendre le graphique plat et de réduire le nombre de ms max.

Au lieu de vérifier toutes les entités en un seul tick tous les 5 ticks, nous allons vérifier 1/5 de toutes les entités à chaque tick, vérifiant ainsi uniformément une quantité spécifiée d'entités à chaque tick.

Pour mettre en œuvre ce système, je vais séparer les entités en 5 groupes. Pour ce faire, je vais créer un score qui oscillera entre 0 et 4 et progressera avec chaque entité ajoutée au groupe, et je donnerai un tag correspondant au groupe de l'entité. J'utilise un tag au lieu d'un score parce qu'en plus de la vérification des tags qui est plus rapide, Minecraft a un bug avec les scores sur les entités qui disparaissent qui peut même corrompre votre monde au fil du temps.

Tout d'abord, ajoutons le score au cycle des groupes

```mcfunction
#> on_load.mcfunction
scoreboard objectives add sky_burn_entity_group dummy
scoreboard players set .current_group sky_burn_entity_group 0
joueurs du tableau de bord set .current_group_to_assign sky_burn_entity_group 0
scoreboard players set .group_count sky_burn_entity_group 5
```

Ensuite, créons une fonction pour assigner une étiquette de groupe à une entité

```mcfunction
#> assign_group.mcfunction
# Cyclage du groupe
scoreboard players add .current_group_to_assign sky_burn_entity_group 1
opération des joueurs du tableau de bord .current_group_to_assign sky_burn_entity_group %= .group_count sky_burn_entity_group

# Définition du groupe d'étiquettes
execute if score .current_group_to_assign sky_burn_entity_group matches 0 run tag @s add sky_burn_group_0
execute si le score .current_group_to_assign sky_burn_entity_group correspond à 1 run tag @s add sky_burn_group_1
exécuter si le score .current_group_to_assign sky_burn_entity_group correspond à 2 run tag @s add sky_burn_group_2
exécuter si le score .current_group_to_assign sky_burn_entity_group correspond à 3 run tag @s add sky_burn_group_3
execute si le score .current_group_to_assign sky_burn_entity_group correspond à 4 run tag @s add sky_burn_group_4

# Définir une balise indiquant que cette entité a été regroupée
tag @s add sky_burn_has_group
```

Maintenant, ajoutons la fonction tick pour assigner un groupe aux entités qui n'ont pas encore de groupe

```mcfunction
#> on_tick.mcfunction
execute as @e[tag=!sky_burn_has_group] run function example:assign_group
```

Ceci est déjà suffisant pour donner un groupe à chaque entité. Notez que j'ai également supprimé les fonctions schedule du chargement, puisque nous ne les utiliserons plus. J'ai également supprimé la fonction `schedule _5t.mcfunction`.

Maintenant, nous devons modifier notre tick pour sélectionner un des groupes à mettre à jour, et mettre à jour le groupe correspondant. C'est ce que nous allons faire.

```mcfunction
#> on_tick.mcfunction
# Attribuer un groupe aux entités
execute as @e[tag=!sky_burn_has_group] run function example:assign_group

# Cycle du groupe en cours de mise à jour
les joueurs du tableau de bord ajoutent .current_group sky_burn_entity_group 1
opération des joueurs du tableau de bord .current_group sky_burn_entity_group %= .group_count sky_burn_entity_group

# Exécute la vérification sur l'entité de ce groupe
execute if score .current_group sky_burn_entity_group matches 0 as @e[tag=sky_burn_group_0] run function example:try_sky_burn
execute si le score .current_group sky_burn_entity_group correspond à 1 as @e[tag=sky_burn_group_1] run function example:try_sky_burn
execute si le score .current_group sky_burn_entity_group correspond à 2 as @e[tag=sky_burn_group_2] run function example:try_sky_burn
execute si le score .current_group sky_burn_entity_group correspond à 3 as @e[tag=sky_burn_group_3] run function example:try_sky_burn
execute si le score .current_group sky_burn_entity_group correspond à 4 as @e[tag=sky_burn_group_4] run function example:try_sky_burn
```

Maintenant, rechargeons et vérifions le graphique... et si tout fonctionne toujours, bien sûr

![TPS avec répartition de charge appropriée](../../../assets/load_balancing/graph4.png)

Beaucoup mieux, et tout fonctionne encore ! Chaque entité individuelle est vérifiée tous les 5 ticks, mais pas toutes en même temps.

Notez que les pics dans ce graphique sont dus au jeu lui-même, et non à notre code.

Nous avons réussi à équilibrer la charge en la répartissant entre les ticks ! Yay 🥳🥳🥳

Les lecteurs les plus attentifs et expérimentés se rendront compte que vérifier si une entité est exposée au ciel est quelque chose de trivial en utilisant `positioned over`. J'ai ajouté la ligne `execute as @e as @e as @e` à la fin de la fonction juste pour l'alourdir et pouvoir mieux démontrer les choses ;)_

### Quand ne pas utiliser le Load Balancing

Vous avez probablement remarqué que bien que nous ayons appliqué correctement la répartition de charge, notre datapack a une charge de travail beaucoup plus élevée puisqu'il doit gérer et exécuter des commandes basées sur des groupes. Dans ce cas, c'était une bonne chose, car cela rendait le datapack plus efficace. Mais ce n'est pas toujours le cas

Si vous avez appliqué correctement l'équilibrage de charge et que le nombre maximal de ms du graphique est toujours supérieur à ce qu'il serait sans l'équilibrage de charge, cela signifie que les frais généraux sont beaucoup plus importants que l'opération et que vous ne devriez pas l'appliquer. En d'autres termes, ne l'utilisez pas si elle n'apporte pas de bénéfice en termes de performances.

### Une autre façon de résoudre le problème (mais elle est incohérente)

Une autre façon de résoudre le problème serait d'utiliser un prédicat aléatoire et de sélectionner les entités en fonction de celui-ci, par exemple

```mcfunction
#> fonction exécutée en tant qu'entité
execute if predicate example:20_percent_chance run function example:random_entity_check
```

Mathématiquement, 20% de chance par entité reviendrait à diviser 1/5 des entités par exécution, avec presque aucun surcoût et beaucoup de simplicité, n'est-ce pas ?

Enfin... en quelque sorte. Le problème est **l'incohérence**. Bien qu'en moyenne, il s'exécute sur 1/5 des entités, ce n'est pas toujours le cas. Il peut y avoir des ticks où beaucoup d'entités sont exécutées en même temps, et il peut y avoir des ticks où presque aucune entité n'est exécutée ! Cela signifie que les commandes peuvent provoquer des **pics aléatoires** !

Dans cet exemple, j'ai fixé un délai de 5 ticks pour chaque entité, mais dans un scénario réel, j'aimerais un délai précis de 20 ticks d'intervalle, pour pouvoir m'aligner sur le temps de dégâts du feu et causer 1 dégât par seconde. Donc, dans ce scénario, je ne pourrais pas du tout utiliser le hasard.

Mais, en raison de l'énorme réduction de l'overhead, cela peut en fait être une possibilité d'optimisation. J'éviterais surtout d'utiliser le hasard en raison de son incohérence et je préférerais les options qui me permettent de contrôler réellement ce qui se passe. Gardez à l'esprit que nous voulons un graphique plat, pas plein de vagues. En tant que développeur, c'est à vous d'analyser et de voir si cela vaut la peine d'être utilisé ou non dans votre cas.

### Conclusion

Nous devrions utiliser l'équilibrage de charge pour distribuer le poids de nos commandes en les répartissant sur les ticks.

Sachant cela, gardez à l'esprit que ce n'est pas la seule utilisation possible de l'équilibrage de la charge. Vous pourriez vouloir répartir vos commandes d'une autre manière, par exemple en modifiant de grandes zones de blocs et en les séparant en petites parties sur plusieurs ticks. Vous pouvez également avoir deux programmes qui s'exécutent tous les deux ticks entrecoupés, chacun effectuant une fonction complètement différente ou entrelacée. L'important est de répartir le poids de vos commandes entre les ticks afin de maintenir l'appartement du graphe et d'éviter la surcharge des ticks !

# Références et sources


[← Retour au README](../../../README.md)
