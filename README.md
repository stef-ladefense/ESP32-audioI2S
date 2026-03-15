# ESP32-audioI2S

:warning: **This library only works on multi-core chips like ESP32, ESP32-S3 and ESP32-P4. Your board must have PSRAM! It does not work on the ESP32-S2, ESP32-C3 etc** :warning:

***
# Changes applied to ESP32-audioI2S (v3.4.5d)

This document provides a detailed breakdown of the modifications, improvements, and bug fixes applied to the library. Each change aims to optimize resource usage (Flash, RAM, CPU) and improve system stability.

## 1. Modular Configuration & Codec Selection
**Context:** By default, the library compiles all decoders and features, consuming significant Flash memory even if only one format is used.
**Improvement:**
- **Codec Selection:** Selective enable/disable via `#define` macros in `Audio.h`.
- **OGG Pack:** Enabling `AUDIO_CODEC_OGG` activates the OGG container and the **Opus** decoder (modern high-quality streams). **Vorbis** support (legacy) can be added optionally.
- **M4A/AAC:** Enabling `AUDIO_CODEC_M4A` (container) automatically activates the **AAC** decoder, as M4A files almost exclusively contain AAC-encoded audio.
- **Log Level:** `AUDIO_LOG_LEVEL` (0-4) to strip log strings from the binary at compile-time.
- **Volume Ramp:** `AUDIO_VOLUME_RAMP_STEPS` to adjust fade speed (default 50, use 25 for faster response).

### 📊 Resource Savings (Flash & Static RAM)
  ┌───────────────────────┬────────────┬────────────┬──────────────┐
  │ Module Disabled       │ Flash Gain │ RAM Gain   │ Impact       │
  ├───────────────────────┼────────────┼────────────┼──────────────┤
  │ Codec OPUS            │ +170.7 KB  │ +4.3 KB    │ 🔴 Very High │
  │ Codec VORBIS          │ +108.6 KB  │ +160 bytes │ 🟠 High      │
  │ Codec AAC             │ +107.1 KB  │ -          │ 🟠 High      │
  │ Codec MP3             │ +66.0 KB   │ +24 bytes  │ 🟠 High      │
  │ Codec FLAC            │ +32.2 KB   │ +72 bytes  │ 🟡 Medium    │
  │ Codec M4A (Container) │ +14.6 KB   │ +160 bytes │ 🟢 Low       │
  │ File System (FS)      │ +3.3 KB    │ +24 bytes  │ 🟢 Low       │
  │ Codec WAV             │ +2.9 KB    │ +16 bytes  │ 🟢 Low       │
  └───────────────────────┴────────────┴────────────┴──────────────┘

**Benefit:** Significant reduction in resource usage. Combined, these optimizations can save up to **~487KB of Flash memory**.

## 2. Robust I2S State Management
**Context:** The library often generated `E (10013)` errors during zapping or station changes because it tried to disable an already disabled I2S channel.
**Improvement:**
- Secured `I2Sstop()` by synchronizing the software state (`m_f_i2s_channel_enabled`) with the hardware.
- The software flag only resets to `false` if the hardware confirms a successful stop (`ESP_OK`).
**Benefit:** Eliminates `E (10013)` errors during rapid zapping and ensures reliable hardware/software synchronization.

## 3. Restored "v3.4.4" Volume Curves
**Context:** Standard volume curves can feel unnatural or too slow at low levels.
**Improvement:**
- Re-implemented **Square** (Quadratic) and **Logarithmic** (authentic 3.4.4) curves.
- **Mode 0 (Square):** Punchy and very responsive.
- **Mode 1 (Log):** Natural feel, faithful to the 3.4.4 version.
- **Switch:** A `#define CurveOff345` macro is available in `Audio.h` to force the use of the original v3.4.5 polynomial curves if preferred.
**Benefit:** Offers an immediate power increase and a more natural auditory feeling, while maintaining compatibility with the original v3.4.5 behavior via the macro.

## 4. Adjustable Volume Ramp Speed
**Improvement:** Added `#define AUDIO_VOLUME_RAMP_STEPS` in `Audio.h`.
**Benefit:** Controls the "fade" speed during Mute or volume changes.
- **Value 50 (Default):** Smooth standard fade.
- **Value 25:** Doubles the response speed (more reactive).

## 5. Modular Speech Features (TTS) & String Elimination
**Context:** The TTS features heavily relied on the Arduino `String` class, causing heap fragmentation through dynamic concatenations.
**Improvement:**
- Added `AUDIO_ENABLE_SPEECH` macro to completely disable TTS code if unused.
- **Refactoring:** Completely rewrote `openai_speech()` to use `const char*`, `ps_ptr` buffers, and `snprintf`.
**Benefit:** Eliminates heap fragmentation risks and saves Flash memory when the module is disabled.

## 6. Modular File System Support
**Context:** Unconditional inclusion of FS libraries (SD, SD_MMC, etc.) is unnecessary for pure WebRadio projects.
**Improvement:**
- Added `AUDIO_ENABLE_FS` macro to conditionally include or exclude all local file processing code.
**Benefit:** Removes unnecessary dependencies and significantly reduces the binary footprint.

## 7. Granular Log Control
**Improvement:** Added `AUDIO_LOG_LEVEL` macro (0=None to 4=Debug).
**Benefit:** Saves ~10-15KB of Flash memory. Since logs are implemented as preprocessor macros, choosing a lower level prevents the literal strings (text messages) from being compiled into the binary. If a log level is disabled, its associated text is physically removed from the Flash memory.

## 8. Performance & Memory Optimizations
**Improvement:**
- **Dynamic DSP Bypass:** Skips the filter chain (Equalizer) calculations if all gains are zero.
- **Static Constexpr:** Moved internal lookup tables to Flash (PROGMEM).
- **Persistent Resample Buffer:** Implemented a persistent `ps_ptr` buffer for 48kHz resampling.
- **STL Removal:** Replaced remaining `std::string` usage with C-strings.
**Benefit:** 10-15% CPU savings during playback without EQ and reduced stress on the memory allocator.

***

# Changements appliqués à ESP32-audioI2S (v3.4.5d)

Ce document détaille les modifications, améliorations et corrections apportées à la bibliothèque. Chaque changement vise à optimiser l'utilisation des ressources (Flash, RAM, CPU) et à améliorer la stabilité du système.

## 1. Configuration Modulaire et Sélection des Codecs
**Contexte :** Par défaut, la bibliothèque compile tous les décodeurs et fonctionnalités, consommant beaucoup de mémoire Flash même si un seul format est utilisé.
**Amélioration :**
- **Sélection des Codecs :** Activation sélective via des macros `#define` dans `Audio.h`.
- **Pack OGG :** L'activation de `AUDIO_CODEC_OGG` active le conteneur OGG et le décodeur **Opus** (haute qualité moderne). Le support **Vorbis** peut être ajouté en option.
- **M4A/AAC :** L'activation de `AUDIO_CODEC_M4A` (conteneur) active automatiquement le décodeur **AAC**, car les fichiers M4A contiennent presque exclusivement de l'audio encodé en AAC.
- **Niveau de Log :** `AUDIO_LOG_LEVEL` (0-4) pour retirer les messages de log du binaire à la compilation.
- **Rampe de Volume :** `AUDIO_VOLUME_RAMP_STEPS` pour ajuster la vitesse du fondu (défaut 50, régler à 25 pour plus de réactivité).

### 📊 Économies de Ressources (Flash & RAM Statique)
  ┌──────────────────────────┬────────────┬─────────────┬───────────────┐
  │ Module Désactivé         │ Gain Flash │ Gain RAM    │ Impact        │
  ├──────────────────────────┼────────────┼─────────────┼───────────────┤
  │ Codec OPUS               │ +170.7 Ko  │ +4.3 Ko     │ 🔴 Très Élevé │
  │ Codec VORBIS             │ +108.6 Ko  │ +160 octets │ 🟠 Élevé      │
  │ Codec AAC                │ +107.1 Ko  │ -           │ 🟠 Élevé      │
  │ Codec MP3                │ +66.0 Ko   │ +24 octets  │ 🟠 Élevé      │
  │ Codec FLAC               │ +32.2 Ko   │ +72 octets  │ 🟡 Moyen      │
  │ Codec M4A (Conteneur)    │ +14.6 Ko   │ +160 octets │ 🟢 Faible     │
  │ Système de fichiers (FS) │ +3.3 Ko    │ +24 octets  │ 🟢 Faible     │
  │ Codec WAV                │ +2.9 Ko    │ +16 octets  │ 🟢 Faible     │
  └──────────────────────────┴────────────┴─────────────┴───────────────┘

**Bénéfice :** Réduction drastique de l'utilisation des ressources. Cumulées, ces optimisations permettent d'économiser jusqu'à **~487 Ko de Flash**.

## 2. Gestion d'État I2S Robuste
**Contexte :** Des erreurs `E (10013)` apparaissaient fréquemment lors du zapping ou du changement de station car la bibliothèque tentait de désactiver un canal I2S déjà arrêté.
**Amélioration :**
- Sécurisation de `I2Sstop()` en synchronisant l'état logiciel (`m_f_i2s_channel_enabled`) avec le matériel.
- Le flag logiciel ne repasse à `false` que si le matériel confirme l'arrêt effectif (`ESP_OK`).
**Bénéfice :** Élimine les erreurs `E (10013)` lors des changements rapides et garantit une synchronisation fiable.

## 3. Restauration des Courbes de Volume "v3.4.4"
**Contexte :** Les courbes de volume standard peuvent paraître peu naturelles ou trop lentes à bas volume.
**Amélioration :**
- Ré-implémentation des courbes **Square** (Quadratique) et **Logarithmique** (authentique 3.4.4).
- **Mode 0 (Square) :** Très réactif, idéal pour un réglage rapide.
- **Mode 1 (Log) :** Sensation naturelle, fidèle à la version 3.4.4.
- **Commutateur :** Une macro `#define CurveOff345` est disponible dans `Audio.h` pour forcer l'utilisation de la courbe polynomiale originale de la v3.4.5 si nécessaire.
**Bénéfice :** Offre une montée en puissance immédiate et un ressenti auditif plus naturel, tout en permettant de revenir au comportement standard de la v3.4.5 via la macro.

## 4. Vitesse de Rampe du Volume Ajustable
**Amélioration :** Ajout du paramètre `#define AUDIO_VOLUME_RAMP_STEPS` dans `Audio.h`.
**Bénéfice :** Permet de régler la vitesse du "fondu" (Fade in/out) lors du Mute ou des changements de volume.
- **Valeur 50 (Défaut) :** Fondu standard et fluide.
- **Valeur 25 :** Double la vitesse de réaction (plus réactif).

## 5. Modularité TTS et Suppression de String
**Contexte :** Les fonctions de synthèse vocale utilisaient la classe Arduino `String`, provoquant une fragmentation de la mémoire (tas/heap).
**Amélioration :**
- Ajout de la macro `AUDIO_ENABLE_SPEECH` pour désactiver complètement le TTS.
- **Refactorisation :** Réécriture complète de `openai_speech()` en utilisant des `const char*` et `snprintf` avec des buffers `ps_ptr`.
**Bénéfice :** Élimination des risques de fragmentation mémoire et économie de Flash si le module est désactivé.

## 6. Modularité du Système de Fichiers (FS)
**Contexte :** L'inclusion systématique des librairies de fichiers (SD, SD_MMC, etc.) est inutile pour une WebRadio pure.
**Amélioration :**
- Ajout de la macro `AUDIO_ENABLE_FS`.
- Encadrement conditionnel de toutes les méthodes liées au système de fichiers.
**Bénéfice :** Supprime les dépendances inutiles et allège considérablement le binaire final.

## 7. Contrôle Granulaire des Logs
**Amélioration :** Ajout de la macro `AUDIO_LOG_LEVEL` (0=Aucun à 4=Debug).
**Bénéfice :** Économie de 10 à 15 Ko de Flash. Les logs étant gérés par le préprocesseur, les chaînes de caractères (textes des messages) sont physiquement retirées du binaire final lors de la compilation si le niveau est désactivé. Si un message n'est pas compilé, il n'occupe aucun octet en mémoire Flash.

## 8. Optimisations Performance & Mémoire
**Amélioration :**
- **Bypass DSP Dynamique :** Saut automatique des calculs de l'égaliseur (IIR) si tous les gains sont à 0dB.
- **Constexpr :** Déplacement des tables de recherche internes en Flash (PROGMEM).
- **Buffer Persistant :** Utilisation d'un buffer `ps_ptr` persistant pour le ré-échantillonnage 48kHz.
- **Suppression STL :** Remplacement des derniers `std::string` par des chaînes C.
**Bénéfice :** Gain de 10 à 15% de cycles CPU lors de la lecture sans égalisation et réduction du stress sur l'allocateur mémoire.

***

Plays mp3, m4a and wav files from SD card via I2S with external hardware.
HELIX-mp3 and faad2-aac decoder is included. There is also an OPUS decoder for Fullband, an VORBIS decoder and a FLAC decoder.
Works with MAX98357A (3 Watt amplifier with DAC), connected three lines (DOUT, BLCK, LRC) to I2S. The I2S output frequency is always 48kHz, regardless of the input source, so Bluetooth devices can also be connected without any problems.
For stereo are two MAX98357A necessary. AudioI2S works with UDA1334A (Adafruit I2S Stereo Decoder Breakout Board), PCM5102A and CS4344.
Other HW may work but not tested. Plays also icy-streams, GoogleTTS and OpenAIspeech. Can be compiled with Arduino IDE. [WIKI](https://github.com/schreibfaul1/ESP32-audioI2S/wiki)

```` c++
#include "Arduino.h"
#include "WiFi.h"
#include "Audio.h"

// Digital I/O used
#define I2S_DOUT      25
#define I2S_BCLK      27
#define I2S_LRC       26

String ssid =     "*******";
String password = "*******";

Audio audio;

// callbacks
void my_audio_info(Audio::msg_t m) {
    Serial.printf("%s: %s\n", m.s, m.msg);
}

void setup() {
    Audio::audio_info_callback = my_audio_info; // optional
    Serial.begin(115200);
    WiFi.begin(ssid.c_str(), password.c_str());
    while (WiFi.status() != WL_CONNECTED) delay(1500);
    audio.setPinout(I2S_BCLK, I2S_LRC, I2S_DOUT);
    audio.setVolume(21); // default 0...21
    audio.connecttohost("http://stream.antennethueringen.de/live/aac-64/stream.antennethueringen.de/");
}

void loop(){
    audio.loop();
    vTaskDelay(1);
}

````
You can find more examples here: https://github.com/schreibfaul1/ESP32-audioI2S/tree/master/examples

````c++
// detailed cb output
void my_audio_info(Audio::msg_t m) {
    switch(m.e){
        case Audio::evt_info:           Serial.printf("info: ....... %s\n", m.msg); break;
        case Audio::evt_eof:            Serial.printf("end of file:  %s\n", m.msg); break;
        case Audio::evt_bitrate:        Serial.printf("bitrate: .... %s\n", m.msg); break; // icy-bitrate or bitrate from metadata
        case Audio::evt_icyurl:         Serial.printf("icy URL: .... %s\n", m.msg); break;
        case Audio::evt_id3data:        Serial.printf("ID3 data: ... %s\n", m.msg); break; // id3-data or metadata
        case Audio::evt_lasthost:       Serial.printf("last URL: ... %s\n", m.msg); break;
        case Audio::evt_name:           Serial.printf("station name: %s\n", m.msg); break; // station name or icy-name
        case Audio::evt_streamtitle:    Serial.printf("stream title: %s\n", m.msg); break;
        case Audio::evt_icylogo:        Serial.printf("icy logo: ... %s\n", m.msg); break;
        case Audio::evt_icydescription: Serial.printf("icy descr: .. %s\n", m.msg); break;
        case Audio::evt_image: for(int i = 0; i < m.vec.size(); i += 2){
                                        Serial.printf("cover image:  segment %02i, pos %07lu, len %05lu\n", i / 2, m.vec[i], m.vec[i + 1]);} break; // APIC
        case Audio::evt_lyrics:         Serial.printf("sync lyrics:  %s\n", m.msg); break;
        case Audio::evt_log   :         Serial.printf("audio_logs:   %s\n", m.msg); break;
        default:                        Serial.printf("message:..... %s\n", m.msg); break;
    }
}
````
<br>

|Codec       | ESP32       |ESP32-S3 or ESP32-P4         |                          |
|------------|-------------|-----------------------------|--------------------------|
| mp3        | y           | y                           |                          |
| aac        | y           | y                           |                          |
| aacp       | y (mono)    | y (+SBR, +Parametric Stereo)|                          |
| wav        | y           | y                           |                          |
| flac       | y           | y                           |blocksize max 24576 bytes |
| vorbis     | y           | y                           | <=196Kbit/s              |
| m4a        | y           | y                           |                          |
| opus       | y           | y                           |                          |

<br>

***
Wiring
![schematic](https://github.com/user-attachments/assets/77ce30d2-acb1-4b5d-a9d6-4f1e3d56e385)

***
Impulse diagram
![Impulse diagram](https://github.com/schreibfaul1/ESP32-audioI2S/blob/master/additional_info/Impulsdiagramm.jpg)
***
Yellobyte has developed an all-in-one board. It includes an ESP32-S3 N8R2, 2x MAX98357 and an SD card adapter.
Documentation, circuit diagrams and examples can be found here: https://github.com/yellobyte/ESP32-DevBoards-Getting-Started
![image](https://github.com/user-attachments/assets/4002d09e-8e76-4e08-9265-188fed7628d3)

