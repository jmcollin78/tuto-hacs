J'ai mis longtemps à comprendre qu'il faut savoir qu'un appareil n'a pas de code, ni de déclaration. Il est simplement créé lorsqu'on déclare une entité et qu'on l'a relie à un appareil.



Pour terminer ce tuto, on va relier cette entité à un appareil (device). Pour cela, c'est très simple, il suffit de déclarer dans la classe de l'entité à quel device elle appartient.

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
            identifiers={(DOMAIN, self._config_id)},
            name=self._device_name,
            manufacturer=DEVICE_MANUFACTURER,
            model=DOMAIN,
        )
```

On redémarre le tout et on constate dans la liste des appareils, un nouvel appareil 



