---
layout: post
title:  "Coupler une base existante NoSQL avec ElasticSearch"
date:   2014-10-09 19:00:00
tags: [elasticsearch, nosql, mongodb]
categories: jekyll update
---

Dans le précédent post, je vous ai présenté le fonctionnement et l'utilisation du moteur de recherche *ElasticSearch*, dans 
un cas d'utilisation où nous utilisons à la fois les mécanismes de recherche mais aussi la couche de persistence.

      
Dans ce billet, je vais expliquer comment ajouter un service de recherche ElasticSearch dans une application utilisant déjà une couche de persistence 
existante de type NoSQL comme [**MongoDB**](http://www.mongodb.org/), [**Redis**](http://redis.io/) ou [**Couchbase**](http://www.couchbase.com/fr) par exemple.

#Plugin "River" d'ElasticSearch

ElasticSearch dispose d'un mécanisme de gestion de plugin permettant d'augmenter certaines fonctionnalités comme l'analyse de données (plugins *analysis*), 
la visualisation et la supervision d'un cluster ElasticSearch (plugins *site*), ou encore l'indexation de données provenant 
de sources différentes (plugin *river*). Ces plugins peuvent être soit développés et supportés directement par ElasticSearch ou soit 
fournis par la communauté open-source puisque une API est disponible afin d'écrire soit-même son propre plugin.
Ici, nous allons nous intéressé au plugin *river*, et plus particulièrement celui qui s'interface avec **MongoDB** écrit 
par Richard Louapre ([**MongoDB river plugin**](https://github.com/richardwilly98/elasticsearch-river-mongodb)) pour illustrer 
ce cas d'utilisation.

<i class="fa fa-arrow-circle-right"></i> Le fonctionnement du plugin *river* doit effectivement être vu comme la métaphore de la rivière. Il permet ainsi de déverser
 le contenu d'une source de données vers ElasticSearch.
 
![Plugin River](/images/River.png){: .centeredImage}

####Installation automatique

Dans le cas où vous utilisez la version 1.2.2 d'ElasticSearch, vous pouvez utiliser l'installation automatique:

``$ <ELASTIC_HOME_DIRECTORY>/bin/plugin --install com.github.richardwilly98.elasticsearch/elasticsearch-river-mongodb/2.0.1``

<i class="fa fa-info-circle"></i> Sur son dépôt Github, Richard Louapre détaille les versions compatibles d'ElasticSearch avec celles de son plugin. 

####Installation manuelle

Dans mon environnement, j'utilise la version 1.3.4 d'ElasticSearch et la version 2.6.5 de MongoDB. Je suis dans l'obligation 
de faire une installation manuelle car ces versions ne sont pas encore supportées:

* Cloner tout d'abord le dépot **elasticsearch-river-mongodb**:

``$ git clone https://github.com/richardwilly98/elasticsearch-river-mongodb.git``

* Compiler le project:

``$ mvn clean package -DskipTests``

* Installer le plugin dans ElasticSearch:

``$ <ELASTIC_HOME_DIRECTORY>/bin/plugin --url "file://<PATH_ELASTIC_RIVER_PLUGIN>/target/elasticsearch-river-mongodb-<VERSION>.jar" --install elasticsearch-river-mongod``

* Copie du driver mongodb

``$ cp mongo-java-driver-2.12.3.jar <ELASTIC_HOME_DIRECTORY>/lib``
 
Et voilà, l'installation du plugin **elasticsearch-river-mongodb** est terminée.

#MongoDB
  
Ici, imaginons que nous utilisons MongoDB afin de stocker un blog contenant des posts. Je vais donc créer une 
base de données *blog* avec une collection *post*:

* Création du répertoire de stockage:

``$ mkdir <PATH>/data``

* Démarrage du process **mongod**:

``$ <MONGO_INSTALLATION>/bin/mongod --dbpath <PATH>/data --replSet rs0`` 

<i class="fa fa-info-circle"></i> L'argument **\--replSet** permet de spécifier que notre instance appartient à un groupe de replicas.
Cet argument est obligatoire car le plugin *elasticsearch-river-mongodb* utilise le mécanisme du **oplog** 
(les logs d'opérations) afin de synchroniser et indexer les données contenues dans MongoDB. 

* Démarrage du process **mongod**:

``$ <MONGO_INSTALLATION>/bin/mongod --dbpath <PATH>/data --replSet rs0`` 

* Déployer le groupe de replicas:

``$ echo 'rs.initiate()' | mongo``

Ajoutons maintenant un petit jeu de données:

    $ mongo
    > use blog
    > db.post.save({"title": "My first post", "text": "A simple text for my first post", "author": "Antoine JULLIEN"})
    > db.post.save({"title": "My second post", "text": "A simple text for my second post", "author": "Antoine JULLIEN"})
    > db.post.save({"title": "My third post", "text": "A simple text for my third post", "author": "Antoine JULLIEN"})
     
#ElasticSearch

A partir de là, c'est quasiment terminé. Nous pouvons démarrer ElasticSearch:

``$ <ELASTIC_HOME_DIRECTORY>/bin/elasticsearch -f``

et configurer le plugin *river* afin d'indexer tous les documents *post* de la base de données *blog* dans l'index ElasticSearch 
*blog* avec le type *post*:

    curl -XPUT 'http://localhost:9200/_river/mongodb/_meta' -d '{ 
        "type": "mongodb", 
        "mongodb": { 
          "db": "blog", 
          "collection": "post"
        }, 
        "index": {
          "name": "blog", 
          "type": "post" 
        }
    }'

Vous pouvez vérifier que vos documents contenus dans MongoDB sont correctement indexés:

``$ curl -XGET 'localhost:9200/blog/post/_search?pretty'``

    {
        "took" : 1,
        "timed_out" : false,
        "_shards" : {
            "total" : 3,
            "successful" : 3,
            "failed" : 0
        },
        "hits" : {
            "total" : 3,
            "max_score" : 1.0,
            "hits" : [ {
                "_index" : "blog",
                "_type" : "post",
                "_id" : "543c341154bb4eb903463c47",
                "_score" : 1.0,
                "_source":{"author":"Antoine JULLIEN","_id":"543c341154bb4eb903463c47","text":"A simple text of my first post","title":"My first post"}
                }, 
                {
                "_index" : "blog",
                "_type" : "post",
                "_id" : "543c4ff19e7c795cbfc0fe13",
                "_score" : 1.0,
                "_source":{"author":"Antoine JULLIEN","_id":"543c4ff19e7c795cbfc0fe13","text":"A simple text of my second post","title":"My second post"}
                }, 
                {
                "_index" : "blog",
                "_type" : "post",
                "_id" : "543c340b54bb4eb903463c46",
                "_score" : 1.0,
                "_source":{"author":"Antoine JULLIEN","_id":"543c340b54bb4eb903463c46","text":"A simple text of my third post","title":"My third post"}
                } 
            ]
        }
    }



#Conclusion

// TODO



