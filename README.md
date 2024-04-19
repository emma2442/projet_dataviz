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


