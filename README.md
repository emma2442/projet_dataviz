# Projet dataviz
Théo Chaloyard & Emma Lovisi

## Prise en main de Kibana avec elk-demo

# Partie 1

Nous utilisons une vm parrot équipée de docker

<div align="center">
  <img src="https://github.com/emma2442/projet_dataviz/assets/102244339/660bcd4b-7913-46c9-8ed3-92f0be0e61ca" width="500">

  <img width="1437" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/f6916171-e19f-4ee8-9651-ee62cd1f0204">
</div>

# Partie 2

Pas de difficulté rencontrée

<div align="center">
  <img src="https://github.com/emma2442/projet_dataviz/assets/102244339/29767519-9ce5-4e5a-8497-e45939960ef9" width="700">
</div>

# Partie 3

Nous utilisons une vm parrot équipée de docker

<div align="center">
  <img src="https://github.com/emma2442/projet_dataviz/assets/102244339/2cbc74a7-fec5-4ae9-bd0b-f5988eb02848" width="500">
</div>

<div align="center">
  <img src="https://github.com/emma2442/projet_dataviz/assets/102244339/9ee35b3e-fce5-464e-a3f0-85b85eb18af5" width="500">
</div>

(soucis avec le remove field de message)

# Projet

sujet : velib (vélos et bornes)
source de données : https://opendata.paris.fr/explore/dataset/velib-disponibilite-en-temps-reel/api/?disjunctive.name&disjunctive.is_installed&disjunctive.is_renting&disjunctive.is_returning&disjunctive.nom_arrondissement_communes

L'API nous permet de récupérer les données en JSON, 100 stations à la fois. Il y en a 1467 en tout. On incrémente l'offset dans l'url de 100 à la fois afin de récupérer 

commande pour entrer dans le docker : docker exec -it elk_logstach /bin/bash

exporter les visualisations pour pas les faire

pour les points geo : créer un template d'index dans cerebro de la forme
PUT /votre_index
{
  "mappings": {
    "properties": {
      "coordonnees_geo": {
        "type": "geo_point"
      }
    }
  }
}

# Fonctionnement de notre projet 
Voici donc l'architecture que nous avons choisi pour notre projet. Nous sommes donc parti de l'architecture que nous avons pu récupérer sur le GitHub suivant : https://gist.github.com/labynocle/35a4c5b7411c5497c0e30398531b580a

Nous l'avons adapté à nos besoins, en effet nous avons donc un nouveau container qui est concacré à la récupération des données.
Au sein de ce container nous avons donc un script qui va aller récupérer toutes les données concernant les Vélib à une intervalle de 5 minutes. Par la suite nous retrouvons donc le répertoire data qui contient les données récupéré par notre script. Nous allons égallement retrouver un repertoure classique Kibana ainsi que logstash.

```
projet_dataviz/
├── data/
│   ├── stations.json
│   └── uuid
├── kibana/
│   ├── config/
│   │   └── kibana.yml
│   ├── Dockerfile
│   └── entrypoint.sh
├── logstash/
│   ├── config/
│   │   └── logstash.conf
│   └── Dockerfile
└── recuperationdesdonnees/
    ├── Dockerfile
    ├── recup.sh
    └── README.md
└── docker-compose.yml
```
## Fonctionnement individuel de chaque container. 
### Récupération des données

Afin de récupérer nos données nous utilisons donc un container utilisant la configuration suivante :

```ruby
# Use Alpine for a minimal base image
FROM alpine:latest

# Install bash, curl and jq
RUN apk add --no-cache bash curl jq

# Copy the script into the container
COPY recup.sh /recup.sh

# Ensure the script is executable
RUN chmod +x /recup.sh

# Command to run the script
CMD ["/recup.sh"]
```
Nous pouvons voir que nous construisons notre container avec une image alpine et que nous installons "curl" ainsi que "jq" afin de pouvoir récupérer et manipuler nos données dans le container. Par la suite nous allons copier le script à la racine de notre container, lui donner les droits d'éxécution pour enfin l'éxécuter.

Ici nous allons donc venir récupérer les données de nos vélib grâce à notre script recup.sh qui est donc en Bash

Au sein de notre script nous pouvons donc voir le fichier "stations.json" qui est donc notre fichier de sortie. Celui-ci est accéssible grâce à notre configuration sur le docker-compose. $${\color{red}ATTENTION}$$ Il faut impérativement changer le chemin du volumes où est présent le répertoir data du projet avant les ":", la seconde partie est donc le chemin au sein du container donc il ne faut pas le changer.

```ruby
data_recuperation:
    build: ./recuperationdesdonnees
    container_name: data_recuperation
    privileged: true
    volumes:
      - /home/etudiant/Desktop/pro/projetdata/data:/usr/share/recuperationdesdonnees/data
```

Les URL sont pour le moment en static, elles sont donc toutes parcourues une à une par la boucle et leur contenu est ajouté au fichier "stations.json".

```ruby
#!/bin/bash

# Définir le chemin complet du fichier de sortie
output="/usr/share/recuperationdesdonnees/data/stations.json"

# Vérifier si le fichier existe déjà
if [ ! -f "$output" ]; then
    # Créer un fichier vide si nécessaire
    touch "$output"
fi

urls=("Liens*")

# Boucle infinie pour exécuter le script toutes les 5 minutes
while true; do

    # Boucle sur chaque URL
    for url in "${urls[@]}"
    do
        # Télécharger le contenu JSON de l'API
        json=$(curl -s "$url")

        # Utiliser jq pour itérer sur chaque station dans la liste 'results' et les écrire dans le fichier de sortie
        # En ajoutant chaque nouvelle entrée à la fin du fichier, chaque objet sur une nouvelle ligne
        echo "$json" | jq -c '.results[]' >> "$output"
    done

    # Afficher le chemin du fichier de sortie
    echo "Les données sont écrites dans $output"

    # Attendre 5 minutes avant de recommencer
    sleep 900

done
```

### Data

Une fois nos données récupérées elle sont donc dans notre répertoire data où nous allons retrouver le fichier "stations.json" contenant tous les informations des Vélib.
Maintenant il s'agit de les récupérer et de les utiliser.

### Logstash

Création du container avec l'import de l'image docker : 

```ruby
FROM logstash:7.5.0
```

Logstash va maintenant nous servir à pouvoir récupérer les données ainsi qu'à les traiter. Pour ce faire nous avons donc la configuraiton docker-compose suivante :

```ruby
logstash:
    build: logstash/
    container_name: elk-logstash
    command: logstash -f /etc/logstash/conf.d/logstash.conf --config.reload.automatic
    volumes:
      - ./logstash/config:/etc/logstash/conf.d
      - /home/etudiant/Desktop/pro/projetdata/data:/usr/share/logstash/data
    ports:
      - "5000:5000"
    links:
      - elasticsearch
```
Au sein de cette configuration nous allons principalement retrouver le fichier de conf que nous allons configurer juste après ainsi que les fichier qui doivent être accéssible. Par la suite nous avons donc le port sur lequel le service va donc écouter et être disponible. $${\color{red}ATTENTION}$$ Tel que pour le container précédent il faut changer le lien du répertoire data.

Dans notre cas notre fichier "stations.json" contenant les données est allimenté toutes les 5 minutes donc il nous faut le regarder en continu et lors de chaque ajout aller chercher pour récupérer les nouvelles données et les rendre disponible sur nos interfaces visuelles.

Nous allons donc configurer tout cela dans le fichier :
```
├── logstash/
│   ├── config/
│   │   └── logstash.conf
```
```ruby
input {
	file {
    path => "/usr/share/logstash/data/stations.json"
  }
}

filter {
	json {
		source => "message"
		remove_field => "message"
	}
	date {
		match => [ "[duedate]", "ISO8601" ]
		target => "[duedate]"
		timezone => "Europe/Paris"
	}
	mutate {
		convert => [ "[location]", "float" ]

	}
}

output {
	elasticsearch {
		hosts => "elasticsearch:9200"
	}
}
```

Nous vennons donc commencer par regarder le fichier "stations.jon", ensuite nous allonsvenir traiter les données en récupérer le contenu des objets "message". Ensuite nous allons mettre la date au format souhaité avec le bon bon timezone, nous allons égallement convertir les valeurs de "longitude" et de "latitude" en "float" pour avoir des données à virgules.

Une fois tout cela fait nous allons donc envoyer nos données vers elasticsearch qui lui écoute sur le port 9200.

### Elastic Search 

Elastic Search n'est lui configuré que au sein de notre docker-compse, il va venir récupérer un image puis nous allons le lancer avec certains critères tel que l'adresse ip host, ou encore le port. Et enfin nous allons venir lui attribuer les ports. 

Elastic Search lui nous sert donc à venir stocker nos données correctement.
```ruby
 elasticsearch:
    image: elasticsearch:7.5.0
    container_name: elk-es
    command: elasticsearch
              -Enetwork.host=0.0.0.0
              -Ehttp.cors.enabled=true
              -Ehttp.cors.allow-origin=/.*/
              -Ediscovery.type=single-node
              -Etransport.host=localhost
              -Etransport.tcp.port=9300
              -Ehttp.port=9200
    ports:
      - "9200:9200"
      - "9300:9300"
```
### Cerebro 

Cerebro va donc nous offrir une interface afin de pouvoir afficher le stockage de nos données, sa configuration est relativement simple car nous récupérer tout simplement son image, nous lui indiquons qu'il dépend de elasticsearch ainsi que le port sur lequel il doit écouter.

```ruby
  cerebro:
    image: lmenezes/cerebro:0.8.5
    container_name: elk-cerebro
    ports:
      - "9000:9000"
    links:
      - elasticsearch
```

### Kibana

Kibana est l'interface que nous utilisons afin de pouvoir créer des visuel sur nos différentes données. Pour accéder à cette interface il nous faut donc créer le Dockerfile suivant :

```ruby
FROM kibana:7.5.0

USER root

RUN yum update -y && yum install -y nmap

COPY entrypoint.sh /tmp/entrypoint.sh
RUN chmod +x /tmp/entrypoint.sh

USER kibana

CMD ["/tmp/entrypoint.sh"]
```

Ici tout comme les précéndents container nous venons chercher une image, ne vennons mettre à jour la bibliothèque yum puis nous installons nmap. Nous vennons également récupérer notre script "entrypoint.sh", lui donner les droits d'éxécution puis nous vennons l'éxécuter.

Voici son contenu :

```ruby
#!/usr/bin/env bash

# Wait for the Elasticsearch container to be ready before starting Kibana.
echo "Stalling for Elasticsearch"
while true; do
	# CentOS uses ncat (from the nmap package)
	nc -z elasticsearch 9200 2>/dev/null && break
done

echo "Starting Kibana"
exec kibana
```

Ce script sert à attendre que Elasticsearch soit bien setup avant de lancer Kibana et une fois cela confirmé il lance Kibana.

Maintenant regardons la configuration que nous avons pour le docker-compose :

```ruby
  kibana:
    build: kibana/
    container_name: elk-kibana
    volumes:
      - ./kibana/config/:/opt/kibana/config/
    ports:
      - "5601:5601"
    links:
      - elasticsearch
```

Dans notre cas elle est relativement faible puisque nous retrouvons seulement les fichier auquels il à accès, la dépendance avec elasticsearch et enfin le port "5601".

Concernant le fichier de config "kibana.yml", il permet à toutes les adresses IP d'accéder au service mais également il vient configurer le monitorinf et le nom du server.

```ruby
# From https://github.com/elastic/dockerfiles/tree/v6.7.0/kibana
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true
```
