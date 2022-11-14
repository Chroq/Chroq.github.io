---
title: "101 - La concurrence en Go"
date: 2022-11-14T11:11:24+02:00
draft: false
github_link: "https://github.com/Chroq/tutorial/tree/main/001_concurrency"
author: "Christophe Lecroq"
tags:
  - Golang
  - Concurrence
  - Goroutines
  - Asynchrone
image: /images/001_concurrence_go/001_concurrence_go.webp
description: "Ce premier article a pour but d'expliquer les concepts relatifs à la concurrence en Go. Il servira de base aux prochains articles qui aborderont des notions plus avancées."
---

Ce premier article a pour but d'expliquer les concepts de base de la concurrence en Go : goroutine, channels et select.  

<!--more-->

## Pré-requis

Pour bien suivre ce tutoriel, vous aurez besoin :
- d'une installation fonctionnelle de Golang (je travaille sur la version 1.19 mais les concepts expliqués ici existent dans les versions antérieures
- d'une connaissance de base du langage Go

Vous pouvez retrouver tout le code source des exemples (001_*) <a href="https://github.com/Chroq/tutorial" target="_blank">ici</a>.

## Qu'est ce que la concurrence ?

Précisons d'abord ce qu'est la concurrence en programmation. Le concept,  de manière simplifiée, pourrait se résumer de la manière suivante : **effectuer des traitements multiples de durées indéfinies sans notion d'ordre**.

L'exemple du serveur web HTTP revient fréquemment quand on évoque le principe de la concurrence. Il reçoit, traite et distribue les requêtes de manière concurrente. 

Sans rentrer dans le détail, sachez quand même qui existe une différence entre la concurrence et le parallélisme. Si vous souhaitez plus d'informations sur la question, je vous redirige vers <a href="https://www.youtube.com/watch?v=Y1pgpn2gOSg" target="_blank">cette vidéo</a>.

## La concurrence en Go

### Les Goroutines

Quand vous souhaitez détacher une tâche de votre programme principal pour qu'elle s'exécute de manière concurrente, vous utiliserez une `goroutine`. 
Pour la déclarer, Golang fournit l'instruction éponyme `go`, c'est dire comme le concept est central pour ce langage.

```golang
go myFunction()
```

Placé devant l'appel d'une fonction, ce mot-clé vous permet de détacher une nouvelle goroutine (un processus léger). Ce sous-programme va s'exécuter dans un `thread` géré par le runtime de Golang. Je ne rentrerais pas ici dans le détail de ce qu'est le runtime car la notion mériterait un article en tant que tel, comprenez juste qu'il s'agit de l'interface fournie par Golang entre votre programme et le système d'exploitation.

### Les channels

les `channels` permettent de manipuler les données entre vos sous-programmes (goroutines).

Comme vous le savez le Go est un langage à typage fort et chaque `channel` a un type propre (int, string,`channel`, etc). 

```golang
var myChan = make(chan int)
var myChan2 chan int
```

Vous pouvez utiliser les deux manières d'initialiser un channel. 

Les `channels` sont bi-directionels (entrée/sortie) et **bloquants**, c'est à dire qu'une fois la valeur écrite à l'intérieur, vous devez la lire pour continuer le traitement. 

## Programmes concurrents

### Utilisation basique des goroutines  
Puisqu'on en a finit avec la terminologie, voyons un peu de code !

Le but est de lancer deux goroutines et de retarder l'affichage du nom passé en paramètre en fonction de sa longueur.

```golang
package main

import (
	"fmt"
	"strings"
	"time"
)

// shout prints an uppercased name n times, the longer the name the longer the delay
func shout(number int, name string) {
	for j := 1; j <= number; j++ {
		fmt.Println(strings.ToUpper(name))
		time.Sleep(time.Duration(len(name)*100) * time.Millisecond)
	}
}

func main() {
	go shout(3, "Galadrielle")
	go shout(3, "Arwen")
	var a string
	fmt.Scanln(&a)
}
```

J'appelle donc deux fois cette fonction, la première avec un prénom long (donc qui s'affichera moins rapidement) et la seconde avec un prénom court.

Vous noterez deux choses. La première c'est que devant les deux appels de fonctions se trouve le mot clé `go`. La seconde c'est que j'appelle la fonction `fmt.Scanln()` pour que le programme ne s'arrête pas une fois les instructions lancées. En effet, votre programme principal n'a pas connaissance (**Spoiler alert** pour l'instant) de l'exécution de sous-programmes et va se terminer sans attendre le traitement des `goroutines`.

Donc, si vous lancez le programme vous obtiendrez la sortie suivante : 

```
GALADRIELLE
ARWEN
ARWEN
ARWEN
GALADRIELLE
GALADRIELLE
```

Les deux premières lignes correspondent respectivement à l'exécution des premières itérations de chaque fonction avec `Galadrielle` et `Arwen`.

À ce stade-là, les fonctions `Sleep` sont appellées. Le délai pour `Arwen` étant beaucoup plus court (500 millisecondes) que celui de `Galadrielle` (1100 millisecondes), la boucle se termine avant celle de `Galadrielle`.

### Communication entre Goroutines

Le premier programme ne permet pas de faire grand chose. En effet, chaque goroutines va s'exécuter de manière indépendante sans échanger quoi que ce soit avec notre programme principal ou une autre `goroutine`. 

Voyons maintenant le code suivant : 

```golang
package main

import (
	"fmt"
	"time"
)

const BalrogHP = 20

func LegolasShootArrow(damage chan int) {
	for damage != nil {
		damage <- 1
		time.Sleep(100 * time.Millisecond)
	}
}

func GandalfCastsSpell(damage chan int) {
	for damage != nil {
		damage <- 5
		time.Sleep(250 * time.Millisecond)
	}
}

func DisplayBalrogHP(LegolasDamage, GandalfDamage chan int) {
	var balrogHP = BalrogHP
	for LegolasDamage != nil && GandalfDamage != nil {
		var incomingDamage int
		select {
		case incomingDamage = <-LegolasDamage:
			fmt.Println("Legolas shoots an arrow !")
		case incomingDamage = <-GandalfDamage:
			fmt.Println("Gandalf casts a spell !")
		}

		balrogHP -= incomingDamage

		fmt.Printf("Balrog HP : %d\n", balrogHP)

		if balrogHP <= 0 {
			LegolasDamage, GandalfDamage = nil, nil
			fmt.Printf("Balrog is dead !")
		}
	}
}

func main() {
	var LegolasDamage, GandalfDamage = make(chan int), make(chan int)

	go LegolasShootArrow(LegolasDamage)
	go GandalfCastsSpell(GandalfDamage)
	go DisplayBalrogHP(LegolasDamage, GandalfDamage)

	var a string
	fmt.Scanln(&a)
}
```

Supposons que vous soyez obligé d'affronter un <a href="https://fr.wikipedia.org/wiki/Balrog" target="_blank">Balrog</a>. La tâche nécessite le concours de plusieurs protagonistes <a href="https://fr.wikipedia.org/wiki/Legolas" target="_blank">le rapide Legolas</a> et <a href="https://fr.wikipedia.org/wiki/Gandalf" target="_blank">le puissant (mais plus lent) Gandalf</a>. Ici, chacun des héros attaque avec sa vitesse et ses dommages propres.

Revenons à nos `channels`, l'idée principale est de déclarer un channel pour chacune des fonctions liées aux personnages et de les passer à une fonction d'affichage de la vie du Balrog. Comme vous l'aurez remarqué, chaque appel de fonctions se fait de manière concurrente.

Si on execute le programme, on aura la sortie suivante : 
```golang
Gandalf casts a spell !
Balrog HP : 15
Legolas shoots an arrow !
Balrog HP : 14
Legolas shoots an arrow !
Balrog HP : 13
Legolas shoots an arrow !
Balrog HP : 12
Gandalf casts a spell !
Balrog HP : 7
Legolas shoots an arrow !
Balrog HP : 6
Legolas shoots an arrow !
Balrog HP : 5
Gandalf casts a spell !
Balrog HP : 0
Balrog is dead !
```

NB : Si vous exécutez le code chez vous, il est possible que la sortie diffère un petit peu. Les fonctions étant appelées l'une derrière l'autre, il arrive que la fonction de Gandalf (comme sur l'exemple ci-dessus) s'exécute en premier. C'est normal et vous devrez vous habituer quand vous travaillez en asynchrone à ce que l'ordre d'exécution puisse être aléatoire si les fonctions se déclenchent quasiment en même temps.

Le déroulement est donc ici le suivant :

- Le programme lance les trois `goroutines`
- le `select` se comporte ici comme un `switch`. C'est une instruction spécifique aux channels qui va attribuer la valeur écrite dans le `channel` à une variable et exécuter du code spécifique.
- On remarque bien que le `channel LegolasDamage` est écrit plus fréquemment que le `channel GandalDamage`
- Enfin, pour éviter que le programme ne continue de s'exécuter, je place les deux `channels` à nil pour sortir des boucles. 

### Conclusion

J'espère que vous avez désormais une meilleure vision de ce que sont les `goroutines`, `channels` et de la concurrence en Go. Il s'agit d'une première approche pour appréhender les concepts de base de la concurrence et le prochain article traitera plus en profondeur des channels et du package sync.