# Tutoriel pour la création d'une API de données à partir de données existantes grâce à l'outil Netlify
## Notre exemple : création d’une API pour les dates de jours fériés en France



### Netlify
[Netlify](https://www.netlify.com/) est un outil qui propose des services d'hébergement web pour des sites web statiques, par exemple à partir de dépôts Github.
Dans ce tutoriel nous allons l'utiliser pour créer une API Rest à partir de données d'un dépôt Github.

### 1. Contraintes
Une API construite avec Netlify est restreint par quelques contraintes :
<ul>
<li> L'API permet uniquement de consulter les données, impossible de les modifier ou les supprimer.
<li> Le nombre de fichiers composant les jeux de données ne doit pas être trop grand. (Quelques centaines au maximum)
<li> L'API doit respecter les conditions d'utilisation de Netlify :https://www.netlify.com/tos/ . Les quotas pour des comptes gratuits: 100GB/ mois de bande passante et 100GB de stockage.
</ul>

### 2. Mise en place

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

DATA_GOUV = 'https://www.data.gouv.fr/fr/datasets/r/'

modes = [
    ('data/', DATA_GOUV + 'cc620384-4ccf-41ae-a7ba-9eceacb7b6db'),
    ('data/alsace-moselle/', DATA_GOUV + '944504ac-2592-4503-acd9-6befe8942ae2'),
]

for mode in modes:
    base_path, url = mode
    os.makedirs(base_path, exist_ok=True)

    response = urlopen(url).read().decode('utf-8')
    reader = csv.DictReader(StringIO(response))

    data_by_year = defaultdict(list)
    for row in reader:
        year = int(row['date'][0:4])
        del row['est_jour_ferie']
        data_by_year[year].append(row)

    if max(data_by_year.keys()) < datetime.today().year + 15:
        raise ValueError('We should have bank holidays for 15 years')

    for year in data_by_year:
        filename = base_path + str(year) + '.json'
        with open(filename, 'w') as f:
            json.dump(data_by_year[year], f, ensure_ascii=False)

```


Enfin, nous allons créer un fichier `_redirects` qui va rediriger les pages de requêtes directement sur le résultat JSON, et configurer la page d'accueil comme la page du dépôt.
```
/ https://github.com/GaelleMarais/tuto-fr-api-netlify 301
/api/alsace-moselle/:year /data/alsace-moselle/:year.json 200
/api/:year /data/:year.json 200
```

### 3. Déploiement avec netlify

Nous sommes prêts à héberger notre API.

Sur la page d’accueil de [Netlify](https://www.netlify.com/), il faut se connecter avec son compte Github puis cliquer sur “create a new site from GitHub” et accepter les autorisations demandées par Netlify.

On choisi ensuite le dépôt  :

![Capture d’écran de 2019-08-13 11-34-44](https://user-images.githubusercontent.com/14167172/62934329-9276a700-bdc4-11e9-9914-6008ee4d144c.png)

Et on configure la construction de notre application web :

![Capture d’écran de 2019-08-13 14-03-45](https://user-images.githubusercontent.com/14167172/62940164-44b56b00-bdd3-11e9-8dd0-558f13dd311d.png)

### 4. Personnalisation

Par défaut, Netlify donne un nom à notre site web, que l'on peut modifier comme suit :

![Capture d’écran de 2019-08-13 12-09-59](https://user-images.githubusercontent.com/14167172/62934536-1597fd00-bdc5-11e9-918b-a44fe1e8565e.png)

![Capture d’écran de 2019-08-13 12-10-18](https://user-images.githubusercontent.com/14167172/62934540-1761c080-bdc5-11e9-9ff6-ed594c06795a.png)

Et voilà !
Nous avons créé une API très basique pour consulter des données au format JSON.

### 5. Exemple

La requete https://tuto-fr-api-netlify.netlify.com/api/2019 renvoie vers la page :


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
