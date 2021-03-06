# Utilisation du cache et des artefacts

Dans cet exercice nous allons partager des éléments entre les jobs à l'aide des notions de cache et d'artefacts. 

## 1. Le cache

La commande `cache` est utilisée pour spécifier une liste de fichiers et de répertoires à mettre en cache entre les `jobs`. 
Seuls les chemins situés dans l'espace de travail du projet sont accessibles.

Si le `cache` est défini en dehors des `jobs`, cela signifie qu'il est global et que tous les `jobs` utiliseront cette définition.

* Créer un pipeline constitué de deux jobs :
    * Sans définir le cache pour le moment.
    * Définir un job `build` et scripter la création d'un fichier `build-info.txt` dans un répertoire `build`.
    * Définir un job `test` et vérifier l'existence de ce fichier (`if [ ! -f build/build-info.txt ]; then exit 1; fi`).
* Le fichier produit par le job `build` est-il disponible dans le job `test` ?
* Déclarer un cache :
    * Path `./build/`.
    * Donner un identifiant (`key`) au cache afin de l'isoler entre deux lancements du pipeline.
* Vérifier que le pipeline passe au vert.

<details>
<summary>Solution</summary>
<p>

```yaml
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - ./build/

build:
  stage: build
  before_script:
    - rm -rf ./build
    - mkdir ./build
  script:
    - echo "test" > build/build-info.txt

test:
  stage: test
  script:
    - if [ ! -f build/build-info.txt ]; then exit 1; fi
```

</p>
</details>

## 2. Les artefacts

`artefacts` est la commande utilisée pour spécifier une liste de fichiers et de répertoires qui seront attachés au `job` en cas de succès.
Une fois le `job` terminé, les `artefacts` seront envoyés à GitLab Server et pourront être téléchargés depuis l'interface web.

* Ajouter un job `dist` (stage `deploy`) construisant un `tar.gz` avec le contenu de répertoire `build`
> `tar zcvf dist/artifact.tar.gz build/`
* Déclarer le `tar.gz` comme artefact du build. 
* Cet artefact doit :
    * Etre disponible dans le répertoire `dist`.
    * Avoir une durée de vie de `5 minutes`.
    * Etre téléchargable dans un zip nommé `<projet>-<branche>-<sha1 du commit>` (cf. les [variables d'environnement d'un pipeline](https://docs.gitlab.com/ce/ci/variables/predefined_variables.html))

<details>
<summary>Solution</summary>
<p>

```yaml
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - ./build

stages:
  - build
  - test
  - deploy

build:
  stage: build
  before_script:
    - rm -rf ./build
    - mkdir ./build
  script:
    - echo "test" > ./build/build-info.txt

test:
  stage: test
  script:
    - if [ ! -f ./build/build-info.txt ]; then exit 1; fi

dist:
  stage: deploy
  before_script:
    - rm -rf ./dist
    - mkdir ./dist
  script:
    - tar zcvf ./dist/artifact.tar.gz ./build 
  artifacts:
    name: "$CI_PROJECT_NAME-$CI_COMMIT_REF_NAME-$CI_COMMIT_SHORT_SHA"
    paths:
      - dist/
    expire_in: 5 mins
```

<p>
<img src="artefact.png" height="200">
</p> 

</p>
</details>

## 3. Lier les jobs

En s'inspirant du pipeline de l'exercice [2.2](../exercice_2) créer un pipeline contenant :
* Deux jobs de build
    * Produisant chacun un fichier texte avec pour contenu le résultat de la commande `ruby -v`
    * Exécutés avec deux versions différentes de l'image Ruby 
* Deux jobs de test
    * Affichant le contenu du fichier texte construit pendant le job de build

Au final le graphe de dépendance entre les jobs doit être le suivant : 

>build:X -> test:X
>
>build:Y -> test:Y

<details>
<summary>Solution</summary>
<p>

```yaml
stages:
  - build
  - test

build:2.6:
  stage: build
  image: ruby:2.6-alpine
  script:
    - ruby -v > build_2.6.txt
  artifacts:
    paths:
      - build_2.6.txt
    expire_in: 1 min

build:2.5:
  stage: build
  image: ruby:2.5-alpine
  script:
    - ruby -v > build_2.5.txt
  artifacts:
      paths:
        - build_2.5.txt
      expire_in: 1 min

test:2.6:
  stage: test
  image: ruby:2.6-alpine
  script:
    - cat build_2.6.txt
  dependencies:
    - build:2.6

test:2.5:
  stage: test
  image: ruby:2.5-alpine
  script:
    - cat build_2.5.txt
  dependencies:
    - build:2.5
```

<p>
<img src="link_jobs.png" height="200">
</p> 

</p>
</details>

    
[< Previous](../exercice_2) | [Home](../README.md) | [Next >](../exercice_4)