## L'objectif de cet article est d'enrichir fonctionnellement notre entité
Il s'inscrit dans la suite des articles dont le sommaire est [ici](/README.md).

> ![Tip](/images/tips.png?raw=true) Les fichiers sources complets en version finales sont en fin d'article. Cf [Fichiers sources du tuto](#fichiers-sources-du-tuto)

# Pre-requis
Avoir déroulé avec succès les deux premiers articles [tuto1](/tuto1.md) et [tuto2](/tuto2.md). Vous devez donc avoir une entité avec un état qui est une mesure en secondes.

# Les points abordés
Dans cet article, tu vas apprendre à :
1. utiliser l'objet `hass`,
2. déclencher périodiquement la mise à jour d'une entité,
3. mettre à jour l'état de l'entité,
4. écouter et générer des évènements,
5. TODO

# L'objet `hass`

Si il y a bien un objet important dans le developpement Home Assistant, c'est l'objet `hass`. De type `HomeAssistant`, on l'a déjà rencontré dans le tuto2 sans l'expliquer, on va le faire ici.
Cet objet est partout. **Il représente l'instance Home Assistant** sur laquelle l'entité est configurée et permet d'accéder à toutes les objets manipulés par Home Assistant : les configurations, les états, les entités, les bus d'évènements, ...

**Comme il est indispensable**, on va le mémoriser dès qu'on peut - à la construction de l'entité - dans les attributs de l'entité ; en variable privée (donc commençant par un `_` selon la règle de nommage Python.
Pour cela, on définit un attribut `self._hass` dans la fonction d'init de l'entité. Et on stocke l'objet `hass` dedans :

```python
class TutoHacsElapsedSecondEnity(SensorEntity):
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

class TutoHacsElapsedSecondEnity(SensorEntity):
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

class TutoHacsElapsedSecondEnity(SensorEntity):
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

