---
title: "Focus sur les channels"
date: 2022-11-21T11:11:24+02:00
draft: false
github_link: "https://github.com/Chroq/tutorial/tree/main/002_in_depth_channel"
author: "Christophe Lecroq"
tags:
  - Golang
  - Concurrence
  - Channel
  - Asynchrone
image: /images/002_focus_on_channel/002_focus_on_channel.webp
description: "Ce second article reprends la notion de channel en Go et détaille les différents usages dans un contexte de "
---

Cet article est le second de la série consacrée à la concurrence en Go. Il reprend en détail l'utilisation des `channels`. 

<!--more-->

## Pré-requis

Pour bien suivre ce tutoriel, vous aurez besoin :
- d'une installation fonctionnelle de Golang (je travaille sur la version 1.19 mais les concepts expliqués ici existent dans les versions antérieures
- d'une connaissance de base du langage Go. Si vous n'êtes pas encore familier avec la concurence en Go, je vous encourage à aller lire <a href="/blogs/001_concurrence_go/" target="_blank">le premier article</a> de la série. 

Vous pouvez retrouver tout le code source des exemples (002_*) <a href="https://github.com/Chroq/tutorial" target="_blank">ici</a>.

## Les buffered channels

### Bloquant vs non bloquant

En Go, un `channel` est bloquant par défaut. C'est à dire qu'il est nécessaire d'attribuer une valeur au channel pour pouvoir la lire et ainsi continuer l'exécution du programme.
Si vous lancez ce code :
```golang
1.	c, c2 := make(chan bool), make(chan bool)
2.
3.	go func() {
4.		c <- true
5.		c2 <- false
6.	}()
7.
8.	fmt.Println(<-c2,<-c)
```

Vous recevrez un message du type : `fatal error: all goroutines are asleep - deadlock! goroutine 1 [chan receive]:`. Lorsque le programme va s'exécuter, il écrit dans le channel `c` (ligne 4) puis cherche à lire cette valeur. Ici, c'est `c2` qui est appelé en premier sauf que lui n'a aucune valeur. Le programme détecte alors un `deadlock` puisqu'il ne peut plus continuer son exécution.

Le même code en réordonnant la lecture des channels ne générera plus d'erreur.

```golang
	c, c2 := make(chan bool), make(chan bool)

	go func() {
		c <- true
		c2 <- false
	}()

	fmt.Println(<-c, <-c2)
```
L'exécution du code suivant retournera : `true false`

Il existe plusieurs manières de rendre non bloquant un channel, je ne parlerais ici que de celle concernant `les channels` mais sachez qu'il est possible aussi de contourner le mécanisme avec l'instruction `select`. 

```golang
	c, c2 := make(chan bool, 1), make(chan bool, 1)

	go func() {
		c <- true
		c2 <- false
	}()

	fmt.Println(<-c2, <-c)
```

Dans l'extrait ci-dessus, on déclare un channel avec un buffer de 1. Les channels ayant une taille de buffer dans l'instruction `make` sont **non bloquants**, c'est à dire que le traitement va continuer même si le channel n'est pas lu directement.

Notez que j'ai replacé les paramètres de l'instruction `Println` dans l'ordre qui avait généré le `deadlock` tout à l'heure. Si vous exécuter ce code, vous aurez comme sortie `false true`

Pour conclure sur cette notion de blocage, il s'agit d'un mécanisme pratique pour ne pas avoir à gérer explicitement le blocage des channels. Cependant, vous aurez sans doute des cas de figure dans lesquels vous souhaitez gérer vous-même ce comportement pour couvrir votre cas d'usage.

#### Itérer sur un channel

Pour parcourir un `buffered channel`, Golang permet à l'aide de la boucle `for` et du mot clé `range` d'itérer sur les valeurs de votre channel. Vous pouvez donc organiser vos traitements et leurs résultats associés. 


```golang
package main

import "fmt"

func main() {
	c := make(chan int, 2)
	go func() {
		c <- 2
		c <- 1
		close(c)
	}()

	for v := range c {
		fmt.Println(v)
	}
}
```

La sortie sera donc : 
```
2
1
```

Selon ce que vous cherchez à réaliser, l'utilisation d'un channel avec buffer peut impliquer l'utilisation de la fonction `close` sur votre channel. En effet, si vous n'indiquez pas à votre programme que vous ne souhaitez plus continuer à utiliser ce channel, le programme considérera que vous devez encore écrire dedans et paniquera avec une erreur `deadlock`.

### Cas pratique

Reprenons l'exemple de l'article précédent, Legolas et Gandalf versus le Balrog. Pour démontrer l'intéret des `buffered channels`, j'ai ajouté quelques contraintes :

- Legolas a un nombre limité de flèches. Dans mon exemple, je limite la taille du buffer à 3.
- Laisser Legolas agir en premier et Gandalf terminer en boucle jusqu'à la fin du "traitement"
- Utiliser un channel `dead` pour gérer l'information de fin de traitement plutôt que de passer les channels à nul. Il s'agit d'une façon plus idomatique de faire. 

```golang
package main

import (
	"fmt"
)

const BalrogHP = 20

func LegolasShootArrows(damage chan int) {
	damage <- 1
	damage <- 2
	damage <- 3
	close(damage)
}

func GandalfCastsSpell(dead chan bool, damage chan int) {
	for !<-dead {
		damage <- 5
	}
}

func DisplayBalrogHP(dead chan bool, LegolasDamage, GandalfDamage chan int) {
	var balrogHP = BalrogHP
	fmt.Print("Balrog is coming ! ")
	for balrogHP > 0 {
		if balrogHP > 0 {
			dead <- false
			fmt.Printf("Balrog HP : %d\n", balrogHP)
		} else {
			dead <- true
		}

		var incomingDamage int
		for damage := range LegolasDamage {
			fmt.Printf("Legolas shoots an arrow and deals %d damages ! \n", damage)
			incomingDamage += damage
		}

		var open bool
		if incomingDamage, open = <-GandalfDamage; open {
			fmt.Printf("Gandalf casts a spell and deals %d damages ! \n", incomingDamage)
		}

		balrogHP -= incomingDamage
	}

	fmt.Printf("Balrog is dead !")
}

func main() {
	dead, LegolasDamage, GandalfDamage := make(chan bool), make(chan int, 3), make(chan int)

	go LegolasShootArrows(LegolasDamage)
	go GandalfCastsSpell(dead, GandalfDamage)

	go DisplayBalrogHP(dead, LegolasDamage, GandalfDamage)

	var a string
	fmt.Scanln(&a)
}
```

La sortie sera la suivante : 
```
Balrog is coming ! Balrog HP : 20
Legolas shoots an arrow and deals 1 damages !
Legolas shoots an arrow and deals 2 damages !
Legolas shoots an arrow and deals 3 damages !
Gandalf casts a spell and deals 5 damages ! 
Balrog HP : 15
Gandalf casts a spell and deals 5 damages ! 
Balrog HP : 10
Gandalf casts a spell and deals 5 damages ! 
Balrog HP : 5
Gandalf casts a spell and deals 5 damages ! 
Balrog is dead !
```

Vous noterez que le code a été agencé un peu différemment que dans la première version. Tout d'abord, pour ordonner les actions, j'ai supprimé l'instruction `select` pour gérer la boucle `for` puis la structure conditionnelle `if`.

Enfin, vous aurez peut-être noté la présence d'un booléen `open` utilisé lors de l'assignation de la valeur stockée dans le `channel` de Gandalf, ligne 40. Étant donné que je ne gère pas un `buffered channel`, ce booléen sera toujours à `true`. Je n'ai cependant pas le choix que de le gérer si je veux vous montrer comment récupérer une valeur d'un channel dans une condition.

## Conclusion

Les `channels` sont les outils de base pour gérer les échanges d'informations entre vos différentes `goroutines`. N'hésitez pas à vous entrainez pour bien appréhender leurs principes de fonctionnement. Le prochain article de la série continuera de traiter de la concurrence mais au travers du package `sync`.