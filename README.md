# Presostat ESPHome

Projet ESPHome pour piloter un pressostat basé sur un ESP32-S3. La configuration gère :

- La lecture d'un capteur de pression analogique.
- Le pilotage d'un relais de pompe avec hystérésis (seuil bas/haut).
- Un bouton physique pour forcer la pompe.
- Un écran OLED SSD1306 pour afficher l'état.
- Une sécurité de pression basse persistante.

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

### 2) Ajuster les substitutions matérielles

Dans `install.yaml`, adaptez si besoin :

- `board` (ex: `esp32-s3-devkitc-1`).
- Pins : `pressure_adc_pin`, `relay_pin`, `button_pin`, `i2c_sda_pin`, `i2c_scl_pin`.
- Plages de calibration :
  - `pressure_voltage_min` / `pressure_voltage_max`
  - `pressure_bar_min` / `pressure_bar_max`
- Seuils d'hystérésis :
  - `pressure_hold_min` (marche pompe)
  - `pressure_hold_max` (arrêt pompe)
- Sécurité : `pressure_fault_min` (défaut si pression trop basse pendant 5 min).

### 3) Calibration du capteur

Le calcul de pression est linéaire entre `pressure_voltage_min` et
`pressure_voltage_max`, reporté dans la plage `pressure_bar_min` à
`pressure_bar_max`.

Adaptez ces valeurs aux caractéristiques de votre capteur pour obtenir une
mesure fiable.

## Utilisation

1. Installez ESPHome (CLI ou via Home Assistant).
2. Lancez la compilation et le flash du firmware avec ESPHome.
3. Surveillez les entités créées (pression, relais, défaut) via l'API ESPHome.

## Remarques

- Le relais est configuré avec `restore_mode: RESTORE_DEFAULT_OFF`.
- Le défaut de pression basse coupe la pompe automatiquement.
- L'écran OLED affiche l'état de la pompe, la pression et le défaut.
