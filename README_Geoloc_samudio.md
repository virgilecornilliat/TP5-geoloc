# TP géoloc addresses

> **⚠️ à télécharger**
>
> pour faire cette activité sur votre ordi, commencez par télécharger le zip : [ARTEFACTS-addresses.zip](./ARTEFACTS-addresses.zip)

ce TP vise à vous faire découvrir quelques possibilités de manipulation et d'affichage de données géographiques

> **survol**
>
> voici les différentes étapes du TP en résumé
>
> 1. on part d'un gros paquet d'adresses parisiennes
> 2. on va commencer par les géolocaliser: trouver leurs latitude/longitude
> 3. on va ensuite les afficher sur une carte
> 4. puis on va ajouter dans le mélange [les 80 quartiers parisiens](https://fr.wikipedia.org/wiki/Liste_des_quartiers_administratifs_de_Paris); d'abord en les affichant simplement sur la carte
> 5. puis on va voir comment on ferait pour associer chaque adresse dans un quartier
> 6. et pour vérifier qu'on ne s'est pas trompé, on affichera les quartiers et les adresses avec des couleurs
> 7. enfin on verra comment passer à l'échelle pour visualiser plusieurs dizaines de milliers de points

> **les notions au coeur de ce TP**
>
> - faire des requêtes HTTP en Python (`requests`), en GET et en POST
> - manipuler des données géographiques avec `geopandas` (une `GeoDataFrame` a une colonne "active" qui porte la géométrie)
> - la **jointure spatiale** : comme un `merge`, mais le critère d'appariement est un prédicat géographique (`contains`) plutôt qu'une égalité
> - afficher tout ça avec `folium`

le résultat final sera de produire une carte comme ceci

[Voir la carte finale](media/addresses-final.html)

> **librairies utilisées**
>
> on va utiliser principalement
>
> * `geopandas`: comme son nom l'indique, [`geopandas`](https://geopandas.org/en/stable/) est une librairie pour faire .. un peu de géo
>
>   pour faire court, elle permet de manipuler un dataframe avec des coordonnées géographiques
>
> * `folium` pour l'affichage de cartes
>
> * et aussi au début un peu de `requests` pour envoyer des requêtes à l'API de géolocalisation

## échauffement : construire une URL avec des paramètres

avant d'attaquer l'API, un petit rappel sur la syntaxe d'une URL avec paramètres :

```
http://le.host.name/le/path?param1=truc&param2=bidule&param3=machinechose
```

**Énoncé :**

Écrivez une f-string qui construit l'URL `https://api-adresse.data.gouv.fr/search/?q=3+rue+tourelles,Paris&limit=1` à partir de variables `num=3`, `typ="rue"`, `nom="tourelles"`.

---

## échauffement : sélectionner des lignes avec `iloc` et `loc`

```python
import pandas as pd

df = pd.DataFrame({
    "number": [3, 18, 7, 12, 5],
    "name": ["tourelles", "BERNARDINS", "Bouvart", "Rivoli", "Seine"],
})
sub = df[df.number > 10]
```

**Énoncé :**

1. Que contient `sub` ? Quel est son index ?
2. Prédire puis vérifier le résultat de `df.iloc[:2]` et de `df.loc[:2]`. Quelle différence ?
3. Même question avec `sub.iloc[:2]` et `sub.loc[:2]`.
4. Que donnent `sub.iloc[0]` et `sub.loc[0]` ?
5. Comment obtenir la colonne `name` de la première ligne de `sub` avec `iloc` ? avec `loc` ?

<details>
<summary><b>Indice</b></summary>

`iloc` raisonne en **positions** (0, 1, 2...), `loc` en **étiquettes** d'index  
après un filtrage, l'index n'est plus `0, 1, 2...` : il garde les étiquettes d'origine
</details>

## géolocalisation

pour obtenir des coordonnées latitude/longitude à partir d'une adresse, en France on peut utiliser un service public et gratuit ici  
**<https://adresse.data.gouv.fr/api-doc/adresse>**  
lisez bien cette page, et notamment tout en bas il y a une zone où vous pouvez faire une recherche en ligne

```python
# for starters, we only need the 'regular fit' pandas

import pandas as pd
```

vous pouvez charger le fichier `data/addresses.csv`; toutes ces adresses sont situées à PARIS

```python
# load the data in data/addresses.csv
# and display a few first lines

addresses = pd.read_csv('data/addresses.csv')
addresses.head(4)
```

et la première chose qu'on va faire, c'est naturellement d'utiliser cette API pour géolocaliser ces adresses

mais avant cela, je vous recommande de produire un fichier `addresses-small.csv` qui contient un petit extrait, disons les 10 ou 20 premières lignes; ce sera très utile pour débugger

```python
# produce a small extract into data/addresses-small.csv

# your code here
```

<details>
<summary><b>Indice</b></summary>

`.iloc[:20]` pour prendre les 20 premières lignes, puis `.to_csv(...)` pour sauver  
pensez à `index=False` sinon pandas rajoute une colonne d'index inutile dans le fichier
</details>

### une par une

c'est très pratique de pouvoir faire une recherche des adresses 'une par une'; voici comment ça on s'y prendrait:

- après avoir bien lu la doc de l'API, commencez par taper dans un navigateur  
  `https://api-adresse.data.gouv.fr/search/?q=3+rue+tourelles,Paris`
- observez le format de ce que vous recevez, qu'est-ce que ça peut être ?
- ajoutez `&limit=1` à l'url pour que cela donne  
  `https://api-adresse.data.gouv.fr/search/?q=3+rue+tourelles,Paris&limit=1`
  quelle est la différence ?

> **? et & dans l'URL**
>
> ici lorsqu'on envoie donc une requête vers l'URL  
> `https://api-adresse.data.gouv.fr/search/?q=18+rue+BERNARDINS,Paris&limit=1`  
> les caractères `?` et `&` **jouent un rôle particulier**
>
> pour information, la syntaxe générale c'est
>
> ```
> http://le.host.name/le/path?param1=truc&param2=bidule&param3=machinechose
> ```
>
> et donc de cette façon, c'est un peu comme si on appelait une fonction à distance, en lui passant
>
> - `q=18+rue+BERNARDINS,Paris` (`q` pour *query*)
> - et `limit=1` (pour avoir seulement la première réponse)

<details>
<summary><b>pour vous faire réfléchir</b></summary>

il se passerait quoi si par exemple dans la colonne `name` il y avait un caractère `&` (imaginez la rue *Bouvart & Ratinet*)
</details>

fort de toutes ces observations, allez chercher sur Internet la doc d'une librairie Python qui s'appelle `requests`  
et notamment vous devriez tomber sur une page qui commence avec cet exemple

```
>>> r = requests.get('https://api.github.com/user', auth=('user', 'pass'))
>>> r.status_code
200
>>> r.headers['content-type']
'application/json; charset=utf8'
>>> r.encoding
'utf-8'
>>> r.text
'{"type":"User"...'
>>> r.json()
{'private_gists': 419, 'total_private_repos': 77, ...}
```

qui devrait vous inspirer pour l'écriture de la fonction `localize_one(num, typ, nom)`, qui construit l'URL, envoie la requête, et retourne le JSON décodé (ou `None` si le code HTTP n'est pas dans `[200, 300[`).

```python
# requests is the swiss knife for doing http

import requests
```

```python
# votre code

def localize_one(num, typ, nom):
    pass
```

<details>
<summary><b>Indice</b></summary>

réutilisez la f-string de l'échauffement pour construire `url`, appelez `requests.get(url)`, vérifiez `response.status_code`, puis retournez `response.json()`
</details>

```python
# and here is how we could use it

localize_one(18, 'rue', 'BERNARDINS')
```

**⚠️ MAIS** on ne va pas faire comme ça... pourquoi d'après vous ?

<details>
<summary><b>Indice</b></summary>

* mesurez combien de temps ça a pris de résoudre cette adresse
    * pour cela vous pouvez utiliser la *magic* `%%timeit`
    * ou encore le module `time`, sachant que `time.time()` compte des secondes
* combien de temps il faudrait pour faire ça 30.000 fois ?
* pourquoi est-ce que c'est aussi lent de faire comme ça ?
</details>

```python
# try to estimate how long it would take 
# to resolve 20_000 addresses this way

# your code here
```

<details>
<summary><b>Indice</b></summary>

chronométrez un seul appel avec `time.time()` avant/après, puis extrapolez en multipliant par 20000
</details>

### en un seul coup

si vous avez bien lu la page qui décrit l'API, vous devez avoir remarqué qu'il y a une autre façon de lui soumettre une recherche  
c'est ce qui est indiqué ici (**cherchez `search/csv` dans la page de l'API**)

```
curl -X POST -F data=@path/to/file.csv -F columns=voie -F columns=ville https://api-adresse.data.gouv.fr/search/csv/
```

<details open>
<summary><b>mais comment ça se lit ce bidule ?</b></summary>

* `curl` est un programme qu'on peut utiliser directement dans le terminal pour faire des requêtes http
* dans son utilisation la plus simple, il permet par exemple d'aller chercher une page web: vous copiez l'URL depuis le navigateur, et vous la donnez à `curl`, qui peut ranger la page dans un fichier

  ```bash
  curl -o lapageweb.html http://github.com/flotpython/
  ```

* quand on utilise une API, comme on vient de le faire pour aller chercher la position de la rue des bernardins, on doit **passer des paramètres** à l'API  
et pour faire ça dans une requête http, il y a **deux mécanismes: GET et POST**

  <details>
  <summary><b>GET</b></summary>

  **`GET`**: c'est le comportement par défaut de `curl`  
  dans ce mode de fonctionnement les paramètres sont passés **directement dans l'URL** comme on l'a fait tout à l'heure quand on avait vu ceci

  ```
  localize_one is fetching page
  https://api-adresse.data.gouv.fr/search/?q=18+rue+BERNARDINS,Paris&limit=1
  ```
  </details>

  <details>
  <summary><b>POST</b></summary>

  **`POST`**: dans ce mode-là, on ne **passe plus** les paramètres dans l'URL, mais **dans les headers HTTP**  
  bon je sais ça ne vous parle pas forcément, et ce n'est pas hyper important de savoir exactement ce que ça signifie, mais le point important c'est qu'**on ne va plus passer les paramètres de la même façon**
  </details>

et donc pour revenir à notre phrase:

```
curl -X POST -F data=@path/to/file.csv -F columns=voie -F columns=ville https://api-adresse.data.gouv.fr/search/csv/
```

ce qu'il se passe ici, c'est qu'on utilise `curl` pour envoyer une requête `POST` avec des paramètres `data` et `columns`  
le bon sens nous dit que

* `data` désigne un fichier csv qui contient les données à géolocaliser, une par ligne
* `columns` désigne les noms des colonnes qui contiennent (les morceaux de) l'adresse

</details>

#### en Python

sauf que nous, on ne veut pas utiliser `curl`, on veut faire cette requête en Python; voici comment on va s'y prendre

1. en une seule requête à l'API, on va envoyer **tout le fichier** csv, en lui indiquant juste quelles sont les colonnes qui contiennent les morceaux de l'adresse
2. le résultat - toujours au format csv - pourra être également transformé en dataframe
3. qu'il ne restera plus qu'à `merge` (ou `join` si vous préférez) avec la dataframe de départ, pour ajouter les résultats de la géolocalisation dans les données de départ  
   pour cette étape on peut envisager de ne garder que certaines colonnes de la géolocalisation (assez bavarde par ailleurs), je vous recommande de conserver uniquement:

   * `latitude`, `longitude` - *of course*
   * `result_city` pour pouvoir vérifier la validité des résultats - ici on devrait toujours avoir `Paris`
   * `result_type` qui devrait toujours renvoyer `housenumber`, ça permet à nouveau de vérifier qu'on a bien une adresse connue

> **💡 tip**
>
> * pour envoyer un POST avec des paramètres, on peut faire
>   ```python
>   response = requests.post(url, file=some_dict, data=another_dict)
>   ```
>
> * et donc dans notre cas, puisque `data` est un paramètre de type fichier, alors que `columns` est un paramètre usuel, on fera
>   ```python
>   response = requests.post(url, files={'data': file_like}, data={'columns': ['col1', ...]})
>   ```
>   dans lequel `file_like` désigne le résultat d'un `open()`
>
> * enfin, `pd.read_csv` s'attend à un paramètre de type fichier, i.e. du genre de ce qu'on obtient avec `open()`  
>   et du coup pour reconstruire une dataframe à partir du texte obtenu dans la requête http, on a deux choix
>   1. soit on commence par sauver le texte dans un fichier temporaire (juste faire attention à choisir un nom de fichier qui n'existe pas, de préférence dans un dossier temporaire, voir le module `tempfile`)
>   1. soit on triche un peu, et grâce à `io.StringIO` on peut transformer une chaine en fichier !
>   
>   c'est ce qu'on va faire dans notre solution, mais la première option est tout à fait raisonnable aussi

je vous donne ma solution pour `mass_post`; essayez par contre d'écrire vous-même `localize_many`, qui doit enchainer : lire le fichier d'origine, appeler `mass_post`, ne garder que les colonnes utiles du résultat, puis fusionner avec la dataframe de départ

```python
# requests is the swiss knife for doing http

import requests
# this one is to fake a file from a string
import io

def mass_post(filename, col_number, col_type, col_name, col_city):

    # we don't care about the index for now
    # but note that if we want to merge the result
    # with the initial data, there is a need 
    # for a unique index...

    with open(filename) as feed:
        response = requests.post(
            "https://api-adresse.data.gouv.fr/search/csv/", 
            files={'data': feed},
            data={'columns': [col_number, col_type, col_name, col_city]})
    if not (200 <= response.status_code < 300):
        print(f"OOPS, got {response.status_code}")
        return None

    # this time the output is csv, not json
    # note that read_csv requires a file-like object
    # and we have a str, so io.StringIO comes to the rescue
    result = pd.read_csv(io.StringIO(response.text))

    return result
```

```python
# votre code

def localize_many(filename, col_number, col_type, col_name, col_city):
    pass
```

<details>
<summary><b>Indice 1</b></summary>

`mass_post` vous donne le résultat de la géoloc, mais sans les colonnes d'origine ; il vous faut aussi relire le fichier d'origine avec `pd.read_csv`
</details>

<details>
<summary><b>Indice 2 : la fusion</b></summary>

l'API retourne les lignes dans le **même ordre** que le fichier d'entrée, et ni l'une ni l'autre des deux dataframes n'a d'index explicite (donc un `RangeIndex` 0, 1, 2...) : on peut donc fusionner directement sur cet index avec `left_index=True, right_index=True`
</details>

```python
# try your code on the small sample for starters

addresses_small = localize_many("data/addresses-small.csv", "number", "type", "name", "city")
addresses_small
```

```python
# sanity check : look at the entries that have
# result_city != 'Paris'
# (or result_type != 'housenumber')
# and decide what to do with them

# your code
```

<details>
<summary><b>Indice</b></summary>

l'API ne trouve pas toujours la bonne adresse : elle peut renvoyer une rue homonyme dans une autre ville (par exemple `La Roche-sur-Yon`), ou un résultat moins précis qu'un numéro (`street`, `locality`)  
un `assert` qui plante ne vous dit pas *quelles* lignes posent problème ; commencez plutôt par les afficher  
attention aux valeurs manquantes (`NaN`) dans `result_city` : pensez à `fillna("")` avant de comparer
</details>

```python
# when you think you're ready, go full scale
# be ready to wait for at least 40-60s 

# optional: try to record the time it takes !
# tip: see for example time.time()

# addresses = localize_many("data/addresses.csv", "number", "type", "name", "city")
```

<details>
<summary><b>Indice 1 : traitement par lots (optionnel)</b></summary>

envoyer 20 000 lignes d'un seul coup peut dépasser les limites de l'API (taille du fichier, timeout)  
une approche plus robuste : découper `addresses.csv` en lots de 1000 lignes (`iloc[i:i+batch_size]`), sauver chaque lot dans un fichier temporaire, appeler `localize_many` dessus, puis concaténer tous les résultats avec `pd.concat(..., ignore_index=True)`
</details>

<details>
<summary><b>Indice 2 : le cache</b></summary>

cette étape prend du temps : si le fichier `data/addresses-geoloc.csv` existe déjà, inutile de la refaire  
voir `os.path.exists`, et `os.remove` pour supprimer les fichiers temporaires des lots
</details>

```python
# sanity check

# len(addresses)
```

```python
# at this point you should store the data
# it's just good practice, as you've done one important step of the process
# plus it takes quite some time to redo

# so: store geolocalized addresses in data/addresses-geoloc.csv

# your code
```

## afficher sur une carte

à présent qu'on a une position, on va pouvoir afficher ces adresses sur une carte

et pour ça il y a plein de libs disponibles, on va choisir `folium`

si nécessaire, il faut l'installer (comment on fait déjà ?)

```python
import folium
```

pour commencer, allez chercher la documentation; le plus simple c'est de demander à google 
[`folium python`](https://www.google.com/search?q=folium+python) ?

et surtout (regardez les deux premiers pour l'instant):

* l'objet `Map` <https://python-visualization.github.io/folium/quickstart.html#Getting-Started>
* l'objet `Marker` <https://python-visualization.github.io/folium/quickstart.html#Markers>
* et un peu plus tard on utilisera aussi des overlays <https://python-visualization.github.io/folium/quickstart.html#GeoJSON/TopoJSON-Overlays>

```python
# ~ chatelet

CENTER = 48.856542, 2.347614
```

### le fond de carte

```python
# pour commencer on va recharger la dataframe précédente
import pandas as pd

addresses = pd.read_csv("data/addresses-geoloc.csv")

# et en faire un petit échantillon

addresses_small = addresses.iloc[:20]
```

```python
# créez une map centrée sur ce point et de zoom 13
# n'oubliez pas de l'afficher,
# et vérifiez que vous voyez bien Paris, 
# que vous pouvez zoomer et vous déplacer, ...

# votre code
def paris_map():
    ...

# pour l'afficher
paris_map()
```

<details>
<summary><b>Indice</b></summary>

regardez l'exemple du *Getting Started* de folium : `folium.Map(location=..., zoom_start=...)`  
`location` attend un couple (latitude, longitude), exactement la forme de `CENTER`

si le fond de carte par défaut (OpenStreetMap) ne s'affiche pas chez vous, passez `tiles=None` et ajoutez vous-même un `folium.raster_layers.TileLayer` avec un autre fournisseur de tuiles
</details>

### on ajoute les adresses

pareil mais vous ajoutez les adresses qui se trouvent dans la dataframe  
éventuellement, vous pouvez comme sur l'exemple du *Getting Started* ajouter un tooltip avec l'adresse complète

```python
# your code here

def map_addresses(geoloc):
    """
    creates folium map centered on Paris, with the various addresses
    contained in the input dataframe shown as a marker
    """
    pass
```

<details>
<summary><b>Indice</b></summary>

partez de `paris_map()`, puis itérez sur les lignes de `geoloc` (`for _, row in geoloc.iterrows()`), et pour chacune ajoutez un `folium.Marker(location=(row['latitude'], row['longitude']), tooltip=...).add_to(map)`
</details>

```python
# and try it out

# make sure you use A SMALL DATAFRAME because with this method
# trying to display tens of thousands addresses
# is going to be SUPER SLOOOOOW !

map_addresses(addresses_small)
```

### on sauve la carte

une fonction très sympatique de `folium`, c'est qu'on peut sauver cette carte sous la forme d'un fichier html, on dit *standalone*, c'est-à-dire autosuffisant, pas besoin de Python ni de Jupyter pour la regarder

```python
map_small = map_addresses(addresses_small)

# je commente car à ce stade dans le notebook étudiant
# on a map_small is None et donc si on essaie de faire cela
# le notebook plante
# map_small.save("addresses-small.html")
```

et maintenant pour voir le résultat, ouvrez le fichier dans un autre tab du navigateur

## les quartiers de Paris

maintenant on va ranger les adresses par quartier; pour cela nous avons dans le dossier `data/` le fichier `quartier_paris.zip` qui contiennent la définition des 80 quartiers qui recouvrent Paris intra-muros

<details>
<summary><b>source</b></summary>

obtenu ici <https://opendata.paris.fr/explore/dataset/quartier_paris/export/?disjunctive.c_ar>  
en choisissant le format "Shapefile"
</details>

ça se présente comme ceci:

```python
# à ce stade on a besoin de geopandas
# installez-le si besoin

import geopandas as gpd
```

```python
# ça se lit facilement
quartiers = gpd.read_file("data/quartier_paris.zip", encoding="utf8")

# et le résultat est .. une dataframe
type(quartiers)
```

```python
# qu'on peut afficher normalement

quartiers
```

```python
# et on peut même l'afficher sommairement avec matplotlib

# bon c'est loin d'être parfait mais ça nous suffit à ce stade
# on va voir plus loin comment incorporer ça dans la carte

quartiers.plot();
```

pour afficher cela sur la carte, il faut se livrer à une petite gymnastique

1. on construit la map comme on l'a vu jusqu'ici
2. on passe à `folium.GeoJson()` la geo-dataframe, pour créer un objet
3. qu'on ajoute dans la map

ce qui nous donne ceci:

```python
def paris_map():
    """
    create a map of Paris with its 80 quartiers
    """
    map = folium.Map(
        location=CENTER,
        zoom_start=13,
        # width='80%',
    )

    # create a folium JSON object from the geo df
    folium.GeoJson(
        data=quartiers,
        # optionnally we could also tweak things on the way
        # try to uncomment these
        # style_function=lambda x: {"fillColor": "#45a012", "color": "#881212"}
        # the name is in the l_qu column
        # tooltip=folium.GeoJsonTooltip(fields=['l_qu'], aliases=['quartier']),
    # and add it to the map
    ).add_to(map)

    # and that's it
    return map
```

```python
paris_map()
```

### un peu de couleurs

maintenant c'est à vous: il s'agit d'améliorer cela pour ajouter des couleurs aux quartiers:

1. écrivez une fonction `random_color` qui renvoie une couleur au hasard, i.e. une chaine comme `#12f285`
2. ajoutez dans la dataframe `quartiers` une colonne contenant une couleur aléatoire
3. écrivez `map_addresses_in_quartiers_2`, une variante qui affiche les quartiers avec chacun une couleur

### échauffement : le formatage hexadécimal

une couleur comme `#12f285` est constituée de 3 octets (rouge, vert, bleu) écrits en hexadécimal, 2 chiffres chacun

```python
x = 10
f"{x:02x}"
```

**Énoncé :**

1. Que donne `f"{x:02x}"` pour `x = 255` ? pour `x = 3` ?
2. À quoi sert le `0` dans `02x`, et à quoi sert le `2` ?
3. En déduire comment transformer 3 entiers entre 0 et 255 en une chaine `"#rrggbb"`

<details>
<summary><b>Indice</b></summary>

`x` est converti en hexadécimal (`x`), sur 2 caractères minimum (`2`), en complétant avec des zéros à gauche si besoin (`0`)  
sans ce zéro de remplissage, `3` donnerait `"3"` au lieu de `"03"`, ce qui casserait la longueur fixe attendue pour une couleur
</details>

### couleurs: étape 1

```python
# your code here

def random_color():
    """
    returns a random color as a string like e.g. #12f285
    that is to say containing 3 bytes in hexadecimal
    """
    # of course this is not the right answer
    return "#12f285"
```

<details>
<summary><b>Indice</b></summary>

tirez 3 entiers au hasard entre 0 et 255 (`random.randint`), puis réutilisez le formatage de l'échauffement
</details>

```python
random_color(), random_color()
```

### couleurs: étape 2

```python
# add a random color column in quartiers

# your code here
```

<details>
<summary><b>Indice</b></summary>

il faut une couleur **par ligne** de `quartiers`, donc une liste de `len(quartiers)` couleurs, assignée à une nouvelle colonne
</details>

```python
quartiers.head(3)
```

### couleurs: étape 3

> **💡 tip**
>
> ça m'a pris un peu de temps à trouver comment faire, voici un indice
>
> <https://stackoverflow.com/questions/72983410/folium-geojson-how-to-apply-the-stylefunction-within-a-loop>

```python
# display the quartiers with their individual color

# your code

def paris_map():
    """
    create a map of Paris with its 80 quartiers
    each quartier is shown in its individual color
    """
    pass
```

<details>
<summary><b>Indice</b></summary>

`folium.GeoJson` accepte un paramètre `style_function` : une fonction qui reçoit **un feature** (une ligne, au format GeoJSON) et retourne un dictionnaire de style  
la couleur de ce feature est accessible via `feature['properties']['color']`
</details>

```python
# display it
paris_map()
```

## spatial join

ce qu'on aimerait bien faire à présent, c'est de trouver le quartier de chaque adresse  
c'est-à-dire en pratique de rajouter dans la dataframe des adresses une ou des colonnes indiquant le quartier

et pour faire cela il existe un **outil très intéressant** dans geopandas [qui s'appelle le *spatial join* et qui est décrit ici](https://geopandas.org/en/stable/gallery/spatial_joins.html#Spatial-Joins)

en deux mots, l'idée c'est de faire comme un JOIN usuel (ou un `pd.merge()` si vous préférez)  
mais pour décider si on doit aligner deux lignes (une dans la df de gauche et l'autre dans la df de droite):

- au lieu de vérifier **l'égalité** entre deux colonnes
- on va cette fois utiliser un **prédicat** entre deux colonnes géographiques, comme par exemple ici le prédicat **`within`** (le point est à l'intérieur du polygone), ou son inverse **`contains`**

ce qui signifie, en pratique, qu'on va faire ceci

1. transformer la dataframe d'adresses en une `GeoDataFrame` et remplacer les colonnes `latitude` et `longitude` par une colonne `position`, cette fois dans un format connu de `geopandas`
2. et ainsi on pourra appliquer un *spatial join* entre la (géo)dataframe d'adresses et la (géo)dataframe des quartiers, en choisissant ce prédicat **`within`**
3. et du coup modifier la carte pour afficher les adresses dans la bonne couleur - celle du quartier - pour vérifier qu'on a bien fait correctement le classement en quartiers

on doit donc obtenir autant de ligne que d'adresses, mais avec une ou des colonnes en plus (couleur, nom du quartier, ...) qui caractérisent le quartier dans lequel se trouve l'adresse

> **merge classique vs spatial join**
>
> dans un `pd.merge()` classique, on aligne deux lignes quand une colonne de gauche **est égale** à une colonne de droite (une clé)  
> dans un *spatial join*, on aligne deux lignes quand un prédicat géométrique est vérifié entre leurs géométries respectives : ici, "le point de l'adresse est **à l'intérieur** (`within`) du polygone du quartier"  
> c'est le même principe de jointure, seul le critère d'appariement change

### spatial join: étape 1

je vous donne le code; ce qu'il faut savoir notamment c'est qu'en `geopandas` il y a la notion de 'colonne active', celle qui contient les informations géographiques; ça n'est pas forcément hyper-intuitif la première fois...

<https://geopandas.org/en/stable/docs/reference/api/geopandas.GeoDataFrame.set_geometry.html>

```python
def convert_lat_lon(df):
    """
    the input df is expected to have 2 columns named 'latitude' and 'longitude'

    this function will create a GeoDataFrame based on that, where 'latitude' and 'longitude'
    are replaced by a new column named 'position' 
    also this new column becomes the active geometry for future operations 
    (in our case the spatial join)
    """

    # we need a geopandas-friendly df
    geo_df = gpd.GeoDataFrame(df)

    # beware, here LONGITUDE comes first !
    geo_df['position'] = gpd.points_from_xy(df.longitude, df.latitude)

    # it would make sense to clean up
    # BUT
    # our map-production code relies on these columns, so...
    # del geo_df['latitude']
    # del geo_df['longitude']

    # declare the new column as the active one
    geo_df.set_geometry('position', inplace=True)

    return geo_df
```

```python
# let's apply that to our small input

geoaddresses_small = convert_lat_lon(addresses_small)
```

```python
# we will also need to set the active column in the quartiers (geo)dataframe

quartiers.set_geometry('geometry', inplace=True)
```

### spatial join: étape 2

Il ne nous reste plus qu'à faire ce fameux *spatial join*, je vous laisse trouver le code pour faire ça

```python
# spatial join allows to extend the addresses dataframe
# with quartier / arrondissement information

# your code

def add_quartiers(gdf):
    """
    given an addresses geo-dataframe, 
    (i.e. typically produced as an output of convert_lat_lon)
    this function will use spatial join with the quartiers information
    and return a copy of gdf extended with columns such as
    l_qu : name of the quartier
    c_ar : arrondissement number
    color: the (random) color of that quartier
    ...

    """
    # this is not the right answer...
    return gdf
```

<details>
<summary><b>Indice 1</b></summary>

cherchez `geopandas sjoin` : `gdf.sjoin(autre, how=..., predicate=...)`  
`how='left'` garde toutes les lignes de la dataframe de gauche (donc toutes les adresses), même celles sans correspondance
</details>

<details>
<summary><b>Indice 2 : le prédicat</b></summary>

la dataframe de gauche contient des **points** (les adresses), celle de droite des **polygones** (les quartiers)  
le point doit être **à l'intérieur** du polygone : quel prédicat choisir ? (`within` ou `contains`, suivant le sens dans lequel vous écrivez la jointure)
</details>

<details>
<summary><b>Indice 3 : les colonnes</b></summary>

inutile de rapatrier toutes les colonnes de `quartiers` ; ne gardez que `l_qu`, `c_ar`, `color` et la géométrie
</details>

```python
# try your code

# xxx you can safely ignore this warning...
# UserWarning: CRS mismatch between the CRS of left geometries and the CRS of right geometries

geoaddresses_small_extended = add_quartiers(geoaddresses_small)
geoaddresses_small_extended.head(2)
```

```python
# verify your code

# make sure you have the right number of lines in the result

geoaddresses_small_extended.shape
```

### spatial join: étape 3

on va donc maintenant récrire `map_addresses`; la logique reste la même mais on veut afficher chaque adresse avec une couleur qui provient de son quartier

comme vous allez le voir, l'objet `folium.Marker` ne peut pas s'afficher avec une couleur arbitraire - il semble qu'il y ait [seulement une petite liste de couleurs supportées](https://stackoverflow.com/questions/75011160/how-do-assign-color-to-folium-map-based-on-another-column-value)

pour contourner ça, utilisez à la place un objet de type `folium.CircleMarker`

```python
# rewrite map_addresses so that each address is shown 
# in the color of its quartier

def map_addresses(gdf):
    """
    (slightly) rewrite the first version of this function

    your input is now a geopandas dataframe, with the information
    about the 'quartier'

    and your job is to display all the addresses on the map, now with
    the color of the 'quartier'

    return a folium map of paris with the adresses displayed
    """
    return
```

<details>
<summary><b>Indice</b></summary>

même squelette que la première version de `map_addresses`, mais `folium.CircleMarker(location=..., radius=..., color=row['color'], fill=True, fill_color=row['color'])`
</details>

```python
# test the new function

map = map_addresses(geoaddresses_small_extended)

map
```

### on sauve la carte

c'est sans doute un bon moment pour sauver tout ce qu'on a fait:

- sauvez la geo-dataframe dans un fichier `addresses-small-extended.csv`
- sauvez la carte au format html (même nom sinon) pour une utilisation en *standalone* (par exemple pour la publier sur un site web indépendant des notebooks et de jupyter et tout ça)

```python
# votre code
```

## on clusterise

pour pouvoir passer à l'échelle, il est indispensable de *clusteriser*; c'est-à-dire de grouper les points en fonction du niveau de zoom; voyons ce que ça donne

ne perdez pas de temps à chercher le code vous-même, c'est un peu *hacky* comme on dit, voici comment il faut faire

> **pourquoi une fonction JavaScript ?**
>
> avec des dizaines de milliers de `Marker` Python, la carte devient très lente : chaque marker est un objet géré côté Python puis sérialisé en HTML  
> `FastMarkerCluster` fait l'inverse : on lui donne juste les données brutes, et c'est le **navigateur** (en JavaScript) qui se charge de créer les cercles à la volée, uniquement pour ceux qui sont visibles ; d'où le `callback`, une fonction JS qu'on embarque telle quelle dans une chaine Python

```python
from folium import plugins

def map_addresses(gdf):

    # start like before
    if 'human' not in gdf.columns:
        # create a column with a human-readable address
        gdf['human'] = gdf['number'].astype(str) + ', ' + gdf['type'] + ' ' + gdf['name']

    map = paris_map()


    # we need a JavaScript function...
    # this is required to get  good performances
    callback = """
function (row) {
  // if you need to debug it
  // console.log(row)
  let circle = L.circleMarker(
      // the position
      new L.LatLng(row[0], row[1]),
      // styling
     {color: row[3], radius: 8},
    )
  // add the tooltip
  circle.bindTooltip(row[2]).openTooltip()
  return circle
}
"""

    # compute an extract with fewer columns
    # not strictly necessary but easier to deal with column names
    extract = gdf[['latitude', 'longitude', 'human', 'color']]
    cluster = plugins.FastMarkerCluster(
        extract, callback=callback,
        # nicer behaviour
        disableClusteringAtZoom=18,
    ).add_to(map)

    return map
```

### sur l'échantillon

```python
# let's first test it on the small extract

map_addresses(geoaddresses_small_extended)
```

### sur le dataset entier

```python
# and if all goes well we can try and display the full monty

# first prepare the full dataset

geoaddresses = add_quartiers(convert_lat_lon(addresses))
final_map = map_addresses(geoaddresses)

final_map
```

```python
# makes sense to save the hard work

# geoaddresses.to_csv("addresses-final.csv")
# final_map.save("addresses-final.html")
```


## Extrat credit : quel quartier a le plus d'adresses ?
 
On dispose de `geoaddresses`, où chaque adresse connait son quartier (colonne `l_qu`)  
l'idée : produire une **carte choroplèthe**, c'est-à-dire où chaque quartier est coloré selon le **nombre d'adresses** qu'il contient (plus c'est foncé, plus il y en a)
 
**Énoncé :**
1. Calculez le nombre d'adresses par quartier, sous la forme d'une dataframe à deux colonnes : le nom du quartier et le nombre d'adresses.
2. Affichez les 5 quartiers qui en ont le plus.
3. Produisez la carte choroplèthe correspondante, et sauvez-la dans `addresses-choropleth.html`.

<details>
<summary><b>Indice 1 : compter</b></summary>
c'est un `groupby` sur la colonne `l_qu`, suivi d'un comptage du nombre de lignes de chaque groupe (`size()`)  
pensez à `reset_index()` pour retrouver une vraie dataframe à deux colonnes
</details>
<details>
<summary><b>Indice 2 : les 5 premiers</b></summary>
`sort_values(..., ascending=False)` puis `head(5)`
</details>
<details>
<summary><b>Indice 3 : la carte</b></summary>
cherchez `folium.Choropleth` ; les paramètres importants sont
- `geo_data` : la géométrie des quartiers (votre `GeoDataFrame` `quartiers`)
- `data` et `columns` : la dataframe des comptages, et les deux colonnes à utiliser (clé, valeur)
- `key_on` : comment retrouver la clé dans chaque feature du GeoJSON, sous la forme `feature.properties.<colonne>`
- `fill_color` : le nom d'une palette, par exemple `'YlOrRd'`
</details>

## références

le jeu de données utilisé ici provient à l'origine de  
<https://nakala.fr/10.34847/nkl.3038f62v#e6f991e37ab1fadb388262b57472a703ba379d48>, légèrement retravaillé pour les besoins du TP