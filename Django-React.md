

# Création d'une application Django-React avec Docker

 
### 1.  Créer une image et un projet Django
### 2. Créer une image et un projet React avec create-react-app
### 3. Implémenter les deux services à l'aide de docker-compose qui construira les deux images
### 4. Connecter le frontend à une api backend pour y chercher des données

## L'arborescence du projet

Avant la création de projets:

$ sudo mkdir project
$ sudo mkdir project/frontend
$ sudo mkdir project/backend

$ sudo touch project/backend/Dockerfile
$ sudo touch project/backend/requirements.txt
$ sudo touch project/frontend/Dockerfile

├── backend
│   ├── Dockerfile
│   └── requirements.txt
└── frontend
    └── Dockerfile
```
## 1. Création du projet Django et de son image 
- Editer requirements.txt et y ajouter la version de`django et psycopg2 pour qu'il s'installe dans l'image
  - Django: 2.2.6
  - psycopg2: 2.8.4

- La Dockerfile du backend: 
```
# Official python image (stable one)
FROM python:3.7.5

# Adding backend directory 
WORKDIR /app/backend

# Install Python dependencies
COPY requirements.txt /app/backend
RUN pip3 install --upgrade pip -r requirements.txt

# Add the rest of the code
COPY . /app/backend

# Make port 8000 available for the app
EXPOSE 8000

# Default command when image is run
CMD python3 manage.py runserver 0.0.0.0:8000
```

- On construit l'image en allant chercher la Dockerfile se trouvant dans le dossier backend. On lui donne le nom *backend* et le tag *latest* à l'aide du flag *-t* :

```
$ docker build -t backend:latest backend
```
- Ensuite on crée le projet Django start_student dont on lie les données en local au volume du container. On utilise pour se faire l'image qu'on vient de construire (*backend:latest*), ce qui crée un nouveau container:
 ```
$ docker run -v $PWD/backend:/app/backend backend:latest django-admin startproject hello_world . 
```
- pour finir, nous lançons un nouveau container qui va utiliser la commande par défaut de l'image *backend:latest* (c'est à dire le runserver de django). la commande va mapper les ports 8000 de l'hôte au port 8000 du container grâce au tag *-p*:

```
$ docker run -v $PWD/backend:/app/backend -p 8000:8000 backend:latest
```

- il suffit d'aller sur [localhost:8000](localhost:8000) pour voir l'app Django! 


## 2. Création du projet React et de son image

- Voici la `Dockerfile` du frontend:

```
# Official stable image
FROM node:12.13.0

WORKDIR /app/

# Install dependencies
# COPY package.json yarn.lock /app/

# RUN npm install

# Add rest of the client code
COPY . /app/

EXPOSE 3000

# CMD npm start
```

celle-ci se base sur une image officielle de node. Tout est en place dans node pour pouvoir créer notre application React grâce à l'outil *create-react-app*. 

- Construisons l'image:

```
$ docker build -t frontend:latest frontend
```

- Créons notre app react à l'aide de notre image:

```
$ docker run -v $PWD/frontend:/app frontend:latest npx create-react-app start_react
```

- nettoyons un peu les dossiers:

```
$ mv frontend/start_react/* frontend/hello-world/.gitignore frontend/ && rmdir frontend/start_react
```

- il suffit maintenant de lancer l'appli avec la commande `npm start` sur le port 3000:

```
$ docker run -v $PWD/frontend:/app -p 3000:3000 frontend:latest npm start
```

- Direction [localhost:3000](localhost:3000) ça marche!

## 3. Implémenter les services avec docker-compose

Avant de commencer, nous allons stopper les containers pour ne pas avoir de conflits de ports. 

```
$ docker stop $(docker ps -q)
```

Cette commande va stopper tous les containers actifs 

- créons un fichier *docker-compose.yml* à la racine du projet

```
version: "3.7"
services:
  backend:
    build: ./backend
    volumes:
      - ./backend:/app/backend
    ports:
      - "8000:8000"
    stdin_open: true
    tty: true
    command: python3 manage.py runserver 0.0.0.0:8000
  frontend:
    build: ./frontend
    volumes:
      - ./frontend:/app
      # One-way volume to use node_modules from inside image
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    depends_on:
      - backend
    command: npm start

```
Celui-ci implémente les différents services (dans ce cas-ci le *frontend* et *backend*) en construisant automatiquement les images (*build*), en créant les volumes, et en lancant des commandes. Il suffit maintenant de la commande suivante pour tout construire et lancer: 

```
$ docker-compose up
```

Cela fonctionne comme avant, sur les ports 3000 et 8000! 

## 4. Connecter le frontend à une api backend.

On va voir maintenant comment transmettre des données entre le back et frontend.

- Créons d'abord une application Django qui va compter le nombre de caractère dans un mot:

```
$ docker-compose run --rm backend python3 manage.py startapp char_count
```

- ensuite on va ajouter cette fonction dans `
backend/char_count/views.py` qui va renvoyer le nombre de caractères d'un mot en format json:

```
from django.http import JsonResponse


def char_count(request):
    text = request.GET.get("text", "")

    return JsonResponse({"count": len(text)})
```

- n'oublions pas d'ajouter l'application au `INSTALLED_APPS` de `settings.py`:

```
"char_count.apps.CharCountConfig"
```

et d'ajouter un URL au projet pour accéder à l'application dans `urls.py`: 

```
from django.contrib import admin
from django.urls import path
from char_count.views import char_count

urlpatterns = [
    path('admin/', admin.site.urls),
    path('char_count', char_count, name='char_count'),
]
```
 
- On doit maintenant redémarrer les process pour que les changements soient effectifs:

```
$ docker-compose stop
```

et 

```
$ docker-compose up
```

- allons voir le lien [localhost:8000/char_count?text=hello world]((localhost:8000/char_count?text=hello%20world)) . ça marche? 

- C'est maintenant qu'on va lier le frontend pour que l'utilisateur puisse y entrer un mot, qui sera envoyé au backend qui lui renverra le nombre de caractère. 
- Dans `settings.py`, on ajoute `backend` à `ALLOWED_HOSTS`
- dans `package.json`, on ajoute `"proxy": "http://backend:8000"`. 
- Voilà, les deux services sont configurés pour communiquer entre eux à travers le port 8000. 
- Il faudra maintenant retourner sur la `Dockerfile` du `frontend` pour décommenter le nécessaire pour que tout fonctionne
- Nous allons d'abord ajouter le packet `axios`qui nous permettra de faire communiquer les deux services en l'ajoutant au `package.json` avec cette commande:
```
$ docker-compose run --rm frontend npm add axios
```

- ensuite on compose-down et up en reconstruisant l'image pour que le package.json soit importé dans le container:

```
$ docker-compose down
$ docker-compose up --build
```

- si tout fonctionne sur les deux ports, copier ceci dans `App.js` du projet React: 

```
import React from 'react';
import axios from 'axios';
import './App.css';

function handleSubmit(event) {
  const text = document.querySelector('#char-input').value

  axios
    .get(`/char_count?text=${text}`).then(({data}) => {
      document.querySelector('#char-count').textContent = `${data.count} characters!`
    })
    .catch(err => console.log(err))
}

function App() {
  return (
    <div className="App">
      <div>
        <label htmlFor='char-input'>How many characters does</label>
        <input id='char-input' type='text' />
        <button onClick={handleSubmit}>have?</button>
      </div>

      <div>
        <h3 id='char-count'></h3>
      </div>
    </div>
  );
}

export default App;
```


- rdv sur [localhost:3000](localhost:3000) :)
