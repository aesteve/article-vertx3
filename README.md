Bonjour à tous,

Ce 22 juin 2015, la version 3 de [Vert.x](http://vertx.io) a vu le jour. L'occasion pour moi de vous présenter cette boîte à outils pour la Machine Virtuelle Java (JVM).


# Qu'est-ce-que Vert.x, en quelques mots ?

Il s'agit d'une boîte à outils polyglote pour construire des applications distribuées (web mais pas seulement) asynchrones pour la JVM. Si vous connaissez [Node.js](https://nodejs.org/), ces mots devraient vous être relativement familiers. Et d'ailleurs pour l'anecdote, la première version de Vert.x s'appelait _Node.X_... Rien d'innocent là-dedans.

Le [cœur de Vert.x](https://github.com/eclipse/vert.x) est développé sous la coupe de la fondation Eclipse, [les projets "satellites"](https://github.com/vert-x3) (extensions, support de différents langages, clients asynchrones pour différents SGBD, ...) sont des projets 100% communautaires. L'ensemble est disponible sous [licence Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0). 

L'idée fondatrice de Vert.x est de construire son application par ensemble de petites briques, appelées micro-services, qui peuvent fonctionner sur une même machine ou sur des machines différentes à travers le réseau de façon transparente pour le développeur. De même, chaque brique peut-être, sans aucun effort et surtout sans aucun changement dans son code, déployée sous forme d'une ou plusieurs instances. Permettant ainsi une grande scalabilité des applications.

Avant de rentrer plus en détail dans le fonctionnement de Vert.x, écrivons le sacro-saint Hello, World.

En Java 8 :

```java
public class Server extends AbstractVerticle {
  public void start() {
    vertx.createHttpServer().requestHandler(req -> {
      req.response().end("Hello from Vert.x!");
    }).listen(8080);
  }
}
```

Son petit frère en Javascript :

```javascript
var vertx = require('vertx');
vertx.createHttpServer().requestHandler(function (req) {
  req.response().end("Hello from Vert.x!");
}).listen(8080);
```

Leur cousin en Groovy :

```Groovy
vertx.createHttpServer().requestHandler({ req ->
  req.response().end("Hello from Vert.x!")
}).listen(8080)
```

Enfin, le petit dernier de la famille, monsieur Ruby :

```Ruby
vertx.create_http_server().request_handler() { |req|
  req.response().end("Hello from Vert.x!")
}.listen(8080)
```

Pour les cousins germains que sont Scala, Python, Clojure, Ceylon, ... Il faudra attendre [la version 3.1 de Vert.x](https://github.com/vert-x3/wiki/wiki/Vert.x-Roadmap#vertx-31).

Deux points majeurs à noter dans ce tout petit bout de code.

En premier lieu, il n'y a pas besoin d'énormément de code pour préparer un serveur HTTP, le lancer et lui dire comment il doit répondre aux requêtes HTTP. Encore heureux, c'est un peu le but du jeu et l'exemple n'est pas bien complexe.

Ensuite, on remarquera la cohérence de l'API dans les différents langages. L'équipe de Vert.x a choisi de conserver une API très semblable (qu'elle qualifie "d'idiomatique") dans les différents langages pour lesquels elle est disponible. Ainsi on retrouve un objet `vertx`, et les méthodes `createHttpServer`, `requestHandler` et `listen`. 

Si on regarde plus en détails la méthode répondant aux requêtes HTTP, on se rend compte qu'elle prend plusieurs formes:

* une lambda expression en Java 8
* une fonction anonyme en Javascript
* une Closure en Groovy
* un bloc en Ruby

Ce sont les moyens d'expression d'une [fermeture](https://fr.wikipedia.org/wiki/Fermeture_(informatique)) (ou clôture, en Angais *closure*) dans ces trois langages. C'est vraiment le souhait de l'équipe de développement de Vert.x. Exporter, dans chaque langage supporté, les mêmes primitives à l'aide des possibilités offertes par le langage. Nous reviendrons sur le "comment" un peu plus tard dans cet article.

Revenons sur ce qui donne à Vert.x sa particularité en parcourant la description présente dans la [documentation officielle](http://vertx.io/docs/).

## Vert.x est *reactive* (réactif)

Le paradigme de programmation choisi pour l'API de Vert.x est *event-driven* et *non-blocking*. Qu'est-ce-que cela signifie en pratique ?

En gros un seul credo: "ne m'appelez pas, c'est moi qui vous appellerai". On écrit son code de façon descriptive, en expliquant ce qu'il faut faire, et dans quel cas le faire. Ce que l'on écrit, ce sont des "réactions" à des *stimuli* (*event-driven*). On n'appelle pas de primitives directement (ou très rarement), mais on écrit plutôt : 

> Quand une requête te parvient, voici comment la traiter.

Encore une fois, ce n'est pas nouveau dans le monde de ces technologies asynchrones.

[[question]]
| Pourquoi cette approche ? Et sous le capot, comment ça fonctionne ?

En réalité, le constat de base de tous ces nouveaux outils de développement d'applications asynchrones ([Node.js](https://nodejs.org/), [undertow](http://undertow.io/), [akka](http://akka.io/), [Tornado](http://www.tornadoweb.org/), ...) partent d'un même constat : une application fonctionnant sur le réseau passe son temps à attendre. Mais à attendre quoi, au juste ? A vrai dire, tout plein de choses. Un cas que l'on retrouve fréquemment dans une application web par exemple, est le requêtage d'une base de données, parfois localisée sur la même machine, parfois sur un point distant du réseau. Dans ce cas, le schéma d'éxécution est le suivant : 

* l'utilisateur fait une requête au serveur
* le serveur web reçoit la requête, en extrait les informations qui l'intéressent (paramètres, en-têtes, ...)
* le serveur web interroge la base de données (requêtes SQL, procédure stockée, récupération d'un document, d'un *set*, suivant le type de stockage : SQL ou noSQL ou autre)
* la base de données (ou plus globalement : le système de stockage) répond au serveur web
* le serveur web formate la réponse comme il le souhaite (appel d'un moteur de template pour formater une page HTML, [marshalling](https://en.wikipedia.org/wiki/Marshalling_(computer_science)) XML, JSON ou autre s'il s'agit d'une [API REST](https://en.wikipedia.org/wiki/Representational_state_transfer), ...)
* l'utilisateur reçoit la requête, le navigateur l'affiche ou la traite (dans le cas d'un appel Ajax par exemple)

On voit donc, que du côté du serveur web, il y a une durée non-négligeable, et surtout non maîtrisable passée à attendre une réponse (dans l'exemple : du système de stockage, mais cela pourrait être des I/O du système, ou encore tout plein d'autres choses). 

Or, les serveurs web classiques ("historiques" dirons-nous) et surtout les frameworks associés (JavaEE en tête) fonctionnent de la façon suivante : lorsqu'une requête parvient au serveur, le serveur alloue un *thread* à son traitement. Une fois la réponse envoyée au client, le *thread* est désalloué.

Et du coup, on a alloué un *thread* à... 

... attendre majoritairement, ce qui est somme toute un peu dommage. Et on a surtout l'impression que ce sont des ressources simplement perdues. Pire encore, si on a besoin de gérer un grand nombre de requêtes en parallèle (un trafic important), il nous faut un grand nombre de threads (un nombre quasi-directement proportionnel au nombre de requêtes que l'on souhaite traiter). Une ressource coûteuse (spécialement en Java)... Et qui semble bien mal employée.

L'idée est donc d'ordonnancer les traitements au lieu de les paralléliser. En partant du principe que les traitements (réels, pas le temps passé à attendre) sont généralement plutôt courts, voire quasi instantanés. Vous en conviendrez, générer un petit bout de page HTML (finalement une bête chaîne de caractère) c'est loin d'être très coûteux. 

Ensuite, au lieu d'attendre une réponse, on rend la main. On "enregistre" auprès de Vert.x (ou node, ou autre) le code à appeler et quand est-ce-qu'il sera appelé.

Concrètement, dans l'exemple précédent, le développeur aurait décrit le fonctionnement suivant : 

* quand on te demande une page, exécute *cette* requête vers la base de données, puis rend la main
* quand la base de données t'aura répondu, renvoie *telle* réponse au client (en fonction des données reçues)

Charge maintenant à Vert.x de s'en débrouiller, et d'ordonnancer tout ce petit monde. Le but du jeu étant qu'aucun thread ne soit inactif (en attente d'I/O, ...).

Pour ceux que cela intéresse, et qui désireraient en savoir plus on parle de [patron de conception reactor](https://en.wikipedia.org/wiki/Reactor_pattern).

Ainsi, toute la mécanique de gestion de la concurrence, des threads, etc. est gérée par Vert.x (à l'aide d'une *event-loop* comme décrit dans l'article cité précédemment). Vous écrivez du code sans vous soucier du contexte dans lequel il sera exécuté. Et même mieux, vous avez l'assurance que pour une même requête, votre code sera toujours exécuté dans le même contexte. C'est la garantie qu'offre Vert.x. Finis les `synchronized`, les `volatile`, et autres verrous potentiels. La seule chose dont vous devez vous soucier et de **ne jamais** (sous aucun prétexte) bloquer la sacro-sainte event-loop (donc rendre la main lorsque vous exécutez une opération qui prend du temps et l'éxécuter de façon asynchrone). C'est la clef du succès.

Pour autant, il serait dommage de n'utiliser qu'un seul *thread* de votre machine, d'autant plus sur une machine disposant de plusieurs cœurs. Du coup, Vert.x propose (sans que vous n'ayez à vous en préoccuper) par défaut plusieurs event-loops, proportionnellement au nombre de cœurs disponibles sur votre machine. C'est une différence majeure avec Node.js qui pour obtenir le même fonctionnement nécessite d'instancier plusieurs "nœuds".

Cette différence a valu par le passé à Vert.x le petit surnom de "Node.js sous stéroïdes", notamment lorsqu'est sorti [ce test de performance](https://www.techempower.com/benchmarks/#section=data-r8&hw=i7&test=plaintext) réalisé par Techempower.

Les développeurs de Vert.x ont donné à ce mode de fonctionnement le petit nom de [**multi**-reactor pattern](http://vertx.io/docs/vertx-core/java/index.html#_reactor_and_multi_reactor) par opposition au *reactor pattern* présenté plus haut. 

Au niveau de la machinerie interne, pour accomplir tout cela, Vert.x repose sur [Netty](http://netty.io/). Netty fournit une API pour gérer des canaux de communication : entrées / sorties ou réseau de façon asynchrone, en utilisant [NIO](https://en.wikipedia.org/wiki/Non-blocking_I/O_(Java)), pour [*Non-blocking IO*](https://en.wikipedia.org/wiki/Asynchronous_I/O). En elle-même, cette nouvelle façon de gérer les entrées / sorties de façon non bloquante pourrait faire l'objet d'un article entier ;)

## Vert.x est polyglote

On l'a vu dans l'exemple ci-dessus. Vert.x fournit une API en Java, Javascript, Ruby et Groovy pour l'instant et les développeurs s'attachent désormais à fournir son équivalent en Python, Scala, Clojure et Ceylon.

L'idée, c'est que l'équipe de développement n'a pas envie de prendre partie dans une guerre de langage (qui tourne parfois à la guerre de clochers, ...), et se fiche pas mal de "quel langage est le meilleur".

C'est à vous de choisir le langage que vous préférez, ou dans lequel vous vous sentez le plus compétent. De même, étant donné la nature assez peu dogmatique de Vert.x (vous pouvez l'utiliser pour ce que vous voulez, application web, file de messages, event-bus distribué, ...) : en fonction du micro-service que vous écrivez, Java peut paraître plus adapté, tandis que pour un autre, Python vous semblera meilleur. Vert.x s'en fiche. "Chat perché".

## Vert.x est *unopiniated* (non-dogmatique)

Un anglicisme barbare. Qu'est-ce-que ça veut dire ? 

Un peu de la même façon que l'équipe se fiche pas mal de quel langage est le meilleur, Vert.x n'est pas un framework dans le sens : "il n'y a qu'une bonne façon d'écrire une application". Si vous connaissez Symfony, Rails, Spring MVC, vous savez sans doute de quoi je parle. On a des vues, des modèles, des templates, un moteur de templates, des services, un ORM, ...

Ce sont des frameworks extrêmement performants pour conduire votre développement, ils vous fournissent un canvas, balisent le chemin pour vous. Vous écrivez des contrôleurs, des vues, des services, une couche d'accès aux données etc. Ce qui donne une grande cohérence dans les applications écrites à l'aide de ces frameworks et guide parfaitement les débutants vers de bonnes pratiques. Ce qui sont deux arguments de taille.

Vert.x fait tout l'inverse, pour une simple raison : il n'y a pas vraiment d'application-type. Qu'il s'agisse de gérer une application "middle-tiers" ou un serveur d'API REST, Vert.x sera adapté à ces deux tâches, si tant est que vous écriviez votre application correctement.

Ce choix est un choix très fort, d'un coté extrêmement intéressant mais parfois également critiqué. Il faut bien comprendre qu'au contraire de ce qu'on peut penser, Vert.x n'est pas véritablement un framework, mais plus une boîte à outils. Vous avez tous les outils à disposition pour faire absolument tout ce que vous voulez, de façon asynchrone. Par contre, si vous écrivez une application web plus classique (rigoureusement MVC, que je qualifierais de *model-driven*) vous vous retrouverez sans doute à écrire beaucoup plus de code qu'avec Rails, Django ou Symfony par exemple. A vous de juger !

## Légèreté

Vert.x est extrêmement léger. Le noyau de l'API (sans brique additionnelle) pèse à peu près 650kB et ne vient pas avec une méga-tonne de dépendances transitives. Cela n'a l'air de rien, mais dans le monde de la JVM c'est assez rare pour être souligné.


## Rapide

Depuis la version 2, les développeurs de Vert.x se sont attachés à en faire un produit performant, capable de tirer profit de la machine sur laquelle il tourne (notamment le nombre de cœurs) sans que le développeur n'ait à s'en préoccuper. Capable également d'une grande scalabilité. En une option, vous pouvez passer d'un thread à plusieurs, sans rien n'avoir à changer au code que vous avez écrit. J'ai cité un test de performance datant de la version 2 de Vert.x, sachez que Vert.x 3 et en train d'être passé au crible pour [battre à plates coutures son aîné](https://twitter.com/timfox/status/610100718058500096).

## Modulaire

Au contraire de ses aïeux du monde Java EE (Tomcat, JBoss, Websphere, ...) Vert.x **n'est pas** un conteneur d'application (là encore, avantage/inconvénient, à vous de voir en fonction de votre besoin). 

Chaque micro-service est une application à part entière. Vert.x sait déployer ces applications (les fameux "micro-services") programmatiquement ou via la ligne de commande, mais rien ne vous oblige à la faire. Vous pouvez tout simplement écrire un `main` et y coller :

```java
Vertx vertx = Vertx.vertx();
```

Et c'est parti, vous avez accès à la boîte à outil.

Libre à vous d'oganiser vos applications comme vous le souhaitez. En fonction de ce que vous désirez faire.

Si vous choisissez l'approche "service" (appelés `Verticle`), sachez que vous disposez d'un grand nombre d'outils pour les déployer, programmatiquement ou non.

Vous pouvez ainsi choisir que votre serveur web doit fonctionner sur quatre instances en parallèle et disposer d'une haute disponibilité (`high-availability`, i.e. Vert.x s'occupe de le redéployer s'il détecte une panne) tandis que votre petit service de supervision devrait quant à lui se contenter de peu de ressources (parce qu'il n'a pas grand chose d'autre à faire qu'envoyer un mail de temps en temps) et que vous le redémarrerez à la main, voire pas du tout. Vous avez besoin d'un autre service, lui beaucoup plus lourd et effectuant des traitement bloquants (parfois on n'a malheureusement pas le choix...), vous allez alors demander à Vert.x de le déployer en tant que *worker verticle*, lui allouant un thread à part, et surtout pas une event-loop, histoire qu'il ne la bloque pas.


## Un Event-Bus distribué

Encore un barbarisme qu'il est nécessaire d'expliquer... 

Nous avons vu qu'on pouvait disposer de tout plein de petits services, éparpillés aux quatre coins du réseau sans que cela ne pose de réel problème. Mais que se passe-t-il lorsque ces services ont besoin de communiquer entre eux ? 

Réponse simple : on leur colle un serveur web à chacun ? Mmm, oui mais non...

En réalité, Vert.x propose un [distributeur de message](https://en.wikipedia.org/wiki/Message_broker) (ou *Message Broker*, ou *Event Bus* en Anglais dans le texte) dont le rôle, vous l'aurez deviné, est de distribuer des messages, des réponses, etc. à tout notre petit monde, un facteur, en gros. 

Peu importe que vos services fonctionnent sur la même machine ou sur deux nœuds distincts du réseau. C'est le problème de Vert.x, pas le votre. Vous disposez d'une API pour envoyer et recevoir des messages, Vert.x s'occupe de les distribuer. Il s'agit d'une implémentation du patron de conception [publish / subscribe](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) pour les plus curieux.

Ceux ayant déjà implémenté ce genre de système doivent ce dire: "ca va être la foire". Or justement, Vert.x 3 propose une [fonctionnalité](https://github.com/vert-x3/vertx-service-proxy) afin de masquer ces messages, et invoquer des _services_ distants sans ce soucier des messages a envoyer et recevoir.


[[secret]]
| # Quoi de neuf dans la version 3 ?
| 
| ## Gestion des langages alternatifs
| 
| L'aspect polyglote de Vert.x est donc un argument important aux yeux des développeurs. Dans la version 2, c'est la communauté qui s'occupait de porter l'API dans le langage de leur choix. Il existait donc un module Groovy, un module Scala, etc.
| 
| Approche intéressante mais néanmoins un peu limitée.
| 
| Ainsi, dès les prémices de la version 3, l'équipe à fait le choix de générer automatiquement l'API dans tous les langages supportés, à partir du code Java. C'est le projet [vertx-codegen](https://github.com/vert-x3/vertx-codegen). Je ne rentrerais pas dans les détails ici, mais pour ceux que cela intéresse, sachez que son fonctionnement repose sur des templates [MVEL](http://johannburkard.de/blog/programming/java/mvel-templating-introduction.html) et [l'Annotation Processing Tool](http://docs.oracle.com/javase/7/docs/technotes/guides/apt/) de Java. Ainsi, votre code est analysé lors de sa compilation en Java, et les autres APIs sont générées automatiquement.
| 
| C'est un outil important essentiellement pour les développeurs de Vert.x ou de clients tiers (Redis, MongoDB, ...) mais pourquoi pas vous ? Si vous écrivez un petit service en Java pour vos besoins et que vous décidez de changer de langage, pourquoi pas utiliser codegen ?
| 
| C'est le cas [des exemples](https://github.com/vert-x3/vertx-examples) fournis par Vert.x. Chacun d'eux est automatiquement retranscrit dans l'ensemble des langages supportés par Vert.x.
| 
| ## Exit le système de modules
| 
| Que serait une boîte à outils sans outils ? Pas grand chose...
| 
| Dans la version 2, Vert.x proposait un système de modules. Par exemple, si vous souhaitiez interagir avec MongoDB, vous deviez utiliser le module vertx-mongodb. Et vous pouviez déclarer votre dépendance à ce module lors du démarrage de votre application, Vert.x allait le chercher, le télécharger, et le lancer tout seul comme un grand.
| 
| Le soucis, c'est que ce système de module était propre à Vert.x, avec un format particulier, son propre descripteur, et packagé sous forme de zip.
| 
| Intéressant, mais un peu limitant parfois.
| 
| L'équipe de développement a donc fait table rase de ce système pour partir du principe que n'importe quelle dépendance pouvait être un module (un jar, une gem, un module CommonJS, ...).
| 
| Et du coup, repose désormais sur des *factories* à aller fouiller à la recherche de dépendances. Ainsi, si vous souhaitez utiliser, dans votre code, le module XXX et que vous travaillez en Java, vous allez déclarer une dépendance à ce module (via Maven ou Gradle par exemple) dans votre projet comme vous le feriez dans n'importe quel autre projet. Par contre, si vous souhaitez lancer le module XXX en tant que service (par exemple une base MongoDB embarquée, ...), alors vous allez déployer programmatiquement ce service de la façon suivante dans votre code : 
| 
| ```java
| vertx.deployVerticle("maven:com.mycompany:main-services:1.2::my-service")
| ```
| La dépendance sera chargée automatiquement au lancement de votre application.
| 
| Le développeur dispose du coup d'une plus grande souplesse. Soit il s'agit d'une dépendance forte (il a besoin du service-tiers dans son code) soit d'une dépendance au sens micro-service (j'ai besoin d'avoir une base MongoDB qui tourne quelque part pour mes tests, par exemple).
| 
| ## Une palette d'outils pour les applications web
| 
| A l'instar de son prédécesseur [Yoke](https://github.com/pmlopes/yoke), [Vert.x Web](https://github.com/vert-x3/vertx-web) fournit un ensemble d'outils à destination des développeurs d'application web pour gérer tout un tas de petites choses dont ils ont systématiquement besoin : la session HTTP, les différents mécanismes d'authentification (Basic, OAuth, ...), les cookies, le routage et la déclaration de routes, la lecture de paramètres de requêtes, la gestion de différents MIME-TYPES, les websockets et protocoles assimilés, les moteurs de templates, ...
| 
| C'est un grand pas pour l'ouverture de Vert.x vers un monde un peu différent, partant du pur "middle-tiers" (services réseau bas niveau) vers des applications qu'on qualifie aujourd'hui de *fullstack*, un terme un peu flou comme le web nous en réserve parfois mais qui signifie grosso-modo "qui s'occupe de tout" depuis la plomberie de bas-niveau (files de message entre nœuds du réseau, tâches périodiques, reprise sur erreur, ...) jusqu'à la présentation de données à des clients modernes (différents moteurs de template, gestion des websockets, ...).
| 
| Pour faire la comparaison, si vous connaissez Node.js, Vert.x dispose aujourd'hui de l'équivalent d'[Express](http://expressjs.com/) et ses [middlewares](http://expressjs.com/guide/using-middleware.html) ou si vous êtes plutôt Ruby, de [Sinatra](http://www.sinatrarb.com/).
| 
| ## Proxifier des services
| 
| L'Event-Bus est l'une des pierres angulaires de Vert.x depuis sa première version, et il n'a pas énormément changé dans cette version. 
| 
| Une des nouveautés introduites cependant, est la possibilité de définir n'importe quel service comme étant invocable (appelable) depuis l'event-bus, simplement en annotant son service et en laissant le compilateur (Java) se débrouiller et générer le code nécessaire.
| 
| Cela permet, à l'instar de protocoles comme WAMP ou RPC, d'appeler des méthodes à distance et de façon asynchrone depuis n'importe quel point du réseau.
| 

# Conclusion

C'en est terminé de cette présentation de Vert.x, vous l'aurez compris assez similaire à Node.js mais aussi sensiblement différent sur quelques axes majeurs.


# Pour aller plus loin 

* Le site de [Vert.x](http://vert-x3.github.io)
* Le [repo du projet sur GitHub](http://github.com/vert-x3)
* L'interview du créateur de Vert.x sur [InfoQ](http://www.infoq.com/articles/vertx-3-tim-fox)
* Le [groupe de discussion](https://groups.google.com/forum/#!forum/vertx) sur Google Groups
