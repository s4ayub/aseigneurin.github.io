---
layout: post
title:  "Utiliser Apache Spark en cluster"
date:   2014-11-12 11:00:00
tags: spark cluster
language: FR
---
Dans les précédents posts, nous avons utilisé Apache Spark avec un exécuteur unique. Spark étant un framework de calcul distribué, nous allons maintenant monter un cluster en mode *standalone*.

# Topologie

Un cluster Spark se compose d'un **master** et d'un ou plusieurs **workers**. Le cluster doit être démarré et rester actif pour pouvoir exécuter des **applications**.

Le master a pour seul responsabilité la gestion du cluster et il n'exécute donc pas de code MapReduce. Les workers, en revanche, sont les exécuteurs. Ce sont eux qui apportent des *ressources* au cluster, à savoir de la mémoire et des cœurs de traitement.

Pour exécuter un traitement sur un cluster Spark, il faut *soumettre une application* dont le traitement sera piloté par un **driver**. Deux modes d'éxécution sont possibles :

* mode *client* : le driver est créé sur la machine qui soumet l'application
* mode *cluster* : le driver est créé à l'intérieur du cluster.

# Communication au sein du cluster

Les workers établissent une communication bidirectionnelle avec le master : le worker se connecte au master pour ouvrir un canal dans un sens, puis le master se connecte au worker pour ouvrir un canal dans le sens inverse. Il est donc nécessaire que les différents noeuds du cluster puissent se joindre correctement (résolution DNS...).

La communication entre les nœuds s'effectue avec le framework [Akka](http://akka.io/). C'est utile à savoir pour identifier les lignes de logs traitant des échanges entre les noeuds.

Les noeuds du cluster (master comme workers) exposent par ailleurs une interface Web permettant de surveiller l'état du cluster ainsi que l'avancement des traitements. Chaque nœud ouvre donc deux ports :

* un port pour la communication interne : port 7077 par défaut pour le master, port aléatoire pour les workers
* un port pour l'interface Web : port 8080 par défaut pour le master, port 8081 par défaut pour les workers.

# Création du cluster

## Démarrage du master

Le master peut être démarré via le script `sbin/start-master.sh`. 
Les options disponibles sont :

- `-i IP`, `--ip IP` : adresse IP sur laquelle se *binder*
- `-p PORT`, `--port PORT` : port utilisé pour la communication interne (7077 par défaut sur le master, random sur les workers)
- `--webui-port PORT` : port pour l'interface Web (8080 par défaut sur le master, 8081 sur les workers)

Nous pouvons lancer le script sans options :

{% highlight bash %}
$ sbin/start-master.sh
starting org.apache.spark.deploy.master.Master, logging to /Users/aseigneurin/logiciels/spark-1.1.0-bin-hadoop2.4/sbin/../logs/spark-aseigneurin-org.apache.spark.deploy.master.Master-1-MacBook-Pro-de-Alexis.local.out
{% endhighlight %}

Le programme se lance en daemon et il rend donc la main. Les logs sont écrits dans des fichiers tournants. On peut y voir que l'interface Web est disponible sur le port 8080. On peut également lire l'URL du cluster (`spark://MacBook-Pro-de-Alexis.local:7077`) que l'on devra indiquer pour démarrer les workers.
    
{% highlight bash %}
$ tail -f logs/spark-aseigneurin-org.apache.spark.deploy.master.Master-1-MacBook-Pro-de-Alexis.local.out
...
14/11/12 14:29:09 INFO Remoting: Starting remoting
14/11/12 14:29:09 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkMaster@MacBook-Pro-de-Alexis.local:7077]
14/11/12 14:29:09 INFO Utils: Successfully started service 'sparkMaster' on port 7077.
14/11/12 14:29:09 INFO Master: Starting Spark master at spark://MacBook-Pro-de-Alexis.local:7077
14/11/12 14:29:09 INFO Utils: Successfully started service 'MasterUI' on port 8080.
14/11/12 14:29:09 INFO MasterWebUI: Started MasterWebUI at http://10.10.200.112:8080
14/11/12 14:29:09 INFO Master: I have been elected leader! New state: ALIVE
{% endhighlight %}

Nous pouvons alors afficher l'interface Web du cluster :

<a class="image-popup" href="/images/spark_cluster_master.png">
  <img src="/images/spark_cluster_master.png" width="50%">
</a>

## Démarrage des workers

Nous allons ensuite démarrer 3 workers via la commande `bin/spark-class org.apache.spark.deploy.worker.Worker`. Celle-ci attend en argument l'URL du cluster ainsi que d'éventuelles options. Les options possibles sont les mêmes que celles disponibles sur le master, auxquelles s'ajoutent :

- `-c CORES`, `--cores CORES` : nombre de coeurs alloués (uniquement sur les workers)
- `-m MEM`, `--memory MEM` : quantité de mémoire allouée (uniquement sur les workers)

Notez qu'un worker utilise par défaut toute la mémoire de la machine moins 1 Go laissé à l'OS. Dans notre cas, nous allons démarrer plusieurs workers sur le même noeud physique. Nous allons donc limiter la mémoire allouée au process à 4 Go via l'option `--memory 4G`, et restreindre à l'utilisation de 2 coeurs via l'option `--cores 2`.

Ici, le programme ne rend pas la main et on voit le log s'afficher directement sur la console. On peut voir que le port de communication interne est choisi au hasard (ici, 64570), que l'interface Web est accessible sur le port 8081, et que la communication a été établi avec le master.

{% highlight bash %}
$ bin/spark-class org.apache.spark.deploy.worker.Worker spark://MacBook-Pro-de-Alexis.local:7077 --cores 2 --memory 4G
...
14/11/12 14:29:14 INFO Remoting: Starting remoting
14/11/12 14:29:14 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkWorker@10.10.200.112:64570]
14/11/12 14:29:14 INFO Utils: Successfully started service 'sparkWorker' on port 64570.
14/11/12 14:29:14 INFO Worker: Starting Spark worker 10.10.200.112:64570 with 2 cores, 4.0 GB RAM
14/11/12 14:29:14 INFO Worker: Spark home: /Users/aseigneurin/logiciels/spark-1.1.0-bin-hadoop2.4
14/11/12 14:29:14 INFO Utils: Successfully started service 'WorkerUI' on port 8081.
14/11/12 14:29:14 INFO WorkerWebUI: Started WorkerWebUI at http://10.10.200.112:8081
14/11/12 14:29:14 INFO Worker: Connecting to master spark://MacBook-Pro-de-Alexis.local:7077...
14/11/12 14:29:15 INFO Worker: Successfully registered with master spark://MacBook-Pro-de-Alexis.local:7077
{% endhighlight %}

Notez qu'une ligne supplémentaire apparait dans le log du master, confirmant ainsi la connexion entre le master et le worker :

{% highlight bash %}
14/11/12 14:29:15 INFO Master: Registering worker 10.10.200.112:64570 with 2 cores, 4.0 GB RAM
{% endhighlight %}

On peut voir apparaître le worker dans l'interface Web :

<a class="image-popup" href="/images/spark_cluster_master-with-one-worker.png">
  <img src="/images/spark_cluster_master-with-one-worker.png" width="50%">
</a>

Via cette interface, il est possible de voir le statut du worker :

<a class="image-popup" href="/images/spark_cluster_worker.png">
  <img src="/images/spark_cluster_worker.png" width="50%">
</a>

Nous démarrons donc deux workers supplémentaires. Dans l'interface Web principale, on peut voir que le nombre de coeurs et la mémoire des workers se cumulent, totalisant ainsi 6 coeurs et 12 Go de mémoire :

<a class="image-popup" href="/images/spark_cluster_master-with-three-workers.png">
  <img src="/images/spark_cluster_master-with-three-workers.png" width="50%">
</a>