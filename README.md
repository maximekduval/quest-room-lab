# Room Twin

Prototype WebXR (Three.js) pour Meta Quest : lit le scan de la pièce, en construit
un **jumeau virtuel fidèle à l'échelle 1:1**, et fait fondre le passthrough pour te
laisser dans la version virtuelle de ton propre logement.

## Le pipeline

1. **Lecture** — plans sémantiques (`plane-detection`) + maillage 3D (`mesh-detection`)
2. **Compréhension** — chaque élément est typé : structure (sol/mur/plafond),
   ouverture (porte/fenêtre) ou volume (table/canapé/lit/rangement…)
3. **Reconstruction** — les plans deviennent de la géométrie volumétrique : murs
   extrudés en épaisseur, meubles extrudés de leur plateau jusqu'au sol
4. **Bascule** — grip droit : fondu de 0,9 s du réel vers le virtuel. Un dôme opaque
   masque le passthrough, la pièce virtuelle reste alignée au centimètre sur la vraie
5. **Export** — `room-scan.json` (données brutes) et `room-twin.glb` (modèle 3D
   ouvrable dans Blender)

Trois styles pour le jumeau : Argile, Blueprint, Néon (grip gauche).

## Ce que ça fait

| Fonction | API WebXR | Casques |
|---|---|---|
| Plans sémantiques (sol, mur, table, canapé, porte, fenêtre…) | `plane-detection` | Quest 2 / 3 / 3S / Pro |
| Maillage 3D de la pièce (murs + meubles, triangles) | `mesh-detection` | Quest 3 / 3S (scan de pièce) |
| Viser une surface réelle (réticule) | `hit-test` | tous |
| Poser un repère qui reste fixe dans le monde réel | `anchors` | tous |
| Passthrough couleur | `immersive-ar` | Quest 3 / 3S / Pro (N&B sur Quest 2) |
| Mains nues (pincement = gâchette) | `hand-tracking` | tous |
| Balles qui rebondissent sur les vrais murs et meubles | raycast sur la géométrie détectée | tous |

Les fonctions non supportées sont simplement désactivées : la démo reste utilisable
sur Quest 2 (plans uniquement, pas de maillage).

## Commandes dans le casque

- **Gâchette droite** : tirer une balle (elle rebondit sur la géométrie réelle et se pose)
- **Gâchette gauche** : poser un repère ancré sur la surface visée
- **Grip** : cycler l'affichage — plans + maillage / plans seuls / maillage seul / passthrough pur
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
