## Introduction
Bonjour, si vous êtes développeurs de bots discord, vous avez sans doute entendu parler des `shards`.
C'est peut-être encore un peu brouillon pour vous, mais ne vous inquietez pas, à la sortie de ce tutoriel le sharding n'aura plus aucun secret pour vous.
J'ai fait ce tuto car peu de tutos sur les shards en francais existent, et je suis bien placé pour en parler sachant que je suis repsonsable de l'infrastructure et du développement d'un bot comptant plusieurs dizaines de milliers de serveurs discord.

* * *

Dans ce tutoriel nous verrons ce qu'est le sharding pour ceux qui ne connaissent pas encore, ensuite nous expliquerons à quoi il sert et quand l'implémenter sur son bot, enfin nous mettrons en pratique plusieurs exemples de shardings en Javascript avec nodeJs.

Vous êtes préts ? 

* * *


## Le Sharding, Kézako ?

Au fur et à mesure que votre bot grandis et est invité sur de plus en plus de serveurs, arrive un moment où votre machine ne suffit plus pour le faire tourner, c'est assez problématique n'est-ce pas ? Donc la solution est de diviser le bot et de le mettre sur 2 machines, mais nous rencontrons vite un probleme... Le bot répondra 2 fois à vos messages, en clair il fera tout en double. Donc il faudrais faire en sorte que votre Machine#1 ne réponde qu'a `x` serveurs uniques, et Machine#2 repondra à `Y`serveurs uniques, mais c'est long et fastidieux à mettre en plus surtout pour savoir quels serveurs seront sur telle machine... Donc Discord à mis en place le "Sharding", pour la faire simple, vous dites à discord combien vous avez de shards au total, et lequel est le premier et le dernier de la machine. tout ca dans les parametres de connexion Gateway de l'api de Discord, Pour faire simple, Machine#1 aura un client qui s'occupera de 3 shards, le shards 0 jusqu'au 2, et la Machine#2 s'occupera de 2 shards, vous l'aurez deviner ce client s'occupera du shard 3 au 4, Si vous savez compter vous avez lancer 5 shards sur 2 machines. Tada votre bot est sur 2 machines, chaque machine recois uniquement les données des serveurs compris dans les shards dont il s'occupe, rien d'autre.

Bien sur nous pouvons imaginer que l'ont peut aussi faire ca sur la même machine (il suffit juste de lancer 2 clients sur 1 machine au lieu de 2 sur 2 machines, et que chaque client s'occupe de ses shards).

En bref, le sharding sert à diviser votre bot sur plusieurs machines ou plusieurs instances, pour limiter le trop de requetes et se coltiner un rate limit de l'api, ou alors tout simplement pour faire du multi Thread ou juste soulager la machine si elle à peu de performances (d'oû le principe d'en mettre sur 2 machines).

Voila, vous connaissez le principe du Sharding d'un bot Discord.


* * *


## Un Shard ou Une Shard ?
Sur des serveurs d'entraides de développement, vous verrez beaucoup passer des gens qui disent UN ou UNE shard, et bien on ne sais pas trop, on pourrais dire que une shard est une instance, donc au féminin, on pourrais aussi dire que les shards sont des Clients Discord eux même, donc au masculin.

Enfin bref prononcez le comme vous le voudrez, on ne vous jugera pas (c'est promis).

* * *

## Je ne sais pas quand mettre en place le sharding, Comment je fais ?
Beaucoup de gens pensent que mettre un systeme de shard est bien dés le début car "Comme ca c'est fait, j'aurais pas à le faire le moment venu", c'est une mauvaise idée, votre bot marchera tout aussi bien, mais lancer votre bot sur 1 shard c'est comme le lancer normalement car de base il est lancé sur 1 shard.
Le sharding est obligatoire aprés le passage des 2500 serveurs, il est conseillé de mettre environ 1000 serveurs par shards, mais tout dépend de votre situation, si c'est un bot qui envoie juste des statistiques depuis des données receuillies sur un api, vous pouvez en mettre 2000-2500, si votre bot a besoin de performance mettez en vers les 1000, aprés il faut que vous essayez avec plusieurs palliers (2000, 1500, 1000, 500, 250, 100). Il est conseillé d'implémenter le systeme de shard assez tot avant le passage des 2500 serveurs, si votre bot gagne 100 serveurs/jours, il faut s'y prendre trés tot, sinon envisagez aux 1500-2000 serveurs, vous serrez large.

*** 
## Comment récupérer les Messages Privés ?
Il faut savoir que tous les messages privés du bot sont sur le premier shard, donc le shard #0.

*** 

## Mise en Pratique avec [Discord.js](https://discord.js.org)

* - ### Le Sharding Manager
Dans leur librairie, Discord.js propose un systeme de shard déja tout fait et trés simple d'utilisation, ce dernier ne va pas se contenter de lancer un client avec x shards sur son instance, il va lancer autant d'instances que de shards, voici un schéma pour comprendre son fonctionnement:![Sharding Manager](http://i.sayrix.fr/QfdJ.png)
Pour mieux comprendre le fonctionnement, imaginez vous avez votre `index.js` avec le contenu suivant: 
```js
const { Client } = require('discord.js') ;
const client = new Client() ;
```
Et votre fichier `sharder.js` avec le systeme de sharding manager va lancer le fichier `index.js` autant de fois qu'il y a de shards sous forme de "Worker" autonomme.
Donc pour résumer, chaque shard aura son propre client, son propre processus, son propre cache avec un processus Maître qui va gérer ces derniers, à savoir que contrairement à l'internal sharding, pour accéder aux informations des autres shards, il faut effectuer un broadcast et les informations demandées  seront retournées par tous les shards (null si rien trouvé).
```js
//Demmander le nombre de serveurs de tous les shards
client.shard.broadcastEval(c => c.guilds.cache.size))
    .then(console.log) ;
//OU
client.shard.fetchClientValues('guilds.cache.size')
    .then(console.log) ;
/* OUTPUT
[
    956,
    963,
    945
]
*/
//Demander un serveur en particulier avec l'id
client.shard.broadcastEval(c => c.guilds.cache.get("223070469148901376")))
    .then(console.log) ;
/* OUTPUT
[
    null,
    Guild (223070469148901376),
    null
]
*/
//Le serveur de Michel est sur le shard #1

```

Lancer le Sharding Manager:

`sharder.js`
```js
const { ShardingManager } = require('discord.js') ;

const manager = new ShardingManager('./index.js', { token: 'MicheL_T0K3N' }) ;

manager.on('shardCreate', shard => console.log(`Launched shard ${shard.id}`)) ;

manager.spawn() ;
```
`index.js`
```
const { Client, Intents } = require('discord.js') ;

const client = new Client({ intents: [Intents.FLAGS.GUILDS] }) ;

/*
    Code de votre bot
*/
client.login('MicheL_T0K3N');
```

* - ### L'internal Sharding

L'internal sharding c'est pour ceux qui ne veulent pas avoir un shard par instance, pour le mettre en place il suffit tout simplement mettre le nombre de shards à lancer sur le client.

![Internal Sharding](http://i.sayrix.fr/6TbD.png)

Internal sharding avec partage des shards automatique:
```js
const Discord = require('discord.js') ;
const client = new Discord.Client({
    shards : 'auto'
}) ;
```
Internal sharding avec nombre de shards définis:
```js
const Discord = require('discord.js') ;
const client = new Discord.Client({
    shards : [0, 1, 2],
    shardCount : 3 
}) ;
```
!! Attention de ne pas mettre plus de shards qu'il y'en a dans `shardCount` !!

* * *

## Liens Utiles

* - [Documentation Discord.js](https://discord.js.org/#/docs)
* - [Guide Discord.js](https://discordjs.guide/)
* - [Documentation API Discord](https://discord.com/developers/docs/intro)

* * *

## Conclusion

Et voila, maintenant vous savez tout sur le sharding discord, Si vous avez des questions n'hésitez pas à me mentionner sur le [serveur de Michel](https://discord.com/invite/gca).

Merci d'avoir lu ce tutoriel en esperant vous avoir éclairer sur le sujet.
