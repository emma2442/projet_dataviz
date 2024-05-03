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
  <img src="https://github.com/emma2442/projet_dataviz/assets/102244339/c8eea5be-0e45-4214-b12b-6b9d24a68806" width="700">
</div>

# Partie 3

<div align="center">
  <img src="https://github.com/emma2442/projet_dataviz/assets/102244339/2cbc74a7-fec5-4ae9-bd0b-f5988eb02848" width="500">
</div>

<div align="center">
  <img src="https://github.com/emma2442/projet_dataviz/assets/102244339/9ee35b3e-fce5-464e-a3f0-85b85eb18af5" width="500">
</div>

Nous n'avons pas réussi à retirer le champ "message", cela doit être dû à la version car en effectuant la même chose sur differents PC parfois ça fonctonnait et d'autres fois non.


# Projet

sujet : velib (vélos et bornes)
source de données : https://opendata.paris.fr/explore/dataset/velib-disponibilite-en-temps-reel/api/?disjunctive.name&disjunctive.is_installed&disjunctive.is_renting&disjunctive.is_returning&disjunctive.nom_arrondissement_communes

L'API nous permet de récupérer les données en JSON, 100 stations à la fois. Il y en a 1467 en tout. On incrémente l'offset dans l'url de 100 à la fois afin de récupérer toutes les stations.

commande pour entrer dans le docker : docker exec -it elk-logstash /bin/bash


# Fonctionnement de notre projet 

Nous sommes parti de l'architecture que nous avons pu récupérer sur le GitHub suivant : https://gist.github.com/labynocle/35a4c5b7411c5497c0e30398531b580a

Nous l'avons adaptée à nos besoins, en effet nous avons un nouveau container qui est concacré à la récupération des données.
Au sein de ce container nous avons un script qui récupère toutes les données concernant les Vélibs à une intervalle de 5 minutes. Par la suite, nous retrouvons le répertoire data qui contient les données récupérées par notre script. Nous allons également retrouver un repertoire classique Kibana ainsi que logstash.

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
## Fonctionnement individuel de chaque container
### Récupération des données

Afin de récupérer nos données nous utilisons un container avec la configuration suivante :

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

Ici nous allons venir récupérer les données de nos vélibs grâce à notre script recup.sh qui est donc en Bash.

Au sein de notre script nous pouvons voir le fichier "stations.json" qui est notre fichier de sortie. Celui-ci est accéssible grâce à notre configuration sur le docker-compose. $${\color{red}ATTENTION}$$ Il faut impérativement changer le chemin du volume où est présent le répertoir data du projet avant les ":", la seconde partie est le chemin au sein du container donc il ne faut pas le changer.

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

Une fois nos données récupérées, elles sont placées dans notre répertoire data où nous allons retrouver le fichier "stations.json" contenant toutes les informations des vélibs.

### Logstash

Création du container avec l'import de l'image docker : 

```ruby
FROM logstash:7.5.0
```

Logstash va nous servir à récupérer les données ainsi qu'à les traiter. Pour ce faire nous avons donc la configuraiton docker-compose suivante :

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
Au sein de cette configuration nous allons principalement retrouver le fichier de conf que nous allons configurer juste après, ainsi que les fichiers qui doivent être accéssibles. Il y a ensuite le port sur lequel le service va écouter et être disponible. $${\color{red}ATTENTION}$$ Tel que pour le container précédent il faut changer le lien du répertoire data.

Dans notre cas notre fichier "stations.json" contenant les données est allimenté toutes les 5 minutes donc il faut le regarder en continu et, lors de chaque ajout, aller chercher et récupérer les nouvelles données et les rendre disponibles sur nos interfaces visuelles.

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
}

output {
	elasticsearch {
		hosts => "elasticsearch:9200"
	}
}
```

Nous commençons par regarder le fichier "stations.jon", ensuite nous allons venir traiter les données et récupérer le contenu des objets "message". Nous mettons enfin la date au format souhaité avec la bonne timezone.

Une fois tout cela fait nous envoyons nos données vers elasticsearch qui lui écoute sur le port 9200.

### Elastic Search 

Elastic Search n'est configuré que au sein de notre docker-compose, il va venir récupérer une image puis nous allons le lancer avec certains critères tel que l'adresse ip host, ou encore le port. Enfin, nous lui attribuons les ports. 

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

Cerebro offre une interface afin de pouvoir afficher le stockage de nos données, sa configuration est relativement simple car nous récupérons tout simplement son image. Nous lui indiquons qu'il dépend de elasticsearch ainsi que le port sur lequel il doit écouter.

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

Kibana est l'interface que nous utilisons afin de pouvoir créer des visuel sur nos différentes données. Pour accéder à cette interface il nous faut créer le Dockerfile suivant :

```ruby
FROM kibana:7.5.0

USER root

RUN yum update -y && yum install -y nmap

COPY entrypoint.sh /tmp/entrypoint.sh
RUN chmod +x /tmp/entrypoint.sh

USER kibana

CMD ["/tmp/entrypoint.sh"]
```

Ici, comme les containers précédents, nous venons chercher une image, puis nous mettons à jour la bibliothèque yum et nous installons nmap. Nous venons également récupérer notre script "entrypoint.sh", lui donnons les droits d'éxécution puis l'éxécutons.

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

Dans notre cas elle est relativement rapide puisque nous retrouvons seulement les fichier auquels il à accès, la dépendance avec elasticsearch et enfin le port "5601".

Concernant le fichier de config "kibana.yml", il permet à toutes les adresses IP d'accéder au service mais également de configurer le monitorinf et le nom du server.

```ruby
# From https://github.com/elastic/dockerfiles/tree/v6.7.0/kibana
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

# Visualisation dans Kibana

Nos données sont structurées de la manière suivante :  
  
Pour chaque station de vélib :  
	- stationcode  
	- name  
	- is_installed  
	- capacity  
 	- numdocksavailable  
	- numbikesavailable  
	- mechanical  
	- ebike  
	- is_renting  
	- is_returning  
	- duedate  
	- coordonnees_geo (avec sous-champs lon et lat)  
 	- nom_arrondissement_communes  

Nous commençons par compter le nombre de stations et de villes (nom_arrondissement_communes) différents que nous avons.

<div align="center">
<img width="692" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/e28208c0-fba8-47da-91ee-8ec0b0abbfa6">
</div>

On peut voir que si on filtre sur paris, nous avons 971 stations et evidemment une ville.

<div align="center">
<img width="695" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/cb3dc831-569a-4cb9-be7f-1d810995833f">
</div>

Pour avoir le nombre de station par ville, on fait un bar chart (échelle logarithmique car paris a beaucoup plus de stations que les autres villes ce qui rend le graphe illisible avec une échelle linéaire).

<div align="center">
<img width="1385" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/a0aff792-eb9f-47cb-822c-25f7c75da7b6">
</div>

Ici nous avons le nombre de vélos méchaniques et électriques disponibles par ville.

<div align="center">
<img width="1387" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/7e941ac6-aa45-4ef2-b41c-343c274bef1b">
</div>

Voici un pie chart des stations en service, nous pouvons voir que 0.07% des stations ne le sont pas, ce qui est très faible.

<div align="center">
<img width="690" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/c75e0a7a-ec86-4b8e-a57a-5ffb54644a71">
</div>

Ici nous avons le nombre moyen de docks disponibles par station, ainsi que le nombre de vélos mécaniques et électriques. On observe qu'il y a en moyenne plus de docks disponibles que de vélos mécaniques et électriques réunis.

<div align="center">
<img width="693" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/552daa06-4601-4f9a-8af2-bbe927283fb7">
</div>

En filtrant sur une station (Place Nelson Mandela) nous pouvons voir que les proportions sont différentes et il y a beaucoup moins de vélo disponibles.

<div align="center">
<img width="690" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/5c475559-7ad0-4332-b20b-46c4e698f012">
</div>

Nous avons représenté l'évolution temporelle du nombre de vélos (mécaniques et électriques) disponibles, il serait aussi interessant de filtrer par station ou par ville.

<div align="center">
<img width="691" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/d5052c26-1d3c-40a8-b107-57436e4e97e0">
</div>

Une visulatisation interessante serait d'avoir une carte qui affiche toutes les stations de vélib. Nous avons les coordonnées de chaque station, mais pour créer une map dans kibana il faut que ce champ soit qualifié de géo_point et ce n'est pas le cas automatiquement.

Nous devons donc créer un template d'index dans cerebro qui indique le mapping tel que :

```ruby
  "mappings": { - 
      "properties": { - 
        "@timestamp": { - 
          "type": "date"
        },
        "coordonnees_geo2": { - 
          "type": "geo_point"
        },
```

Le souci est que le champ est de la forme : "coordonnees_geo": {"lon": 2.2754641655975, "lat": 48.822036188382} et kibana accepte le champ en tant que geo_point seulement si c'est une liste avec longitude en premier puis latitude mais dans les "lon" et "lat". Nous avons donc aussi modifié le logstash.


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
                add_field => { "[coordonnees_geo2]" => "%{[coordonnees_geo][lat]},%{[coordonnees_geo][lon]}" }
        }
        mutate {
                remove_field => ["coordonnees_geo"]
        }
}
output {
        elasticsearch {
                hosts => "elasticsearch:9200"
                index => "logs"
        }
}
```

Maintenant que les coordonnées sont bien des geo_points, nous avons fait une première visualisation de toutes les stations. La taille du point est proportionelle à la capacité de la station. Si la station est complète (aucun dock disponible) alors il apparait en rouge. Enfin, pour chaque station, nous affichons le nom, le nombre de vélos et docks disponibles, et l'heure à laquelle l'information a été récupérée.

<div align="center">
<img width="692" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/a847084b-4248-4253-8013-640d21fac09e">
<img width="691" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/5d10a3eb-bd0c-4644-9972-fd56d6de58cd">
</div>

De la même manière, nous avons fait une map des stations mais cette fois-ci oritentée sur le nombre de vélo disponibles, lorsque le symbol de vélo est rouge c'est qu'il n'y en a aucun de disponible.

<div align="center">
<img width="698" alt="image" src="https://github.com/emma2442/projet_dataviz/assets/102244339/2edb5f91-182b-4919-abcd-55b0d774da23">
</div>
