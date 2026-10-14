# ADR MCU1 : architecture sécurité et actionnement

**Projet** : NEURODEAD-TESTBED · **Cible** : NUCLEO-F446RE, FreeRTOS · **Version** : 1.0 (2026-10-14)
**Source** : sessions des 02/10, 12/10, 13/10 et 14/10/2026. Complète `README_mcu1.md` v0.5, qui contient les interfaces, la machine d'états et les points ouverts.

**Statuts** : *Accepté* = décidé par Antoine · *Proposé* = suggéré, non validé · *Ouvert* = à trancher.
**Identifiants** : ADR-M1-xx est provisoire, à aligner avec le registre du projet (`DECISIONS_PROJET.md`). La correspondance avec les entrées M1-xx du README est donnée dans chaque ADR.

| ADR | Titre | Statut |
|---|---|---|
| M1-01 | RTOS et partition des responsabilités | Accepté |
| M1-02 | Lien RK1 : trame série custom, pas micro-ROS | Accepté |
| M1-03 | E-stop S3 : timer matériel, non-latching, verrou logiciel (amende D-005) | Accepté |
| M1-04 | Kill switch = coupure de l'alimentation | Accepté (matériel à spécifier) |
| M1-05 | Liens de sécurité directs MCU2 → MCU1 et MCU3 → MCU1 | Accepté (valeurs proposées) |
| M1-06 | Réarmement : sol uniquement, refus tant qu'une cause est active | Accepté (nonce proposé) |
| M1-07 | Politique de heartbeat : arrêt doux puis dur | Accepté |
| M1-08 | Arrêt doux à reprise automatique | Accepté |
| M1-09 | Variantes de firmware AUTO et DEV | Accepté (garde-fous proposés) |
| M1-10 | Perte de lien radio et perte de localisation | Accepté (mode manuel proposé) |
| M1-11 | Ultrasons : capteur actif selon le sens de marche | Principe accepté, périmètre ouvert |

---

## ADR-M1-01 : RTOS et partition des responsabilités

- **Statut** : Accepté (02/10 et 12/10). Correspond à M1-01 et M1-02.
- **Contexte** : les encodeurs sont câblés sur les RoboClaw, pas sur MCU1. Trois RoboClaw partagent un UART à 115200 baud : ~0,87 ms par trame, soit ~2,6 ms pour les trois. Même à 460800 baud la marge reste faible. Un PID à 1 kHz sur MCU1 est donc hors budget.
- **Décision** : FreeRTOS préemptif à priorités fixes. Les RoboClaw ferment leur boucle de vitesse (réglages via Motion Studio). Le mixage cmd_vel vers roues se fait sur RK1. MCU1 convertit les vitesses en QPPS et les envoie à 20 ms.
- **Alternatives écartées** : PID logiciel à 1 kHz sur MCU1 ; Zephyr (envisagé, non retenu : FreeRTOS est natif dans CubeIDE via CMSIS-RTOS v2).
- **Justification** : le budget UART tranche. Chaque domaine garde ce qu'il maîtrise : boucle rapide dans le contrôleur moteur, géométrie sur RK1, sécurité sur MCU1.
- **Conséquences** :
  - Le RoboClaw est une boîte noire (firmware propriétaire) : on **mesure** sa latence, on ne l'analyse pas.
  - L'analyse RMA (Rate Monotonic) ne s'applique qu'à MCU1. RK1 et le Jetson relèvent du temps réel souple, avec mesures.
- **À réviser si** : la latence mesurée de la chaîne RoboClaw est incompatible avec la distance d'arrêt exigée.

## ADR-M1-02 : Lien RK1, trame série custom, pas micro-ROS

- **Statut** : Accepté (12/10). Correspond à M1-03.
- **Contexte** : RK1 est le nœud 1 d'une Turing Pi 2.5. Le header GPIO 40 broches du nœud offre un UART 3,3 V.
- **Décision** : UART 3,3 V, trame custom (synchronisation, longueur, type, séquence, charge utile, CRC), avec un nœud passerelle ROS2 sur RK1.
- **Alternatives écartées** : micro-ROS ; USB CDC.
- **Justification** : le domaine de sécurité reste isolé, avec un temps d'exécution maximal borné et sans allocation dynamique.
- **Conséquences** :
  - Travail annexe accepté explicitement (changement de périmètre assumé) : spécification de trame, passerelle ROS2, tests.
  - Le format de trame est à rédiger dans un document séparé.
- **À réviser si** : la charge de maintenance de la passerelle compromet les jalons.

## ADR-M1-03 : E-stop S3, timer matériel, non-latching, verrou logiciel

- **Statut** : Accepté. Correspond à M1-04 et M1-05. **Amende D-005** (14/10).
- **Contexte** :
  - La broche S3 du RoboClaw est active à l'état bas. Son pull-up interne a une valeur non documentée : un pull-down externe est obligatoire.
  - Test banc (12/10) : le mode latching ne se libère qu'en coupant l'alimentation du RoboClaw.
  - Une tâche RTOS périodique dépendrait du CPU et de l'ordonnanceur sur un monocœur.
- **Décision** :
  - S3 est généré par un timer en output-compare (niveau statique), hors ordonnanceur. Un IWDG couvre le cas « CPU bloqué, GPIO figé à l'état sûr ».
  - S3 est configuré en **non-latching**. Le verrou est logiciel, dans MCU1.
  - **S3 = tout arrêt dur** : arrêt radio, watchdog, vitesse 0 non confirmée, heartbeat MCU3 perdu, arrêt doux répété. Le kill switch n'y est plus (voir ADR-M1-04).
  - Trou et obstacle passent par une commande série vitesse = 0, pas par S3.
- **Alternatives écartées** : mode latching ; génération de S3 par une tâche RTOS.
- **Justification de l'amendement (arrêt radio sur S3)** : l'arrêt radio sert quand personne n'est à portée du rover. Il doit donc être **inconditionnel**, c'est-à-dire indépendant de l'UART et du RoboClaw. L'arrêt doux dépend des deux, S3 non.
- **Conséquences** :
  - Pull-down externe obligatoire sur S3.
  - À mesurer au banc : comportement physique de S3 (freinage ou roue libre), qui compte sur pente avec un réducteur 26,9:1.
  - Ouverts : une sortie timer pour les trois S3 en parallèle (courant, niveau logique) ; chemin de l'arrêt radio, ISR qui force S3 ou superviseur (README O3).
- **À réviser si** : le test banc montre un comportement de S3 incompatible avec l'arrêt sur pente.

## ADR-M1-04 : Kill switch = coupure de l'alimentation

- **Statut** : Accepté (14/10). Correspond à M1-06. Matériel à spécifier.
- **Contexte** : il faut un arrêt qui ne dépend d'aucun logiciel.
- **Décision** : le kill switch est l'interrupteur principal et coupe l'alimentation de tout le rover.
- **Alternative écartée** : simple signal d'entrée vers MCU1.
- **Justification** : seul arrêt indépendant de tout firmware, y compris MCU1.
- **Conséquences** :
  - MCU1 ne lit plus le kill switch. L'entrée E6 et la condition « kill switch actif » disparaissent.
  - À la remise sous tension, MCU1 démarre en `ESTOP_LATCHED`. Le relâchement du kill switch ne réarme pas.
  - Coupure brutale de RK1 et du Jetson : risque de corruption de fichiers et perte de logs. Coût connu, acceptable pour un arrêt d'urgence.
  - Matériel à spécifier : coupure du bus batterie après la jonction des deux LiPo, calibre en courant continu cohérent avec les limites des RoboClaw (provisoires), tenue à l'arc sur charge inductive. Un contacteur est une option. Article à ajouter à la liste d'achats.
  - Critère de test : à la mise sous tension, aucun moteur ne bouge avant `ARMED`.
- **À réviser si** : le dimensionnement du courant impose un contacteur piloté.

## ADR-M1-05 : Liens de sécurité directs MCU2 → MCU1 et MCU3 → MCU1

- **Statut** : Accepté (13/10). Correspond à M1-07 et M1-08. Valeurs proposées.
- **Contexte** :
  - Anti-retournement : l'inclinaison est mesurée sur MCU2.
  - Arrêt radio : la trame est validée sur MCU3.
  - RK1 tourne sous Linux (PREEMPT_RT et cœurs dédiés possibles). Un heartbeat seul ne couvre pas un nœud ROS2 bloqué, un message perdu ou une décision tardive sous charge SLAM.
- **Décision** :
  - MCU2 → MCU1 en direct : GPIO `TILT_OK` et jeton de validité renouvelé (compteur, attitude, horodatage, santé capteurs). L'absence du signal positif déclenche l'arrêt.
  - MCU3 → MCU1 en direct : GPIO `ESTOP_REQ` (chemin rapide) et trame série (contexte, compteur, source). Jamais via RK1.
  - MCU1 publie son état à RK1 **en continu** (20 ms) : un état, pas un événement, car une trame envoyée une seule fois peut se perdre.
- **Alternatives écartées** : MCU2 → RK1 → MCU1 avec jeton (défendable, mais impose une analyse de latence de RK1) ; arrêt radio via RK1.
- **Justification** : retire Linux de la chaîne de sécurité pour un coût d'une broche et d'un message. L'arrêt radio est une commande positive : une commande perdue ne s'exécute pas, donc l'absence de signal ne peut pas servir de déclencheur.
- **Formulation pour la documentation** : « arrêt radio = commodité opérationnelle ; sûreté = kill switch + arrêts autonomes ». La manette passe par RK1 : confort, pas argument de sécurité.
- **Conséquences** :
  - Faux arrêt = côté sûr, arrêt manqué = côté dangereux : CRC, compteur de séquence et motif d'arrêt répété sur au moins 2 trames (proposé).
  - Authentification (MAC avec clé partagée) : hors périmètre, à assumer ou non.
  - Latence `ESTOP_REQ` vers S3 bas : cible < 1 ms, à mesurer.
- **À réviser si** : la mesure montre que le chemin GPIO n'est pas plus rapide que la trame série.

## ADR-M1-06 : Réarmement, sol uniquement, refus tant qu'une cause est active

- **Statut** : Accepté (13/10). Correspond à M1-09. Nonce proposé.
- **Contexte** : arrêter est sûr, réarmer ne l'est pas. Un faux réarmement remet les moteurs sous tension.
- **Décision** :
  - En mode AUTO, le réarmement vient du sol via MCU3. Manette et kill switch n'ont aucune autorité.
  - MCU1 **refuse** tant qu'une cause d'arrêt est active et renvoie le masque des conditions bloquantes. Aucun forçage depuis le sol.
  - Démarrage ou reset de MCU1 : toujours `ESTOP_LATCHED`.
  - Réarmement en deux temps avec nonce (`ARM_REQ`, puis `ARM_CONFIRM(nonce)` avant expiration) : **proposé**.
  - Seuil de relâchement distinct du seuil d'arrêt (hystérésis) et durée de stabilité. Une condition illisible compte comme bloquante.
- **Alternatives écartées** : forçage depuis le sol (l'opérateur voit moins que les capteurs du rover) ; réarmement local de maintenance (cavalier ou bouton), écarté sur recommandation.
- **Justification** : le réarmement revient à l'autorité la plus éloignée du danger, avec plus de vérifications que l'arrêt.
- **Conséquences** :
  - **Limite connue** : une cause qui ne disparaît pas (IMU morte, par exemple) verrouille le rover. Un redémarrage n'aide pas. La récupération est physique, sur site.
  - Un rover immobilisé par une panne radio est un défaut accepté pour un démonstrateur, avec HOLD et balise périodique.
- **À réviser si** : la limite connue se produit en essais.

## ADR-M1-07 : Politique de heartbeat, arrêt doux puis dur

- **Statut** : Accepté (14/10). Correspond à M1-11.
- **Contexte** : tout heartbeat perdu pouvait déclencher l'arrêt dur, ce qui était le souhait initial. Un heartbeat absent est un symptôme, pas un diagnostic : pour RK1, une charge SLAM ou un nœud bloqué suffit.
- **Décision** :
  - Heartbeat MCU3 perdu (variante AUTO) : **arrêt dur** (S3 et verrou).
  - Heartbeat RK1 perdu ou jeton MCU2 expiré : **arrêt doux** (vitesse 0 par série), puis arrêt dur si l'arrêt doux se **répète** (N fois dans une fenêtre T, à mesurer).
- **Alternative écartée** : arrêt dur immédiat sur tout heartbeat.
- **Justification** :
  - L'arrêt dur se libère uniquement par un réarmement sol via MCU3. Une coupure de RK1 d'une seconde en zone morte radio immobiliserait le rover.
  - Si RK1 ou MCU2 tombe, MCU1 reste vivant et peut envoyer vitesse 0 lui-même. Si l'arrêt n'est pas confirmé, il escalade déjà vers S3.
  - MCU3 est différent : c'est un canal de sécurité (arrêt radio, balise), sa perte n'est pas récupérable par logiciel.
- **Conséquences** :
  - Si RK1 meurt définitivement, le rover reste à l'arrêt doux et la règle de retour au départ (qui vit sur RK1) ne s'exécute pas. MCU1 signale « RK1 absent » à MCU3 pour la balise. Limite connue.
  - Distinction à garder : une liaison radio coupée (zone morte) n'est pas un MCU3 muet (panne interne).
  - Paramètres à mesurer : T_rk1, T_jeton, N, T.
- **À réviser si** : les essais montrent des arrêts doux répétés sans cause réelle.

## ADR-M1-08 : Arrêt doux à reprise automatique

- **Statut** : Accepté (14/10). Correspond à M1-12.
- **Décision** : l'arrêt doux se relâche seul quand la cause disparaît, aux conditions suivantes :
  1. L'arrêt a été confirmé. Un arrêt non confirmé escalade vers S3.
  2. Seuil de relâchement distinct du seuil d'arrêt, avec hystérésis et durée de stabilité.
  3. Ultrasons : N lectures libres consécutives et durée de maintien minimale.
  4. Reprise sur consigne fraîche uniquement, jamais la dernière consigne rejouée. Reprise par rampe.
  5. Compteur d'arrêts répétés : au-delà de N dans une fenêtre T, escalade vers l'arrêt dur.
- **Alternative écartée** : verrou jusqu'au réarmement sol pour tout arrêt.
- **Justification** : éviter d'immobiliser le rover pour une cause transitoire, sans remettre en mouvement un état incertain. Sur gravier, un écho qui sautille ne doit pas relancer le rover vers un trou.
- **Conséquences** : MCU1 publie la cause de l'arrêt doux à RK1 toutes les 20 ms, pour que le nœud de mission ne cumule pas de timeouts. Valeurs à mesurer.

## ADR-M1-09 : Variantes de firmware AUTO et DEV

- **Statut** : Accepté (14/10). Correspond à M1-10. Garde-fous proposés.
- **Contexte** : MCU3 devient obligatoire pour les missions autonomes. En téléopération au jardin, tu as le kill switch à portée.
- **Décision** :
  - Variante choisie **à la compilation** : `FW_VARIANT=AUTO` ou `DEV`.
  - AUTO : MCU3 requis et surveillé, arrêt radio actif, réarmement par le sol.
  - DEV : MCU3 absent toléré, pas d'arrêt radio (kill switch seul), réarmement par la manette (exception à ADR-M1-06, car l'opérateur est sur place).
  - Garde-fous proposés pour DEV : le code de réarmement par RK1 n'existe pas dans le binaire AUTO ; réarmement en deux temps avec nonce ; vitesse et consigne nulles au moment du réarmement ; le nœud de mission RK1 refuse le mode autonome si DEV ; `FW_VARIANT` annoncée dans chaque `ESTOP_STATE` et journalisée.
- **Alternatives écartées** : drapeau modifiable à l'exécution (finirait en production) ; MCU3 obligatoire pour tout usage ; réarmement par câble de debug (moins pratique).
- **Justification** : un code absent du binaire ne peut pas s'exécuter, contrairement à un test à l'exécution.
- **Conséquences** :
  - La manette passe par RK1 (Linux, USB) : un nœud bloqué ou un bouton coincé ne doit pas pouvoir réarmer, d'où les garde-fous.
  - Test d'acceptation à ajouter : mission autonome refusée en DEV.
- **À réviser si** : un incident de réarmement involontaire se produit en DEV.

## ADR-M1-10 : Perte de lien radio et perte de localisation

- **Statut** : Accepté (13/10). Correspond à M1-13. Mode manuel proposé. Rattaché à D-007.
- **Contexte** : le site (carrière, avec un réseau de tunnels) coupe la radio par nature. Arrêter à chaque coupure ferait échouer la mission là où elle doit marcher.
- **Décision** :
  - Perte de lien radio sans arrêt actif : la mission autonome continue, bornée par le délai X (constante du rover, bornée par la charge de la batterie). En cas d'échec du retour : HOLD et balise radio périodique.
  - Un arrêt déjà verrouillé suspend la mission : la perte de radio ensuite ne la reprend pas.
  - Perte de localisation : arrêt et HOLD, radio ou pas. Cette décision se prend dans le nœud de mission de RK1.
  - Mode manuel : perte de manette = arrêt (**proposé**, à trancher pour J2).
- **Alternative écartée** : arrêt systématique à la perte de lien.
- **Justification** : la poursuite n'est défendable que si le rover reste sûr sans l'opérateur. Le cas le plus dangereux, perdu et sans radio, est donc traité par l'arrêt.
- **Conséquences** : la sûreté repose sur le kill switch et les arrêts autonomes. X et l'énergie de retour se **mesurent** (J2, wattmètre sur le bus batterie).

## ADR-M1-11 : Ultrasons, capteur actif selon le sens de marche

- **Statut** : principe Accepté (12/10) ; nombre, géométrie et seuils **ouverts**. Correspond à M1-14.
- **Contexte** : les sites sont fermés au public et tu y es seul, donc l'obstacle dynamique est secondaire. Le rôle principal est la détection de trous et falaises. À 1 m/s, un cycle de 60 ms représente 6 cm parcourus, un cycle de 200 ms 20 cm, avant même le freinage.
- **Décision** :
  - HC-SR04, un seul capteur actif, celui du sens de marche. Interface prévue pour N capteurs configurables (masque par mode et âge de mesure).
  - Cinq capteurs en main (4 prévus à l'origine). Le nombre final (5 ou 6) dépend d'une matrice de couverture (mode × trace de roue) à remplir avec les cotes du CAD.
  - Seuils, angles et période réglés sur le châssis monté.
- **Alternative écartée** : mesure des quatre capteurs en continu.
- **Justification** : la période de rafraîchissement du capteur utile est le paramètre qui compte pour la distance parcourue. La rotation sur place est le cas le moins couvert : l'arrière semble sous-couvert avec un seul capteur (hypothèse à vérifier sur plan).
- **Conséquences** :
  - Supports de capteurs réglables en angle et en hauteur recommandés.
  - Passer à 5 ou 6 capteurs change les broches (~21 à 23 signaux), ajoute un diviseur 5 V vers 3,3 V par écho (1 kΩ / 2 kΩ) et modifie la répartition des rails 5 V.
  - Période 40 ou 60 ms : parquée, à fixer sur châssis (paramètre de configuration, pas une constante dans le code).
- **À réviser si** : la matrice de couverture montre des cases sans capteur.
