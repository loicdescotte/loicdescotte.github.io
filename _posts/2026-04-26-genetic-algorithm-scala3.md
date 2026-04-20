---
layout: post
title: Faire évoluer des agents pour un jeu de plateforme avec un algorithme génétique en Scala 3
tags:
 - Scala
 - Scala 3
 - Genetic Algorithm
 - Neural Network
 - FP
 - Java2D
---

# Faire évoluer des agents pour un jeu de plateforme avec un algorithme génétique en Scala 3

## Introduction

Ce projet explore une idée simple mais fascinante : plutôt que de programmer manuellement l'intelligence d'un agent joueur, on la fait émerger par sélection naturelle.

Concrètement, une population de 50 agents tente de traverser un niveau de plateforme généré procéduralement. Chaque agent est piloté par un réseau de neurones dont les paramètres (poids et biais) constituent son "génome". Les agents qui progressent le plus loin survivent, se reproduisent et transmettent leurs gènes à la génération suivante. Ceux qui tombent dans les trous ou se font toucher par un ennemi sont éliminés.

Au bout de quelques générations, des comportements émergent : les agents apprennent à sauter au bon moment, à esquiver les ennemis, à ajuster leur trajectoire en fonction des obstacles. Aucune règle explicite ne leur enseigne ces comportements — tout vient de la pression de sélection.

L'ensemble est écrit en **Scala 3**, s'appuie sur **Java2D** pour le rendu et fonctionne entièrement en local (pas de GPU, pas de framework ML). L'accent est mis sur la pureté fonctionnelle : pas d'état mutable partagé, pas d'effets de bord cachés.

---

## Modèle de données

### Approche purement fonctionnelle

Toute la logique du jeu est **pure et immuable**. Chaque frame produit un nouveau `GameState` plutôt que de modifier l'ancien. L'aléatoire est passé explicitement comme paramètre — jamais via un état global.

Cette contrainte a des avantages concrets : on peut rejouer n'importe quelle frame, comparer deux états, ou évaluer 50 individus en parallèle sans risque d'interférence.

### Système de coordonnées

```
(0,0) ──────────────────────► X  (positif vers la droite)
  │
  │   [tuiles] [joueur] [ennemis]
  │
  ▼
  Y  (positif vers le bas — convention écran Java2D)
```

L'axe Y pointe **vers le bas**, ce qui est la convention habituelle de Java2D. Un saut correspond donc à une vélocité Y **négative**.

### Types principaux

**`Vec2`** — vecteur 2D pour positions et vitesses, en pixels :

```scala
case class Vec2(x: Double, y: Double):
  def +(v: Vec2): Vec2   = Vec2(x + v.x, y + v.y)
  def -(v: Vec2): Vec2   = Vec2(x - v.x, y - v.y)
  def *(s: Double): Vec2 = Vec2(x * s, y * s)
```

**`Tile`** — ADT à deux cas représentant une cellule de la grille du monde :

```scala
enum Tile:
  case Air    // espace vide, traversable
  case Ground // sol ou plateforme solide
```

**`World`** — grille immuable de tuiles, indexée par `tiles(col)(row)` :

```scala
case class World(tiles: Vector[Vector[Tile]], width: Int, height: Int)
```

**`Player`** — état du joueur à un instant donné :

```scala
case class Player(
  position: Vec2,    // coin supérieur-gauche de la hitbox (pixels)
  velocity: Vec2,    // vitesse courante (pixels/frame)
  onGround: Boolean, // posé sur une surface solide
  alive:    Boolean, // faux si mort (trou ou ennemi)
  maxX:     Double   // position X maximale atteinte — mesure la progression
)
```

**`Enemy`** — ennemi patrouillant entre deux bornes fixes :

```scala
case class Enemy(
  position:    Vec2,
  velocity:    Vec2,
  direction:   Direction,
  patrolLeft:  Double,
  patrolRight: Double
)
```

**`Actions`** — sorties du réseau de neurones, décodées en booléens :

```scala
case class Actions(moveLeft: Boolean, moveRight: Boolean, jump: Boolean)
```

Plusieurs actions peuvent être actives simultanément (par exemple `moveRight + jump`).

**`GameState`** — l'état complet du jeu, transmis frame après frame :

```scala
case class GameState(
  world:   World,
  player:  Player,
  enemies: Vector[Enemy],
  frame:   Int,
  over:    Boolean
)
```

La fonction de fitness est définie directement sur `GameState` :

```scala
def fitness: Double =
  val distance   = player.maxX / GameConfig.TileSize
  val speedBonus = 1.0 - frame.toDouble / GameConfig.MaxFrames
  distance + speedBonus
```

---

## Réseau de neurones

### Architecture

Le réseau est un perceptron multicouche feedforward à deux couches :

```
Capteurs (11 entrées)
     ↓
Couche cachée : 16 neurones, activation ReLU
     ↓
Couche de sortie : 3 neurones, activation Sigmoid
     ↓
Actions : moveLeft | moveRight | jump
```

Chaque neurone calcule `activation(Σ(entrée_i × poids_i) + biais)`. La propagation avant est une opération **pure** — aucun état modifié.

### Génome : une séquence plate de 243 nombres réels

Chaque individu est représenté par un `Genome`, soit un `Vector[Double]` de 243 valeurs. Ces gènes encodent tous les paramètres du réseau dans l'ordre suivant :

| Segment | Contenu | Taille |
|---|---|---|
| 1 | Poids couche 1 (11 × 16 connexions) | 176 |
| 2 | Biais couche 1 (16 neurones) | 16 |
| 3 | Poids couche 2 (16 × 3 connexions) | 48 |
| 4 | Biais couche 2 (3 neurones) | 3 |
| **Total** | | **243** |

Cette représentation plate permet d'appliquer croisement et mutation de façon uniforme, sans connaître la structure interne du réseau.

### `Genome.decode()`

La fonction `decode` reconstruit un `Network` depuis un `Genome` et une `NetworkConfig`. Elle parcourt les gènes séquentiellement : pour chaque couche, elle lit d'abord les `inSize × outSize` poids, puis les `outSize` biais.

```scala
Genome (243 gènes)
  └─ Genome.decode() ──► Network (poids + biais)
                              │
GameState ──► Agent.observe() ──► [11 floats] ──► Network.forward() ──► [3 floats]
                                                                              │
                                                                    Agent.act() ──► Actions
                                                                                        │
                                                              Engine.step(state, actions) ──► next GameState
```

---

## Capteurs : les 11 entrées du réseau

A chaque frame, `Agent.observe()` transforme l'état brut du jeu en un vecteur de 11 valeurs numériques normalisées que le réseau peut traiter.

| Index | Feature | Plage | Description |
|---|---|---|---|
| 0 | Sol présent à 1 tuile devant | 0 ou 1 | Détecte les bords de trous immédiats |
| 1 | Sol présent à 2 tuiles devant | 0 ou 1 | |
| 2 | Sol présent à 3 tuiles devant | 0 ou 1 | |
| 3 | Sol présent à 4 tuiles devant | 0 ou 1 | |
| 4 | Distance au prochain trou | [0, 1] | Normalisée sur 8 tuiles ; 0 = sauter maintenant, 1 = aucun trou en vue |
| 5 | Distance horizontale ennemi devant | [0, 1] | Normalisée sur 12 tuiles ; 1 = aucun ennemi |
| 6 | Distance verticale ennemi devant | [-1, 1] | Normalisée sur 4 tuiles ; négatif = ennemi en hauteur |
| 7 | Vitesse verticale du joueur | [-1, 1] | Normalisée par `MaxFallSpeed` ; négatif = monte |
| 8 | Joueur au sol | 0 ou 1 | Flag `onGround` |
| 9 | Clearance verticale devant | [0, 1] | 0 = plafond collé, 1 = hauteur de saut complète libre |
| 10 | Direction de l'ennemi | -1, 0 ou +1 | +1 = approche, -1 = s'éloigne, 0 = aucun ennemi |

Les capteurs 0 à 3 forment un "regard" sur le sol : ils permettent au réseau de détecter un trou avant d'y tomber. Le capteur 4 affine ce signal avec une précision pixel. Le capteur 9 évite que l'agent saute sous un plafond bas — une information absente de la première version du projet, dont l'ajout a nettement amélioré les comportements en zone de plateformes.

### Actions

Les 3 sorties Sigmoid du réseau produisent des valeurs dans [0, 1]. Un seuil à 0.5 les convertit en booléens :

```
outputs(0) > 0.5  →  moveLeft  = true
outputs(1) > 0.5  →  moveRight = true
outputs(2) > 0.5  →  jump      = true
```

---

## Algorithme génétique

### Représentation

Chaque individu est une paire `(Genome, fitness)`. Le génome est un `Vector[Double]` de 243 valeurs — un encodage direct de tous les poids et biais du réseau. L'évaluation consiste à décoder le génome en réseau, faire jouer l'agent jusqu'à sa mort ou le timeout de 2500 frames, et noter le score obtenu.

### Population initiale : des agents volontairement mauvais

La génération 0 est peuplée d'individus "naïfs" construits via `Genome.naive()`. L'idée est de démarrer avec des comportements idiots mais différenciés, pour que la sélection ait matière à travailler dès les premiers croisements.

Trois archétypes sont générés aléatoirement :

- **Fonceur (50%)** : biais de sortie orientés pour spammer `moveRight` et bloquer `jump`. L'agent court à droite et tombe invariablement dans le premier trou. C'est le comportement le plus "visible" en génération 0.
- **Sauteur maladroit (25%)** : `moveRight` encouragé, `jump` permanent. L'agent avance en sautant en continu — il loupe les atterrissages et saute là où il ne faut pas.
- **Erratique (25%)** : poids très saturés (σ=3.0), comportement chaotique. Il oscille, recule, ou reste sur place.

Dans tous les cas, les poids généraux sont à σ=1.5 (contre σ=0.5 pour un réseau aléatoire classique), ce qui sature les activations et empêche le réseau de traiter correctement les capteurs. L'évolution doit donc apprendre à la fois à calibrer les poids et à construire des stratégies de saut.

### Fonction de fitness

```
fitness = maxX / TileSize + (1 - frame / MaxFrames)
```

La composante principale est la **distance en tuiles** atteinte. Le **bonus de vitesse** vaut 1.0 si l'agent finit instantanément, 0.0 s'il atteint le timeout de 2500 frames. Deux agents atteignant la même distance sont donc départagés par leur rapidité. MaxFrames correspond à environ 42 secondes à 60 fps.

### Sélection par tournoi (k=3)

On tire 3 individus aléatoirement dans la population et on conserve le meilleur. Cette opération est répétée pour choisir chaque parent. La pression de sélection est modérée : un individu faible a toujours une probabilité non nulle d'être sélectionné s'il tombe sur deux adversaires encore plus faibles.

### Croisement BLX-α (α=0.3)

Chaque gène de l'enfant est une interpolation aléatoire des deux gènes parentaux, légèrement extrapolée au-delà de leur intervalle :

```
lo  = min(g1, g2) - α × |g2 - g1|
hi  = max(g1, g2) + α × |g2 - g1|
gene_enfant = lo + u × (hi - lo),  u ~ Uniforme[0, 1]
```

Avec α=0.3, l'enfant peut légèrement dépasser les valeurs parentales. Ce croisement préserve mieux la cohérence inter-couches du réseau qu'un crossover à point unique, où une coupure dans le milieu du génome peut rompre des dépendances entre poids de couches adjacentes.

### Mutation gaussienne

Chaque gène a une probabilité de 10% d'être perturbé par un bruit gaussien N(0, σ=0.4). Cela introduit de la diversité sans détruire les bonnes solutions.

### Elitisme

Les 5 meilleurs individus de chaque génération passent **inchangés** à la suivante. Cela garantit que les meilleures solutions ne sont jamais perdues par la mutation.

### Détection de stagnation

Si la meilleure fitness reste identique pendant 8 générations consécutives, le taux de mutation passe de 10% à 15% et σ de 0.4 à 0.55. Le message suivant est logué :

```
[GEN 14] Stagnation détectée — mutation boostée (rate=0.15, strength=0.55)
```

Ce mécanisme permet d'échapper aux optima locaux sans augmenter la mutation de façon permanente.

---

## Boucle de simulation

La simulation est une machine d'état à deux phases, représentée par l'ADT `SimState` :

```scala
enum SimState:
  case Evaluating(population, remaining, evaluated)
  case Showcasing(population, gameState, network, framesDone)
```

### Phase 1 : Evaluating

Aucun rendu. 8 individus sont évalués par tick Swing. Pour chaque individu, on décode son génome en réseau, puis on boucle sur `Engine.step()` jusqu'à `state.over`, en appelant `Agent.act()` à chaque itération. Cette boucle prend moins de 2 ms par individu sur un CPU moderne — les 50 individus d'une génération sont traités en une fraction de seconde.

Une fois tous les individus évalués, les fitness sont calculées, la population est triée, et la simulation passe en phase Showcasing.

### Phase 2 : Showcasing

Le meilleur individu de la génération joue en temps réel à 60 fps, pendant au maximum 2500 frames. Le rendu Java2D est actif : on peut observer le comportement évolué, voir le personnage sauter les trous, esquiver les ennemis, accélérer vers la droite.

A la fin du showcase (mort ou timeout), `Evolution.evolve()` est appelé pour produire la génération suivante, et le cycle recommence en phase Evaluating.

Il est aussi possible de passer le showcase manuellement via `skipShowcase()`, ce qui déclenche immédiatement l'évolution.

---

## Génération du monde

Le niveau est généré **une seule fois** au démarrage avec la graine fixe `42L`. Tous les individus de toutes les générations jouent sur le même niveau, ce qui garantit une comparaison de fitness équitable.

La génération se déroule en quatre passes :

**Passe 1 — Sol de base :** les rangées `FloorRow` et `FloorRow+1` sont remplies de `Ground` sur toute la largeur du monde (150 tuiles).

**Passe 2 — Trous :** de gauche à droite, des trous de 2 à 3 tuiles de large sont creusés avec une probabilité croissante (de 20% à 40% selon la progression dans le niveau). Un espacement minimum de 5 à 10 tuiles solides sépare deux trous. Les trous font toujours 2 ou 3 tuiles — suffisamment franchissables avec un bon timing de saut.

**Passe 3 — Plateformes flottantes :** des plateformes de 4 à 7 tuiles de large sont placées 3 à 5 tuiles au-dessus du sol. Une passe de réparation rebouche ensuite les trous qui se trouvent sous une plateforme basse, pour éviter des configurations impossibles à traverser.

**Passe 4 — Ennemis :** des ennemis sont placés sur les segments de sol d'au moins 6 tuiles (avec 80% de probabilité) et sur les plateformes d'au moins 4 tuiles (70%). Chaque ennemi patrouille en aller-retour dans les bornes de son segment.

Ce monde fixe, combiné à la fitness basée sur la distance, crée une pression évolutive claire et reproductible : chaque génération est évaluée contre exactement le même obstacle.
