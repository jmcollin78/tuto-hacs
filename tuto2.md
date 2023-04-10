## L'objectif de cet article est de créer une première intégration et une prmière entité simple
Il s'inscrit dans la suite des articles dont le sommaire est [ici](/README.md).

> ![Tip](/images/tips.png?raw=true) Les fichiers sources complets sont en fin d'article. Cf [Fichiers sources du tuto](#fichiers-sources-du-tuto)

# Pre-requis
Avoir déroulé avec succès l'installation de l'environnement de dev décrit [ici](/tuto1.md).

# Sommaire
- [Pre-requis](#pre-requis)
- [Sommaire](#sommaire)
- [Créer et déclarer son intégration](#créer-et-déclarer-son-intégration)
  - [Créer un répertoire sous custom\_components](#créer-un-répertoire-sous-custom_components)
  - [Transformer le répertoire en package Python](#transformer-le-répertoire-en-package-python)
  - [Déclarer l'intégration](#déclarer-lintégration)
  - [Voir nos logs](#voir-nos-logs)
  - [Redemarrer Home Assistant](#redemarrer-home-assistant)
  - [Instancier notre intégration](#instancier-notre-intégration)
  - [Corriger les erreurs de compilation](#corriger-les-erreurs-de-compilation)
    - [Could not be resolved](#could-not-be-resolved)
    - [Unused argument](#unused-argument)
- [Créer une première entité simple](#créer-une-première-entité-simple)
  - [Ajouter une plate-forme](#ajouter-une-plate-forme)
  - [Ajouter le code source de la plate-forme `sensor`](#ajouter-le-code-source-de-la-plate-forme-sensor)
  - [Configurer une entité](#configurer-une-entité)
  - [Donner un état et des attributs à l'entité](#donner-un-état-et-des-attributs-à-lentité)
  - [Relier l'entité à un appareil](#relier-lentité-à-un-appareil)
- [Debugger notre code](#debugger-notre-code)
  - [Configurer le debugger](#configurer-le-debugger)
    - [Ajouter une configuration de lancement dans VSC](#ajouter-une-configuration-de-lancement-dans-vsc)
  - [Lancer Home Assistant en mode debug](#lancer-home-assistant-en-mode-debug)
- [Conclusion](#conclusion)
- [Fichiers sources du tuto](#fichiers-sources-du-tuto)
  - [`manifest.json`](#manifestjson)
  - [`__init__.py`](#__init__py)
  - [`const.py`](#constpy)
  - [`sensor.py`](#sensorpy)
  - [`launch.json`](#launchjson)
  - [`tasks.json`](#tasksjson)
  - [`configuration.yaml`](#configurationyaml)

# Créer et déclarer son intégration

Les étapes pour créer et initialiser son intégration sont les suivantes :
1. créer un répertoire qui va accueillir le code de l'intégration,
2. transformer le répertoire de package Python,
3. déclarer l'intégration à l'aide d'un fichier manifest

## Créer un répertoire sous custom_components

Une intégration HACS est un `custom_component` et doit être installer dans le répertoire `config/custom_components`. Au démarrage, HA parcours tous les sous-répertoires de `custom_components` et créé les intégrations qu'il y trouve.

Dans le navigateur, cliques droit sur `config`, "Nouveau dossier", "custom_components/tuto_hacs" (ou tout autre nom qui te plait). On peut créer les 2 répertoires en une seule fois.

> ![Tip](/images/tips.png?raw=true) Le choix du nom de l'intégration est important : il va rester, il sera compliqué de le changer ensuite et surtout il ne doit pas entrer en collision avec une intégration HACS déjà existante. Une petite recherche sur internet avec le nom que tu as choisi est fortement conseillé à ce niveau là.

Tu dois avoir quelque-chose qui ressemble à ça :
> ![Custom_components](/images/custom_components.png?raw=true)

## Transformer le répertoire en package Python

Le répertoire de l'intégration étant vide il ne va pas être reconnu par HA comme une intégration. Il faut le transformer en un module Python à part entière.
Ca se fait en ajoutant notre premier fichier source Pyhton qui se nomme `__init__.py` (attention, il y a 2 '_' devant et après `init`). Il contient le code source de l'initialisation du package et donc de notre intégration (qui est un package Python).

C'est une convention de nommage de Python. Tu peux approfondir le sujet avec cette article au besoin : [les packages Python](https://docs.python.org/fr/3.5/tutorial/modules.html#packages).

On en profite pour ajouter un fichier nommé `const.py` dans lequel on va déclarer toutes nos différentes constantes qui seront nécessaires à notre intégration.

Ce fichier `const.py` doit contenir, les constantes suivantes :
```python
""" Les constantes pour l'intégration Tuto HACS """

from homeassistant.const import Platform

DOMAIN = "tuto_hacs"
PLATFORM: list[Platform] = []
```
- La première ligne est un commentaire qui explique ce que contient le fichier.
- La ligne `from homeassistant.const import Platform` permet d'importer la définition de Platform depuis la librairie Home Assistant. Elle va nous permettre de déclarer toutes les plates-formes utilisées par notre intégration. Dans le langage HA, une plate-forme est un type d'entité (`switch`, `light`, `climate`, `sensor`, ...). C'est ce qui se trouve devant le `.` dans le nom d'une entité.
- La ligne `DOMAIN = "tuto_hacs"` définit le domaine de notre intégration. Un domaine est un nom d'intégration. Tous les appareils et entités de notre intégration feront parties de domaine. Le domaine doit être **le même que le nom du répertoire de l'intégration** : `tuto_hacs` dans notre cas.
- Ensuite on définit la liste des plate-formes utilisées par l'intégration avec la ligne: `PLATFORM: list[Platform] = []`. On déclare une liste de Platform (`list[Platform]`) et on l'initialise avec rien pour l'instant (` = []`)


Le code d'initialisation dans le fichier `__init__.py` est le suivant :
```python
"""Initialisation du package de l'intégration HACS Tuto"""
import logging

from homeassistant.core import HomeAssistant
from homeassistant.config_entries import ConfigEntry

from .const import DOMAIN, PLATFORMS

_LOGGER = logging.getLogger(__name__)


async def async_setup(
    hass: HomeAssistant, config: ConfigEntry
):  # pylint: disable=unused-argument
    """Initialisation de l'intégration"""
    _LOGGER.info(
        "Initializing %s integration with plaforms: %s with config: %s",
        DOMAIN,
        PLATFORMS,
        config,
    )

    # Mettre ici un eventuel code permettant l'initialisation de l'intégration
    # (pas nécessaire pour le tuto)

    # L'argument config contient votre fichier configuration.yaml
    my_config = config.get(DOMAIN)  # pylint: disable=unused-variable

    # Return boolean to indicate that initialization was successful.
    return True
```

> ![Tip](/images/tips.png?raw=true)
> - La fonction `async_setup` est appelée par Home Assistant lors de la découverte de l'intégration. Vous pourrez y mettre tout le code nécessaire à son initialisation,
> - L'argument config contient le `configuration.yaml`. On pourrait accéder à d'éventuels paramètre de l'intégration avec le code suivant `config.get(DOMAIN)`.

## Déclarer l'intégration
La déclaration de l'intégration a Home Assistant se fait via un fichier de conf nommé `manifest.json`. Les fichiers manifest sont des fichiers descriptifs qui sont utilisés au démarrage de Home Assistant dans la phase de découverte des intégrations. Il doit contenir les infos suivantes :
```json
{
    "domain": "tuto_hacs",
    "name": "Tuto HACS",
    "codeowners": [
        "@jmcollin78"
    ],
    "config_flow": false,
    "documentation": "https://github.com/jmcollin78/tuto_hacs",
    "issue_tracker": "https://github.com/jmcollin78/tuto_hacs/issues",
    "integration_type": "device",
    "iot_class": "calculated",
    "quality_scale": "silver",
    "version": "3.0.0"
}
```

Les valeurs à déclarer sont les suivantes :
1. `domain` : notre domaine. Doit être égal à la constante `DOMAIN` de notre `const.yaml`,
2. `name` : le nom de l'intégration tel qu'il s'affichera dans le menu "Ajouter une intégration",
3. `codeowners` : les noms Github des propriétaires du code. Mettez le votre,
4. `config_flow` : présence ou non d'une interface de configuration de l'intégration. On en reparlera en détail dans un prochain tuto,
5. `documentation` : le lien Github vers la documentation,
6. `issue_tracker` : le lien Github vers la déclaration des report de bugs ou anomalies de fonctionnement,
7. `integration_type` : plusieurs type d'intégration sont possibles. Le type device permet d'indiquer que l'intégration va créer des appareils (devices) et des entités,
8. `iot_class` : plusieurs "IOT class" sont disponibles. Cette option définie comment notre intégration intéragit avec les appareils. Les plus communs sont : `cloud_polling` (les appareils / entités se mettent à jour en interrogeant régulièrement le Cloud), `local_polling` intérrogation régulière d'un appareil en local sur le réseau, `local_push` l'appareil en local envoi les nouvelles valeurs en cas de changement (pas besoin de l'interroger)
9. `quality_scale`: le niveau de qualité de votre intégration,
10. `version`: la version du `manifest.json`. La dernière en date doit être 3.0.0.

La documentation complète est [ici](https://developers.home-assistant.io/docs/creating_integration_manifest).

## Voir nos logs
Pour debugger et suivre le bon fonctionnement de votre intégration, tu vas avoir besoin de configurer les logs.
Cela se fait en modifiant le fichier `configuration.yaml` de la façon suivante :

```yaml
logger:
    default: info
    logs:
        custom_components.tuto_hacs: debug
```

## Redemarrer Home Assistant
Lances Home Assistant en utilisant les tâches faites au tuto1 (Command + Shift + P / Tâches: exécuter la tâche / Run Home Assistant on port 9123).
Pour rappel, tu dois avoir le port 9123 ouvert si le démarrage est bon : ![Port ouvert](/images/port-ouvert.png?raw=true)

Regardes les logs de Home Assistant (soit dans le Terminal de la tâche "Run Home Assistant.." dans directement dans le fichier `home-assistant.log`).
Tu devrais voir le log suivant :
```log
2023-04-09 08:10:22.372 WARNING (SyncWorker_0) [homeassistant.loader] We found a custom integration tuto_hacs which has not been tested by Home Assistant. This component might cause stability problems, be sure to disable it if you experience issues with Home Assistant
```

> ![Tip](/images/tips.png?raw=true?raw=true) Ca montre que notre intégration est bien reconnue par Home Assistant.
> 
> Par contre, on ne voit pas notre log qui correspond à la ligne `_LOGGER.info("Initializing %s integration with plaforms: %s", DOMAIN, PLATFORMS)` ce qui indique que notre intégration n'est pas utilisée. On va y remedier un peu en-dessous.

## Instancier notre intégration
Dans l'interface web, menu intégration, ajouter une intégration, on voit bien notre intégration :

> ![Tuto HACS](/images/new-integration.png?raw=true)

Si tu essaies de l'ajouter, HA prévient que l'ajout ne peut être fait qu'à la main dans le `configuration.yaml` car l'intégration n'a pas d'interface de configuration (option `"config_flow": false` du fichier `manifest.json`).

> ![Tuto HACS](/images/integration-manuelle.png?raw=true)

C'est ce qu'on va faire. On va ajouter cette simple ligne dans notre `configuration.yaml` pour utiliser notre intégration :
```yaml
tuto_hacs:
```

On redémarrage (Command + Shift + P / taches ...), on regarde les logs et on voit :
```log
2023-04-09 08:40:40.253 WARNING (SyncWorker_0) [homeassistant.loader] We found a custom integration tuto_hacs which has not been tested by Home Assistant. This component might cause stability problems, be sure to disable it if you experience issues with Home Assistant
...
2023-04-09 08:40:40.973 INFO (MainThread) [homeassistant.bootstrap] Setting up stage 2: {... , 'tuto_hacs', ... }
...
2023-04-09 08:40:40.987 INFO (MainThread) [homeassistant.setup] Setting up tuto_hacs
2023-04-09 08:40:40.987 INFO (MainThread) [custom_components.tuto_hacs] Initializing tuto_hacs integration with plaforms: []
2023-04-09 08:40:40.987 INFO (MainThread) [homeassistant.setup] Setup of domain tuto_hacs took 0.0 seconds
...
```

On retrouve bien, cette fois, notre log d'initialisation !

## Corriger les erreurs de compilation
Si tu regardes dans l'onglet Problèmes, tu verras un certain nombre d'erreurs ou de warning :

> ![Compilation problèmes](/images/compilation-problemes.png?raw=true)

L'idée est que cette liste soit toujour vide. Cette liste se met à jour en fur et à mesure de la frappe du code et se rafraichit lors d'une sauvegarde des fichiers (Command + Shift + S sur Mac).

### Could not be resolved
Les erreurs du type `import "homeassistant.core" could not be resolved` se corrige facilement en indiquant à VSC quel interpréteur Python il doit utiliser. En l'occurence, on doit lui indiquer celui du container dans lequel le package homeassistant a été installé (souviens toi de : `pip install -r requirements.txt` qui installe le package homeassistant). Pour faire ça, il faut :
Command + Shift + P / "Python sélectionner un interpréteur" et choisir "Utiliser Python à partir du paramètre `python.defaultInterpreterPath` qu'on a renseigné dans notre `devcontainer.json`

> ![Tip](/images/interpreteur-python.png?raw=true)

Ca devrait supprimer toutes ces erreurs.

### Unused argument
Ces erreurs sont signalées lors de la déclaration de la fonction `async_setup` qui prend 2 arguments hass et config mais qui ne sont pas utilisés pour l'instant.
> ![Tip](/images/unused-argument.png?raw=true)

Pour les faire disparaitre, 4 possibilités :
1. on utilise les arguments dans notre fonction. Dans notre exemple, on n'a pas l'occasion,
2. on supprime les arguments inutiles,
3. on met en commentaires les arguments parce-qu'on pense qu'on va en avoir besoin un jour : `async def async_setup(): # hass: HomeAssistant, config: ConfigEntry ):`
4. on met un tag qui indique au linter d'ignorer ces erreurs. Ca se fait en ajoutant le commentaire suivant sur la ligne en question : `# pylint: disable=unused-argument`

La 4ème méthode est de loin la plus propre si on veut garder les arguments pour un prochain usage.

Ma recommandation est de **garder cette liste d'erreur vide**. Ca permet de tout de suite **prendre les bonnes habitudes** et de voir instantanément, au cours de la frappe si quelque-chose ne va pas.
Si tu as 98 erreurs et que le compteurs passe à 99, tu ne la verras pas et tu vas potentiellement louper quelque-chose et perdre du temps.

> ![Tip](/images/tips.png?raw=true) A ce stade, on a :
> - une intégration reconnue par Home Assistant,
> - instanciée et initialisée par Home Assistant
>
> **Il ne nous "reste" plus qu'à lui faire faire des 'trucs'.**


# Créer une première entité simple

On va créer une entité qui expose une valeur en secondes pour démarrer doucement. Ca va nous permettre de voir pas mal de concepts clés.

La démarche pour déclarer une entité est la suivante :
1. ajouter une plate-forme pour notre entité (sensor dans notre cas de test),
2. donner le code de notre entité,
3. configurer notre entité dans le `configuration.yaml`,
4. donner un état et des attributs à l'entité,
5. relier l'entité à un appareil.

## Ajouter une plate-forme
On déclare que notre intégration utilise la plate-forme `sensor` dans le fichier `const.yaml`, en ajoutant le code suivant :

```python
PLATFORMS: list[Platform] = [Platform.SENSOR]
```
Ca indique a Home Assistant qu'il doit trouver un fichier source nommé `sensor.py` dans notre package `tuto_hacs`. Ce code source est en charge d'instancier TOUS les sensors. **On ne fait pas un code source par sensor mais bien un code source par plate-forme.**

Pour avoir un code maintenable, on va créer une classe par entité.

## Ajouter le code source de la plate-forme `sensor`
Pour faire ça, on ajoute le fichier `sensor.py` dans notre intégration, et on met le code le classe de notre entité dedans :
```python
""" Implements the VersatileThermostat sensors component """
import logging

from homeassistant.core import HomeAssistant
from homeassistant.config_entries import ConfigEntry
from homeassistant.helpers.entity_platform import AddEntitiesCallback
from homeassistant.components.sensor import SensorEntity

_LOGGER = logging.getLogger(__name__)


async def async_setup_platform(
    hass: HomeAssistant,
    entry: ConfigEntry,
    async_add_entities: AddEntitiesCallback,
    discovery_info=None,  # pylint: disable=unused-argument
):
    """Configuration de la plate-forme tuto_hacs à partir de la configuration
    trouvée dans configuration.yaml"""

    _LOGGER.debug("Calling async_setup_entry entry=%s", entry)

    entity = TutoHacsElapsedSecondEnity(hass, entry)
    async_add_entities([entity], True)


class TutoHacsElapsedSecondEnity(SensorEntity):
    """La classe de l'entité TutoHacs"""

    def __init__(
        self,
        hass: HomeAssistant,  # pylint: disable=unused-argument
        entry_infos,  # pylint: disable=unused-argument
    ) -> None:
        """Initisalisation de notre entité"""
        self._attr_name = entry_infos.get("name")
        self._attr_unique_id = entry_infos.get("entity_id")
        self._attr_has_entity_name = True

```
La fonction `async_setup_platform` est appelée par Home Assistant lorsqu'une entité est de type `sensor`, pour notre domaine est configurée (dans `configuration.yaml`). Elle prend en argument, l'objet `hass` (qu'on verra plus en détail dans le prochain tuto), la configuration trouvée dans le `configuration.yaml` et une fonction `async_add_entities` qui doit être appelée pour ajouter les entités.

Elle instancie notre entité à partir de sa classe qui le représente (`TutoHacsElapsedSecondEnity`) et appelle `async_add_entities` avec un tableau des classes d'entités créées.

Ce fichier contient aussi la déclaration de la classe `TutoHacsElapsedSecondEnity`, ne faisant pas grand chose mais qui représente notre entité. Elle dérive de `SensorEntity` qui est la classe de base de toutes les entités de type Sensor.

Pour l'instant cette classe, ne fait rien d'autre qu'initialiser les 2 attributs `_attr_name` et `_att_unique_id` qui sont nécessaire à la création de l'entité. Comme a donné un nom à notre entité, on l'indique à HA (sinon il lui affecte un nom par défaut) avec la ligne: `self._attr_has_entity_name = True`.

On va redémarrer Home Assistant et vérifier que tout se passe bien (command + Shift + P). Les logs ne montrent pas grand chose de plus que ci-dessus ; ceci est normal car nous n'avons pas configuré d'entité dans le fichier `configuration.yaml`.

## Configurer une entité

On ajoute le bloc suivant dans le `configuration.yaml` pour déclarer un Sensor, sur notre plate-forme `tuto_hacs` avec les 2 attributs `entity_id` et `name` :
```yaml
sensor:
  - platform: tuto_hacs
    entity_id: tuto_hacs_entité
    name: Tuto HACS Entité
    dautres: attributs utiles
```

On redémarre Home Assistant et cette fois on voit un peu plus de log :
```log
# L'initialisation de notre domaine:

2023-04-09 22:28:25.604 INFO (MainThread) [custom_components.tuto_hacs] Initializing tuto_hacs integration with plaforms: [<Platform.SENSOR: 'sensor'>] with config: {'default_config': {}, 'frontend': {'themes': {}}, 'tts': [{'platform': 'google_translate'}], 'automation': {}, 'script': {}, 'scene': {}, 'logger': {'default': 'info', 'logs': {'custom_components.tuto_hacs': 'debug'}}, 'tuto_hacs': {}, 'sensor': [{'platform': 'tuto_hacs', 'entity_id': 'tuto_hacs_entité', 'name': 'Tuto HACS Entité'}]}
2023-04-09 22:28:25.604 INFO (MainThread) [homeassistant.setup] Setup of domain tuto_hacs took 0.0 seconds

...
# L'initialisation de notre entité

2023-04-09 22:28:25.734 INFO (MainThread) [homeassistant.components.sensor] Setting up sensor.tuto_hacs
2023-04-09 22:28:25.735 DEBUG (MainThread) [custom_components.tuto_hacs.sensor] Calling async_setup_entry entry={'platform': 'tuto_hacs', 'entity_id': 'tuto_hacs_entité', 'name': 'Tuto HACS Entité'}
```
Si on regarde sur le web [ici](http://localhost:9123/config/entities), on peut voir notre entité :

> ![Tip](/images/entite-1.png?raw=true)

## Donner un état et des attributs à l'entité
L'entité existe mais n'a pas d'état (`undefined`), pas d'unité de mesure et pas d'icône. On va corriger tout ça en ajoutant le code suivant dans notre classe d'entité :
```python
from homeassistant.const import UnitOfTime
from homeassistant.components.sensor import (
    SensorEntity,
    SensorDeviceClass,
    SensorStateClass,
)

...
class TutoHacsElapsedSecondEnity(SensorEntity):
...

    @property
    def icon(self) -> str | None:
        return "mdi:timer-play"

    @property
    def device_class(self) -> SensorDeviceClass | None:
        return SensorDeviceClass.DURATION

    @property
    def state_class(self) -> SensorStateClass | None:
        return SensorStateClass.MEASUREMENT

    @property
    def native_unit_of_measurement(self) -> str | None:
        return UnitOfTime.SECONDS
```

On a déclaré la proprité `icon` en donnant le nom de l'icone a utiliser pour cet entité, la classe du device qui est ici une durée (`SensorDeviceClass.DURATION`), une classe d'état qui est ici une mesure (`SensorStateClass.MEASUREMENT`) et une unité de la mesure qui est ici des secondes (`UnitOfTime.SECONDS`).

Plein d'autres combinaisons sont possibles, la doc pour aller plus loin sur le sujet est [ici](https://developers.home-assistant.io/docs/core/entity/sensor).

On va lui donner une valeur d'état (ie. le `state`) en ajoutant la ligne suivante dans la fonction `__init__` de la classe :

```python
    def __init__(
        self,
        hass: HomeAssistant,  # pylint: disable=unused-argument
        entry_infos,  # pylint: disable=unused-argument
    ) -> None:
        """Initisalisation de notre entité"""
        ... 
        self._attr_native_value = 12
```

et enfin, on indique à Home Assistant que notre entité ne doit pas être pollée puisque pour l'instant ca valeur est fixe :
```python
    @property
    def should_poll(self) -> bool:
        """Do not poll for those entities"""
        return False
```

On redémarre Home Assistant et si on regarde maintenant sur le web :

> ![Tip](/images/entite-2.png?raw=true)
>
> On voit bien l'icone, la valeur 12 et l'unité en secondes.

## Relier l'entité à un appareil
Après pas mal de recherche, il n'est pas possible de relier une entité créée par `configuration.yaml` à un appareil. On verra cette partie dans le tuto 4.

# Debugger notre code
Lorsque ça se passe mal et qu'on souhaite débugger notre code, 2 possibilités s'offre à nous :
1. **ajouter des logs**. On a vu plusieurs exemple ci-dessus. Cf [Voir nos logs](#voir-nos-logs),
2. **exécuter le code pas-à-pas**, inspecter les variables et comprendre ce qui se passe. C'est ce dernier point qu'on va voir ici pour terminer ce tuto

## Configurer le debugger
Il faut indiquer à Home Assistant de s'éxécuter en mode debug. Pour cela, on ajoute le bloc de code suivant dans le `configuration.yaml` :

```yaml
debugpy:
  start: true
  wait: false
  port: 5678
```

Avec cette configuration, on indique :
1. qu'on veut activer le debugger Python (`debugpy`),
2. qu'on veut le démarrer tout de suite (`start: true`),
3. que Home Assistant doit se mettre en attente de la connexion du debugger au démarrage (`wait: true`). Mets le à `false` si tu ne veux pas attendre au démarrage. Comme c'est le debugger qui lance Home Assistant cette valeur peut rester sur `false` sans soucis,
4. et que le port du debugger est le port 5678.

### Ajouter une configuration de lancement dans VSC
Cliques sur le bouton du debugger dans VSC : ![Debugger bouton](/images/debugger-bouton.png?raw=true).
Appuies sur "Créer un fichier launch.json" : ![Launch.json](/images/creer-launch.png?raw=true).

Un fichier `launch.json` permet de créer des configuration de lancement de nos applications. On va en créer une qui démarre Home Assistant en mode debug dans le debugger.

Choisis ensuite "Suggestions" en face de Python : ![Launch.json](/images/debugger-python.png?raw=true), puis enfin "Module" : ![Debugger module](/images/debugger-module.png?raw=true).

Donne alors `homeassistant` comme nom de module à débugger : ![Debugger module](/images/debugger-homeassistant.png?raw=true) et appuies sur entrée.

VSC t'as créé un fichier `launch.json` qui contient presque tout ce qu'on a besoin pour debugger notre intégration (et Home Assistant au passage comme on va le voir ci-dessous !).
Modifies le fichier `launch.json` pour ajouter la ligne `args`, changes `justMyCode` en `false` pour pouvoir debugger Home Assistant. Profites en pour donner un `name` plus clair.

Le fichier `launch.json` doit maintenant contenir :

```yaml
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Home Assistant (debug)",
            "type": "python",
            "request": "launch",
            "module": "homeassistant",
            "justMyCode": false,
            "args": [
                "--debug",
                "-c",
                "config"
            ]
        }
    ]
}
```

Notre configuration de lancement est maintenant visible en haut à gauche : ![Launch créé](/images/launch-cree.png?raw=true).

## Lancer Home Assistant en mode debug
Pour vérifier que ça marche, on va mettre un point d'arrêt dans notre code.
Sélectionne le fichier `sensor.py` et clique dans la marge en face de la ligne suivante : ![Debugger module](/images/debugger-breakpoint.png?raw=true).
Un point rouge s'affiche pour indiquer qu'un point d'arrêt a bien été positionné sur cette ligne.

Tous les points d'arrêt sont visibles à bas à gauche dans la fenêtre "POINTS D'ARRET" :

> ![Debugger module](/images/debugger-breakpoint2.png?raw=true)

Il est possible de **désactiver, réactiver, supprimer les points d'arrêt** directement depuis cette fenêtre.

Lances Home Assistant en mode débugger en appuyant sur la flêche verte : ![Launch créé](/images/launch-cree.png?raw=true).

> ![Tip](/images/tips.png?raw=true) Vérifies bien que Home Assistant est stoppé avant de lancer le debugger. Comme les 2 utilises le même port, tu ne peux avoir qu'une seule instance de Home Assistant qui tourne.

Home Assistant se lance et au bout de quelques-instants, le lancement se bloque sur notre point d'arrêt. On a alors l'affichage suivant :

> ![Debugger Stop](/images/debugger-stop.png?raw=true)
> On peut voir :
> 1. l'exécution est stoppée sur notre point d'arrêt à la ligne surlignée en jaune pale,
> 2. l'état des différentes variables à ce moment de l'exécution,
> 3. la pile des appels,
> 4. les logs de Home Assistant,
> 5. une barre d'outil qui va nous permettre d'éxécuter pas à pas le code. Attention : cette barre est relativement invisible et pas souvent bien positionnée. Tu peux la déplacer avec la "poignée" à gauche de la barre.

En passant la souris sur une variable, on peut inspecter sa valeur, ce qui est super pratique.

Appuies sur les boutons de la d'outil du debugger pour "Continuer l'execution" ![Continuer](/images/debugger-bouton-continue.png?raw=true), "Sauter l'exécution de l'instruction courante" ![Sauter](/images/debugger-bouton-saute.png?raw=true), "Entrer dans l'appel de la fonction" ![Entrer](/images/debugger-bouton-entre.png?raw=true), "Sortir de la fonction courante" ![Sortir](/images/debugger-bouton-sort.png?raw=true), "Stopper le debugger" ![Sopper](/images/debugger-bouton-stop.png?raw=true).

Plus d'informations sur le debugger [ici (VSC)](https://code.visualstudio.com/docs/editor/debugging) et [ici (Home Assistant)](https://www.home-assistant.io/integrations/debugpy/)

# Conclusion

Dans ce tuto, tu as appris à :
1. créer une intégration et faire en sorte qu'elle soit reconnue par Home Assistant,
2. créer une entité dans cette intégration en lui donnant quelques caractéristiques de base (unité, icone, valeur, classe),
3. debugger le code avec le debugger intégré de VSC.


# Fichiers sources du tuto

L'ensemble du code résultat est remis ici :

## `manifest.json`

```json
{
    "domain": "tuto_hacs",
    "name": "Tuto HACS",
    "codeowners": [
        "@jmcollin78"
    ],
    "config_flow": false,
    "documentation": "https://github.com/jmcollin78/tuto_hacs",
    "integration_type": "device",
    "iot_class": "calculated",
    "issue_tracker": "https://github.com/jmcollin78/tuto_hacs/issues",
    "quality_scale": "silver",
    "version": "3.0.0"
}
```

## `__init__.py`

```python
"""Initialisation du package de l'intégration HACS Tuto"""
import logging

from homeassistant.core import HomeAssistant
from homeassistant.config_entries import ConfigEntry

from .const import DOMAIN, PLATFORMS

_LOGGER = logging.getLogger(__name__)


async def async_setup(
    hass: HomeAssistant, config: ConfigEntry
):  # pylint: disable=unused-argument
    """Initialisation de l'intégration"""
    _LOGGER.info(
        "Initializing %s integration with plaforms: %s with config: %s",
        DOMAIN,
        PLATFORMS,
        config,
    )

    # Mettre ici un eventuel code permettant l'initialisation de l'intégration
    # (pas nécessaire pour le tuto)

    # L'argument config contient votre fichier configuration.yaml
    my_config = config.get(DOMAIN)  # pylint: disable=unused-variable

    # Return boolean to indicate that initialization was successful.
    return True

```

## `const.py`

```python
""" Les constantes pour l'intégration Tuto HACS """

from homeassistant.const import Platform

DOMAIN = "tuto_hacs"
PLATFORMS: list[Platform] = [Platform.SENSOR]

CONF_NAME = "name"
CONF_DEVICE_ID = "device_id"

DEVICE_MANUFACTURER = "JMCOLLIN"
```

## `sensor.py`

```python
""" Implements the VersatileThermostat sensors component """
import logging

from homeassistant.const import UnitOfTime
from homeassistant.core import HomeAssistant
from homeassistant.config_entries import ConfigEntry
from homeassistant.helpers.entity_platform import AddEntitiesCallback
from homeassistant.components.sensor import (
    SensorEntity,
    SensorDeviceClass,
    SensorStateClass,
)

from homeassistant.helpers.entity import DeviceInfo, DeviceEntryType

from .const import DOMAIN, DEVICE_MANUFACTURER, CONF_DEVICE_ID, CONF_NAME

_LOGGER = logging.getLogger(__name__)


async def async_setup_platform(
    hass: HomeAssistant,
    entry: ConfigEntry,
    async_add_entities: AddEntitiesCallback,
    discovery_info=None,  # pylint: disable=unused-argument
):
    """Configuration de la plate-forme tuto_hacs à partir de la configuration
    trouvée dans configuration.yaml"""

    _LOGGER.debug("Calling async_setup_entry entry=%s", entry)

    entity = TutoHacsElapsedSecondEnity(hass, entry)
    async_add_entities([entity], True)


class TutoHacsElapsedSecondEnity(SensorEntity):
    """La classe de l'entité TutoHacs"""

    def __init__(
        self,
        hass: HomeAssistant,  # pylint: disable=unused-argument
        entry_infos,  # pylint: disable=unused-argument
    ) -> None:
        """Initisalisation de notre entité"""
        self._attr_has_entity_name = True
        self._attr_name = entry_infos.get(CONF_NAME)
        self._device_id = entry_infos.get(CONF_DEVICE_ID) # Pas utilisé pour le moment
        self._attr_unique_id = self._device_id + "_seconds"
        self._attr_native_value = 12

    @property
    def should_poll(self) -> bool:
        """Do not poll for those entities"""
        return False

    @property
    def icon(self) -> str | None:
        return "mdi:timer-play"

    @property
    def device_class(self) -> SensorDeviceClass | None:
        return SensorDeviceClass.DURATION

    @property
    def state_class(self) -> SensorStateClass | None:
        return SensorStateClass.MEASUREMENT

    @property
    def native_unit_of_measurement(self) -> str | None:
        return UnitOfTime.SECONDS
```

## `launch.json`

```yaml
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Home Assistant (debug)",
            "type": "python",
            "request": "launch",
            "module": "homeassistant",
            "justMyCode": false,
            "args": [
                "--debug",
                "-c",
                "config"
            ]
        }
    ]
}
```

## `tasks.json`

```yaml
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Run Home Assistant on port 9123",
            "type": "shell",
            "command": "hass -c ./config",
            "problemMatcher": []
        },
        {
            "label": "Restart Home Assistant on port 9123",
            "type": "shell",
            "command": "pkill hass ; hass -c ./config",
            "problemMatcher": []
        }
    ]
}
```

## `configuration.yaml`

```yaml
# Loads default set of integrations. Do not remove.
default_config:

# Load frontend themes from the themes folder
frontend:
  themes: !include_dir_merge_named themes

# Text to speech
tts:
  - platform: google_translate

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

logger:
  default: info
  logs:
    custom_components.tuto_hacs: debug

tuto_hacs:
  - not_used: non utilisé

sensor:
  - platform: tuto_hacs
    device_id: tuto_hacs_device
    name: Tuto HACS Entité
    entry_id: tuto_hacs_entry

# If you need to debug uncommment the line below (doc: https://www.home-assistant.io/integrations/debugpy/)
debugpy:
  start: true
  wait: false
  port: 5678
```