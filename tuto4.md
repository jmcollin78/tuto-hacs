## L'objectif de cet article est d'ajouter une IHM de paramétrage à notre intégration
Il s'inscrit dans la suite des articles dont le sommaire est [ici](/README.md).

> ![Tip](/images/tips.png?raw=true)
> Les fichiers sources complets en version finales sont en fin d'article. Cf [Fichiers sources du tuto](#fichiers-sources-du-tuto)

# Pre-requis
Avoir déroulé avec succès les trois premiers articles [tuto1](/tuto1.md), [tuto2](/tuto2.md) et [tuto3](/tuto3.md). Vous devez donc avoir une entité avec un état qui est une mesure en secondes et une deuxième entité qui écoute la première et stocke dans son état la date heure du dernier changement.

# Sommaire

- [Pre-requis](#pre-requis)
- [Sommaire](#sommaire)
- [Les points abordés](#les-points-abordés)
- [Contexte](#contexte)
- [Activer l'IHM de configuration](#activer-lihm-de-configuration)
- [Ajouter une étape de configuration](#ajouter-une-étape-de-configuration)
  - [Ajout de libellés dans notre formulaire](#ajout-de-libellés-dans-notre-formulaire)
- [Comprendre les schémas](#comprendre-les-schémas)
    - [Voluptuous](#voluptuous)
    - [Les Helpers Home Assistant](#les-helpers-home-assistant)
    - [Les Selectors Home Assistant](#les-selectors-home-assistant)
- [Ajouter une deuxième étape](#ajouter-une-deuxième-étape)
- [Créer une entité à partir d'une configuration](#créer-une-entité-à-partir-dune-configuration)
  - [Mémoriser les éléments saisis](#mémoriser-les-éléments-saisis)
  - [Créer une entrée de configuration (ConfigEntry)](#créer-une-entrée-de-configuration-configentry)
  - [Créer une entité à partir d'une entrée de configuration](#créer-une-entité-à-partir-dune-entrée-de-configuration)
  - [Relier les entités à un appareil (device)](#relier-les-entités-à-un-appareil-device)
- [Modifier une configuration](#modifier-une-configuration)


# Les points abordés
Dans cet article, tu vas apprendre à :
1. activer l'IHM de configuration,
2. paramétrer une étape de configuration,
3. comprendre les schémas,
4. ajouter des étapes,
5. créer une entité à partir d'une configuration,
6. modifier la configuration d'une entité existante

# Contexte

Dans les tutos précédents, notre intégration et nos entités étaient paramétrées via le fichier `config/configuration.yaml` global a Home Assistant. La modification de ce fichier, se fait en yaml, est complexe et est sujette à des beaucoup d'erreurs : indentation stricte, syntaxe particulière, ...

Dans Home Assistant il existe un moyen beaucoup plus "user-friendly" de paramétrer ses appareils et ses entités : le `config flow` (ou flot de configuration).

Cela correspond à toutes les fenêtres de configuration plus ou moins complexes que l'on peut trouver dans la plupart des intégrations récentes.
Exemple avec Versatile Thermostat :

![Versatile Thermostat](/images/vtherm-config-main.png?raw=true)

Exemple avec le panneau de configuration de Sonoff

![Sonoff](/images/sonoff-config.png?raw=true)

Ces panneaux de configuration s'ouvre lorsqu'on ajoute une intégration ou lorsqu'on veut modifier la configuration d'une intégration existante.

> ![Tip](/images/tips.png?raw=true)
> Une configuration se fait potentiellement en plusieurs étapes qui s'enchainent en cliquant sur `Valider`. Chaque étape peut dépendre de ce qui a été saisi à la précédente. On arrive donc à définir un parcours de configuration (le `flow`) dont **la dernière étape est la création de l'entité** elle-même.


# Activer l'IHM de configuration
Pour utiliser cette IHM de configuration, la première chose à faire est d'indiquer à Home Assistant que notre intégration possède un flot de configuration.
Cela se passe dans le `manifest.yaml`, on indique :

```yaml
   "config_flow": true,
```

A la lecture de cet attribut, Home Assistant va chercher le code du `configFlow` dans le fichier nommé `config_flow.py` à la racine de notre intégration. Ce nom de fichier n'est pas paramétrable. Il doit impérativement s'appeler comme ça.

On va donc créer un fichier `config_flow.yaml` le plus simple possible pour l'instant :

```python
""" Le Config Flow """

import logging

from homeassistant.config_entries import ConfigFlow
from .const import DOMAIN

_LOGGER = logging.getLogger(__name__)


class TutoHACSConfigFlow(ConfigFlow, domain=DOMAIN):
    """La classe qui implémente le config flow pour notre DOMAIN.
    Elle doit dériver de FlowHandler"""

    # La version de notre configFlow. Va permettre de migrer les entités
    # vers une version plus récente en cas de changement
    VERSION = 1
```

Vérifies les erreurs de compilation et corriges les au besoin et redémarres Home Assistant.

Lorsqu'on créé une intégration de type TutoHACS ("Paramètres / Intégrations / Ajouter une intégration") :

![Ajout intégration](/images/ajout-integration.png?raw=true)

On obtient la fenêtre de configuration suivante :

![ConfigFlow vide](/images/config-flow-vide.png?raw=true)

Pour rappel, dans le tuto1, lorsqu'on avait fait l'ajout de notre intégration, on avait eu le message suivant :

![ConfigFlow vide](/images/integration-manuelle.png?raw=true)

> ![Tip](/images/tips.png?raw=true)
> A ce stade, Home Assistant nous permet de configurer notre intégration. Mais comme aucune étape de configuration n'est codée il ne se passe rien lorsqu'on clique sur "Fermer".

# Ajouter une étape de configuration

Pour ajouter une étape de configuration (`step`), il faut ajouter une méthode par étape dans notre classe de configuration. La première étape qui va être appelée par Home Assistant doit avoir un nom fixé à l'avance qui dépend de comment a été découverte l'intégration.

Dans notre cas, l'intégration a été ajoutée par l'utilisateur, donc la méthode qui implémente la première étape doit avoir le nom suivant : `async_step_user`.
Si notre intégration avait été découverte automatiquement par le bluetooth par exemple, elle aurait du s'appeler, `async_step_bluetooth`.

> ![Tip](/images/tips.png?raw=true)
> Cette façon de faire est assez perturbante si tu développes depuis un certain temps. Le développement dans Home Assistant fait beaucoup appel à ces noms de fichiers, de classe, de méthodes dont le nom est fixeet auquel on ne peut pas déroger.

Bref, c'est comme ça et il faut faire avec.

On va donc ajouter une méthode nommée `async_step_user` puisque notre intégration est ajoutée manuellement par un utilisateur :

```python
    import voluptuous as vol
    ...

    async def async_step_user(self, user_input: dict | None = None) -> FlowResult:
        """Gestion de l'étape 'user'. Point d'entrée de notre
        configFlow. Cette méthode est appelée 2 fois :
        1. une première fois sans user_input -> on affiche le formulaire de configuration
        2. une deuxième fois avec les données saisies par l'utilisateur dans user_input -> on sauvegarde les données saisies
        """
        user_form = vol.Schema({vol.Required("name"): str})

        if user_input is None:
            _LOGGER.debug(
                "config_flow step user (1). 1er appel : pas de user_input -> on affiche le form user_form"
            )
            return self.async_show_form(step_id="user", data_schema=user_form)

        # 2ème appel : il y a des user_input -> on stocke le résultat
        # TODO: utiliser les user_input
        _LOGGER.debug(
            "config_flow step user (2). On a reçu les valeurs: %s", user_input
        )
```

Comme indiqué dans le commentaire, cette méthode va être appelée 2 fois :
1. une première fois sans user_input. Home Assistant s'attend à ce qu'on lui donne alors, le formulaire a afficher à l'utilisateur,
2. une deuxième fois, cette fois avec des données dans user_input. user_input contient alors un dictionnaire avec les valeurs du formulaire saisies par l'utilisateur. On va voir ce qu'on fait de ses valeurs ensuite. Pour l'instant, on va juste les logger.

Note : le code qui initialise le formulaire `user_form = vol.Schema({vol.Required("name"): str})`sera expliqué ci-dessous TODO .

Après relance de Home Assistant, si on tente de créer une intégration de type TutoHACS, on obtient cette fois cette page de configuration :

![ConfigFlow vide](/images/config-flow-1.png?raw=true)

On est bien rentré dans le config flow et Home Assistant nous affiche le formulaire qui contient un champ "name".

Saisis un nom dans le champ et appuis sur "Valider". Tu dois voir les 2 logs suivants :

```log
2023-04-22 10:31:15.284 DEBUG (MainThread) [custom_components.tuto_hacs.config_flow] config_flow step user (1). 1er appel : pas de user_input -> on affiche le form user_form
...
2023-04-22 10:31:19.752 DEBUG (MainThread) [custom_components.tuto_hacs.config_flow] config_flow step user (2). On a reçu les valeurs: {'name': 'xxxxxx'}
```
Ca fonctionne bien, notre methode `async_step_user` a bien été appelée 2 fois, une fois sans valeur et une fois avec les valeurs saisies dans le formulaire.

> ![Tip](/images/tips.png?raw=true)
> 1. il n'est pas facile pour l'utilisateur de savoir ce qu'il doit saisir. On va ajouter juste en dessous des libellés pour notre formulaire pour y remédier,
> 2. l'appui sur "Valider" se termine avec une erreur. C'est parce-que notre méthode ne retourne rien lors du 2ème passage. On va y remédier aussi un peu en dessous. A ce stade, c'est normal.

## Ajout de libellés dans notre formulaire
On va ajouter des libellés à ce formulaire en ajoutant le fichier `strings.json` suivant à la racine de notre intégration :

```json
{
    "title": "TutoHACS",
    "config": {
        "flow_title": "TutoHACS configuration",
        "step": {
            "user": {
                "title": "Vos infos de connexion",
                "description": "Donnez vos infos de connexion",
                "data": {
                    "name": "Nom"
                },
                "data_description": {
                    "name": "Nom de l'intégration"
                }
            }
        }
    }
}
```

Tu donnes dans ce fichier les différents libellés qui accompagnent les formulaires :
- `title` est le nom de l'intégration,
- le bloc `config` contient les libellés du config flow,
- `flow_title` est le titre du flot de configuration,
- le bloc `step` contient les libellés des étapes de la configuration,
- le bloc `user` contient les libellés de l'étape `user`. Il y a la possibilité de mettre un titre et une description
- le bloc `data` contient les libellés des datas du formulaire `user`. 2 libellés sont possibles : le libellé de nos champs (ici `name`)
- le bloc `data_description` contient une description optionnelle pour chaque champ du formulaire. Dans notre exemple, il n'y a pas `name`

Ensuite on va créer une copie de ce fichier dans un sous-répertoire de notre intégration nommé `translations`. Ce répertoire doit contenir, les traductions du fichier `strings.json` dans toutes les langues supportées par notre intégration. La langue par défaut affichées à l'utilisateur sera sa langue configurée dans Home Assistant.

On doit donc avoir l'arborescence suivante :

![Arborescence](/images/arbo-tuto-hacs.png?raw=true)

Les fichiers `strings.json` et `translations/fr.json` sont identiques. Pour une vraie intégration, il est préférable que les libellés du fichier `strings.json` soient en anglais.

On redémarre Home Assistant et on tente de recréer l'intégration.

> ![Tip](/images/tips.png?raw=true)
> On constate que nos libellés **NE SONT PAS** pris en compte ! En effet, ils sont mis en cache dans le navigateur pour éviter de trop souvent interroger le serveur. Il va falloir vider ce cache (command-shift-suppr / "Images et fichiers en cache" sur Chrome sous Mac).

Vides le cache, recharges la page, crées l'intégration TutoHACS et cette fois tu dois avoir ça :

![Arborescence](/images/config-flow-2.png?raw=true)

![Attention](/images/attention.png?raw=true)
> **Attention :** en cas d'erreur de syntaxe dans un fichier de libellés, aucune erreur ne sera signalée nul part et seule la dernière version valide sera prise en compte. Combiné avec le cache navigateur qui reste aussi sur la dernière version valide, il est parfois très compliqué de comprendre pourquoi ses modifications ne pas prisent en compte.

# Comprendre les schémas

Dans le code de la fonction `async_step_user` ci-dessus, on a une ligne qui n'a pas été expliquée. Elle initialise le formulaire affiché dans l'étape `user` : 

```python
user_form = vol.Schema({vol.Required("name"): str})
```
Ce petit bout de code qui n'a l'air de rien mériterait à lui tout seul un tuto complet tellement il est puissant mais complexe et mal documenté. Je vais vous donner quelques clés pour comprendre comment il marche.

### Voluptuous

Les formulaires sont créés à partir du package Python [Voluptuous](https://github.com/alecthomas/voluptuous) qui permet de créer des schémas. Un schéma est une librairie de validation des données. Sa première intention est de valider syntaxiquement et sémantiquement des données reçues par un logiciel. On s'en sert ici pour décrire le formulaire qui est présenté à l'utilisateur et pour valider les données du formulaire saisies par l'utilisateur.

`vol.Schema` instancie une classe de type `Schema` du package `vol` qui est le nom de donné à l'import Voluptuous : `import voluptuous as vol`.

Ce constructeur prend en argument un objet json dont chaque attribut est un élément du formulaire. Exemple :

```python
vol.Schema({
    <premier element du formulaire>,
    <deuxième element du formulaire>,
    <troisième element du formulaire>
    ...
    })
```

Chaque élement de formulaire (chaque ligne), est lui même un objet qui dit si l'élément est faculatif ou obligatoire et on lui donne un nom :
```python
vol.Schema({
    vol.Required("nom du 1er champ obligatoire"): <Validator>,
    vol.Required("nom du 2ème champ obligatoire"): <Validator>,
    vol.Optional("nom du 3ème champ optionnel"): <Validator>>
    ...
    })
```

Dans le constructeur `Required` ou `Optional`, il est possible de donner une valeur par défaut au champ (si non saisi par l'utilisateur) ainsi qu'une valeur suggérée (valeur proposée à l'utilisateur mais qui ne sera pas retenue si l'utilisateur laisse le champ vide). La nuance valeur par défaut / valeur suggérée est subtile mais importante. Un champ qui a une valeur par défaut, **ne pas ne pas voir de valeur**, alors qu'en cas de valeur suggérée, l'utilisateur peut supprimer la valeur proposée et ainsi saisir la valeur vide.

```python
vol.Schema({
    vol.Required("nom du 1er champ obligatoire", default=12): <Validator>,
    vol.Required("nom du 2ème champ obligatoire", default="valeur par defaut"): <Validator>,
    vol.Optional("nom du 3ème champ optionnel", suggested_value="valeur suggérée"): <Validator>>
    ...
    })
```

Chaque champ a un type qu'il faut mettre à la place de <Validator> en fonction de ce qui est attendu par l'utilisateur. Un type est une classe proposée par le package Voluptuous lui-même. Par exemple :
- `str` : string,
- `Boolean` : booleen,
  
mais aussi des classes plus complexes :
- `Range` : une plage de valeur admises. Exemple : `vol.Range(min=-90, max=90)`,
- `Coerce(type)` : permet de convertir la valeur en un type (en argument). Exemple: `vol.Coerce(float)` pour traduire le champ en `float`,
- `Match(regexp)`: le champ est valide si l'expression régulière est vraie,
- `In([])` : la valeur doit être une des valeurs du tableau donné en argument.

mais aussi des Validator qui combinent d'autres validator :
- `All(list(Validator))` : est vrai si tous les `Validator` de la liste sont vérifiés. Exemple: `vol.All(vol.Coerce(float), vol.Range(min=-90, max=90)` 
- `Any(list(Validator))` : est vrai si au moins un `Validator` de la liste est vérifié. Exemple: `vol.Any("valeur1", "valeur2")`

Exemple un peu plus complet :
```python
vol.Schema({
    # On attend un entier
    vol.Required("nom du 1er champ obligatoire", default=12): vol.Coerce(int),
    # On attend une chaine qui peut être "valeur1" ou "valeur2"
    vol.Required("nom du 2ème champ obligatoire", default="valeur1"): vol.Any("valeur1", "valeur2"),
    # On attend une longitude (un float compris entre -90° et +90°)
    vol.Optional("nom du 3ème champ optionnel", suggested_value="1.112"): vol.All(vol.Coerce(float), vol.Range(min=-90, max=90))
    ...
    })
```

### Les Helpers Home Assistant

Pour aider dans la rédaction des formulaire Home Assistant fournit le package `homeassistant.helpers.config_validation` qui contient des Validator prêts à l'emploi. Par exemple :
- `byte` : un octet (définit comme `vol.All(vol.Coerce(int), vol.Range(min=0, max=255))`),
- `small_float` : un float entre 0 et 1,
- `positive_int` : un entier positif,
- `latitude` : un float entre -90 et +90,
- `time` : une valeur de temps,
- `date` : une date,
- etc

Il serait impossible de tous les listés ici donc il est conseillé de regarder ce qui est contenu dans le package lui-même.

### Les Selectors Home Assistant

Home Assistant permet d'utiliser les `Selector` comme des Validators. Pour rappel, les `Selector` sont listés [ici](https://www.home-assistant.io/docs/blueprint/selectors/).
On va donc pouvoir très facilement demander à Home Assistant de valider le champ si le champ correspond bien à une entité d'un domaine par exemple. Et dans ce cas, le formulaire affichera que les entités du ou des domaines.

Exemple pour sélectionner des entités :
```python
vol.Schema({
    # On attend un entity id du domaine climate
    vol.Required("climate_id"): selector.EntitySelector(
        selector.EntitySelectorConfig(domain=CLIMATE_DOMAIN),
    ),
    # On attend un entity id d'un switch ou d'un input_boolea,
    vol.Optional("switch_id"): selector.EntitySelector(
        selector.EntitySelectorConfig(
            domain=[SWITCH_DOMAIN, INPUT_BOOLEAN_DOMAIN]
        ),
    )
})
```

> ![Tip](/images/tips.png?raw=true)
> C'est très puissant mais vraiment très mal documenté. Souviens toi, en introduction de ces tutos, je disais qu'il fallait aller voir ce qu'on fait les autres (**Open Source !**), c'est primodial d'appliquer cette règle ici. Fork le repo de Home Assistant, parcours le code, fait des recherches dedans et tu vas apprendre plein de choses.

Pour les curieux, voici le schéma complet de la prmeière page de configuration du Versatile Thermostat :
```python
vol.Schema(  # pylint: disable=invalid-name
            {
                vol.Required(CONF_NAME): cv.string,
                vol.Required(
                    CONF_THERMOSTAT_TYPE, default=CONF_THERMOSTAT_SWITCH
                ): selector.SelectSelector(
                    selector.SelectSelectorConfig(
                        options=CONF_THERMOSTAT_TYPES, translation_key="thermostat_type"
                    )
                ),
                vol.Required(CONF_TEMP_SENSOR): selector.EntitySelector(
                    selector.EntitySelectorConfig(
                        domain=[SENSOR_DOMAIN, INPUT_NUMBER_DOMAIN]
                    ),
                ),
                vol.Required(CONF_EXTERNAL_TEMP_SENSOR): selector.EntitySelector(
                    selector.EntitySelectorConfig(
                        domain=[SENSOR_DOMAIN, INPUT_NUMBER_DOMAIN]
                    ),
                ),
                vol.Required(CONF_CYCLE_MIN, default=5): cv.positive_int,
                vol.Required(CONF_TEMP_MIN, default=7): vol.Coerce(float),
                vol.Required(CONF_TEMP_MAX, default=35): vol.Coerce(float),
                vol.Optional(CONF_DEVICE_POWER, default="1"): vol.Coerce(float),
                vol.Optional(CONF_USE_WINDOW_FEATURE, default=False): cv.boolean,
                vol.Optional(CONF_USE_MOTION_FEATURE, default=False): cv.boolean,
                vol.Optional(CONF_USE_POWER_FEATURE, default=False): cv.boolean,
                vol.Optional(CONF_USE_PRESENCE_FEATURE, default=False): cv.boolean,
            }
        )
```

Le résultat est le suivant :
![Versatile Thermostat](/images/vtherm-config-main.png?raw=true)

# Ajouter une deuxième étape
Pour ajouter une deuxième étape de configuration, on ajoute une méthode (une méthode par étape) et on l'appelle à la fin de la première étape. Le code ressemble à ça :

```python
    # Cette fois on est libre sur le nommage car ce n'est pas le point d'entrée
    async def async_step_2(self, user_input: dict | None = None) -> FlowResult:
        """Gestion de l'étape 2. Mêmes principes que l'étape user"""
        step2_form = vol.Schema(
            {
                # On attend un entity id du domaine sensor
                vol.Optional("sensor_id"): selector.EntitySelector(
                    selector.EntitySelectorConfig(domain=SENSOR_DOMAIN),
                )
            }
        )

        if user_input is None:
            _LOGGER.debug(
                "config_flow step2 (1). 1er appel : pas de user_input -> "
                "on affiche le form step2_form"
            )
            return self.async_show_form(step_id="2", data_schema=step2_form)

        # 2ème appel : il y a des user_input -> on stocke le résultat
        # TODO: utiliser les user_input
        _LOGGER.debug("config_flow step2 (2). On a reçu les valeurs: %s", user_input)
```

et en fin de la méthode `async_step_user` on va appeler le step 2 explicitement :

```python
    async def async_step_user(self, user_input: dict | None = None) -> FlowResult:
        ...
        if user_input is None:
            ...

        ...
        # On appelle le step 2 ici
        return await self.async_step_2()
```

On ajoute les imports qui manquent :
```python
from homeassistant.helpers import selector

import voluptuous as vol
import homeassistant.helpers.config_validation as cv
from homeassistant.components.sensor import DOMAIN as SENSOR_DOMAIN
```

On corriger les erreurs et on relance Home Assistant. Si on configure une intégration, on a bien maintenant notre page 2 de la configuration après avoir "Valider" la première page :

![Config flow page 2](/images/config-flow-3.png?raw=true)

On constate qu'il manque quelques traductions pour notre page 2. On les ajoute dans le fichiers `strings.json` qu'on recopie dans `translations/fr.json`, on redémarre, on vide le cache du navigateur et cette fois, on a la page suivante :

```json
{
    "title": "TutoHACS",
    "config": {
        ...
        "step": {
            "user": ...,
            "2": {
                "title": "Page 2",
                "description": "Une deuxième page de configuration",
                "data": {
                    "sensor_id": "Sensor"
                },
                "data_description": {
                    "sensor_id": "Le capteur permettant d'utiliser les selector dans ce beau tuto."
                }
            }
        }
    }
}
```

![Config flow page 2](/images/config-flow-4.png?raw=true)

> ![Tip](/images/tips.png?raw=true)
> 1. comme au-dessus, la validation de la 2ème page de configuration génère une erreur. A ce stade, c'est normal puisque notre méthode `async_step_2` ne renvoie rien,
> 2. dans notre première méthode, lorsqu'on appelle la 2ème, il est possible d'avoir de la logique pour router vers la page 2 ou tout autre page de notre choix. C'est comme ça qu'on va pouvoir avoir un parcours de paramétrage différent en fonction de la configuration que l'on veut atteindre.


# Créer une entité à partir d'une configuration
On a définit un parcours de configuration (le fameux `configFlow`) et maintenant il va falloir créer une entité en fin de ce parcours avec les éléments saisis.

Pour cela, il faut :
1. mémoriser les éléments saisis à chaque étape,
2. créer une entrée de configuration,
3. créer les entités avec l'ensemble des éléments saisis,
4. relier les entités à un appareil (device)

## Mémoriser les éléments saisis
Pour mémoriser les éléments saisis, il faut ajouter un réceptacle des saisies de l'utilisateur :

```python
class TutoHACSConfigFlow(ConfigFlow, domain=DOMAIN):
    """La classe qui implémente le config flow pour notre DOMAIN.
    Elle doit dériver de FlowHandler"""

    ...
    # le dictionnaire qui va recevoir tous les user_input. On le vide au démarrage
    _user_inputs: dict = {}
```

et la mémorisation dans le réceptacle des user_infos à chacune de nos étapes :
```python
    async def async_step_user(self, user_input: dict | None = None) -> FlowResult:
        ...
        if user_input is None:
            ...
        
        # On mémorise les user_input
        self._user_inputs.update(user_input)

    async def async_step_2(self, user_input: dict | None = None) -> FlowResult:
        ...
        if user_input is None:
            ...
            
        # On mémorise les user_input
        self._user_inputs.update(user_input)
        _LOGGER.info(
            "config_flow step2 (2). L'ensemble de la configuration est: %s",
            self._user_inputs,
        )
```

Si on relance en l'état et qu'on ajoute une intégration Tuto HACS, on obtient le log suivant après avoir validé la dernière étape :
```log
2023-04-22 16:43:03.973 DEBUG (MainThread) [custom_components.tuto_hacs.config_flow] config_flow step2 (2). On a reçu les valeurs: {'sensor_id': 'sensor.sun_next_setting'}
2023-04-22 16:43:03.973 INFO (MainThread) [custom_components.tuto_hacs.config_flow] config_flow step2 (2). L'ensemble de la configuration est: {'name': 'le nom', 'sensor_id': 'sensor.sun_next_setting'}
```

Notre objet `_user_inputs` contient bien les 2 champs des 2 formulaires de configuration.

## Créer une entrée de configuration (ConfigEntry)

L'entrée de configuration ou ConfigEntry, est ce qui permet de rendre les configurations des intégrations persistantes dans le temps. Après une relance de Home Assistant, toutes les entités sont créées à partir des configEntry sauvegardées sur le disque. C'est ce qui remplace le `configuration.yaml`.
Tu peux retrouver toutes les configEntry sur ton disque dur, dans le fichier `config/.storage/core.config_entries`.

Donc à la fin du configFlow, après avoir collecté tous les éléments de configuration, on va demander à Home Assistant de créer ou de mettre à jour un configEntry. Cela se fait très simplement avec le code suivant :

```python
self.async_create_entry(title="titre de l'entrée", data=self._user_inputs)
```

On va ajouter une constante CONF_NAME qui définit le nom de l'élément de config `name` au lieu de l'avoir en dur et on va utiliser l'élément de configuration `name` comme `title` pour ce configEntry. La deuxième méthode devient donc :

```python
    async def async_step_2(self, user_input: dict | None = None) -> FlowResult:
        """Gestion de l'étape 2. Mêmes principes que l'étape user"""
        step2_form = vol.Schema(
            {
                # On attend un entity id du domaine sensor
                vol.Optional("sensor_id"): selector.EntitySelector(
                    selector.EntitySelectorConfig(domain=SENSOR_DOMAIN),
                )
            }
        )

        if user_input is None:
            _LOGGER.debug(
                "config_flow step2 (1). 1er appel : pas de user_input -> "
                "on affiche le form step2_form"
            )
            return self.async_show_form(step_id="2", data_schema=step2_form)

        # 2ème appel : il y a des user_input -> on stocke le résultat
        _LOGGER.debug("config_flow step2 (2). On a reçu les valeurs: %s", user_input)

        # On mémorise les user_input
        self._user_inputs.update(user_input)
        _LOGGER.info(
            "config_flow step2 (2). L'ensemble de la configuration est: %s",
            self._user_inputs,
        )

        return self.async_create_entry(
            title=self._user_inputs[CONF_NAME], data=self._user_inputs
        )
```

On relance Home Assistant, on créé une intégration de type Tuto HACS et on doit avoir le résultat suivant :
![Config flow réussi](/images/config-flow-5.png?raw=true)

On constate aussi qu'une intégration a été créée :
![Config flow échec](/images/config-flow-6.png?raw=true)
mais elle est en échec.

Allons voir les logs et on constate ceci :

```log
2023-04-22 20:59:40.958 DEBUG (MainThread) [custom_components.tuto_hacs.config_flow] config_flow step2 (2). On a reçu les valeurs: {'sensor_id': 'sensor.sun_next_setting'}
2023-04-22 20:59:40.958 INFO (MainThread) [custom_components.tuto_hacs.config_flow] config_flow step2 (2). L'ensemble de la configuration est: {'name': 'La première', 'sensor_id': 'sensor.sun_next_setting'}
2023-04-22 20:59:40.961 ERROR (MainThread) [homeassistant.config_entries] Error setting up entry La première for tuto_hacs
Traceback (most recent call last):
  File "/home/vscode/.local/lib/python3.11/site-packages/homeassistant/config_entries.py", line 383, in async_setup
    result = await component.async_setup_entry(hass, self)
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^
AttributeError: module 'custom_components.tuto_hacs' has no attribute 'async_setup_entry'

```

La configuration s'est bien passée mais il manque à notre module `custom_components.tuto_hacs` une fonction `async_setup_entry`. Cette fonction va servir à transformer le configEntry en entité. On va voir comment faire ça dans le chapitre suivant.

Si on ouvre le fichier `config/.storage/core.config_entries` et qu'on recherche notre configuration, on doit la trouver et elle doit ressembler à ça :
```yaml
{
  "version": 1,
  "minor_version": 1,
  "key": "core.config_entries",
  "data": {
    "entries": [
        ...
      {
        "entry_id": "e88362b08cbf7774cb2ce61bbc952de3",
        "version": 1,
        "domain": "tuto_hacs",
        "title": "La première",
        "data": {
          "name": "La première",
          "sensor_id": "sensor.sun_next_setting"
        },
        "options": {},
        "pref_disable_new_entities": false,
        "pref_disable_polling": false,
        "source": "user",
        "unique_id": null,
        "disabled_by": null
      }
    ]
  }
}
```

Ce fichier contient bien notre configEntry avec nos paramètres, notamment le `title` qui prend la valeur de `data.name`.

## Créer une entité à partir d'une entrée de configuration
Au chargement ou lors d'une création d'une nouvelle configEntry, il faut indiquer à Home Assistant, comment instancier les entités et les appareils à partir de cette configEntry. Ca se fait simplement en créant une fonction `async_setup_entry` dans le fichier `__init__.py` :

```python
async def async_setup_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    """Creation des entités à partir d'une configEntry"""

    _LOGGER.debug(
        "Appel de async_setup_entry entry: entry_id='%s', data='%s'",
        entry.entry_id,
        entry.data,
    )

    hass.data.setdefault(DOMAIN, {})

    await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)

    return True
```

Ce code positionne le domaine par défaut comme étant notre domain, puis appel `hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)` qui comme son nom l'indique propage le configEntry à toutes les plateformes déclarées dans notre intégration.

Rappelles toi que PLATFORMS contient la liste des plateformes des entités créées par notre intégration (si une intégration doit créer un `sensor` et un `switch`, `PLAFORMS` contiendra `['sensor', 'switch']`). Pour le tuto, `PLATFORM`est initialisé comme suit dans le `const.py` : `PLATFORMS: list[Platform] = [Platform.SENSOR]`.

Donc, dans notre cas, l'effet de cette instruction est d'appeler la fonction `async_setup_entry` de notre `sensor.py`. Comme elle n'existe pas, il faut la créer aussi de la façon suivante :

```python
async def async_setup_entry(
    hass: HomeAssistant, entry: ConfigEntry, async_add_entities: AddEntitiesCallback
):
    """Configuration des entités sensor à partir de la configuration
    ConfigEntry passée en argument"""

    _LOGGER.debug("Calling async_setup_entry entry=%s", entry)

    entity1 = TutoHacsElapsedSecondEntity(hass, entry.data)
    entity2 = TutoHacsListenEntity(hass, entry.data, entity1)
    async_add_entities([entity1, entity2], True)

    # Add services
    platform = async_get_current_platform()
    platform.async_register_entity_service(
        SERVICE_RAZ_COMPTEUR,
        {vol.Optional("valeur_depart"): cv.positive_int},
        "service_raz_compteur",
    )
```
On constate que cette méthode est très proche de la méthode `async_setup_platform` qui initialise les entités à partir de la configuration de notre plateforme trouvée dans le fichier `configuration.yaml`. C'est bien normal puisque les deux font la même chose mais pas à partir de la même source de configuration.

Comme, on n'a pas mis d'élément de configuration donnant le `deviceId`, il va falloir qu'on modifie la façon dont ce deviceId est initalisé. Cf. ci-dessous pour le raccordement des entités à un device.

```python

class TutoHacsElapsedSecondEntity(SensorEntity):
    ...
    def __init__(
        ...
        self._device_id = self._attr_name = entry_infos[CONF_NAME]
    ...

class TutoHacsListenEntity(SensorEntity):
    ...
    def __init__(
        ...
        # On lui donne un nom et un unique_id différent
        self._device_id = entry_infos.get(CONF_NAME)
```

On peut relancer Home Assistant (après corriger les éventuelles erreurs...) et on obtient 2 entités supplémentaires ([ici](http://localhost:9123/config/entities)) :
![Config flow échec](/images/entite-config-flow.png?raw=true)

On constate que les entités précédemment créées par le fichier `configuration.yaml` sont aussi présentes. Il est possible en effet de configurer les entités par les 2 moyens en même temps. Ca ne sert à priori à rien donc on va faire un peu de ménage et supprimer la configuration du `configuration.yaml` :

On supprime tout le bloc :
```yaml
sensor:
  - platform: tuto_hacs
  ...
```

On peut aussi supprimer les fonctions `async_setup_platform` de `__init__.py` et `sensor.py` puisqu'elles ne servent que pour les configurations du `configuration.yaml`.

Après arrêt/relance de Home Assistant, on a plus que nos nouvelles entités qui sont actives.

## Relier les entités à un appareil (device)
Un appareil peut être vu comme un regroupement d'entités chacune exposant une caractéristique d'un même appareil.

J'ai mis longtemps à comprendre qu'il faut savoir qu'un appareil n'a pas de code, ni de déclaration. Il est simplement créé automatiquement lorsqu'on déclare une entité et qu'on l'a relie à un appareil.

Pour terminer cette partie, on va relier nos 2 entités à un même appareil (device). Pour cela, c'est très simple, il suffit de déclarer dans la classe de chacune des entités à quel device elle appartient.

Ca se fait en ajoutant le code suivant dans notre classe d'entité :

```python
from homeassistant.helpers.entity import DeviceInfo, DeviceEntryType

...
class TutoHacsElapsedSecondEntity(SensorEntity):
    ...
    @property
    def device_info(self) -> DeviceInfo:
        """Return the device info."""
        return DeviceInfo(
            entry_type=DeviceEntryType.SERVICE,
            identifiers={(DOMAIN, self._device_id)},
            name=self._device_id,
            manufacturer=DEVICE_MANUFACTURER,
            model=DOMAIN,
        )

class TutoHacsListenEntity(SensorEntity):
    ...
    @property
    def device_info(self) -> DeviceInfo:
        """Return the device info."""
        return DeviceInfo(
            entry_type=DeviceEntryType.SERVICE,
            identifiers={(DOMAIN, self._device_id)},
            name=self._device_id,
            manufacturer=DEVICE_MANUFACTURER,
            model=DOMAIN,
        )
```
Si on avait d'autres entités d'autres domaines, on ferait la même chose pour les relier aussi.

Le manufacturer est une constante définie dans le `const.py` lors du tuto3.

On redémarre le tout et on constate dans la liste des appareils ([ici](http://localhost:9123/config/devices/dashboard)), un nouvel appareil nommé "La première" (le `name` donné à l'intégration), qui contient 2 entités :
![Appareil1](/images/appareil-1.png?raw=true)

Cliques sur l'appareil pour voir ses entités :
![Appareil2](/images/appareil-2.png?raw=true)

> ![Tip](/images/tips.png?raw=true)
> Il est possible de créer autant d'intégration que l'on veut. Il suffit pour cela de cliquer sur "Ajouter une intégration" et de donner les éléments de configuration.


# Modifier une configuration

Il ne nous reste plus qu'à pouvoir modifier la configuration d'une intégration et on aura fait le tour du configFlow. En imaginant qu'on veuille modifier une option sur une de nos intégrations, il serait dommage d'être obligé de la détruire et de la récréer.

Pour faire ça, on va ajouter un menu "Configurer" dans notre intégration qui permettra de dérouler un parcours de configuration.
