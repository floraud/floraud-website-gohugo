+++
title = "Récupérer les données d'un disque dur défaillant avec ddrescue"
categories = ["Tech"]
tags = ["Hardware", "Systeme", "Stockage", "Linux"]
featured_image = "/images/tech/ddrescue_cover_art-wall-kittenprint.webp"
date = 2024-11-10T18:36:34+02:00
+++

Aujourd'hui, un proche est venu me voir avec un disque dur défaillant. Je voudrais vous présenter comment nous avons pu sauver 99.99% des données sans débourser un centime ou utiliser un outil d'une entreprise obscure.

Ça ne fait jamais plaisir en tant qu'informaticien de la famille, lorsque vous ne cessez de sensibiliser vos utilisateurs concernant [la règle des 3-2-1](https://fr.wikipedia.org/wiki/Sauvegarde_(informatique)#Strat%C3%A9gie_3-2-1) pour les sauvegardes de données, quand l'on vient vous voir avec un disque dur externe défaillant sur lequel la personne possède toute sa vie, qu'il semble irrécupérable et que vous êtes sa dernière chance de récupérer tout ça.

Bien qu'il existe des entreprises spécialisées dans le domaine, tout le monde n'a pas des centaines ou milliers d'euros à débourser dans la récupération de leurs données, c'est pourquoi je me suis lancé dans cette aventure.

Avant d'agir, analysons le problème.

**Attention** : **Chaque action** que vous faites sur un stockage défectueux **peut être la dernière** s'il est déjà dans un mauvais état. Effectivement, chaque nouveau travail dessus peut l'user davantage et lui être fatal, donc réfléchissez bien avant d'agir. L'intérêt de l'outil que l'on va utiliser par la suite (ddrescue) est que l'on va créer une image identique du disque que l'on essaye de sauver, ce qui permet de s'affranchir de ces risques par la suite en effectuant une copie brute, bit par bit du disque.

## Analyse de l'état du disque

C'est un SSD externe de 500 Go de stockage et vieux de 12 ans. Lorsqu'il est branché, il émet un sifflement, qui d'après un collègue expérimenté en électronique, pourrait provenir d'un condensateur défectueux. Une panne sérieuse donc.
Pour me démonter le problème, l'utilisateur le branche à son PC (Windows) pour que je puisse observer le comportement. Effectivement, on peut voir les dossiers à la racine du disque dans l'explorateur Windows, mais dès que l'on réalise un clic gauche ou droit sur un dossier, le disque ne répond plus et peut se déconnecter de l'ordinateur, de même pour toute action de copie classique. Cela peut s'expliquer par la tentative de lecture de certains secteurs défectueux.

En le rebranchant, je tente quand même d'utiliser un de mes câbles micro-USB que je sais fonctionnel, mais nous n'avons pas plus de chance.

Je prends la décision de tenter un robocopy Windows qui est une des méthodes de copy les plus fiables sur l'OS de Microsoft.

En cmd, je passe la commande suivante :

`robocopy "E:\" "D:\Recup" /E /Z /R:5 /W:5 /LOG:"D:\Recup\log.txt"` où :
- **"E:\"** : le disque source.
- **"D:\Recup"** : le chemin du dossier de destination.
- **/E** : on copie tous les sous-dossiers/
- **/Z** : on active l'option de redémarrage pour relancer la copie même si la connexion est perdue comme dans le cas de ce disque.
- **/R:5** : on limite le nombre de tentatives de lecture des fichiers problématiques à 5 sinon c'est infini ou presque.
- **/W:5** : on attend 5 secondes entre chaque tentative.
- **/LOG:"D:\Récupération\log.txt"** : on garde un rapport de ce qui a été copié, ce qui pour moi est indispensable dans ce genre de cas.

Le processus est particulièrement long et je décide d'attendre le lendemain pour voir le résultat. Finalement, seulement 17 Go sur les plus de 200 Go de données sont obtenues de cette manière.

Je prends donc la décision de le brancher sur mon système Linux pour voir s'il me permet de mieux accéder aux fichiers.

## Linux et ddrescue

Pas de chance, sous Fedora en interface graphique, j'observe le même problème que sous Windows et en ligne de commande, la commande de copie (cp) ne semble pas répondre correctement et tourne dans le vide sans résultat.

Je me renseigne sur Internet et trouve le logiciel **ddrescue** qui parait prometteur.

En bref **ddrescue permet de copier les données d'un disque et de les reproduire à l'identique sur un autre disque**. Ça tombe bien, je possède un disque sain en plus !

Une fois les disques branchés, j'ai utilisé la commande **lsblk** pour bien vérifier les lettres de ceux-ci afin de ne pas me tromper dans la commande et ne pas copier mon disque vierge sur le disque à récupérer. Cette erreur peut être fatale.

J'ai ensuite lancé la commande `ddrescue -f -d -r 3 /dev/sda /dev/sdb /home/floraud/ddrescue.log` où :
- **-f** : force l’écriture sur le disque cible, je n'ai pas eu le choix de l'utiliser pour lancer ddrescue.
- **-d**  : lit directement depuis le disque en sautant le cache, ça serait utilise pour certains secteurs défectueux et de mon côté le traitement restait bloqué si je ne l'utilisais pas.
- **-r 3** : limite le nombre de tentatives de relecture des secteurs endommagés à 3 fois.
- **/dev/sda** : disque source.récupérés
- **/dev/sdb** : disque destination.
- **/home/user/ddrescue.log** : destination de votre fichier de log. C'est très important car en cas de plantage de la commande ou, si vous souhaitez relancer des actions supplémentaires, il pourra en récupérer le dernier l'état rencontré car il liste les secteurs du disque récupérés ou endommagés.

Un exemple du monitor que vous aurez :

```
ipos: 59073 MB, non-trimmed: 0 B, current rate: 0 B/s
opos: 59073 MB, non-scraped: 49664 B, average rate: 4185 kB/s
non-tried: 0 B, bad-sector: 134144 B, error rate: 2 B/s
rescued: 500107 MB, bad areas: 28, run time: 1d 9h 11m
pct rescued: 99.99%, read errors: 281, remaining time: n/a time since last successful read: 6h 27m 5s
Scraping failed blocks... (forwards)
```

Ça prend du temps, et il est difficile de donner une fourchette car je pense que ça dépend de l'état de santé de votre disque et des capacités matérielles de votre PC. Dans mon cas, pour 500 Go, le temps estimé ne cessait de varier entre 7 et 12 heures de travail mais finalement ça a duré 2 jours. J'ai vaqué à mes occupations le temps de le laisser tourner.

À mon retour la console indiquait **rescued: 99,99%** soit la quasi-totalité des données restaurées, j'étais extrêmement soulagé. J'ai vu que certains tentent de faire un passage en ajoutant à la commande précédente **l'argument -R** qui permet de relancer ce que l'on a fait précédemment mais à l'envers. J'ai donc tenté mais je n'ai pas récupéré un Mb en plus.

**Note** : Sur le disque de destination, vous ne verrez peut-être pas d’évolution de l'espace utilisé en interface graphique. Bien qu’inquiétant, cela n'a pas d'impact ; j’avais bien les données à la fin.

J'ai pu rebrancher le nouveau disque sur l'équipement de mon utilisateur et il ressemblait à ce qu'il avait auparavant sauf que cette fois, on pouvait naviguer dans les dossiers et les copier. Je lui ai fait tout copier ailleurs et le 0,01% devait correspondre à un fichier d'install d'un VST de musique puisqu'il était impossible de le copier. Rien de grave, ouf !

**En bref : je vous recommande ddrescue comme outil pour sauver vos données d'un disque défectueux si vous avez un problème. Prenez bien le temps de vérifier votre commande avant de la lancer, notamment le disque source et destination et de bien lancer le logging des informations. Mais surtout, de bien appliquer [la règle des 3-2-1](https://fr.wikipedia.org/wiki/Sauvegarde_(informatique)#Strat%C3%A9gie_3-2-1), c'est-à-dire que vous devez au minimum avoir 3 copies des données, sur au moins 2 médias différents afin d'éviter des écueils physiques similaires et 1 ailleurs qui peut être le cloud**

*Photo de bannière par [Art Wall - Kittenprint](https://unsplash.com/fr/@artwall_hd?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) sur [Unsplash](https://unsplash.com/fr/photos/plateau-tournant-noir-et-argent-sur-table-noire-9Wq1HpghQ4A?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)*