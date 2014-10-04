---
layout: post
title:  "Indexer vos données via ElasticSearch"
date:   2014-10-02 22:00:24
tags: [elasticsearch, nosql]
categories: jekyll update
---
Il y a quelques temps, j'ai commencé à regarder d'un peu plus près l'outil 
[**ElasticSearch**](http://www.elasticsearch.org/), le super moteur de recherche créé par la société du même nom. 
Je vais tenter d'expliquer dans cet article comment fonctionne ElasticSearch, et de montrer quels sont les avantages 
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
 
{% img centeredImage /images/ES_Architecture.png width height 'Architecture Elastic Search' 'Architecture Elastic Search' %}

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
  
#Démarrage

L'installation et la mise en route de l'outil sont très simples. ElasticSearch a seulement besoin d'une JVM pour tourner 
(OpenJDK ou celle d'Oracle, en version 7 ou 8). Après avoir téléchargé et décompressé l'archive contenant ElastisSearch dans votre répertoire 
favoris, il vous suffit de lancer l'outil via la commande:
    
``$ <ELASTIC_HOME_DIRECTORY>/bin/elasticsearch -f``

Et voilà, votre moteur de recherche est démarré via cette instance et près à être utilisé. Facile, non ?

