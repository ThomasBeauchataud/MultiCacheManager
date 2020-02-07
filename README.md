# MultiCacheManager

*Ce document décrit le fonctionnement du projet MultiCacheManager ainsi que des indications d'utilisations et ses mises à jour / évolutions*

*Created the 26/07/2019 by Thomas Beauchataud*

*Last update the 26/07/2019 by Thomas Beauchataud*

*Note:*
- *26/07/2019 - Tout est bon* 

## Documentation

- [How it works](#how-it-works)
    - [The CacheManagerInterface](#the-cachemanagerinterface)
    - [The CacheMetaManagerInterface](#the-cachemetamanagerinterface)
    - [The CacheValidationInterface](#the-cachevalidationinterface)
    - [Cache Listeners (AOP)](#cache-listeners-aop)
    - [The AbstractCacheManager class](#the-abstractcachemanager-class)
    - [CacheManager services](#cachemanager-services)
        - [FileCacheManager](#filecachemanager)
        - [RedisCacheManager](#rediscachemanager)
        - [SessionCacheManager](#sessioncachemanager)
        - [DatabaseCachegeManager](#databasecachemanager)
    - [Administration](#administration)
        - [The administration page](#administration-page)
        - [Parameters](#parameters)
- [Create a new CacheManager](#create-a-new-cachemanager)
    - [Create the service](#create-the-service)
    - [Plug it](#plug-it)
        
# How it works

> Tout les services et fonctionnalités évoqués sont stockés dans un petit composant (un dossier) lui-même contenu dans CommonBundle\Services\Cache. Toutes les déclarations de services sont faites dans le fichier Ressources\config\cache-manager.yml qui est importé par Ressources\config\services.yml
> A long terme, ces services peuvent être transférés en un package

## The CacheManagerInterface

Pour manager le cache, nous utilisons des services implémentant CacheManagerInterface et possédant les méthodes : 
- get(key) qui permet de récupérer la clé *key* en cache
- add(key, data, options) qui permet de sauvegarder le contenu *data* avec la clé *key* avec les métadonnées *options* en cache
- has(key) qui permet de vérifier si *key* existe en cache
- remove(key) pour supprimer *key* du cache
- keys() pour récupérer toutes les clés du cache
- meta(key = null) pour récupérer toutes les métadonnées du cache ou celles concernant la clé *key*

## CacheMetaManagerInterface

Pour manager les métadonnées du cache, nous utilisons des services implémentant CacheMetaManagerInterface et possédant les méthodes : 
- getMeta(key, metaKey) retourne le champ *metaKey* des métadonnées de *key*
- getAllMeta(key) retourne toutes les métadonnées de *key*
- addMeta(key, metaKey, meta) ajoute *meta* dans le champ *metaKey* des métadonnées de *key*
- removeMeta(key, metaKey) supprime le champ *metaKey* des métadonnées de *key*
- removeAllMeta(key) supprime toutes les métadonnées de *key*

> **Toutes les métadonnées sont stockées parallèlement aux données dans une clé différentes nommée *meta-key***

## The CacheValidationInterface
 
Pour valider ou invalider le cache, nous utilisons des services implémentant CacheValidationInterface et possédant les méthodes:
- isValid(metas, value = null, key = null) verifie si les métadonnées *metas* de la clé *key* avec la valeur *value* est valide ou non

## Cache Listeners (AOP)

Afin de pouvoir facilement gérer des données mis en cache en fonction du retour d’une méthode, un listener a été conçu à cet effet. Il est utilisable en utilisant l’annotation *Cacheable* dans la PHPDoc de la méthode. Cette annotation prend en paramètre :
- le nom de la clé du cache (obligatoire)
- ses métadonnées (facultatif)
- son *CacheManager* (s’il n’est pas défini ou si le *CacheManager* n’existe pas, il prendra celui par défaut, définie dans les paramètres)
- les id des paramètres impactable (-1 pour tous les impacter) (la nom de la clé sera prolongé d’un encodage propre au contenu des paramètres sélectionnés)

Dans l’exemple ci-dessous, nous stockons le retour de la méthode dans la clé *avenantList* et nous considérons tous les paramètres

```
#AvenantClient.php

/**
 * @param $filters
 * @return Avenant[]
 * @throws \Exception
 * @Cacheable(key="avenantList",options={"timeout"=3600},cacheManager="RedisCacheManager",paramsId={-1})
 */
public function getList($filters = array())
{
    $response = $this->middleClient->get(self::MO_ENDPOINT,array('filters' => $filters));

    if (!$response->isSuccess()) {
        RestResponse::ThrowIfError($response->error);
    }

    return $this->serializer->deserialize(json_encode($response->data),'array<Common\MiddleBundle\Model\Avenant>','json');
}
```

Lors que l’annotation est présente dans la PHPDoc de la méthode, celle-ci est donc interceptée par la méthode *intercept* du CacheableInterceptor. Celle-ci regarde si la clé entrée en paramètre dans l’annotation existe en cache. Si elle existe en cache et est valide, alors elle est retournée depuis le cache, sinon, le contenu est récupéré par la fonction initiale, puis stocké en cache avant d’être retourné.

```
#CacheableInterceptor.php

/**
 * Called when intercepting a method call.
 *
 * @param MethodInvocation $invocation
 * @return mixed            the return value for the method invocation
 * @throws Exception       may throw any exception
 */
public function intercept(MethodInvocation $invocation)
{
    $cacheable = $this->annotationReader->getMethodAnnotation($invocation->reflection, 'Common\CommonBundle\Service\Cache\Listener\Cacheable');
    $key = $cacheable->key;
    $options = $cacheable->options;

    if ($this->cacheManager->has($key)){
        return $this->cacheManager->get($key);
    }

    $result = $invocation->proceed();

    $this->cacheManager->add($key, $result, $options);

    return $result;
}
```

## The AbstractCacheManager class

Tous les CacheManagers sont des extensions de la classe AbstractCacheManager implémentant elle-même les interfaces CacheManagerInterface & CacheMetaManagerInterface et possédant toutes les méthodes définies par les interfaces. Il est cependant possible de les Override en les implémentant dans le CacheManager souhaité.

La classe AbstractCacheManager possède également un service CacheValidation implémentant l'interface CacheValidationInterface (dont le rôle est de gérer la validation du cache grâce à ses métadonnées), ce service peut être null et dans ce cas les métadonnées seront ignorées. 

Cette classe possède enfin des méthodes de stockages abstraites qui sont propres à chaque CacheManager. 
- getData(key) retourne les données de la clé *key*
- addData(key, data) ajoute les données data avec la clé *key*
- removeData(key) supprime les données de la clé *key*
- hasKey(key) vérifie si la clé *key* éxiste
- getKeys() retourne toutes les clés

## CacheManager services

### FileCacheManager

Ce service stock le cache dans des fichiers. Ces fichiers porte le nom de la clé est possède l’extension .txt, ils ont donc comme nom final key.txt. 

Le chemin du stockage est défini dans les paramètres et passé en paramètre lors de la construction du service.

```
#FileCacheManager.php

public function __construct($fileCacheDir, CacheValidationInterface $cacheValidation)
{
    $this->cacheDir = $fileCacheDir;
    $this->cacheValidator = $cacheValidation;
    if (!file_exists($this->cacheDir)){
        mkdir($this->cacheDir);
        if(!file_exists($this->cacheDir)){
            throw new Exception("Impossible to create the folder ".$this->cacheDir);
        }
    }
}
```

### RedisCacheManager

Ce service stock le cache dans un serveur Redis qui est configuré dans les paramètres et passé en paramètre lors de la construction du service.

```
#RedisStorageManager.php

public function __construct($host, $port, CacheValidationInterface $cacheValidation)
{
    $this->redis = new Redis();
    $this->cacheValidator = $cacheValidation;
    $this->redis->connect($host, $port);
}
```

###	SessionCacheManager

Ce service stock le cache dans la session en cour. Afin de différencier les clés ajoutés par notre cache des clés utilisés par les différents services de l’application, toutes les clés seront précédées d’un code spécifique (configuré dans les paramètres). Le nom final d’un clé stocké par notre cache dans la session aura donc le nom code-key.

**Attention au découpage, même si tout ceci transparent et géré par les CacheManagers, la clé peut posséder 2 sous clés : *code-meta-key* ou *code-key***

La session en cour doit être passé en paramètre lors de la construction du service.

```
#SessionCacheManager.php

public function __construct($code, SessionInterface $session, CacheValidationInterface $cacheValidation)
{
    $this->session = $session;
    $this->code = $code;
    $this->cacheValidator = $cacheValidation;
}
```

###	DatabaseCacheManager

Ce service stock le cache dans une base de données. 

La base données est passée en paramètres lors de la construction du service doit être une Connection implémentant l’interface DriverConnection de doctrine, autrement dit, avec ce service nous pouvons utilisés plusieurs driver de connexion et donc plusieurs types de base de données (PGSQL, MYSQL, SQLITE, etc.). 

Cette extensibilité est due au fait que les requêtes ne sont pas créées par le DatabaseCacheManager mais par des CacheQueryFactory (il en existe un par type de base de données), qui sont des extension de la classe AbstractCacheQueryFactory implémentant elle-même l’interface CacheQueryFactoryInterface. Tous ces CacheQueryFactory sont distribués par le service CacheQueryFactoryProvider au DatabaseStorageManager en fonction du driver PDO.

Le nom de la table est stocké dans les paramètres et passé en paramètre en second argument lors de la création du service.

```
#DatabaseCacheManager.php

public function __construct($pdo, $tableName, CacheQueryFactoryProviderInterface $CacheQueryFactoryProvider)
{
    $this->pdo = $pdo;

    $pdoDriverName = $this->pdo->getDriver()->getName();
    $serviceName = $this->changeDrivernameIntoServicename($pdoDriverName);

    $this->CacheQueryFactory = $CacheQueryFactoryProvider->getCacheQueryFactory($serviceName);
    $this->CacheQueryFactory->setTableName($tableName);

    $this->init();
}
```

Exemple de l’association entre le DatabaseCacheManager et le CacheQueryFactory :
On suppose que la connexion PDO passée en paramètre dans le constructeur de DatabaseCacheManager utilise un driver PGSQL. Nous récupérons donc le service PGSQLCacheQueryFactory via le CacheQueryFactoryProvider. Nous appelons la méthode get de DatabaseStorageManager (donc de AbstractCacheManager qui fait ensuite appel à la méthode getData de DatabaseStorageManager)

```
#DatabaseCacheManager.php

protected function getData($key)
{
    return $this->decodeBinary(stream_get_contents($this->selectData($key)['datax']));
}
```
qui appelle la méthode selectData
```
#DatabaseCacheManager.php

protected function selectData($key)
{
    $statement = $this->pdo->prepare($this->CacheQueryFactory->createSelectDataQuery());
    $statement->execute(array($key));
    return $statement->fetch();
}
```
qui fait appelle à la méthode createSelectDataQuery du CacheQueryFactory, et dans notre cas PGSQLCacheQueryFactory
```
#PGSQLCacheQueryFactory.php

public function createSelectDataQuery()
{
    return "SELECT datax FROM ".$this->tableName." WHERE keyx = ?";
}
```
cette méthode nous retourne simplement une requête interprétable par une base de données PostgreSQL, qui est exécutée par la méthode précédente.

## Administration

### The administration page

Afin de pouvoir manager le cache facilement, une interface a été créé sur la page Administration

Cette page est chargée par le CacheController avec la route *cache_manager* via */cache-manager*

Afin de pouvoir bien interpréter les données entre le cache et le Controller, un service intermédiaire AdminManagerCache formate les données du cache afin de bien pouvoir les présenter sur la page d’administration.
Cette page nous permet de consulter les différentes clés contenus dans les différents caches, de voir leur détail, de les supprimer, et de les mettre à jour si possible

> (#Update) En passant par le Listener, il est possible de mettre à jour une clé du cache via la page d'administration, il suffit de renseigner le nom de son service lors de la mise en cache dans ses métadonnées. Ainsi ses métadonnées doivent avoir la forme:

```
@Cacheable(key="avenantList",options={"timeout"=3600,"service"="api_avenant_client"},cacheManager="FileCacheManager")
```


### Parameters

Toutes les configurations et tout le paramétrage doivent se faire dans le fichier *parameters.yml* qui doit être constitué comme ci-dessous:

```
#parameters.yml

# Caches : parametres de gestion du cache
redis.host: 127.0.0.1
redis.port: 6379
default_cache_manager: 'RedisCacheManager'
default_query_cache_creator: 'PGSQLCacheQueryQueryFactory'
table_cache_name: 'cachemanager'
file_cache_manager_directory: '%kernel.cache_dir%/filecache/'
cache_session_code: 'xtvtpr-'
```

# Create a new CacheManager

Pour créer un nouveau CacheManager, il suffit de créer le service puis de le brancher sur le CacheManagerProvider comme dans la description ci-dessous

## Create the service

1. Créer le nouveau service dans le dossier CacheManagers
> Le nom de classe du service est le même que le nom du CacheManager
2. Déclarer ce service comme une extension de la classe AbstractCacheManager
3. Implémenter les méthodes manquantes (obligatoirement les méthodes permettant le stockage des données)
4. Réécrire les méthodes souhaitées

## Plug it

5. Déclarer ce service dans les ressources
6. Injecter ce service dans le CacheManagerProvider

```
#cache-services.yml

newservicename_cache_manager:
    class: Common\CommonBundle\Service\Cache\CacheManagers\NewServiceName
    autowire: true
    autoconfigure: true

cache_manager_provider:
    class: Common\CommonBundle\Service\Cache\CacheManagerProvider
    autowire: true
    autoconfigure: true
    arguments:
      - ['@file_cache_manager','@redis_cache_manager','@session_cache_manager','@database_cache_manager','@newservicename_cache_manager']
      - '%default_cache_manager%'
```
