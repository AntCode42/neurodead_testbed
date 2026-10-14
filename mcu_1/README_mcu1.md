# MCU1 : domaine sécurité et actionnement

**Version** : 0.5 (2026-10-14). Remplace le README v0.4 du 12/10 et la spec `mcu1_interface.md` v0.1 du 13/10.
**Cible** : NUCLEO-F446RE (Cortex-M4F, monocœur). Outils : STM32CubeIDE. RTOS : FreeRTOS préemptif à priorités fixes.

**Légende** : **D** = décidé par Antoine · **P** = proposé, à valider · **O** = ouvert.
Les identifiants M1-xx sont provisoires, à aligner avec le registre de décisions du projet. D-005 et D-007 existent déjà.

---

## 1. Rôle

MCU1 est le **domaine moteur et sécurité**. Il doit rester sûr même si RK1 (Linux, ROS2) plante.

| MCU1 fait | MCU1 ne fait pas |
|---|---|
| Convertit les vitesses par roue (rad/s) en QPPS et les envoie aux 3 RoboClaw | PID de vitesse : délégué aux RoboClaw |
| Génère S3 (e-stop dur), rafraîchit l'IWDG, détient le verrou et la machine d'états | Mixage cmd_vel vers roues : fait par RK1 |
| Lit les HC-SR04, applique l'arrêt doux ou dur | Pilotage des Dynamixel : U2D2 direct sur RK1 |
| Relaie l'odométrie à RK1, publie l'état de sécurité | micro-ROS : écarté (voir M1-03) |

```
Sol ──radio──► MCU3 (Ada, CCSDS)
                 │ GPIO ESTOP_REQ + UART
                 ▼
MCU2 ── GPIO TILT_OK + UART (jeton) ──►┌────────┐ UART ×3 ┌──────────┐
(EKF, attitude)                        │  MCU1  ├────────►│ RoboClaw │
RK1 ◄── UART 3,3 V, trames + CRC ─────►│        │         └──────────┘
(passerelle ROS2)                      │        ├─ timer OC ──► S3 (e-stop RoboClaw)
HC-SR04 ×N ───────────────────────────►└────────┘

Kill switch : interrupteur principal, coupe l'alimentation de tout le rover (hors firmware)
```

---

## 2. Journal des décisions

| ID | Décision | Alternative écartée | Justification | Date | Statut |
|---|---|---|---|---|---|
| M1-01 | FreeRTOS préemptif, priorités fixes | Zephyr | Natif CubeIDE (CMSIS-RTOS v2) | 12/10 | **D** |
| M1-02 | PID dans les RoboClaw, mixage sur RK1, MCU1 convertit en QPPS | PID 1 kHz sur MCU1 | 3 RoboClaw sur un UART à 115200 baud : ~0,87 ms par trame, soit ~2,6 ms pour 3, hors budget de 1 ms | 02/10 et 12/10 | **D** |
| M1-03 | Lien RK1 : UART 3,3 V (header GPIO du nœud 1, Turing Pi 2.5), trame série custom avec CRC, passerelle ROS2 sur RK1 | micro-ROS | Garde le domaine de sécurité isolé, WCET borné, pas d'allocation dynamique. Coût accepté : travail annexe | 12/10 | **D** |
| M1-04 | S3 généré par un timer en output-compare (niveau statique) et IWDG | tâche RTOS périodique | La sortie de sécurité ne dépend ni du CPU ni de l'ordonnanceur | 12/10 | **D** |
| M1-05 | D-005 amendé : S3 en non-latching, verrou logiciel dans MCU1, pull-down externe obligatoire. **S3 = tout arrêt dur, radio incluse.** Trou et obstacle = vitesse 0 par série | Mode latching du RoboClaw | Le mode latching ne se libère qu'en coupant l'alimentation du RoboClaw (test banc 12/10) | 12/10, amendé 14/10 | **D** |
| M1-06 | Le kill switch est l'interrupteur principal et coupe l'alimentation de tout le rover | Simple signal vers MCU1 | Seul arrêt indépendant de tout logiciel | 14/10 | **D** |
| M1-07 | MCU2 → MCU1 en direct : GPIO `TILT_OK` + jeton de validité renouvelé (UART) | Passage par RK1 | Retire RK1 de la chaîne de sécurité. Coût : une interface en plus | 13/10 | **D** (principe), **P** (valeurs) |
| M1-08 | Arrêt radio : MCU3 → MCU1 en direct (GPIO + trame série), jamais via RK1 | Passage par RK1 | Pas de dépendance à Linux pour l'arrêt | 13/10 | **D** |
| M1-09 | Réarmement : sol uniquement, via MCU3. MCU1 refuse tant qu'une cause d'arrêt est active, aucun forçage | Forçage depuis le sol | L'opérateur voit moins que les capteurs. Limite connue : un blocage permanent se résout sur site | 13/10 | **D** |
| M1-10 | MCU3 obligatoire pour les missions autonomes seulement. Variante de firmware choisie **à la compilation** : `FW_VARIANT=AUTO` ou `DEV`. En DEV, réarmement par la manette | MCU3 obligatoire pour tout usage ; drapeau modifiable à chaud | Une télé-opération au jardin se fait avec le kill switch à portée | 14/10 | **D** |
| M1-11 | Heartbeat MCU3 perdu (AUTO) = arrêt dur. Heartbeat RK1 ou jeton MCU2 perdu = arrêt doux, puis arrêt dur si l'arrêt doux se **répète** (N fois dans une fenêtre T) | Arrêt dur immédiat sur tout heartbeat | MCU3 est un canal de sécurité non récupérable par logiciel. Un heartbeat RK1 manquant peut venir de la charge SLAM, et l'arrêt dur immobiliserait le rover en zone morte radio | 14/10 | **D** |
| M1-12 | Arrêt doux à reprise automatique, avec 5 conditions (section 5) | Verrou jusqu'au réarmement sol | Éviter d'immobiliser le rover pour une cause qui disparaît | 14/10 | **D** |
| M1-13 | Perte de lien radio sans arrêt actif : la mission continue. Perte de localisation : arrêt et HOLD, radio ou pas | Arrêt systématique | Zones mortes radio (carrière, tunnel) | 13/10 | **D** |
| M1-14 | Ultrasons HC-SR04 : un seul capteur actif, celui du sens de marche. 5 en main, nombre final après matrice de couverture. Seuils et période réglés sur châssis monté | Mesure des 4 capteurs en continu | Réduit le délai entre deux mesures du capteur utile | 12/10, 14/10 | **D** |

**Contexte système** : vitesse max 1,5 m/s (plafond), croisière 1 m/s (**D**).

---

## 3. Interfaces

### Entrées

| # | Source | Support | Contenu | Période | Si absent ou invalide | Statut |
|---|---|---|---|---|---|---|
| E1 | RK1 | UART, trames avec CRC | `CMD_WHEELS` (rad/s par roue), `HB_RK1` | 20 ms | Après T_rk1 : arrêt doux | **D** (lien), **P** (T_rk1) |
| E2 | MCU2 | GPIO `TILT_OK` (haut = OK, pull-down externe) | Inclinaison sous le seuil d'arrêt X2 | niveau | Bas : arrêt doux | **P** |
| E3 | MCU2 | UART | Jeton : compteur, roll, pitch, horodatage, santé capteurs | 20 ms (à fixer) | Pas de jeton frais après T_jeton : arrêt doux | **D** (principe), **P** (valeurs) |
| E4 | MCU3 | GPIO `ESTOP_REQ` (EXTI, haut = arrêt, pull-down externe) | Ordre d'arrêt radio validé | événement | Fil coupé : pas d'arrêt par ce chemin. La trame E5 porte le même ordre | **P** |
| E5 | MCU3 | UART bidirectionnelle | `ESTOP(source, seq)`, `ARM_REQ`, `ARM_CONFIRM(nonce)`, `HB_MCU3` | événement et HB | En AUTO, HB_MCU3 perdu : arrêt dur | **D** |
| E7 | RoboClaw ×3 | UART (packet serial) | Encodeurs, statut | 20 ms | Sans réponse : arrêt doux, escalade dur si vitesse 0 non confirmée | **P** |
| E8 | HC-SR04 ×N | Trig / écho (capture de timer) | Distances (capteur du sens de marche) | à mesurer | Capteur muet : arrêt doux dans ce sens | **P** |

E6 (kill switch) est supprimé : il coupe l'alimentation, MCU1 n'a plus à le lire.

### Sorties

Les sorties sont numérotées O1 à O5 pour éviter la confusion avec la broche S3 du RoboClaw.

| # | Destination | Support | Contenu | Période | Statut |
|---|---|---|---|---|---|
| O1 | RoboClaw ×3 | UART bus | Consignes QPPS par roue | 20 ms | **D** |
| O2 | RoboClaw ×3 | Timer OC vers S3, niveau statique | E-stop dur, non-latching, verrou logiciel, pull-down externe | | **D** |
| O3 | RK1 | UART | `ESTOP_STATE` : état, masque de cause, compteur, `FW_VARIANT`, cause de l'arrêt doux ; odométrie ; distances US ; statut des liens MCU2 et MCU3 | 20 ms, en continu (un état, pas un événement) | **D** (état), **P** (contenu) |
| O4 | MCU3 | UART | État, masque des conditions bloquantes, nonce de réarmement (pour le sol) | événement + 100 ms | **P** |
| O5 | IWDG | registre | Rafraîchi seulement si chaque tâche critique a signalé son passage | 10 ms | **P** |

---

## 4. Machine d'états et niveaux d'arrêt

```
BOOT ─► ESTOP_LATCHED ──ARM_REQ valide + conditions OK──► ARM_PENDING
              ▲                                              │ ARM_CONFIRM(nonce)
              └────────────── tout arrêt dur ─────────── ARMED ◄┘
```

- Démarrage ou reset : toujours `ESTOP_LATCHED`, jamais `ARMED` (**D**).
- Réarmement en deux temps avec nonce à durée de vie courte (**P**).
- Une condition illisible compte comme bloquante (**P**).

| Niveau | Déclencheurs | Action |
|---|---|---|
| **Doux** (reprise automatique) | `TILT_OK` bas, trou ou obstacle, défaut RoboClaw qui répond, HB_RK1 perdu, jeton MCU2 expiré | Vitesse = 0 par UART |
| **Dur** | Arrêt radio, watchdog, vitesse 0 non confirmée, HB_MCU3 perdu (AUTO), arrêt doux répété N fois dans T | S3 + verrou `ESTOP_LATCHED` |
| **Hors firmware** | Kill switch | Coupure de l'alimentation |

**Conditions bloquantes du réarmement** (masque renvoyé au sol) : vitesse non nulle, inclinaison au-dessus du seuil de relâchement (hystérésis et durée de stabilité), HB_RK1 absent, jeton MCU2 expiré, HB_MCU3 absent (AUTO), watchdog ou autotest en défaut.

---

## 5. Arrêt doux : conditions de reprise automatique (M1-12)

1. La reprise n'est possible que si l'arrêt a été **confirmé**. Un arrêt non confirmé escalade déjà vers S3.
2. Le seuil de relâchement est distinct du seuil d'arrêt, avec hystérésis et durée de stabilité.
3. Ultrasons : N lectures libres consécutives et durée de maintien minimale.
4. Reprise sur **consigne fraîche** uniquement. MCU1 ne rejoue jamais la dernière consigne reçue pendant l'arrêt. Reprise par rampe.
5. Compteur d'arrêts doux répétés : au-delà de N dans une fenêtre T, escalade vers l'arrêt dur.

MCU1 publie la cause de l'arrêt doux à RK1 toutes les 20 ms.

---

## 6. Modes AUTO et DEV (M1-10)

| | AUTO | DEV |
|---|---|---|
| Choix | à la compilation (`FW_VARIANT`) | à la compilation |
| MCU3 | requis, heartbeat surveillé | absent toléré |
| Arrêt radio | oui | non (kill switch seul) |
| Réarmement | sol via MCU3 | manette, en deux temps avec nonce |

Garde-fous proposés (**P**) pour la manette en DEV :
- Le code de réarmement par RK1 **n'existe pas** dans le binaire AUTO.
- Réarmement en deux temps (combinaison de 2 boutons, puis une autre) avec le nonce de MCU1.
- MCU1 refuse tant qu'une cause est active. Vitesse et consigne nulles au moment du réarmement.
- Le nœud de mission RK1 refuse le mode autonome si `FW_VARIANT=DEV`. La variante est annoncée dans chaque `ESTOP_STATE` et journalisée par RK1.

**Limites connues** (à garder dans l'ADR) :
- Un blocage permanent (IMU morte, cause d'arrêt coincée) se résout par intervention physique.
- Si RK1 meurt définitivement, le rover reste à l'arrêt doux et la règle de retour au départ, qui vit sur RK1, ne s'exécute pas. MCU1 signale « RK1 absent » à MCU3 pour la balise.
- Le kill switch coupe RK1 et le Jetson brutalement : risque de corruption de fichiers et perte de logs.

---

## 7. Architecture des tâches : en cours de décision

Le README v0.4 prévoyait H0 + T1 à T6, la spec v0.1 en proposait 4. Rien n'est figé : voir les questions O1 à O4 de la section 9.

| Élément | Nature | Rôle | Statut |
|---|---|---|---|
| H0 | matériel (timer OC) | Maintient S3 à l'état sûr, hors ordonnanceur | **D** |
| Superviseur (T1 + T2) | RTOS, priorité la plus haute, ~10 ms | Seul écrivain de l'état de sécurité, IWDG, verrou, machine d'états | **O** (fusion) |
| Ultrasons (T4) | RTOS, périodique | Tir du capteur actif, trou et obstacle | **P** (période à mesurer) |
| Pilote RoboClaw (T5) | RTOS, 20 ms | Seul propriétaire de l'UART RoboClaw | **P** |
| Lien RK1 (T3) | RTOS, 20 ms | Trames, consignes en entrée, état en sortie | **P** |
| Liens internes (MCU2, MCU3) | RTOS, événementiel | Parsing et CRC des trames, messages validés vers le superviseur | **O** |
| Diagnostic (T6) | RTOS, basse priorité | Compteurs d'erreurs, hauts de pile, debug sur USART2 | **O** |

Règles de construction : un propriétaire par ressource partagée (pas de mutex), ISR minimales (DMA puis notification de tâche), écrivain unique de l'état de sécurité, arrêt par défaut.

---

## 8. Paramètres à mesurer (aucune valeur inventée)

| Paramètre | Contrainte | Méthode |
|---|---|---|
| T_rk1 | supérieur à la gigue mesurée de RK1 | `cyclictest` sous charge SLAM |
| T_jeton | supérieur à la gigue, très inférieur au temps pour que la pente devienne dangereuse | mesure et géométrie du rover |
| N et T du compteur d'arrêts doux répétés | à fixer | essais |
| X2 (seuil d'inclinaison), seuil de relâchement, durée de stabilité | issus de la géométrie réelle | après assemblage |
| Latence `ESTOP_REQ` vers S3 bas | cible < 1 ms (à confirmer) | oscilloscope, broche de debug |
| Comportement physique de S3 sur le RoboClaw | freinage ou roue libre | banc, avec un moteur |
| Durée de vie du nonce, délai de confirmation « vitesse 0 » | quelques secondes (à fixer) | essais |
| N lectures libres et durée de maintien des ultrasons | dépend du terrain | essais sur châssis |
| Période de tir des ultrasons (T_slot) et seuils | écho fantôme, empreinte du cône | sur châssis monté |
| Période et gigue de chaque tâche | RMA | analyseur logique ou GPIO + oscilloscope |

---

## 9. Points ouverts

| # | Question | Éléments |
|---|---|---|
| O1 | T1 et T2 fusionnés ou séparés ? | Avis : fusionner si le rafraîchissement de l'IWDG dépend des check-ins. Mesurer le WCET au RMA |
| O2 | Qui parse les trames de MCU2 et MCU3 ? | Option : tâche dédiée sous le superviseur. Le parsing n'est pas sur le chemin de l'arrêt dur (le GPIO est le chemin rapide) |
| O3 | Chemin de l'arrêt radio : ISR qui force S3 puis notifie, ou superviseur qui force S3 ? | L'ISR directe suit le principe M1-04. Priorité d'interruption plus urgente que le seuil masqué par FreeRTOS, pas d'appel à l'API FreeRTOS dans l'ISR |
| O4 | Tâche T6 dans le périmètre de fin octobre ? | Hors RMA strict, comptée dans la charge CPU |
| O5 | Nombre de HC-SR04 : 5 ou 6 ? Position de l'arrière : châssis ou mât du LIDAR ? | Matrice de couverture (mode × trace de roue) à remplir avec les cotes du CAD. Supports réglables en angle et en hauteur recommandés |
| O6 | Matériel du kill switch | Coupe le bus batterie après la jonction des deux LiPo, calibré pour le courant continu et les arcs sur charge inductive. Un contacteur est une option. À ajouter à la liste d'achats |
| O7 | Une sortie timer pour les 3 S3 en parallèle ? | Vérifier courant et niveau logique |
| O8 | Mode manuel : perte de manette = arrêt ? | **P**, à trancher pour J2 |
| O9 | Authentification des trames (MAC) pour l'arrêt et le réarmement | Hors périmètre, à assumer ou non |
| O10 | Format de trame (SOF, longueur, type, séquence, charge utile, CRC16) | Document séparé |
| O11 | Brochage CubeMX : ~21 à 23 signaux, lignes EXTI à numéros distincts | Nombre de broches libres du LQFP64 non vérifié. L'écho 5 V : diviseur 1 kΩ / 2 kΩ ou broches tolérantes |
| O12 | Timeout série du RoboClaw | Réglage qui arrête les moteurs si MCU1 se tait : à vérifier dans la configuration |
| O13 | Odométrie : ticks bruts avec horodatage matériel, conversion sur RK1 (rayon de roue 96 mm) | **P** (proposé le 12/10) |

---

## 10. Critères « prêt à monter » (J1, fin octobre)

Proposés, mis à jour avec les décisions du 13 et 14/10. Un seul cas non testé = critère non rempli.

| # | Critère | Vérification |
|---|---|---|
| 1 | Build depuis un clone propre | 0 warning |
| 2 | S3 à l'état sûr au reset | pull-down externe présent, état mesuré au boot |
| 3 | Arrêt dur | radio, watchdog et vitesse 0 non confirmée testés séparément |
| 4 | Kill switch | test matériel : à la mise sous tension, aucun moteur ne bouge avant `ARMED` |
| 5 | IWDG | une tâche bloquée volontairement provoque un reset et un retour à S3 à l'état sûr |
| 6 | Réarmement | démarrage en `ESTOP_LATCHED` ; refusé tant qu'une cause est active |
| 7 | Arrêt doux | reprise automatique selon la section 5 ; l'escalade vers l'arrêt dur fonctionne |
| 8 | Lien RoboClaw | consignes QPPS envoyées, encodeurs relus |
| 9 | Liens RK1, MCU2, MCU3 | trames avec CRC, trame corrompue rejetée, perte de heartbeat gérée |
| 10 | Variantes | mission autonome refusée en DEV ; `FW_VARIANT` annoncée dans `ESTOP_STATE` |
| 11 | Ultrasons | capteur actif selon le sens de marche. Seuils mesurés sur châssis : peuvent glisser en J2 |
| 12 | Timing | période et gigue mesurées pour chaque tâche |

**Périmètre** : le noyau MCU1 est l'objectif de fin octobre (13 à 15 demi-journées estimées, ±50 %, pour 8 jours disponibles). Les ultrasons sont sacrifiables vers J2. La place de MCU2 en J1 ou en J3, et de l'EKF dans J1, reste à décider.
