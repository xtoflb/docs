# Comment effacer la configuration d'un switch Cisco 2960 avec accès à la console de configuration

Pour effacer complètement la configuration d'un switch Cisco Catalyst 2960 via la console, utilisez les commandes standard en mode privilégié si vous avez déjà accès. Cela supprime la configuration de démarrage (startup-config) et nécessite un redémarrage pour appliquer les changements.

## Procédure standard (accès console existant)

Connectez-vous en mode EXEC utilisateur, puis passez en mode privilégié avec `enable`.

```
enable
erase startup-config
reload
```

- Confirmez l'effacement de la startup-config (tapez `y` si demandé).
- Confirmez le rechargement (le switch redémarre sans configuration persistante).

Après redémarrage, le switch sera aux paramètres d'usine, avec un dialogue de configuration initiale optionnel (répondez `no` pour passer en CLI).

## Vérification préalable

Vérifiez les configurations avant effacement :

- `show running-config` pour la config en cours.
- `show startup-config` pour la config de démarrage.[^5]

**Attention** : Sauvegardez toute config utile avant (`copy running-config startup-config` ou via TFTP). Le processus est irréversible sans backup.

## Alternative si mode ROMMON (sans accès enable)

Si vous n'avez pas les privilèges ou un mot de passe bloque l'accès :

1. Allumez le switch en tenant le bouton **Mode** (façade) pendant ~15-20 secondes jusqu'au prompt `switch:` (LED système passe orange puis verte fixe).
2. Tapez `flash_init`.
3. `delete flash:config.text` (confirmez `y`).
4. `delete flash:vlan.dat` (confirmez `y`).
5. `boot` pour redémarrer.[^4]

Cette méthode remet à zéro total, idéale pour un switch verrouillé.
