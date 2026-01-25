# Presostat ESPHome

Projet ESPHome pour piloter un pressostat basé sur un ESP32-S3. La configuration gère :

- La lecture d'un capteur de pression analogique.
- Le pilotage d'un relais de pompe avec hystérésis (seuil bas/haut).
- Un bouton physique pour forcer la pompe.
- Un écran OLED SSD1306 pour afficher l'état.
- Une sécurité de pression basse persistante.
- Une calibration multi-points du capteur (1 à 6 bar) via l'interface web et le bouton physique.

## Structure du dépôt

- `install.yaml` : fichier principal avec les substitutions et l'inclusion du package.
- `presostat_config.yaml` : logique ESPHome (capteurs, relais, OLED, sécurité).
- `secrets.yaml.example` : exemple de secrets à copier.

## Configuration à apporter

### 1) Secrets Wi‑Fi

Copiez le fichier d'exemple et renseignez vos identifiants :

```bash
cp secrets.yaml.example secrets.yaml
```

Puis éditez `secrets.yaml` :

```yaml
wifi_ssid: "VOTRE_SSID"
wifi_password: "VOTRE_MOT_DE_PASSE"
wifi_ap_password: "MOT_DE_PASSE_AP"
```

Le mot de passe du point d'accès de secours peut aussi être défini directement
via la substitution `wifi_ap_password` dans `install.yaml` si vous préférez ne
pas le mettre dans les secrets.

### 2) Ajuster les substitutions matérielles

Dans `install.yaml`, adaptez si besoin :

- `board` (ex: `esp32-s3-devkitc-1`).
- Pins : `pressure_adc_pin`, `relay_pin`, `button_pin`, `i2c_sda_pin`, `i2c_scl_pin`.
- LED : `led_pin` (LED RGB intégrée, type WS2812 ou équivalent).
- Méthode LED : `led_method_type` / `led_method_channel` (sélection du driver NeopixelBus, ex: `esp32_rmt` + `1` pour ESP32‑S3 si la LED ne répond pas).
- Couleur LED :
  - `led_fault_red` / `led_fault_green` / `led_fault_blue` (couleur en défaut).
  - `led_out_red` / `led_out_green` / `led_out_blue` (couleur pression hors consigne).
  - `led_in_green_min` / `led_in_green_max` (intensité verte en consigne, valeurs 0.0 à 1.0).
  - `led_calibration_red` / `led_calibration_green` / `led_calibration_blue` (couleur pendant la calibration).
  - `led_invalid_red` / `led_invalid_green` / `led_invalid_blue` (couleur quand la valeur du capteur est invalide).
  - `led_off_red` / `led_off_green` / `led_off_blue` (couleur quand la pompe est OFF).
  - `led_rgb_order` (ordre des couleurs, ex: `GRB`).

Sur les modules ESP32‑S3 N16R8 (ex : DevKitC‑1), la LED RGB intégrée est
généralement une WS2812 câblée sur GPIO48 avec un ordre GRB. Cela aide à éviter
les confusions sur les boards S3 N16R8. La doc NeoPixelBus détaille les méthodes
supportées et rappelle que `esp32_i2s` n'est pas disponible sur ESP32‑S3 : https://esphome.io/components/light/neopixelbus.html
- Wi-Fi : `wifi_ap_password` (mot de passe du point d'accès de secours).
- Plages de calibration :
  - `pressure_voltage_min` / `pressure_voltage_max`
  - `pressure_bar_min` / `pressure_bar_max`
- Seuils d'hystérésis :
  - `pressure_hold_min` (marche pompe)
  - `pressure_hold_max` (arrêt pompe)
  - `pressure_hold_min_sensor` / `pressure_hold_max_sensor` (entités Home Assistant optionnelles)
- Sécurité : `pressure_fault_min` (défaut si pression trop basse pendant 5 min).

### 3) Calibration du capteur

Le calcul de pression est linéaire entre `pressure_voltage_min` et
`pressure_voltage_max`, reporté dans la plage `pressure_bar_min` à
`pressure_bar_max`.

Adaptez ces valeurs aux caractéristiques de votre capteur pour obtenir une
mesure fiable.

### 4) Calibration multi-points (1 → 6 bar)

Deux méthodes sont disponibles pour calibrer la pression :

1. **Via l'interface web (ou Home Assistant)** :
   - Renseignez les entités `Calibration X bar (V)` avec la tension lue par le capteur.
   - La conversion en bar devient alors une interpolation linéaire entre les points saisis.
2. **Via le bouton physique** :
   - Appuyez sur l'entité `Calibration Demarrer`.
   - Le texte `Calibration Etat` vous indique le palier à atteindre.
   - Montez la pression au palier (1 bar, 2 bar, …, 6 bar) puis appuyez sur le **bouton physique** pour enregistrer chaque point.
   - L'étape suivante s'affiche automatiquement jusqu'à la fin (6 bar).

Si aucune calibration n'est renseignée, la conversion reste basée sur
`pressure_voltage_min` / `pressure_voltage_max`.

## Utilisation

1. Installez ESPHome (CLI ou via Home Assistant).
2. Lancez la compilation et le flash du firmware avec ESPHome.
3. Surveillez les entités créées (pression, relais, défaut) via l'API ESPHome.
4. Pour suivre les valeurs en temps réel dans le terminal :

```bash
esphome logs install.yaml --device /dev/tty.usbserial-XXXX
```

## Ajuster la plage min/max via number ou sensor

Deux méthodes sont disponibles pour définir les seuils d'hystérésis :

1. **Via les entités number** :
   - `Seuil Pression Min`
   - `Seuil Pression Max`
   Ces valeurs sont persistantes (restore_value) et utilisées par défaut.
2. **Via des entités sensor Home Assistant** :
   - Renseignez `pressure_hold_min_sensor` et `pressure_hold_max_sensor` dans `install.yaml`.
   - Si les capteurs renvoient une valeur valide, ils prennent le dessus sur les numbers.

## Remarques

- Le relais est configuré avec `restore_mode: RESTORE_DEFAULT_OFF`.
- Le défaut de pression basse coupe la pompe automatiquement.
- L'écran OLED affiche l'état de la pompe, la pression et le défaut.
- La LED intégrée suit ces comportements :
  - **État normal** : vert fixe, intensité proportionnelle à la pression (faible proche de `pressure_hold_min`, fort proche de `pressure_hold_max`).
  - **Pompe ON** : vert pulsé lent, modulé autour de l'intensité cible.
  - **Sous pression (< `pressure_hold_min`)** : rouge clignotement lent (flash simple).
  - **Sur pression (>= `pressure_hold_max`)** : rouge double flash lent (flash‑flash‑pause).
  - **Défaut persistant** : rouge clignotement rapide.
  - **Calibration active** : bleu respiration.
  - **Valeur capteur invalide** : violet fixe.
  - **Test LED (Bypass Script)** : blanc fixe tant que l'interrupteur est activé (bypass de la logique).

Note : la configuration LED utilise `neopixelbus` (compatible avec le framework Arduino). Si vous repassez en `esp32_rmt_led_strip`, il faudra basculer sur `esp-idf`. 
