## L'objectif de cet article est de créer une première intégration, un appareil (device) et une entity sur cet appareil
Il s'inscrit dans la suite des articles dont le sommaire est [ici](/README.md).

# Pre-requis
Avoir déroulé avec succès l'installation de l'environnement de dev décrit [ici](/README.md).

- [Pre-requis](#pre-requis)
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



# Créer et déclarer son intégration

Les étapes pour créer et initialiser son intégration sont les suivantes :
1. créer un répertoire qui va accueillir le code de l'intégration,
2. transformer le répertoire de package Python,
3. déclarer l'intégration à l'aide d'un fichier manifest

## Créer un répertoire sous custom_components

Une intégration HACS est un `custom_component` et doit être installer dans le répertoire `config/custom_components`. Au démarrage, HA parcours tous les sous-répertoires de custom_components et créé les intégrations qu'il y trouve.

Dans le navigateur, cliques droit sur `config`, "Nouveau dossier", "custom_components/tuto_hacs" (ou tout autre nom qui te plait). On peut créer les 2 répertoires en une seule fois.

> ![Tip](/images/tips.png?raw=true?raw=true) Le choix du nom de l'intégration est important : il va rester, il sera compliqué de le changer ensuite et surtout il ne doit pas entrer en collision avec une intégration HACS déjà existante. Une petite recherche sur internet avec le nom que tu as choisi est fortement conseillé à ce niveau là.

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


async def async_setup(hass: HomeAssistant, config: ConfigEntry):
    """Initialisation de l'intégration"""
    _LOGGER.info("Initializing %s integration with plaforms: %s", DOMAIN, PLATFORMS)

    # Return boolean to indicate that initialization was successful.
    return True
```

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

Si tu essaies de l'ajouter, HA prévient que l'ajout ne peut être fait qu'à la main dans le `configuration.yaml` car l'intégration n'a pas d'interface de configuration (option `"config_flow": false` du fichier manifest).

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

L'idée est que cette liste soit toujour vide. Cette liste se met à jour en fur et à mesure de la frappe du code et se raffraichit avec une sauvegarde des fichiers (Command + Shift + S sur Mac).

### Could not be resolved
Les erreurs du type `import "homeassistant.core" could not be resolved` se corrige facilement en indiquant à VSC quel interpréteur Python il doit utiliser. En l'occurence, on doit lui indiquer celui du container dans lequel le package homeassistant a été installé (souviens toi de : `pip install -r requirements.txt` qui installe le package homeassistant). Pour faire ça, il faut :
Command + Shift + P / Python sélectionner un interpréteur et choisir "Utiliser Python à partir du paramètre `python.defaultInterpreterPath` qu'on a renseigné dans notre `devcontainer.json`

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

Ma recommandation est de garder cette liste d'erreur vide. Ca permet de tout de suite prendre les bonnes habitudes et de voir instantanément, au cours de la frappe si quelque-chose ne va pas.
Si tu as 98 erreurs et que le compteurs passe à 99, tu ne la verras pas et tu vas potentiellement louper quelque-chose et perdre du temps.

> ![Tip](/images/tips.png?raw=true) A ce stade, on a :
> - une intégration reconnue par Home Assistant,
> - instanciée et initialisée par Home Assistant
>
> **Il ne nous "reste" plus qu'à lui faire faire des trucs.**






