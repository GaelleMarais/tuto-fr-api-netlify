# Tutoriel pour la création d'une API de données à partir de données existantes grâce à l'outil Netlify
## Notre exemple : création d’une API pour les dates de jours fériés en France



### Netlify
[Netlify](https://www.netlify.com/) est un outil qui propose des services d'hébergement web pour des sites web statiques, par exemple à partir de dépôts GitHub.
Dans ce tutoriel nous allons apprendre à mettre en place un site web à partir d'un dépôt GitHub, et ensuite on expliquera en détail comment s'en servir en tant qu'API pour servir des fichiers `.json` .


### 1. Contraintes
Une API construite avec Netlify est restreinte par quelques contraintes :
<ul>
<li> L'API permet uniquement de consulter les données, impossible de les modifier ou les supprimer.
<li> Le nombre de fichiers composant les jeux de données ne doit pas être trop grand. (Quelques centaines au maximum)
<li> L'API doit respecter les conditions d'utilisation de Netlify :https://www.netlify.com/tos/ . Les quotas pour des comptes gratuits: 100GB/ mois de bande passante et 100GB de stockage.
</ul>

### 2. Mise en place de notre site web

Sur la page d’accueil de [Netlify](https://www.netlify.com/), il faut se connecter avec son compte Github puis cliquer sur “create a new site from GitHub” et accepter les autorisations demandées par Netlify.

On choisit ensuite le dépôt  :

![Capture d’écran de 2019-08-13 11-34-44](https://user-images.githubusercontent.com/14167172/62934329-9276a700-bdc4-11e9-9914-6008ee4d144c.png)

Et on configure la construction de notre application web :

![Capture d’écran de 2019-08-13 14-03-45](https://user-images.githubusercontent.com/14167172/62940164-44b56b00-bdd3-11e9-8dd0-558f13dd311d.png)

*( Remarque: Voir la partie 5 du tutoriel pour la case Build Command )*

Par défaut, Netlify recherche un fichier `index.html` pour en faire la page d'accueil de notre site web. Si ce fichier n'existe pas, alors la page affichera une erreur.

### 3. Redirections

Dans notre exemple, on ne veut pas créer de page `index.html` mais par exemple faire en sorte que le lien de notre site web redirige directement vers le dépôt GitHub.
Pour ce faire, on crée un fichier `_redirects` à la racine du dépôt :
```
/ https://github.com/GaelleMarais/tuto-fr-api-netlify 301
```

La syntaxe est la suivante :
```
chemin_originel chemin_vers_lequel_rediriger code_HTTP
```

Le code HTTP `301` indique qu'il s'agit d'une redirection permanente. Voir [la documentation](https://www.netlify.com/docs/redirects/) pour plus de détails.


### 4. Personnalisation

Par défaut, Netlify donne un nom à notre site web, que l'on peut modifier comme suit :

![Capture d’écran de 2019-08-13 12-09-59](https://user-images.githubusercontent.com/14167172/62934536-1597fd00-bdc5-11e9-918b-a44fe1e8565e.png)

![Capture d’écran de 2019-08-13 12-10-18](https://user-images.githubusercontent.com/14167172/62934540-1761c080-bdc5-11e9-9ff6-ed594c06795a.png)

Notre application web est prête ! On peut la consulter à l'adresse https://tuto-fr-api-netlify.netlify.com.

Pour l'instant, elle ne sert pas à grand chose, et nous allons voir comment s'en servir d'une API.


### 5. Création des fichiers JSON

Pour notre exemple, nous utilisons [les dates de jours fériés en France](https://www.data.gouv.fr/fr/datasets/jours-feries-en-france/).

À partir de ces données au format `.csv`, nous allons écrire un script pour créer des fichiers JSON que notre API va permettre de consulter. En particulier, nous voulons obtenir un fichier JSON par année, qui contient tous les jours fériés de l'année en question.

Netlify supporte de nombreux langages pour la phase de construction. (Voir la [documentation](https://www.netlify.com/docs/build-settings/) pour la liste complète.)

Nous allons utiliser un script Python 3.6 `build.py` disponible [ici](https://github.com/AntoineAugusti/api-jours-feries-france/blob/master/build.py).
```python
import csv
import json
import os
from collections import defaultdict
from datetime import datetime
from io import StringIO
from urllib.request import urlopen

# Définition des URLs où récupérer les données
DATA_GOUV = 'https://www.data.gouv.fr/fr/datasets/r/'

modes = [
    ('data/', DATA_GOUV + 'cc620384-4ccf-41ae-a7ba-9eceacb7b6db'),
    ('data/alsace-moselle/', DATA_GOUV + '944504ac-2592-4503-acd9-6befe8942ae2'),
]

# Opérations à faire sur chaque URL
for mode in modes:
    base_path, url = mode
    os.makedirs(base_path, exist_ok=True)  # Création du dossier où stocker les fichiers .json

    response = urlopen(url).read().decode('utf-8') # Ouverture de l'URL qui contient le csv
    reader = csv.DictReader(StringIO(response)) # Ouverture du fichier .csv obtenu

    data_by_year = defaultdict(list)  # Création d'une liste vide
    for row in reader:
        year = int(row['date'][0:4]) # Lecture de la date
        del row['est_jour_ferie']
        data_by_year[year].append(row) # Ajout de la donnée dans la liste

    if max(data_by_year.keys()) < datetime.today().year + 15:
        raise ValueError('We should have bank holidays for 15 years')

    # Création d'un fichier pour chaque année
    for year in data_by_year:
        filename = base_path + str(year) + '.json'
        with open(filename, 'w') as f:
            json.dump(data_by_year[year], f, ensure_ascii=False)

```

Maintenant nous allons ajouter les pages de requêtes dans notre fichier `_redirects` pour les envoyer directement sur le fichier JSON :

```
/ https://github.com/GaelleMarais/tuto-fr-api-netlify 301
/api/alsace-moselle/:year /data/alsace-moselle/:year.json 200
/api/:year /data/:year.json 200
```

### 5. Exemple

La requête https://tuto-fr-api-netlify.netlify.com/api/2019 renvoie vers la page :


![Capture d’écran de 2019-08-13 14-09-42](https://user-images.githubusercontent.com/14167172/62940558-24d27700-bdd4-11e9-94fa-c821b2be227c.png)



### Licence

2019 Direction interministérielle du numérique et du système
d'information et de communication de l'État. <br/>

2019 Les contributeurs accessibles via l'historique du dépôt. <br/>

Les contenus accessibles dans ce dépôt sont placés sous [Licence
Ouverte 2.0](LO.md).  Vous êtes libre de réutiliser les contenus de ce dépôt
sous les conditions précisées dans cette licence. </br>

Ce document est écrit par Gaëlle Marais à Etalab.

### Sources

Ce tutoriel est un équivalent en français de [cet article](https://blog.antoine-augusti.fr/2019/01/serving-a-json-rest-api-without-infrastructure-thanks-to-netlify/) d'Antoine Augusti.
