# Room Twin

Prototype WebXR (Three.js) pour Meta Quest : lit le scan de la pièce, en construit
un **jumeau virtuel fidèle à l'échelle 1:1**, et fait fondre le passthrough pour te
laisser dans la version virtuelle de ton propre logement.

## Le pipeline

1. **Lecture** — plans sémantiques (`plane-detection`) + maillage 3D (`mesh-detection`)
2. **Compréhension** — chaque élément est typé : structure (sol/mur/plafond),
   ouverture (porte/fenêtre) ou volume (table/canapé/lit/rangement…)
3. **Reconstruction** — les murs sont extrudés en épaisseur ; les meubles
   utilisent la boîte englobante que le casque fournit déjà (12 triangles),
   appariée à son plan sémantique pour hériter de son étiquette
4. **Ameublement** — chaque volume étiqueté est remplacé par un meuble généré
   procéduralement à ses dimensions réelles : canapé, table, étagère,
   bibliothèque, lit, rangement, écran, lampe, plante. Aucun asset externe,
   donc aucune dépendance ni licence
5. **Bascule** — grip droit : fondu de 0,9 s du réel vers le virtuel. Un dôme opaque
   masque le passthrough, la pièce virtuelle reste alignée au centimètre sur la vraie
6. **Export** — `room-scan.json` (données brutes) et `room-twin.glb` (modèle 3D
   ouvrable dans Blender)

Trois styles pour le jumeau : Argile, Blueprint, Néon (grip gauche).

## Ce que ça fait

| Fonction | API | Casques |
|---|---|---|
| Plans sémantiques (sol, mur, table, canapé, porte, fenêtre…) | `plane-detection` | Quest 2 / 3 / 3S / Pro |
| Boîte englobante 3D par meuble + maillage global de la pièce | `mesh-detection` | Quest 3 / 3S |
| Viser une surface réelle | `hit-test` | tous |
| Repère fixe dans le monde réel | `anchors` | tous |
| Passthrough couleur | `immersive-ar` | Quest 3 / 3S / Pro |
| Mains nues (pincement = gâchette) | `hand-tracking` | tous |
| **Flux des caméras du monde, en session immersive** | `getUserMedia` | Quest 3 / 3S |
| Balles rebondissant sur les vrais murs et meubles | raycast sur la géométrie détectée | tous |

### Accès caméra : deux chemins, un seul ouvert

Mesuré sur Quest 3, et contre-intuitif :

- le module WebXR **`camera-access` est refusé** — pas de `XRCamera` ;
- **`getUserMedia` fonctionne**, et le flux **survit à la session immersive**.

Le casque expose **trois caméras**. `camera 0, facing front` regarde vers le visage
de l'utilisateur, à travers les lentilles : image floue, surexposée, cerclée
d'anneaux infrarouges. Ce sont `camera 1` et `camera 2, facing back` qui voient la
pièce ; `facingMode: { ideal: 'environment' }` suffit à les obtenir. Les libellés ne
sont lisibles qu'après l'autorisation.

Ce que `getUserMedia` ne fournit pas : les **intrinsèques et la pose** de la caméra
dans le repère XR. Ni l'une ni l'autre n'est fatale — les deux se mesurent.

### Calibration optique : mesurée, pas supposée

La caméra est vissée au casque. Une rotation de tête, que WebXR donne au dixième de
degré, déplace donc l'image d'un nombre de pixels que la corrélation sait mesurer.
La pente pixels-par-radian **est** la distance focale ; le décalage temporel qui
minimise la dispersion du nuage **est** la latence du flux vidéo.

La mesure tourne seule dès que le flux est vivant : il suffit de tourner lentement
la tête, en lacet puis en tangage. Le HUD affiche l'avancement, le résultat part
dans l'export existant. Aucune mire, aucun clic, aucun réglage.

**On tourne la tête autour du cou, pas autour de l'œil.** Un lacet de 10° avec un
pivot à 10 cm derrière déplace déjà la caméra de 1,7 cm, et cette translation
gonfle la focale du facteur (1 + rayon / profondeur) — +3 % dans une pièce. Elle
n'est donc pas filtrée mais **modélisée** : second régresseur dans l'ajustement, ce
qui rend la **profondeur de scène mesurable** au passage. Rotation et translation
sont quasi colinéaires tant qu'on ne fait que tourner la tête ; un pas de côté
pendant la calibration les sépare. Le solveur signale lequel des deux régimes il a
pu résoudre plutôt que de rendre une focale silencieusement biaisée.

Le solveur est vérifié contre une vérité terrain synthétique — scène cylindrique à
profondeur finie, caméra sténopé de focale connue, tête pivotant autour du cou,
flux volontairement retardé de 80 ms :

| | cou seul | cou + pas de côté | vérité |
|---|---|---|---|
| focale | 160,0 px *(biais annoncé)* | 154,8 px | 155 |
| retard | 80 ms | 80 ms | 80 |
| profondeur de scène | non séparable | 3,00 m | 3,0 |
| rayon de rotation | 0,100 m | — | 0,10 |
| résidu de reprojection | 0,03° | 0,03° | ~0 |

Sur la même scène rendue en fisheye équidistant, le rapport flanc/centre tombe de
~0,95 à 0,843 — de quoi trancher la nature de l'objectif.

Ce que la calibration ne donne pas encore : la **rotation caméra→viewer**, trois
angles constants qu'un recalage du filaire de la pièce sur l'image fixera d'un
coup. La translation (~5 cm) vaut 1° de parallaxe à 3 m, sous le budget.

### Ce que le casque a répondu

Mesuré sur Quest 3, navigateur Oculus 150 :

| | |
|---|---|
| `requestVideoFrameCallback` **en session immersive** | oui — 329 rappels sur 345 |
| Retard capture → rappel, annoncé par le navigateur | 16,6 ms |
| Capteur | **1280×1280** — `1600×1200` renvoie du `1200×1280` en portrait |
| Meilleur format 4:3 servi | **1280×960 à 30 im/s** (et non 640×480) |
| Champ horizontal mesuré | ~85°, à confirmer sans le biais de parallaxe |
| Flancs / centre | 0,911 et 0,927 → **image rectilinéaire**, pas de fisheye |
| Les deux caméras du monde **simultanément** | **oui** — disparité 3,81 px, corrélation 0,94 |
| WebGPU | adaptateur Adreno 740, `shader-f16`, `subgroups`, 2 Go |
| Threads WASM | non — `crossOriginIsolated` est faux sur GitHub Pages |
| Cœurs annoncés | 3 |

La stéréo est donc ouverte : base sur profondeur = 0,022, soit une base de ~6,5 cm
pour une scène à 3 m — cohérent avec l'écartement des deux caméras RGB du casque.

### Recalage visuel — **B** en session

Le solveur automatique a rendu 174,5 px puis 228,6 px pour la même caméra sur deux
passages du même code : 31 % d'écart. La statistique ne les départage pas, la pièce
si. **B** (manette droite) ouvre un panneau qui projette le filaire du scan —
métrique, connu au centimètre — dans l'image de la caméra. On ajuste jusqu'à
superposition ; entre 70° et 85° de champ, l'écart est visuellement massif.

Le même geste livre ce que la corrélation ne peut pas donner : les **trois angles
constants entre la caméra et le viewer**, la caméra étant vissée au casque. La
miniature droite règle lacet et tangage, la gauche roulis et focale, **X** (manette
gauche) reprend la valeur mesurée. Le réglage est conservé d'une session à l'autre.

La projection est vérifiée sur quatorze conventions : un lacet de tête de 5° déplace
bien un point fixe de `centre + focale·tan(5°)` — exactement le modèle que la
corrélation ajuste, ce qui referme la boucle entre les deux moitiés.

### Sondes ajoutées

- **« Sonder l'optique »** (page de rapport) : échelle de résolutions réellement
  accordées par caméra, et surtout — les deux caméras « back » s'ouvrent-elles
  **simultanément** ? Si oui, la profondeur devient accessible par stéréo.
- **Sonde de plateforme** (au chargement) : WebGPU, WASM SIMD, threads,
  `requestVideoFrameCallback`, GPU. C'est elle qui décide si un modèle de vision
  peut tourner dans le casque. Note : GitHub Pages ne pose pas COOP/COEP, donc
  `crossOriginIsolated` est faux et les threads WASM sont hors jeu.

## Commandes dans le casque

- **Gâchette droite** : tirer une balle (elle rebondit sur la géométrie réelle et se pose)
- **Gâchette gauche** : poser un repère ancré sur la surface visée
- **Grip droit** : basculer réel ↔ virtuel · **Grip gauche** : changer de style
- **Bouton A** (manette droite) : caméra suivante parmi les trois
- Un petit **HUD** flotte au-dessus de la main gauche (compteurs + étiquettes détectées)

## Prérequis côté casque

1. Faire le **Space Setup** (Paramètres → Espace physique → Configuration de l'espace),
   sinon il n'y a aucun plan ni maillage à lire.
2. Ouvrir la page **en HTTPS** — WebXR refuse le HTTP simple. Un `http://192.168.x.x:8080`
   sur le réseau local ne marchera pas.

## Héberger

Le fichier `index.html` est autonome (une seule dépendance, Three.js via CDN).

GitHub Pages est le chemin le plus simple :

```bash
gh repo create quest-room-lab --public --source=. --push
gh api -X POST repos/:owner/quest-room-lab/pages -f "source[branch]=main" -f "source[path]=/"
```

L'URL `https://<user>.github.io/quest-room-lab/` s'ouvre ensuite directement dans le
navigateur du Quest.

## Pour aller plus loin

- **Occlusion** par la profondeur réelle (`depth-sensing`) : les objets virtuels passent
  derrière tes vrais meubles.
- **Physique complète** (Rapier / cannon-es) avec le maillage de la pièce en collider
  statique, au lieu du raycast simplifié.
- **Multi-joueurs colocalisé** via des ancres partagées.
- Passage à **Unity + Meta XR SDK** si tu veux publier sur le Horizon Store, viser
  90 Hz avec beaucoup d'objets, ou utiliser des fonctions non exposées à WebXR.
