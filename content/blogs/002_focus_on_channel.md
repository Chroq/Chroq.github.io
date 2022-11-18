---
title: "Focus sur les channels"
date: 2022-11-10T11:11:24+02:00
draft: true
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
- d'une connaissance de base du langage Go. Si vous n'êtes pas encore familier avec la conccurence en Go, je vous encourage à aller lire <a href="/blogs/001_concurrence_go/" target="_blank">le premier article</a> de la série. 

Vous pouvez retrouver tout le code source des exemples (002_*) <a href="https://github.com/Chroq/tutorial" target="_blank">ici</a>.

## 