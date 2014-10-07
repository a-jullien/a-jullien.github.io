---
layout: post
title:  "Indexer vos données via ElasticSearch"
date:   2014-10-02 22:00:24
tags: [elasticsearch, nosql]
categories: jekyll update
---
Depuis quelques temps, je commence à regarder d'un peu plus près l'outil 
[**ElasticSearch**](http://www.elasticsearch.org/), un super moteur de recherche créé par la société du même nom. Il a un 
 certain engouement autour du paradigme du "big data" et l'exploitation de ces données dans des environnements 
 [**NoSQL**](http://fr.wikipedia.org/wiki/NoSQL). D'où la nécessité d'avoir des outils simples et performants pour tout se 
 qui touche à la recherche d'information.
Je vais donc tenter d'expliquer dans cet article comment fonctionne ElasticSearch, et de montrer quels sont les avantages 
à l'utiliser dans vos applications.

Tout d'abord, commençons par une petite présentation. ElasticSearch est un moteur de recherche basé sur [**Lucene**](http://lucene.apache.org/).
Lucene fournit effectivement un puissant système basé sur un moteur d'indexation de documents et un moteur de recherche "*full-text*" sur ces indexes.
ElasticSearch exploite ces fonctionnalités en ajoutant une couche "*Elastic*", c'est à dire le moyen de gérer la scalabilité horizontale (*Scale out*),
mais aussi de gérer de manière transparente le basculement (*Failover*) via son système de réplication. Pour finir, 
ElasticSearch fournit une couche d'intéropérabilité, manquante à Lucene, basée sur les standards 
[**REST**](http://fr.wikipedia.org/wiki/Representational_State_Transfer)/HTTP et [**JSON**](http://json.org/) pour le format d'échange des données.

#Architecture et fonctionnement du système
Avant toutes choses, il est nécessaire de se familiariser avec la terminologie d'ElasticSearch. On trouve ainsi:

* **Index**: ElasticSearch enregistre ses données dans un ou plusieurs indexes, qui peuvent être comparés à un espace logique, 
ou souvent comparés à une base de données dans le monde relationnel. C'est un concept qui provient de Lucene et permet 
d'écrire et de lire depuis ses indexes.
* **Type**: Permet simplement de regrouper des documents de même type au sein d'un index. Un document a donc obligatoirement un type
* **Noeud**: c'est simplement une instance d'ElasticSearch
* **Cluster**: l'ensemble de noeuds
* **Partition** (shard): gère le découpage des indexes en plusieurs ensembles afin de distribuer les documents
* **Replica**: Un réplica permet de gérer la montée en charge lorsqu'un noeud n'est pas en mesure de traiter correctement 
toutes les demandes
 
![Architecture Elastic Search](/images/ES_Architecture.png){: .centeredImage}

ElasticSearch fonctionne sur un principe maître - esclave, où dans un cluster un noeud est désigné comme maître (*primary*) 
et les autres noeuds comme esclaves (*replica*).
Il en découle trois règles de fonctionnement au sein d'un cluster ElasticSearch:

* L'écriture d'un document est d'abord exécutée sur une partition dite primaire (*primary*).
* Si l'écriture sur le primaire s'effectue correctement, l'action est propagée à toutes partitions secondaires du cluster.
* Si un noeud primaire meurt, un réplica est alors élu.

Un noeud ElasticSearch a deux facettes. Tout d'abord, l'*arbitre* qui a pout but de traiter les requêtes de recherche 
et les operations. L'arbitre execute ainsi une ou plusieurs opérations de type [**Map/Reduce**](http://fr.wikipedia.org/wiki/MapReduce) 
sur toutes les partitions afin d'optimiser le résultat.
Et enfin un noeud ElasticSearch est bien-sûr un *conteneur de données* responsable du stockage des documents indexés.
On préconise bien entendu de démarrer une seule instance d'ElasticSearch par machine hôte.
  
#Démarrage

L'installation et la mise en route de l'outil sont très simples. ElasticSearch a seulement besoin d'une JVM pour tourner 
(OpenJDK ou celle d'Oracle, en version 7 ou 8). Après avoir téléchargé et décompressé l'archive contenant ElastisSearch dans votre répertoire 
favoris, il vous suffit de lancer l'outil via la commande:
    
``$ <ELASTIC_HOME_DIRECTORY>/bin/elasticsearch -f``

Et voilà, votre moteur de recherche est démarré via cette instance et près à être utilisé. Facile, non ?

Lorsque que votre instance d'ElasticSearch démarre, l'accès au système se fait par l'intermédiaire de deux ports:

* Un port HTTP, le port **9200** par défaut.
* Un port de transport (port à utiliser lorque vous voulez communiquer directement avec la JVM), le port **9300** par défaut.

On peut donc vérifier que notre noeud ElasticSearch est en fonctionnement: 

``$ curl -XGET 'localhost:9200'``

Le résultat de cette requête HTTP doit vous donner quelque chose qui ressemble à ça:

``
{
  "status" : 200,
  "name" : "Jackson Arvad",
  "version" : {
    "number" : "1.0.3",
    "build_hash" : "61bfb72d845a59a58cd9910e47515665f6478a5c",
    "build_timestamp" : "2014-04-16T14:43:11Z",
    "build_snapshot" : false,
    "lucene_version" : "4.6"
  },
  "tagline" : "You Know, for Search"
}
``

Par défaut, ElasticSearch attribut le nom de cluster *elasticsearch* et un nom de noeud de manière automatique, ici *Jackson Arvad*.

<i class="fa fa-exclamation-triangle"></i> Il est important à ce stade de changer la configuration afin d'avoir un nom propre à votre cluster. En effet, par défaut 
le mécanisme de découverte de noeuds est en multicast. Si vous démarrer deux noeuds dans un même réseau sans changer 
la configuration, ils seront automatiquement liés au même cluster *elasticsearch*. Il faut bien l'avoir en tête.

#Configuration

Comme évoqué précedemment, il est important de changer la configuration par défaut d'ElasticSearch. Le fichier de 
configuration à modifier est *\<ELASTIC_HOME_DIRECTORY\>/config/elasticsearch.yml*.

Ici, je vais juste changer le nom du cluster et le nom du noeud:

    cluster.name:   MonSuperCluster
    node.name:      MonSuperNoeud
    
####Configuration réseau
Dans cette section, je vais finir sur la configuration d'ElasticSearch en parlant de la configuration réseau. Par défaut, 
le Multicast est utilisé pour l'autodécouverte des noeuds. Seulement, en production par exemple, il est souvent 
interdit de l'utiliser. Il y a donc un autre mode qui permet de forcer les IPs (Unicast):
 
    cluster.name:                       MonSuperCluster
    node.name:                          MonSuperNoeud
    network.host:                       192.168.0.1
    discovery.zen.ping.unicast.hosts:   ["192.168.0.2", "192.168.0.3"]
    
#API REST
Comme exposé en introduction, ElasticSearch fournit une api REST pour interagir avec le système.
 
####Meta données
<i class="fa fa-arrow-circle-right"></i> Informations sur le statut du cluster: 

``$ curl -XGET 'localhost:9200/_cluster/health'``

<i class="fa fa-arrow-circle-right"></i> Informations sur le cluster et ses noeuds:

``$ curl -XGET 'localhost:9200/_nodes'``

<i class="fa fa-arrow-circle-right"></i> Statistiques sur le cluster

``$ curl -XGET 'localhost:9200/_stats'``

####Opérations
<i class="fa fa-arrow-circle-right"></i> Arrêter un noeud

``$ curl -XPOST 'localhost:9200/_shutdown'``

<i class="fa fa-exclamation-triangle"></i> Pour ce genre d'opération, il est préférable de ne pas exposer directement 
votre cluster directement. C'est à votre soin de protéger votre cluster, par l'ajout d'un *reverse proxy* pour commencer...
   
####Indexer un document

Je veux par exemple indexer un document de type *post* dans l'index *blog*:

    $ curl -XPUT localhost:9200/blog/post/1 -d 
    '{
    "title": "Mon super post de test",
     "text": "Ma description du post",
    "author": "Antoine JULLIEN"
    }'

Le résultat sera alors un document JSON qui ressemblera à ça:

    {
    "ok":true,
    "_index":"blog",
    "_type":"post",
    "_id":1
    }
    
<i class="fa fa-arrow-circle-right"></i> Ici, j'ai directement spécifié un *ID* pour mon document (1). Si rien n'est spécifié dans la requête, un identifiant sera 
généré automatiquement par ElasticSearch.

####Destruction d'un index
``$ curl -XDELETE localhost:9200/blog``

<i class="fa fa-exclamation-triangle"></i> Toutes les données relatives à l'index seront supprimées.

####Fermeture d'un index
``$ curl -XPOST localhost:9200/blog/_close``

<i class="fa fa-arrow-circle-right"></i> Cette opération peut être utile afin de libérer de la ressource CPU & mémoire 
sans supprimer les données.

####Ouverture d'un index
``$ curl -XPOST localhost:9200/blog/_open``

####Rechercher un document
Je veux par exemple faire une recherche *Full-text* dans l'index *blog* des posts où le terme *test* apparait:
   
    $ curl -XGET localhost:9200/blog/post/_search?q=test
    
Le résultat de cette requête de recherche sera de cette forme-là:

    {
    "took":3,"timed_out":false,
    "_shards": {"total":5,"successful":5,"failed":0},
    "hits":{"total":1,"max_score":0.095891505,"hits":
    [{"_index":"blog","_type":"post","_id":"2","_score":0.095891505,
    "_source":{"text": "Mon super post de test", "title": "Mon super post", "author": "Antoine JULLIEN"}}
    ]}
    }
    
La réponse contient effectivement un certain nombre d'information:

* **took**: c'est simplement le temps écoulé pour exécuter la requête (en millisecond).
* **time_out**: si oui ou nom la requête a expiré.
* **_shards**: comment les partitions ont été sollicitées pour cette recherche, avec le nombre de succés et d'échecs. 
* **hits**: correspond au résultat de la recherche, avec le nombre total de documents correspondant aux critères 
(total), le document lui-même (_source), et la pertinence du document par rapport à la requête (_score). Le résultat est trié par 
défaut sur cette pertinence.

<i class="fa fa-info-circle"></i> ElasticSearch affiche seulement les dix premiers documents correspondant à la recherche. 
Il existe une API afin de naviguer dans les résultats.
 
#####Query DSL
ElasticSearch dispose d'un langage puissant permettant de rechercher de manière fine certaines informations. Voici un exemple:

    $ curl -XGET 'localhost:9200/blog/post/_search' -d '
    {"query": 
        {"match": {"text": "test"}}
    }'
    
Je ne vais pas détailler toutes les fonctionnalités offertes par le langage d'interrogation. Je vous invite donc à regarder 
la documentation [**QueryDSL**](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-queries.html).

#Gestion du mapping
Le *mapping* via ElasticSearch permet de définir comment traîter la recherche d'un document et comment les champs doivent
être analysés.

Un mapping est lié à un type de données (string, integer, long, float, double, boolean). Pour chacun des types, des paramètres 
sont disponibles afin de traiter au mieux la recherche et son résultat. 

    $ curl -XPUT 'http://localhost:9200/blog/post/_mapping' -d
    '{
        "post" : {
            "properties" : {
                "author" : {"type" : "string", "index" :"not_analyzed" }
            }
        }
    }'

Ici dans cet exemple, je spécifie que mon attribut *author* de mon document est de type *string* et doit être indexé 
tel quel, en utilisant *not_analysed*.
   
Au même titre que la section **Query DSL**, la gestion du mapping de document est complexe, je ne vais pas détailler entièrement
le processus du mapping. Moi-même je n'ai pas encore exploré toutes les possibilités autour de l'analyse de vos documents.
A voir dans la documentation [**Mapping**](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/mapping.html)
pour plus d'information.


#Agrégations
Un mécanisme d'agrégation est disponible dans ElasticSearch (anciennement appelé *facets*, maintenant *deprecated*) afin 
de pouvoir extraire et grouper vos données afin de les utiliser pour un traitement particulier, comme des statistiques par exemple.

Voici un exemple simple afin d'extraire la liste des auteurs de posts dans le blog: 

    $ curl -XPOST 'localhost:9200/blog/_search?pretty' -d
    '{
        "aggs":{
            "genders" :{
                "group_by_author": {"terms": {"field": "author"}}
            }
        }
    }'

Dans mon jeu de test, le résultat de cette requête ressemble à ceci:
 
    {
        "took" : 4,
        "timed_out" : false,
        "_shards" : {
            "total" : 3,
            "successful" : 3,
            "failed" : 0
        },
        "hits" : {
            "total" : 2,
            "max_score" : 0.0,
            "hits" : [ ... ]
        },
        "aggregations" : {
            "group_by_author" : {
                "buckets" : [ {
                    "key" : "Antoine JULLIEN",
                    "doc_count" : 1
                }, {
                    "key" : "Alex KID",
                    "doc_count" : 1
                }]
            }
        }
    }
    
On voit bien ici que les auteurs des posts sont bien mis en exergue dans le tag *aggregations* du résultat de la requête.    

#Conclusion
Le but de ce premier billet technique était de vous présenter cette technologie d'indexation et de recherche 
que nous fournit ElasticSearch. Nous avons de plus en plus besoin de recherche full-text ultra performantes dans nos applications, 
et cela va de notre simple blog jusqu'à notre middleware distribué contenant des milliers de Teraoctets de données. Et 
ElasticSearch peut effectivement être une solution, au même titre que l'est peut-être son concurrent [**Solr**](http://lucene.apache.org/solr/), 
qui est lui-même encore une fois basé sur **Lucene**. Seulement, je ne peux pas comparer ces deux moteurs puisque je n'ai
pas eu l'occasion de jouer avec **Solr**.

En tout cas pour ElasticSearch, j'ai trouvé que sa mise en place et son utilisation sont vraiment simples et que cela peut
vraiment apporter une valeur ajoutée à vos applications. De plus, l'API cliente est disponible dans la plupart des langages.
Donc à vous de jouer !