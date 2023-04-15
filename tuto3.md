## L'objectif de cet article est d'enrichir fonctionnellement notre entité
Il s'inscrit dans la suite des articles dont le sommaire est [ici](/README.md).

> ![Tip](/images/tips.png?raw=true)
> Les fichiers sources complets en version finales sont en fin d'article. Cf [Fichiers sources du tuto](#fichiers-sources-du-tuto)

# Pre-requis
Avoir déroulé avec succès les deux premiers articles [tuto1](/tuto1.md) et [tuto2](/tuto2.md). Vous devez donc avoir une entité avec un état qui est une mesure en secondes.

# Les points abordés
Dans cet article, tu vas apprendre à :
1. utiliser l'objet `hass`,
2. déclencher périodiquement la mise à jour d'une entité,
3. mettre à jour l'état de l'entité,
4. publier et recevoir des évènements,
5. s'abonner aux changements d'une autre entité
6. implémenter un service

On va couvrir l'ensemble des flux décrit dans Home Assistant Core Architecture ([ici](https://developers.home-assistant.io/docs/dev_101_hass)) :
![ha core architecture](/images/ha-core-architecture.png)

# L'objet `hass`

Si il y a bien un objet important dans le developpement Home Assistant, c'est l'objet `hass`. De type `HomeAssistant`, on l'a déjà rencontré dans le tuto2 sans l'expliquer, on va le faire ici.
Cet objet est partout. **Il représente l'instance Home Assistant** sur laquelle l'entité est configurée et permet d'accéder à toutes les objets manipulés par Home Assistant : les configurations, les états, les entités, les bus d'évènements, ...

**Comme il est indispensable**, on va le mémoriser dès qu'on peut - à la construction de l'entité - dans les attributs de l'entité ; en variable privée  et donc commençant par un `_` selon la règle de nommage Python.
Pour cela, on définit un attribut `self._hass` dans la fonction d'init de l'entité. Et on stocke l'objet `hass` dedans :

```python
class TutoHacsElapsedSecondEntity(SensorEntity):
...
    _hass: HomeAssistant

    def __init__(
        self,
        hass: HomeAssistant,  # pylint: disable=unused-argument
        entry_infos,  # pylint: disable=unused-argument
    ) -> None:
        """Initisalisation de notre entité"""
        ...
        self._hass = hass
```
Plus d'informations sur cet objet [ici](https://developers.home-assistant.io/docs/dev_101_hass/).

# Déclencher périodiquement la mise à jour d'une entité

Pour ce faire, il faut faire générer par Home Assistant un évènement basé sur le temps et capter cet évènement. Lors de la captation de cet évènement, on incrémentera le compteur en secondes de l'entité.

Dans notre sensor, on a besoin d'une fonction spéciale qui est appelée par Home Assistant lorsque l'entité a été prise en compte. C'est la méthode `async_added_to_hass`qui est définie dans la classe de base de toutes les entités et qu'on va surcharger pour ajouter notre comportement souhaité. On marque cette méthode avec l'annotation @callback pour signifier qu'on surcharge une méthode de la classe de base. Même si ce n'est pas indispensable, ca donne des indications au lecteur.

```python
from datetime import timedelta
from homeassistant.core import HomeAssistant, callback
from homeassistant.helpers.event import async_track_time_interval

class TutoHacsElapsedSecondEntity(SensorEntity):
    ...
    @callback
    async def async_added_to_hass(self):
        """Ce callback est appelé lorsque l'entité est ajoutée à HA """

        # Arme le timer
        timer_cancel = async_track_time_interval(
            self._hass,
            self.methode_a_appeler,
            interval=timedelta(seconds=1),
        )
        # desarme le timer lors de la destruction de l'entité
        self.async_on_remove(timer_cancel)
        
```

Le fonction de la méthode `async_added_to_hass` est le suivant :
1. on appelle la fonction helper `async_track_time_interval` qui programme un timer périodique d'interval égal 1 seconde dans l'exemple,
2. on donne à ce helper l'objet `hass`, la méthode de notre entité qui sera appelée à chaque échéance du timer et l'interval,
3. cette fonction retourne une fonction qui doit être appelée pour stopper le timer,
4. on passe cette fonction d'annulation à la méthode de la classe `Entity` nommée `async_on_remove` qui appelle toutes les méthodes qu'on lui aura donné lors de la destruction de l'entité. Si on ne le fait, le timer continuera de poster des évènements dans le vide.

Il ne nous reste plus qu'à créer la méthode qu'on veut appeler toutes les secondes :

```python

class TutoHacsElapsedSecondEntity(SensorEntity):
    ...
    @callback
    async def incremente_secondes(self, _):
        """Cette méthode va être appelée toutes les secondes"""
        _LOGGER.info("Appel de incremente_secondes à %s", datetime.now())
```

Testons pour voir si notre méthode est bien appelée toutes les secondes. Command + Shift + P / Taches...
Si on regarde les logs, on voit bien que :
```log
2023-04-10 21:20:06.027 INFO (MainThread) [custom_components.tuto_hacs.sensor] Appel de incremente_secondes à 2023-04-10 21:20:06.027870
2023-04-10 21:20:07.034 INFO (MainThread) [custom_components.tuto_hacs.sensor] Appel de incremente_secondes à 2023-04-10 21:20:07.034205
2023-04-10 21:20:08.038 INFO (MainThread) [custom_components.tuto_hacs.sensor] Appel de incremente_secondes à 2023-04-10 21:20:08.038764
2023-04-10 21:20:09.040 INFO (MainThread) [custom_components.tuto_hacs.sensor] Appel de incremente_secondes à 2023-04-10 21:20:09.040590
2023-04-10 21:20:10.043 INFO (MainThread) [custom_components.tuto_hacs.sensor] Appel de incremente_secondes à 2023-04-10 21:20:10.043220
...
```

Notre méthode `incremente_secondes` est bien appelée toutes les secondes.

> ![Tip](/images/tips.png?raw=true) Le second argument nommé `_` indique qu'on ne veut pas tenir compte du 2nd argument. Normalement, l'évèvement est passé en 2nd argument mais comme on ne s'en sert pas ici, on remplace le deuxième argument par `_`.

# Mettre à jour l'état de l'entité

Cela se fait en appelant la méthode `async_write_ha_state` définie dans la classe de base `Entity`. On va remplacer notre méthode `incremente_secondes` par celle-ci :

```python

    @callback
    async def incremente_secondes(self, _):
        """Cette méthode va être appelée toutes les secondes"""
        _LOGGER.info("Appel de incremente_secondes à %s", datetime.now())

        # On incrémente la valeur de notre etat
        self._attr_native_value += 1

        # On sauvegarde le nouvel état
        self.async_write_ha_state()
```

On en profite pour initialiser la valeur du compteur à 0 et non pas 12 dans la méthode `__init__`.

On redémarre, on voit toujours les logs bouger toutes les secondes et si on regarde sur le web ([ici](http://localhost:9123/lovelace/0)), on voit bien notre compteur évoluer toutes les secondes :

> ![Compteur](/images/compteur.png?raw=true)

# Publier et recevoir des évènements
Le coeur de Home Assistant est basé sur un bus d'évènements. Il est fondamentale de savoir s'y interfacer puisque c'est par là que va passer toutes communications entre les différents composants de Home Assistant.

Pour ce tuto, on va créer une deuxième entité dans notre intégration qui écoute des évènements publiés par la première.

## Publication d'un évènement

On va modifier notre méthode `incremente_secondes` de notre entité vedette pour envoyer un évènement toutes les 5 secondes :

```python
@callback
    async def incremente_secondes(self, _):
        """Cette méthode va être appelée toutes les secondes"""
        _LOGGER.info("Appel de incremente_secondes à %s", datetime.now())

        # On incrémente la valeur de notre etat
        self._attr_native_value += 1

        # On sauvegarde le nouvel état
        self.async_write_ha_state()

        # Toutes les 5 secondes on envoie un event
        if self._attr_native_value % 5 == 0:
            self._hass.bus.fire(
                "event_changement_etat_TutoHacsElapsedSecondEntity",
                {"nb_secondes": self._attr_native_value},
            )
```

Ca tient en une ligne : `self._hass.bus.fire` qui prend en argument, le type d'évènement et un json qui contient des infos sur l'évèement.

On arrête et on relance Home Assistant. Si on contrôle dans le web / outils de développement / Evènements et qu'on s'abonne à l'évènement `event_changement_etat_TutoHacsElapsedSecondEntity`, on constate ça :

> ![Evènements](/images/evenements.png?raw=true)

Toutes les 5 secondes, on a bien un évènement généré qui contient dans ses data, l'attribut `nb_secondes` qui s'incrémente bien de 5 en 5.

> ![Tips](/images/tips.png?raw=true)
> Si on s'abonne aux évènements de type `state_changed` on voit que toutes les secondes, notre changement d'état fait l'objet d'un évènement. La ligne `self.async_write_ha_state()` génère un évènement de type `state_changed` qui contient les informations suivantes :
```yaml
event_type: state_changed
data:
  entity_id: sensor.tuto_hacs_entite_3
  old_state:
    entity_id: sensor.tuto_hacs_entite_3
    state: "639"
    attributes:
      state_class: measurement
      unit_of_measurement: s
      device_class: duration
      icon: mdi:timer-play
      friendly_name: Tuto HACS Entité
    last_changed: "2023-04-11T17:13:57.377736+00:00"
    last_updated: "2023-04-11T17:13:57.377736+00:00"
    context:
      id: 01GXRMYNC17VXC1H17M771R38E
      parent_id: null
      user_id: null
  new_state:
    entity_id: sensor.tuto_hacs_entite_3
    state: "640"
    attributes:
      state_class: measurement
      unit_of_measurement: s
      device_class: duration
      icon: mdi:timer-play
      friendly_name: Tuto HACS Entité
    last_changed: "2023-04-11T17:13:58.383054+00:00"
    last_updated: "2023-04-11T17:13:58.383054+00:00"
    context:
      id: 01GXRMYPBF28WR4T481B5K0QQ9
      parent_id: null
      user_id: null
origin: LOCAL
time_fired: "2023-04-11T17:13:58.383054+00:00"
context:
  id: 01GXRMYPBF28WR4T481B5K0QQ9
  parent_id: null
  user_id: null
```
> on va pouvoir donc très facilement récupérer les changements d'état des entités : il suffira de s'abonner aux évènements `state_changed`.


## Réception des évènements
On va créer une deuxième entité qui va écouter les évènements de la première et va stocker dans son état la date du dernier évènement reçu (ca sert à rien mais pourquoi pas après tout).

> Si tu es motivé, tu peux le faire sous la forme d'un exercice. A part la réception d'un évènement, tout le reste à déjà été vu dans le [tuto2](/tuto2.md)

Voici ce qu'il faut faire :
1. ajouter une classe pour notre deuxième entité (dans `sensor.py`) et se mettre en écoute des évènements de la première entité,
2. faire en sorte qu'elle soit instanciée au démarrage au setup de la plate-forme (dans `async_setup_platform` de `sensor.py`),
3. interpréter les évènements reçus.

### Ajouter une classe pour notre 2ème sensor

`sensor.py` :


```python
...
class TutoHacsElapsedSecondEntity(SensorEntity):
  ...

class TutoHacsListenEntity(SensorEntity):
    """La classe de l'entité TutoHacs qui écoute la première"""

    _hass: HomeAssistant
    _entity_to_listen: TutoHacsElapsedSecondEntity

    def __init__(
        self,
        hass: HomeAssistant,  # pylint: disable=unused-argument
        entry_infos,  # pylint: disable=unused-argument
        entity_to_listen: TutoHacsElapsedSecondEntity,  # L'entité qu'on veut écouter
    ) -> None:
        """Initisalisation de notre entité"""
        self._hass = hass
        self._attr_has_entity_name = True
        self._device_id = entry_infos.get(CONF_DEVICE_ID)
        # On lui donne un nom et un unique_id différent
        self._attr_name = entry_infos.get(CONF_NAME) + " Ecouteur"
        self._attr_unique_id = self._device_id + "_ecouteur"
        # Pas de valeur tant qu'on n'a pas reçu
        self._attr_native_value = None
        self._entity_to_listen = entity_to_listen

    @property
    def should_poll(self) -> bool:
        """Pas de polling pour mettre à jour l'état"""
        return False

    @property
    def icon(self) -> str | None:
        return "mdi:timer-settings-outline"

    @property
    def device_class(self) -> SensorDeviceClass | None:
        """Cette entité"""
        return SensorDeviceClass.TIMESTAMP

    @property
    def device_info(self) -> DeviceInfo:
        """Donne le lien avec le device. Non utilisé jusqu'au tuto 4"""
        return DeviceInfo(
            entry_type=DeviceEntryType.SERVICE,
            identifiers={(DOMAIN, self._device_id)},
            name=self._attr_name,
            manufacturer=DEVICE_MANUFACTURER,
            model=DOMAIN,
        )

    @callback
    async def async_added_to_hass(self):
        """Ce callback est appelé lorsque l'entité est ajoutée à HA"""

        # Arme l'écoute de la première entité
        listener_cancel = async_track_state_change_event(
            self.hass,
            [self._entity_to_listen.entity_id],
            self._on_event,
        )
        # desarme le timer lors de la destruction de l'entité
        self.async_on_remove(listener_cancel)
```

Ca ressemble beaucoup à la classe créée dans le [tuto2](/tuto2.md), mais il y a quelques substilités :
1. dans `__init__` il faut lui donner un name (resp. unique_id) qui sera unique. Pour cela, on concatène `Ecouteur` (resp. `_ecouteur`) au `name` (resp. `unique_id`)
2. la device class est positionnée à `TIMESTAMP` et non pas `DURATION` car notre entité représente une date absolue et pas une durée,
3. il n'y a pas de `state_class` ni de `native_unit_of_mesurement` puisque l'état de notre entité n'est pas une mesure à proprement parler,
4. dans la méthode `async_added_to_hass` (qui est appelé par HA quand l'entité est ajoutée), on utilise le Helper `async_track_state_change_event` qui permet de se mettre en écoute des changements d'état dont l'entity_id est donné en 2nd paramètre (dans un tableau car on peut en écouter plusieurs). C'est ici que se passe le lien entre les 2 entités : celle qui émet des changements d'état et notre deuxième qui les écoute.
5. comme vu dans le [tuto2](/tuto2.md), lorsqu'on se met en écoute d'un évènement il faut se désabonner lorsque l'entité est supprimée, sinon on continue de recevoir les évents alors que l'entité a été supprimée de HA. Ca se fait avec l'appel à `async_on_remove` qui prend en paramètre le retour de `async_track_state_change_event`. La méthode appelée à chaque changement d'état reçu sera `_on_event` qu'on verra ci-dessous.

### Instancier cette classe au démarrage de la plate-forme
Pour cela, il faut modifier légèrement la fonction `async_setup_platform` (cf. [tuto2](/tuto2.md) au besoin) et ajouter le code suivant :

```python
async def async_setup_platform(
    hass: HomeAssistant,
    entry: ConfigEntry,
    async_add_entities: AddEntitiesCallback,
    discovery_info=None,  # pylint: disable=unused-argument
):
    """Configuration de la plate-forme tuto_hacs à partir de la configuration
    trouvée dans configuration.yaml"""

    _LOGGER.debug("Calling async_setup_entry entry=%s", entry)

    entity1 = TutoHacsElapsedSecondEntity(hass, entry)
    entity2 = TutoHacsListenEntity(hass, entry, entity1)
    async_add_entities([entity1, entity2], True)
```
Tu remarques qu'on passe à notre deuxième classe la première entité créée. C'est utilisé dans `async_added_to_hass` pour se mettre en écoute de ses changements d'état.

### Interpréter les évènements reçus
Cela va se faire dans la méthode `_on_event` qu'il faut ajouter à notre 2nd classe. Le code ressemble à ça :

```python
    @callback
    async def _on_event(self, event: Event):
        """Cette méthode va être appelée à chaque fois que l'entité
        "entity_to_listen" publie un changement d'état"""

        _LOGGER.info("Appel de _on_event à %s avec l'event %s", datetime.now(), event)

        new_state: State = event.data.get("new_state")
        # old_state: State = event.data.get("old_state")

        if new_state is None or new_state.state in (STATE_UNAVAILABLE, STATE_UNKNOWN):
            _LOGGER.warning("Pas d'état disponible. Evenement ignoré")
            return

        # On recherche la date de l'event pour la stocker dans notre état
        self._attr_native_value = new_state.last_changed

        # On sauvegarde le nouvel état
        self.async_write_ha_state()
```

Ce code déroule les étapes suivantes :
1. reçoit un `event` du type `Event` en argument. C'est une core class qu'il est important de connaitre. Un `Event` contient l'`entity_id` de l'émetteur, un champ `data` de type dictonnary qui contient 2 attributs : `new_state` et `old_state` avec respectivement le nouvel état et l'ancien état de l'entité,
2. on commence par récupérer ses ancien et nouveau état : `new_state: State = event.data.get("new_state")`. J'ai mis en commentaire le code qui permet de récupérer l'ancien état (non utilisé ici)
3. l'objet `new_state` est de type `State` qui est aussi une core class à connaitre. Elle contient : l'état dans le champ `state`, les `state_class`, `unit_of_mesurement`, `device_class` de cet état, et 2 timestamp `last_changed` (l'horodatage du changement d'état) et `last_updated` (l'horodatage de la dernière mise à jour). Ces 2 horodatages peuvent être différent dans le cas d'une entité avec polling. Dans ce cas, la date de remontée de l'état (date du poll) n'est pas forcément la date du changement d'état. Si on poll toutes les minutes, il peut y avoir jusqu'à une minute d'écard entre le changement d'état effectif et la date de sa remontée. Dans notre cas et dans la plupart des cas, on préfère utiliser la date du changement d'état effectif et donc `last_changed`
4. on vérifie que l'état est bien positionné avec cette ligne : `if new_state is None or new_state.state in (STATE_UNAVAILABLE, STATE_UNKNOWN)`. Il faut qu'il y ait un `new_state` et que le `state` de ce `new_state` ne soit pas 'Unavailable" ni "Unknown". Si c'est le cas, on arrête ici et on ne met pas à jour notre état,
5. si il y a bien un état valide, on met à jour notre propre état avec l'horodate du dernier changement : `self._attr_native_value = new_state.last_changed`
6. on sauvegarde notre nouvel état avec `self.async_write_ha_state()`

Il faudra ajouter les quelques imports suivants pour que cela fonctionne (en début de fichier) :

```python
from datetime import datetime, timedelta

from homeassistant.const import UnitOfTime, STATE_UNAVAILABLE, STATE_UNKNOWN
from homeassistant.core import HomeAssistant, callback, Event, State
```

> ![Tip](/images/tips.png?raw=true)
> Tu remarques que je **type mes variables ou attributs** de classe quasi systématiquement. Ca se fait avec le `: State` par exemple. Ce n'est pas obligatoire avec Python qui est un langage interprété et non typé par défaut, mais ça a plusieurs vertues de le faire quand même :
> - VSC va vous signaler tout de suite une erreur **si les types ne correspondent pas**,
> - En passant la souris sur le type tu vas avoir **la description de la classe**,
> - en maintenant 'Command (sur Mac)' enfoncée, tu vas pouvoir **ouvrir le code de la classe** et éventuellement **mettre des points d'arrêt** dans cette classe, même si c'est une classe système ou Home Assistant,
> - tu avoir accès à **l'auto-complétion**. Dès que tu vas taper le '.' après ton attribut, la liste des méthodes et attributs utilisables s'affichent avec leur arguments.
>
![autocompletion](/images/autocompletion.png?raw=true)
>
> Bref, je le conseille vivement, ça simplifie beaucoup la phase de développement.


### Démarrage de Home Assistant

Vérifies qu'il n'y a pas d'erreur :
![no probleme](/images/compilation-no-probleme.png?raw=true)
Relance Home Assistant (Command + Shift + P) et regarde les logs. Tu dois voir quelque-chose comme ça :

```log
2023-04-15 08:10:36.889 INFO (MainThread) [custom_components.tuto_hacs.sensor] Appel de incremente_secondes à 2023-04-15 08:10:36.889856
2023-04-15 08:10:36.896 INFO (MainThread) [custom_components.tuto_hacs.sensor] Appel de _on_event à 2023-04-15 08:10:36.896349 avec l'event <Event state_changed[L]: entity_id=sensor.tuto_hacs_entite_3, old_state=<state sensor.tuto_hacs_entite_3=2243; state_class=measurement, unit_of_measurement=s, device_class=duration, icon=mdi:timer-play, friendly_name=Tuto HACS Entité @ 2023-04-15T10:10:35.889760+02:00>, new_state=<state sensor.tuto_hacs_entite_3=2244; state_class=measurement, unit_of_measurement=s, device_class=duration, icon=mdi:timer-play, friendly_name=Tuto HACS Entité @ 2023-04-15T10:10:36.890148+02:00>>
```

Ce log se répête toutes les secondes, puisque l'état de la première entité se met à jour toutes les secondes.

Si on regarde dans l'"Outil de developpement / Etat" ([ici](http://localhost:9123/developer-tools/state)) de notre interface web HA et que l'on cherche "ecouteur", on voit bien notre deuxième entité avec comme état l'horodatage qui change toutes les secondes :
![entité écouteur](/images/entite-ecouteur.png?raw=true)

Le dashboard aperçu (ici) affiche aussi nos 2 entités :
![deux entités](/images/deux-entites.png?raw=true)

# Implémenter un service
Un service est un point d'accès à notre intégration appelable depuis l'extérieur (une autre intégration, une automatisation, ...).

Créer un service se fait très simplement avec les étapes suivantes :
1. déclaration de notre service dans un fichier de description `services.yaml`,
2. enregistrement du service au setup de la plate-forme,
3. implémentation du service à proprement parler.

On va implémenter un service qui permet de remettre à zéro notre compteur pour l'exemple.

## Déclaration du service
Home Assistant découvre les services exposés par les intégrations grace au fichier `services.yaml`. Pour notre exemple, il va ressembler à ça :
```yaml
raz_compteur:
  name: Raz compteur
  description: Remet à zéro le compteur de temps et optionnellement donne la valeur de départ
  target:
    entity:
      integration: tuto_hacs
  fields:
    valeur_depart:
      name: Valeur départ
      description: La valeur de départ du compteur
      required: false
      advanced: false
      example: "10"
      default: "0"
      selector:
        number:
          min: 0
          max: 900
          step: 1
          mode: slider
```

Ce fichier contient :
1. la description d'un service nommé `raz_compteur`, avec un nom et une description pour les utilisateurs à travers "Outil de developpement / Service",
2. la cible `target`. On indique ici que toutes les entités de l'intégration `tuto_hacs` peuvent être ciblées,
3. une description des paramètres dans la structure `fields`:
   1. le nom du paramètre: `valeur_depart` avec un nom pour l'utilisateur et une description,
   2. le caractère obligatoire ou pas du paramètre (ici il est facultatif),
   3. est-ce qu'il apparait seulement en mode 'Avancé'. Ici non puisqu'on veut le voir tout le temps,
   4. un exemple de valeur,
   5. la valeur par défaut,
   6. et un `selector` qui permet à l'utilisateur de choisir la valeur qu'il veut. Ici, on utilise un sélector de type `number` avec une valeur minimale de 0, une valeur maximale de 900, un pas de 1 et l'utilisateur pourra choisir la valeur sur un slider. On verra le rendu un peu en-dessous.

Home Assistant propose un nombre de sélecteurs très impressionant et vraiment très bien foutu. Tu trouveras la liste [ici](https://www.home-assistant.io/docs/blueprint/selectors/).

## enregistrement du service au setup
Lors du setup de notre intégration, on doit enregistrer notre service et dire quelle méthode doit être appelée lorsque le service est invoqué. Ca se fait dans la fonction `async_setup_platform` à l'aide du code suivant :
```python
async def async_setup_platform(
    hass: HomeAssistant,
    entry: ConfigEntry,
    async_add_entities: AddEntitiesCallback,
    discovery_info=None,  # pylint: disable=unused-argument
):
  ...

    # Add services
    platform = async_get_current_platform()
    platform.async_register_entity_service(
        SERVICE_RAZ_COMPTEUR,
        {vol.Optional("valeur_depart"): cv.positive_int},
        "service_raz_compteur",
    )
```

1. on récupère notre plate-forme courante avec : `async_get_current_platform()`,
2. on enregistre notre service avec l'appel à : `platform.async_register_entity_service`. Cet appel prend 3 paramètres :
  - le nom du service que l'on a mis dans la constante `SERVICE_RAZ_COMPTEUR` définie dans notre `const.py`. Elle doit être égale au nom du service dans le `services.yaml` à savoir : `raz_compteur` dans l'exemple,
  - une structure `Voluptuous` qui en charge de vérifier les paramètres donnés lors de l'appel du service. Voir ci-dessous,
  - le nom de la méthode sur notre classe d'entité qui sera appelée lorsque le service est invoqué. Ici, on appellera la méthode : `TutoHacsElapsedSecondEntity.service_raz_compteur`.

On va avoir besoin des imports suivants :

```python
import voluptuous as vol
import homeassistant.helpers.config_validation as cv

from homeassistant.helpers.entity_platform import (
    ...
    async_get_current_platform,
)

from .const import (
    ...
    SERVICE_RAZ_COMPTEUR,
)
```

et on va définir notre constante `SERVICE_RAZ_COMPTEUR` dans notre `const.py` :
```python
...
SERVICE_RAZ_COMPTEUR = "raz_compteur"
```

### Voluptuous
Cette partie est complexe et sera abordée beaucoup plus en détail avec le [tuto4](/tuto4.md). Pour l'instant, on va juste donner une structure qui liste les paramètres "valeur_depart", donne son caractère facultatif (`vol.Optional`) et indique qu'on attend un entier positif (`cv.positive_int`).

C'est une des parties les moins bien documentée à la fois dans Home Assistant mais aussi dans le package `voluptuous` lui-même donc je ne rentre pas plus dans le détail dans ce tuto.

## Redémarrage de Home Assistant
Tu commences à en avoir l'habitude maintenant mais je le répette pour la dernière fois.
On vérifie qu'on n'a pas d'erreur dans l'onglet "PROBLEMES" de VSC et on (re)démarre avec 'Command + Shift + P'.

On ne doit pas voir d'erreur les logs, seulement notre compteur qui tourne toutes les secondes.

Vas ensuite dans les "Outils de développement / Services" ([ici](http://localhost:9123/developer-tools/service)) et tape 'tuto' dans la boite de recherche des services. Tu dois voir notre service :
![Service raz_compteur](/images/service-raz-compteur.png?raw=true)

Sélectionne le et tu vas voir apparaitre l'interface qui permet de configurer l'appel du service :
![Service raz_compteur](/images/service-raz-compteur-config.png?raw=true)

Tu peux :
1. sélectionner des entités et tu constates qu'il n'y a bien que les entités de notre intégration qui sont présentées,
2. modifier la "Valeur départ" avec un slider ou directement en tapant la valeur,
3. ne pas passer de valeur de départ en la décochant. C'est dû au caractère facultatif de notre paramètre "valeur_depart".

L'appel du service provoque une erreur de type :
```log
AttributeError: 'TutoHacsElapsedSecondEntity' object has no attribute 'service_raz_compteur'
```
puisque en effet notre classe `TutoHacsElapsedSecondEntity` n'a pas encore de méthode `service_raz_compteur`. On va y remedier de suite.

## Implémentation du service
Pour cela, c'est très simple, il suffit d'ajouter une méthode `service_raz_compteur` à notre classe `TutoHacsElapsedSecondEntity` :

```python
      async def service_raz_compteur(self, valeur_depart: int):
        """Appelée lors de l'invocation du service 'raz_compteur'
        Elle prend en argument la 'valeur_depart' qui est
        construite à partir du paramètre 'valeur_depart'
        """
        _LOGGER.info(
            "Appel du service service_raz_compteur valeur_depart: %d", valeur_depart
        )
        self._attr_native_value = valeur_depart if valeur_depart is not None else 0

        # On sauvegarde le nouvel état
        self.async_write_ha_state()
```

1. Notre méthode est appelée avec `valeur_depart` en argument typé en `int` dans l'exemple.
2. On loggue l'appel (toujours utile en cas de debug).
3. On affecte notre état avec la valeur passée ou avec 0 si elle est absente (`None`). Python permet de faire tout ça en une seule ligne avec la forme `_attr_native_value = valeur_depart if valeur_depart is not None else 0` qui se lit très bien.
4. On sauvegarde notre nouvel état avec `self.async_write_ha_state()`

Redémarres Home Assistant et vérifies que cet fois l'appel du service se passe bien. Tu dois avoir une coche verte.

Si tu regardes le nouvel état de ton entité ([ici](http://localhost:9123/developer-tools/state) ou [ici](http://localhost:9123/lovelace/0)), tu dois constater le redémarrage du compteur à la valeur spécifiée.

> ![Tip](/images/tips.png?raw=true)
> - si tu appelles le service sur la deuxième entité, tu vas avoir une erreur car nous n'avons définit le service sur la classe de cette entité. Pour éviter ça :
> 1. on peut implémenter le service dans la classe `TutoHacsListenEntity` mais ça fait un appel qui ne sert à rien,
> 2. ou limiter dans les entités `target`. On peut utiliser le paramètre `device_class` du selector à `duration` puisque seule la première classe à cette `device_class`. On a alors une configuration `target` qui ressemble à ça :
>
```yaml
  target:
    entity:
      integration: tuto_hacs
      device_class: duration
  ```
> Après arrêt / relance, tu ne peux plus sélectionner que la première entité dans l'interface de lancement du service.

# Intégrer notre entités dans l'écosystème Home Assistant
Les services sont très utiles pour intégrer notre intégration dans l'écosystème Home Assistant. Grace à lui on va pouvoir faire une automatisation qui raz le compteur sur un évènement particulier ou intégrer le raz dans un script.

Pour cela il faut ajouter le yaml suivant (donné par "Outils de développement / Services / Passez en mode YAML") :

```yaml
service: tuto_hacs.raz_compteur
data:
  valeur_depart: 90
target:
  entity_id: sensor.tuto_hacs_entite_3
```

On aussi utiliser notre entité comme trigger des automatisations. Un exemple complet :
```yaml
alias: Raz le compteur après une minute
description: Remet à zéro le compteur lorsqu'il dépasse 60 (donc toutes les minutes)
trigger:
  - platform: numeric_state
    entity_id: sensor.tuto_hacs_entite_3
    above: 60
condition: []
action:
  - service: tuto_hacs.raz_compteur
    data:
      valeur_depart: "0"
    target:
      entity_id: sensor.tuto_hacs_entite_3
mode: single
```

Cette automatisation se déclenche lorsque la valeur du compteur est supérieure à 60 et remet à zéro le compteur si c'est le cas via l'utilisation du service.
Vérifies [ici](http://localhost:9123/lovelace/0) que cela fonctionne bien.


# Conclusion
Ce tuto t'as permis d'apprendre à créer des entités qui interagissent avec l'extérieur en publiant des états, écoutant les états des autres entités et en publiant des services utilisables par les automatisations et les scripts.

Il est impossible d'être exhaustif tellement l'écosystème Home Assistant est riche. Pour découvrir d'autres façons d'interagir c'est le moment de faire un tour dans la documentation de référence et notamment dans les articles suivants :
 - https://developers.home-assistant.io/docs/integration_listen_events
 - https://developers.home-assistant.io/docs/creating_integration_manifest
 - https://developers.home-assistant.io/docs/configuration_yaml_index
 - https://developers.home-assistant.io/docs/dev_101_services
 - https://developers.home-assistant.io/docs/dev_101_config (non abordé mais important pour récupérer la configuration de Home Assistant)

Dans le prochain [tuto](/tuto4.md), on va apprendre à configurer notre intégration à travers l'interface de Home Assistant et non plus à travers le fichier `configuration.yaml`.

# Listes des fichiers références de ce tuto
Ne sont présents que les fichiers modifiés par rapport au tuto précédent.

## `const.py`

```python
""" Les constantes pour l'intégration Tuto HACS """

from homeassistant.const import Platform

DOMAIN = "tuto_hacs"
PLATFORMS: list[Platform] = [Platform.SENSOR]

CONF_NAME = "name"
CONF_DEVICE_ID = "device_id"

DEVICE_MANUFACTURER = "JMCOLLIN"

SERVICE_RAZ_COMPTEUR = "raz_compteur"
```

## `sensor.py`

```python
""" Implements the VersatileThermostat sensors component """
import logging
from datetime import datetime, timedelta
import voluptuous as vol

from homeassistant.const import UnitOfTime, STATE_UNAVAILABLE, STATE_UNKNOWN
from homeassistant.core import HomeAssistant, callback, Event, State
from homeassistant.config_entries import ConfigEntry
from homeassistant.helpers.entity_platform import (
    AddEntitiesCallback,
    async_get_current_platform,
)
from homeassistant.components.sensor import (
    SensorEntity,
    SensorDeviceClass,
    SensorStateClass,
)

from homeassistant.helpers.event import (
    async_track_time_interval,
    async_track_state_change_event,
)

from homeassistant.helpers.entity import DeviceInfo, DeviceEntryType

import homeassistant.helpers.config_validation as cv

from .const import (
    DOMAIN,
    DEVICE_MANUFACTURER,
    CONF_DEVICE_ID,
    CONF_NAME,
    SERVICE_RAZ_COMPTEUR,
)

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

    entity1 = TutoHacsElapsedSecondEntity(hass, entry)
    entity2 = TutoHacsListenEntity(hass, entry, entity1)
    async_add_entities([entity1, entity2], True)

    # Add services
    platform = async_get_current_platform()
    platform.async_register_entity_service(
        SERVICE_RAZ_COMPTEUR,
        {vol.Optional("valeur_depart"): cv.positive_int},
        "service_raz_compteur",
    )


class TutoHacsElapsedSecondEntity(SensorEntity):
    """La classe de l'entité TutoHacs"""

    _hass: HomeAssistant

    def __init__(
        self,
        hass: HomeAssistant,  # pylint: disable=unused-argument
        entry_infos,  # pylint: disable=unused-argument
    ) -> None:
        """Initisalisation de notre entité"""
        self._hass = hass
        self._attr_has_entity_name = True
        self._attr_name = entry_infos.get(CONF_NAME)
        self._device_id = entry_infos.get(CONF_DEVICE_ID)
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

    @property
    def device_info(self) -> DeviceInfo:
        """Donne le lien avec le device. Non utilisé jusqu'au tuto 4"""
        return DeviceInfo(
            entry_type=DeviceEntryType.SERVICE,
            identifiers={(DOMAIN, self._device_id)},
            name=self._attr_name,
            manufacturer=DEVICE_MANUFACTURER,
            model=DOMAIN,
        )

    @callback
    async def async_added_to_hass(self):
        """Ce callback est appelé lorsque l'entité est ajoutée à HA"""

        # Arme le timer
        timer_cancel = async_track_time_interval(
            self._hass,
            self._incremente_secondes,  # La methode à appeler périodiquement
            interval=timedelta(seconds=1),
        )
        # desarme le timer lors de la destruction de l'entité
        self.async_on_remove(timer_cancel)

    @callback
    async def _incremente_secondes(self, _):
        """Cette méthode va être appelée toutes les secondes"""
        _LOGGER.info("Appel de incremente_secondes à %s", datetime.now())

        # On incrémente la valeur de notre etat
        self._attr_native_value += 1

        # On sauvegarde le nouvel état
        self.async_write_ha_state()

        # Toutes les 5 secondes on envoie un event
        if self._attr_native_value % 5 == 0:
            self._hass.bus.fire(
                "event_changement_etat_TutoHacsElapsedSecondEnity",
                {"nb_secondes": self._attr_native_value},
            )

    async def service_raz_compteur(self, valeur_depart: int):
        """Appelée lors de l'invocation du service 'raz_compteur'
        Elle prend en argument la 'valeur_depart' qui est
        construite à partir du paramètre 'valeur_depart'
        """
        _LOGGER.info(
            "Appel du service service_raz_compteur valeur_depart: %d", valeur_depart
        )
        self._attr_native_value = valeur_depart if valeur_depart is not None else 0

        # On sauvegarde le nouvel état
        self.async_write_ha_state()


class TutoHacsListenEntity(SensorEntity):
    """La classe de l'entité TutoHacs qui écoute la première"""

    _hass: HomeAssistant
    _entity_to_listen: TutoHacsElapsedSecondEntity

    def __init__(
        self,
        hass: HomeAssistant,  # pylint: disable=unused-argument
        entry_infos,  # pylint: disable=unused-argument
        entity_to_listen: TutoHacsElapsedSecondEntity,  # L'entité qu'on veut écouter
    ) -> None:
        """Initisalisation de notre entité"""
        self._hass = hass
        self._attr_has_entity_name = True
        self._device_id = entry_infos.get(CONF_DEVICE_ID)
        # On lui donne un nom et un unique_id différent
        self._attr_name = entry_infos.get(CONF_NAME) + " Ecouteur"
        self._attr_unique_id = self._device_id + "_ecouteur"
        # Pas de valeur tant qu'on n'a pas reçu
        self._attr_native_value = None
        self._entity_to_listen = entity_to_listen

    @property
    def should_poll(self) -> bool:
        """Pas de polling pour mettre à jour l'état"""
        return False

    @property
    def icon(self) -> str | None:
        return "mdi:timer-settings-outline"

    @property
    def device_class(self) -> SensorDeviceClass | None:
        """Cette entité"""
        return SensorDeviceClass.TIMESTAMP

    @property
    def device_info(self) -> DeviceInfo:
        """Donne le lien avec le device. Non utilisé jusqu'au tuto 4"""
        return DeviceInfo(
            entry_type=DeviceEntryType.SERVICE,
            identifiers={(DOMAIN, self._device_id)},
            name=self._attr_name,
            manufacturer=DEVICE_MANUFACTURER,
            model=DOMAIN,
        )

    @callback
    async def async_added_to_hass(self):
        """Ce callback est appelé lorsque l'entité est ajoutée à HA"""

        # Arme l'écoute de la première entité
        listener_cancel = async_track_state_change_event(
            self.hass,
            [self._entity_to_listen.entity_id],
            self._on_event,
        )
        # desarme le timer lors de la destruction de l'entité
        self.async_on_remove(listener_cancel)

    @callback
    async def _on_event(self, event: Event):
        """Cette méthode va être appelée à chaque fois que l'entité
        "entity_to_listen" publie un changement d'état"""

        _LOGGER.info("Appel de _on_event à %s avec l'event %s", datetime.now(), event)

        new_state: State = event.data.get("new_state")
        # old_state: State = event.data.get("old_state")

        if new_state is None or new_state.state in (STATE_UNAVAILABLE, STATE_UNKNOWN):
            _LOGGER.warning("Pas d'état disponible. Evenement ignoré")
            return

        # state.last_changed.astimezone(self._current_tz)

        # On recherche la date de l'event pour la stocker dans notre état
        self._attr_native_value = new_state.last_changed

        # On sauvegarde le nouvel état
        self.async_write_ha_state()
```

## `services.yaml`

```yaml
raz_compteur:
  name: Raz compteur
  description: Remet à zéro le compteur de temps et optionnellement donne la valeur de départ
  target:
    entity:
      integration: tuto_hacs
      device_class: duration
  fields:
    valeur_depart:
      name: Valeur départ
      description: La valeur de départ du compteur
      required: false
      advanced: false
      example: "10"
      default: "0"
      selector:
        number:
          min: 0
          max: 900
          step: 1
          mode: slider
```

## `automations.yaml`

```yaml
- id: '1681554365445'
  alias: Raz le compteur après une minute
  description: Remet à zéro le compteur lorsqu'il dépasse 60 (donc toutes les minutes)
  trigger:
  - platform: numeric_state
    entity_id: sensor.tuto_hacs_entite_3
    above: 60
  condition: []
  action:
  - service: tuto_hacs.raz_compteur
    data:
      valeur_depart: '0'
    target:
      entity_id: sensor.tuto_hacs_entite_3
  mode: single
```
