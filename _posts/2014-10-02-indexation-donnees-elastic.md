---
layout: post
title:  "Indexer vos données via ElasticSearch"
date:   2014-10-02 22:00:24
tags: [elasticsearch, nosql]
categories: jekyll update
---
Il y a quelques temps, j'ai commencé à regarder d'un peu plus près le super outil 
[**ElasticSearch**](http://www.elasticsearch.org/), le moteur de recherche créé par la société du même nom. 
Je vais tenter d'expliquer dans cet article comment fonctionne ElasticSearch, et de montrer quels sont les avantages 
à l'utiliser dans vos applications.

Tout d'abord, commençons par une petite présentation. ElasticSearch est un moteur de recherche basé sur [**Lucene**](http://lucene.apache.org/).
Lucene fournit effectivement un puissant système basé sur un moteur d'indexation et un moteur de recherche sur ces indexes.
ElasticSearch exploite ces fonctionnalités en ajoutant une couche "élastique", c'est à dire le moyen de gérer la scalabilité horizontale (*Scale out*),
mais aussi de gérer de manière transparente le basculement (*Failover*) via son système de réplication. Pour finir, 
ElasticSearch fournit une couche d'intéropérabilité, manquante à Lucene, basée sur les standards 
[**REST**](http://fr.wikipedia.org/wiki/Representational_State_Transfer)/HTTP et [**JSON**](http://json.org/) pour le format d'échange des données.

#Démarrage
L'installation et la mise en route de l'outil est très simple. ElasticSearch a seulement besoin d'une JVM pour tourner 
(OpenJDK ou celle d'Oracle, en version 7 ou 8). Après avoir téléchargé et décompressé l'archive contenant ElastisSearch dans votre répertoire 
favoris, il vous suffit de lancer l'outil via la commande:
    
    $ <ELASTIC_HOME_DIRECTORY>/bin/elasticsearch -f

Et voilà, votre moteur de recherche est démarré et près à être utilisé. Facile, non ?

