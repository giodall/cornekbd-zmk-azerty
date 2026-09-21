# Analyse de la config ZMK — Corne AZERTY

> **État : appliqué le 2026-09-21** — commits `a5954c0` (récupération de la branche) et
> `b2b8f48` (fixes). Non poussé. Firmware non testé sur matériel.
> Ce document reste la référence du *pourquoi* de chaque choix ; voir
> [§9](#9-ce-qui-a-été-appliqué) pour l'écart entre l'analyse et ce qui a réellement été fait.
>
> Analyse initiale réalisée sur `master` @ `d1c112c`.
> Cible identifiée : **macOS, layout « Français » (Apple)** — et non AZERTY Windows.
> Indices : `NUBS` → `@` / `#` et `GRAVE` → `<` / `>`, deux positions spécifiques au layout Apple French.

**Verdict global :** la structure (4 couches, tri-layer, home row mods) est saine et cohérente.
Mais il y a **2 bugs qui coûtent des caractères**, **4 touches système absentes**, et les
home row mods sont configurés avec le flavor le plus hostile.

---

## Sommaire

1. [Correctness du mapping](#1-correctness-du-mapping)
2. [Touches manquantes](#2-touches-manquantes)
3. [Ergonomie — home row mods](#3-ergonomie--home-row-mods)
4. [Build & configuration](#4-build--configuration)
5. [La touche Fn sur Mac](#5-la-touche-fn-sur-mac)
6. [Plan d'action](#6-plan-daction)
7. [Questions ouvertes](#7-questions-ouvertes)
8. [Branche `test-gaming-debounce`](#8-branche-test-gaming-debounce)
9. [Ce qui a été appliqué](#9-ce-qui-a-été-appliqué)
10. [Incident du premier flash](#10-incident-du-premier-flash)

---

## 1. Correctness du mapping

### ① `-` (tiret) est introuvable dans toute la config — BLOQUANT

`config/corne.keymap:68`

Sur Apple FR, le tiret `-` correspond au keycode `N6`. Or `&kp N6` n'apparaît **nulle part**
dans le keymap (vérifié par `grep`). La touche annotée `-` dans le right_layer envoie
`&kp EQUAL`, qui tape `=`.

**Impact réel :** `kebab-case`, `--flags` CLI, `->`, tirets de dialogue, dates `2026-09-21`.

| Option | Détail |
|---|---|
| **(a) `&kp EQUAL` → `&kp N6` sur le right_layer** | **Recommandé.** 1 token. La place est déjà prévue et annotée pour ça. |
| (b) Le mettre sur le default layer, l.54 | Coûte une touche du layer de base pour un caractère déjà prévu ailleurs. |
| (c) Ne rien faire | Non. C'est bloquant au quotidien. |

### ② Le `+` du pavé numérique tape `§`

`config/corne.keymap:65`

```dts
&kp LS(FSLH)   // annoté "+" → tape en réalité "§"
```

Sur AZERTY, `FSLH` = `!`, donc `LS(FSLH)` = `§`. Le vrai `+` est `LS(EQUAL)`.

→ Remplacer par `&kp LS(EQUAL)`.

Combiné au fix ①, le right_layer devient un vrai pavé numérique cohérent :
`* 1 2 3 +` / `/ 4 5 6 -` / `. 7 8 9 0`.

### ③ Les bandeaux de commentaires sont désynchronisés

C'est **la cause racine** : c'est ce désalignement qui a masqué les bugs ① et ②.

| Ligne | Ce que dit le commentaire | Ce que le binding produit |
|---|---|---|
| `:53` / `:54` | `N` `,` `;` `:` `=` `-` | `N` `,` `;` `:` `!` `=` — décalé d'une case |
| `:81` / `:82` | `&kp N8` → `!` | tape `_` |
| `:84` / `:85` | `LS(EQUAL)` → `_` | tape `+` |

**Recommandation :** resynchroniser systématiquement. Sur un keymap AZERTY-sur-QWERTY,
le commentaire *est* la documentation — chaque `&kp X` produit un caractère qui n'a aucun
rapport visuel avec `X`. Si le commentaire ment, tu débugges à l'aveugle (ce qui vient
d'arriver deux fois).

### ④ 4 touches gaspillées en doublons sur le layer de base

`config/corne.keymap:48`, `:51`, `:54`, `:56`

| Touche | Occurrences |
|---|---|
| `ESC` | ligne 1 (`:48`) + pouce droit (`:56`) |
| `TAB` | ligne 2 (`:51`) + pouce droit (`:56`) |
| `BSPC` | ligne 1 (`:48`) + pouce gauche (`:56`) |
| `LSHIFT` | ligne 3 pos. 0 (`:54`) + pouce gauche (`:56`) + 2× en home row |

Soit ~10 % d'un clavier 42 touches immobilisé en redondance.

| Option | Détail |
|---|---|
| **(a) Garder les versions pouce, libérer les versions rangée** | **Recommandé à terme**, mais pas maintenant — voir ci-dessous. |
| (b) Statu quo | Défendable : les doublons rendent les couches plus tolérantes. |

> ⚠️ **À ne faire qu'après avoir stabilisé le reste.** Déplacer `ESC`/`TAB`/`BSPC` casse
> la mémoire musculaire pour un gain purement théorique. À traiter en dernier, si jamais.

---

## 2. Touches manquantes

### ① `&bootloader` / `&sys_reset` — absents

Pour flasher, tu es obligé de double-tapper le bouton reset physique du nice!nano, **sur les
deux moitiés**, à chaque build.

Place disponible : le tri_layer a 6 pouces en `&trans` + les 6 colonnes extérieures.

```dts
// tri_layer, ligne 1 — remplacer les &trans des extrémités
&bootloader  &kp F1 &kp F2 &kp F3 &kp F4 &bt BT_SEL 0   ...   &bootloader
```

### ② `&caps_word` — absent

Pour `CONSTANTES`, acronymes, `SCREAMING_SNAKE_CASE`. Trois `&none` inutilisés dans le
right_layer (`:65`, `:68`, `:71`, colonne 7) attendent exactement ça.

### ③ `&out OUT_TOG` — absent

Quand tu passes du Mac en Bluetooth à un poste en USB-C, aucun moyen de forcer la sortie.
`&bt BT_SEL 0..4` et `&bt BT_CLR` sont bien là, mais pas le toggle USB/BLE.

### ④ Fn / Globe — absent

Voir la [section 5](#5-la-touche-fn-sur-mac), c'est un sujet à part entière.

### Mineurs — à ignorer sauf besoin explicite

- `F13`–`F24` (raccourcis macOS dédiés)
- Mouse keys
- `INS`
- Screenshot direct (`Cmd+Shift+4` reste faisable via home row mod + left_layer)
- `£`, `µ`, `§` (ce dernier est déjà accessible via `Shift` + la touche `!` du default layer)

### Faux positif vérifié

Le tréma `¨` (pour `ë ï ü`) **n'est pas manquant** : il est accessible via `Shift` + la
touche `^` du left_layer (`LBKT`). Le `Shift` du default layer reste atteignable depuis
le left_layer grâce au `&trans` en `:88` pos. 0. Simplement non documenté.

---

## 3. Ergonomie — home row mods

C'est ici que le gain est le plus important.

### ① `flavor = "tap-preferred"` — le pire choix pour des HRM

`config/corne.keymap:26`

Avec `tap-preferred`, le hold ne se déclenche **que** si tu dépasses `tapping-term-ms`
(220 ms). Presser une autre touche ne le déclenche pas. Concrètement : pour faire `Cmd+C`,
tu dois immobiliser le doigt 220 ms **avant** de toucher `C`.

| Option | Détail |
|---|---|
| **(a) `balanced`** | **Recommandé.** Le hold se déclenche si l'autre touche est relâchée en premier. Réactif, faible taux de faux positifs. |
| (b) `hold-preferred` | Plus rapide encore, mais **exige** `hold-trigger-key-positions` sinon les faux positifs explosent. |
| (c) Statu quo | Acceptable si tu tapes lentement et n'as jamais de raté. |

### ② Aucun `hold-trigger-key-positions` (cross-hand)

`config/corne.keymap:22-31`

Un **seul** behavior `ht` est partagé par les deux mains → impossible de définir des
positions asymétriques. Résultat : rouler `a`+`s` rapidement peut déclencher `Shift+Alt`.

**Recommandation : splitter en `hml` / `hmr`**, chacun n'autorisant le hold que si la
touche suivante est sur la **main opposée**. C'est *le* changement qui rend les home row
mods fiables.

Numérotation des positions sur un Corne 42 touches :

```
 0  1  2  3  4  5      6  7  8  9 10 11
12 13 14 15 16 17     18 19 20 21 22 23
24 25 26 27 28 29     30 31 32 33 34 35
         36 37 38     39 40 41
```

```dts
behaviors {
    // Mods main GAUCHE → ne déclenchent le hold que vers la main droite
    hml: home_row_mod_left {
        compatible = "zmk,behavior-hold-tap";
        #binding-cells = <2>;
        flavor = "balanced";
        tapping-term-ms = <220>;
        quick-tap-ms = <150>;
        require-prior-idle-ms = <150>;
        hold-trigger-key-positions = <6 7 8 9 10 11 18 19 20 21 22 23 30 31 32 33 34 35 39 40 41>;
        hold-trigger-on-release;
        bindings = <&kp>, <&kp>;
    };

    // Mods main DROITE → ne déclenchent le hold que vers la main gauche
    hmr: home_row_mod_right {
        compatible = "zmk,behavior-hold-tap";
        #binding-cells = <2>;
        flavor = "balanced";
        tapping-term-ms = <220>;
        quick-tap-ms = <150>;
        require-prior-idle-ms = <150>;
        hold-trigger-key-positions = <0 1 2 3 4 5 12 13 14 15 16 17 24 25 26 27 28 29 36 37 38>;
        hold-trigger-on-release;
        bindings = <&kp>, <&kp>;
    };
};
```

Puis adapter les macros :

```dts
#define HRML(k1,k2,k3,k4) &hml LSHIFT k1 &hml LALT k2 &hml LCTRL k3 &hml LGUI k4
#define HRMR(k1,k2,k3,k4) &hmr RGUI k1  &hmr RCTRL k2 &hmr RALT k3  &hmr RSHIFT k4
```

> ⚠️ Le left_layer utilise aussi `HRML`/`HRMR` (`:85`) — la bascule est donc automatique,
> mais à vérifier au test.

### ③ `global-quick-tap` est déprécié

`config/corne.keymap:29`

Remplacé par `require-prior-idle-ms` dans ZMK récent. Comme `west.yml` pointe sur `main`,
ça peut devenir un warning puis une erreur de build **sans que tu changes une ligne**.

→ `require-prior-idle-ms = <150>;` (déjà intégré dans le snippet ci-dessus).

### ④ Conflit same-hand sur l'activation des couches

`config/corne.keymap:56`

- `&lt LEF BSPC` est sur le pouce **gauche**, mais le left_layer a des symboles sur la
  main gauche (`` ` ``, `/`, `#`, `@`).
- `&lt RIG ESC` est sur le pouce **droit**, mais le right_layer a toute la navigation sur
  la main droite.

Dans les deux cas : tu tiens et tu tapes avec la même main.

| Option | Détail |
|---|---|
| **(a) Ne rien faire** | **Recommandé.** Inhérent à un 3×6 avec 2 couches denses ; le coût de réapprentissage dépasse le gain. |
| (b) Réorganiser les couches en miroir | Gros chantier, bénéfice incertain. |

→ À revoir uniquement si tu sens concrètement la gêne à l'usage.

### ⑤ Aucun combo défini

`grep combos` → rien. Les combos sont le moyen le moins cher d'ajouter des touches sans
toucher aux couches (ex. `J`+`K` → `ESC`, `M`+`,` → `CAPS_WORD`). Piste d'amélioration,
pas un défaut.

---

## 4. Build & configuration

### ① `revision: main` — build non reproductible

`config/west.yml:8`

```yaml
revision: main   # ← suit HEAD upstream
```

Deux builds à deux semaines d'écart ne produisent pas le même firmware. Combiné au
`global-quick-tap` déprécié (§3-③), c'est une panne qui arrivera sans prévenir.

→ **Recommandé :** pinner un tag ou un SHA.

> ✅ **FAIT** (`c3aa604` + `91d95ab`) — mais seulement après que ce risque a provoqué une
> panne totale du clavier le jour même. Voir [§10](#10-incident-du-premier-flash).
>
> ⚠️ **Correction d'une affirmation fausse de ce document.** Il a d'abord été écrit ici que
> « ZMK ne publie pas de releases stables régulières », ce qui a servi à écarter
> l'épinglage. C'est inexact : **v0.1.0** (2024-12), **v0.2.0** / **v0.2.1** (2025-03) et
> **v0.3.0** (2025-08) existent. Rien n'empêchait d'épingler dès le départ.
>
> Pour monter de version plus tard : changer `revision` dans `west.yml`, le `board` dans
> `build.yaml` **et** le `uses:` du workflow, puis tester — les trois vont ensemble.

### ② `corne.conf` est entièrement commenté

Aucun tuning actif. Pistes utiles :

```conf
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_TIMEOUT=30000        # autonomie
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y       # portée / stabilité BLE
```

### ③ `CONFIG_ZMK_HID_CONSUMER_REPORT_USAGES_FULL=y` absent

Obligatoire **si** tu ajoutes `&kp GLOBE` : l'usage `0x029D` dépasse `0xFF` et serait
silencieusement ignoré autrement. C'est le défaut actuel de ZMK, mais l'expliciter protège
d'un changement de défaut upstream.

### ④ README vide

`README.md` contient `# cornekbd`. Sur un keymap AZERTY-sur-QWERTY à 4 couches, un schéma
ASCII des couches vaut de l'or dans 6 mois. Faible priorité — ce fichier-ci fait déjà une
partie du travail.

---

## 5. La touche Fn sur Mac

### Réponse courte : c'est impossible proprement, et ce n'est pas la faute de ZMK

La vraie touche Fn Apple n'est **pas** du HID standard. C'est la page vendeur
`AppleVendor Top Case` (`0xFF`), usage `0x03`, et macOS ne l'accepte **que** d'un
périphérique portant le Vendor ID Apple.

Ton nice!nano ne peut pas l'envoyer. Ce n'est pas une limite de ZMK, c'est un verrou côté
Apple — voir [zmkfirmware/zmk#947](https://github.com/zmkfirmware/zmk/issues/947), où les
mainteneurs expliquent qu'ils refusent d'usurper le VID Apple par défaut.

### Les trois options

#### (a) Remapper Caps Lock → Globe dans macOS — **RECOMMANDÉ**

Chemin exact sous **macOS Tahoe 26** (vérifié le 2026-09-21) :

```
Réglages Système
 └─ Clavier
     └─ bouton « Raccourcis clavier… »          ← le panneau est DERRIÈRE ce bouton
         └─ « Touches de modification »          (barre latérale, tout en bas)
             └─ Sélectionner un clavier : Corne  ← le Corne doit être connecté
                 └─ Touche Verr. maj. (⇪) : 🌐 Globe
```

> ⚠️ « Touches de modification » n'est **pas** une entrée du panneau Clavier : c'est un
> écran *à l'intérieur* de la fenêtre « Raccourcis clavier… ». C'est l'erreur qui a fait
> perdre du temps la première fois.

Deux pièges :

- Le Corne doit être **appairé et allumé** : le réglage est par périphérique, un clavier
  éteint n'apparaît pas dans le menu déroulant.
- macOS liste tous les périphériques d'entrée dans ce menu, **souris comprises**, et en
  présélectionne parfois une. Vérifier la sélection avant de remapper.

Puis dans le keymap, envoyer `&kp CAPS` depuis la touche que tu veux comme Fn.

**Tu récupères le comportement Fn/Globe complet** : picker d'emoji, dictée, `Fn+F1..F12`,
changement de source de saisie, et les raccourcis Globe (`🌐+F` plein écran, `🌐+N` centre
de notifications, `🌐+Q` note rapide…).

- ✅ Le remap étant scopé au clavier « Corne », tes autres claviers ne bougent pas.
- ✅ `&caps_word` (§2-②) applique un **Shift temporaire**, pas un Caps Lock HID : il
  continue de fonctionner après le remap. Les deux mécanismes sont indépendants.
- ⚠️ Tu perds en revanche le **vrai** Caps Lock (le verrouillage permanent) sur ce clavier.
- ⚠️ La dictée se synchronise avec ce réglage : le menu « Appuyer sur 🌐 pour… » de
  Réglages Système › Clavier › Dictée change en même temps.

**Si ça ne fonctionne pas :** Tahoe a remanié la couche de gestion des événements
modificateurs, avec des bugs d'entrée rapportés sur la série 26 (surtout 26.2 et 26.3.1).
Deux symptômes connus : voyant Caps Lock éteint, et Caps Lock qui change la langue de
saisie malgré l'option désactivée. Tester en désactivant Karabiner-Elements /
BetterTouchTool s'ils sont installés — ce sont les suspects habituels.

#### (b) `&kp GLOBE`

Existe bien dans ZMK (consumer usage `0x029D`), mais ne couvre que la fonction *Globe*,
**pas** la fonction *Fn*.

- Fiable sur iPadOS / iOS.
- [Résultats très inégaux sur macOS Sonoma](https://evantravers.com/articles/2023/11/09/zmk-added-the-apple-globe-key/) :
  plusieurs utilisateurs rapportent aucun effet, et un cas où ça ne fonctionne qu'en USB,
  pas en Bluetooth.
- Support partiel : utilisable seulement en touche « tenue », donc combos / tap-dance
  dessus se comportent mal.

→ À ne prendre que si tu vises surtout l'iPad.

#### (c) Ne rien faire

**Sérieusement envisageable.** Tu as déjà `F1`–`F12` sur le tri_layer, plus la luminosité,
le volume et les médias en accès direct. Fn ne t'apporterait plus que le picker d'emoji et
la dictée.

### Sources

- [zmkfirmware/zmk#947 — Apple Fn / Globe key](https://github.com/zmkfirmware/zmk/issues/947)
- [ZMK Added the Apple Globe Key — evantravers.com](https://evantravers.com/articles/2023/11/09/zmk-added-the-apple-globe-key/)
- [QMK Apple Fn — gist fauxpark](https://gist.github.com/fauxpark/010dcf5d6377c3a71ac98ce37414c6c4)
- [Liste des keycodes ZMK](https://zmk.dev/docs/keymaps/list-of-keycodes)

---

## 6. Plan d'action

| # | Action | Section | Effort | Risque |
|---|---|---|---|---|
| **0** | **Récupérer `test-gaming-debounce` (squash)** — à faire *avant* le reste | §8 | 5 min | Aucun (fast-forward) |
| 1 | Fix `-` et `+` (2 tokens) + resync des commentaires | §1-①②③ | 5 min | Aucun |
| 2 | `&bootloader`, `&caps_word`, `&out OUT_TOG` dans les `&none` / `&trans` libres | §2-①②③ | 10 min | Aucun |
| 3 | Caps Lock → Globe côté macOS + `&kp CAPS` sur une touche | §5-(a) | 5 min | Aucun |
| 4 | HRM : `balanced` + `hml`/`hmr` cross-hand + `require-prior-idle-ms` | §3-①②③ | 30 min | Change le ressenti de frappe |
| 5 | Pin `west.yml` + tuning `corne.conf` | §4-①② | 10 min | Aucun |
| 6 | *(optionnel)* Dédoublonner `ESC`/`TAB`/`BSPC`, ajouter des combos | §1-④, §3-⑤ | — | Mémoire musculaire |

Les étapes **1 à 3 et 5** sont des diffs chirurgicaux sans risque.
L'étape **4** doit être revue avant application.
L'étape **6** n'est pas prioritaire.

---

## 7. Questions ouvertes

1. **Tu utilises Caps Lock aujourd'hui ?**
   Si oui, l'option (a) pour la Fn (§5) a un coût réel qu'il faut arbitrer.

2. **Tu as déjà des ratés de home row mods** — un `Shift` ou un `Ctrl` parasite en tapant
   vite ? Ça décide si l'étape 4 du plan est prioritaire ou purement cosmétique.

3. **La branche `test-gaming-debounce`** → analysée, voir [§8](#8-branche-test-gaming-debounce).
   Verdict : à récupérer en squash. Reste à trancher le sort de `&tog GAM` (§8-③).

---

## 8. Branche `test-gaming-debounce`

**Analysée le 2026-09-21.** 9 commits (`a4b6d82` → `7ad167e`), datés de mars 2024.
Un seul fichier touché : `config/corne.keymap` (+30 / −7).

### Contexte

La branche est **strictement linéaire sur `master`** (`d1c112c` est son parent direct) :
une récupération est un fast-forward, **zéro conflit possible**.

> ⚠️ **Le nom de la branche est trompeur.** Aucun debounce n'a jamais été committé :
> `config/corne.conf` n'apparaît dans aucun commit de la branche, et un
> `git log --all -S"DEBOUNCE"` sur tout l'historique ne renvoie rien.
> Ce qui a été tuné, ce sont les **timings hold-tap**, pas le debounce kscan matériel.
> Si le debounce t'intéresse vraiment, c'est un travail qui reste entièrement à faire
> (`CONFIG_ZMK_KSCAN_DEBOUNCE_PRESS_MS` / `_RELEASE_MS` dans `corne.conf`).

### ✅ Ce qui vaut le coup d'être gardé

#### ① Override global de `&lt` — **le vrai gain de la branche**

```dts
&lt {
    flavor = "hold-preferred";
    tapping-term-ms = <160>;
    quick-tap-ms = <130>;
};
```

Les layer-taps de pouce passent de `balanced` implicite / 200 ms à `hold-preferred` / 160 ms.
C'est **le bon réglage** : les pouces ne font pas de rolls, donc `hold-preferred` n'y génère
pas de faux positifs, contrairement aux doigts. Accès aux couches nettement plus réactif.

À garder tel quel.

#### ② Le `game_layer` (layer 4)

Copie du default layer **sans home row mods** (`&kp A` au lieu de `&hmt LSHIFT A`) et
**sans layer-taps** (`&kp BSPC` / `&kp ESC` aux pouces).

C'est le pattern standard et correct : HRM et layer-taps sont inutilisables en jeu
(maintien de touche = hold parasite permanent). À garder.

#### ③ `&kp LSHIFT` (pouce gauche) → `&tog GAM`

Récupère l'une des touches gaspillées identifiées en [§1-④](#-4-touches-gaspillées-en-doublons-sur-le-layer-de-base).
Bonne idée sur le principe.

**Mais :** `&tog GAM` est un *toggle*, placé sur le pouce gauche du layer de base — l'une
des positions les plus sollicitées. Un appui accidentel désactive d'un coup tous les HRM et
toutes les couches, et le clavier paraît « cassé » sans explication.

C'est récupérable (`&tog GAM` est au même endroit dans le game_layer), mais déroutant.

| Option | Détail |
|---|---|
| **(a) Déplacer `&tog GAM` sur le tri_layer** | **Recommandé.** Le gaming se déclenche une fois par session, pas besoin d'un accès direct. Libère à nouveau le pouce gauche. |
| (b) Le garder au pouce | Accès immédiat, mais risque d'activation fantôme. |
| (c) Le remplacer par `&sl GAM` | Non : un sticky layer n'a aucun sens pour du gaming. |

#### ④ Suppression du `label` sur le behavior + renommage `ht` → `hmt`

Cosmétique et correct (`label` est déprécié dans ZMK récent). Neutre, à garder.

### ⚠️ Ce qu'il faut corriger avant de récupérer

#### ⑤ `global-quick-tap` supprimé sans remplacement — RÉGRESSION

```diff
-            global-quick-tap;
```

La branche résout le point [§3-③](#-global-quick-tap-est-déprécié) en supprimant la propriété
dépréciée… mais **sans la remplacer**. `global-quick-tap` empêchait les holds accidentels
pendant la frappe rapide ; sans lui, les faux positifs de home row mods augmentent.

→ Ajouter `require-prior-idle-ms = <150>;` dans le behavior `hmt`. C'est le successeur direct.

#### ⑥ Le `game_layer` duplique les bugs `-` et `+`

Le nouveau layer reprend `&kp EQUAL` annoté `-` et le même bandeau de commentaires décalé.
Les fixes de [§1-①②③](#1-correctness-du-mapping) devront donc être appliqués **à deux
endroits** au lieu d'un.

→ D'où l'ordre du plan d'action : **récupérer la branche d'abord, corriger ensuite.**

#### ⑦ Shift peu praticable dans le `game_layer`

Sans HRM et sans le pouce (devenu `&tog GAM`), le seul `Shift` du game_layer est celui de
la rangée du bas, position 24 (auriculaire gauche). Pour sprinter en jeu, c'est mal placé.

→ Si tu joues à des FPS, prévoir un `Shift` accessible au pouce dans ce layer.
Faible priorité, dépend de tes usages.

#### ⑧ Ligne vide superflue avant la fermeture du bloc `keymap`

Cosmétique.

### Verdict

**À récupérer — mais en squash, pas en fast-forward.**

Les 9 commits sont de l'essai-erreur (`oops`, `oops typo`, `return to tap`, `lt`,
`no binding cell`…). Seul l'état final a de la valeur ; l'historique intermédiaire ne
documente rien d'utile.

| Option | Détail |
|---|---|
| **(a) `git merge --squash`** | **Recommandé.** 1 commit propre, message explicite. |
| (b) `git merge --ff` | Pollue `master` avec 9 commits dont 4 corrections d'erreurs. |
| (c) Cherry-pick sélectif | Inutile ici : tout le contenu final est bon à prendre. |
| (d) Supprimer la branche | Non — on perdrait l'override `&lt`, qui est le meilleur apport. |

Après récupération, la branche distante peut être supprimée : son contenu est intégré et
son nom induit en erreur.

---

## 9. Ce qui a été appliqué

Deux commits, **non poussés**, firmware **non testé sur matériel**.

| Commit | Contenu |
|---|---|
| `a5954c0` | Squash-merge de `test-gaming-debounce` (9 commits → 1) |
| `b2b8f48` | Tous les fixes ci-dessous |

### Décisions prises en s'écartant de l'analyse initiale

#### La touche Fn occupe le pouce gauche (au lieu de `&tog GAM`)

Le pouce gauche (pos. 36) portait `&kp LSHIFT` sur `master`, puis `&tog GAM` sur la branche.
Il porte désormais **`&kp CAPS`**, destiné à devenir la touche Fn via le remap macOS
(§5-a). Justification : c'est la position la plus accessible du clavier, et `Shift` reste
servi par la rangée du bas + les deux home row mods.

#### `&tog GAM` déplacé sur le tri_layer, coin bas droit (pos. 35)

Le tri_layer exige les **deux pouces** (`RIG` + `LEF`) → activation accidentelle
structurellement impossible. Le gaming se déclenche une fois par session ; un accès direct
n'a pas de valeur. Sortie du game_layer à la **même position**, pour ne mémoriser qu'un
seul emplacement.

#### `+` et `-` : doublon assumé

`+` est accessible à deux endroits : home row du left_layer (confortable, `+` est fréquent)
et pavé numérique du right_layer (cohérence du pavé). Doublon volontaire, pas un oubli.

#### Home row mods : `balanced` + cross-hand appliqués

L'analyse recommandait de valider avant application (§3-①②). Appliqué directement sur
instruction (« fais au mieux »). **C'est le seul changement qui modifie le ressenti de
frappe.** Rollback si les holds partent trop facilement :

```dts
// dans hml ET hmr
flavor = "tap-preferred";   // au lieu de "balanced"
```

Si au contraire ils ne partent pas assez : `tapping-term-ms = <180>`.

#### `game_layer` : Shift remonté au pouce

Le layer héritait d'un `Shift` en position 24 (auriculaire, rangée du bas) — injouable
pour sprinter. Déplacé au pouce gauche (§8-⑦).

### Reste à faire

| Sujet | Section | Pourquoi ça n'a pas été fait |
|---|---|---|
| ~~Remap Verr.maj → Globe dans macOS~~ | §5-a | ✅ **Fait et vérifié** — la touche Fn fonctionne. |
| ~~Pin de `west.yml`~~ | §4-① | ✅ **Fait** (`c3aa604` + `91d95ab`), après la panne du §10. |
| **Valider le keymap aux doigts** | §9 | Le firmware tourne, mais les corrections (`-`, `+`, home row mods, game layer) n'ont pas encore été testées à l'usage. |
| Dédoublonner `ESC`/`TAB`/`BSPC` | §1-④ | Coût en mémoire musculaire > gain. Non prioritaire. |
| Combos | §3-⑤ | Piste d'amélioration, pas un défaut. |
| Debounce kscan | §8 | N'a jamais existé malgré le nom de la branche. Chantier entier si besoin. |
| Supprimer `origin/test-gaming-debounce` | §8 | Contenu intégré, nom trompeur. À supprimer après validation du firmware. |

### Vérifications passées

- 5 couches × **42 bindings** exactement
- Accolades et chevrons équilibrés
- Zéro résidu de `&ht` / `&hmt` / `global-quick-tap`
- `dt-bindings/zmk/outputs.h` inclus (requis par `&out OUT_TOG`)

> ⚠️ Ce sont des vérifications **structurelles**, pas une compilation. Le premier build
> GitHub Actions reste le vrai test.

---

## 10. Incident du premier flash

**2026-09-21.** Après le premier flash réussi, le clavier ne tapait plus rien — ni en USB,
ni en Bluetooth. Section écrite pendant la résolution ; le verdict final est en fin de section.

### Symptômes

- Aucune touche, USB comme BLE
- Le Corne n'apparaissait pas dans les appareils Bluetooth à proximité du Mac
- Les deux moitiés répondaient au double-tap reset (`NICENANO` montait)
- Les deux `.uf2` s'étaient écrits correctement (volume démonté de lui-même)

### Commandes de diagnostic utiles (macOS)

À réutiliser tel quel au prochain incident :

```bash
# Le Mac voit-il le clavier sur le bus USB ?
ioreg -p IOUSB -l -w 0 | grep '"USB Product Name"' | sort -u

# Quel firmware tourne ? (VID/PID)
#   0x239A (9114)  = bootloader Adafruit  -> ZMK ne tourne PAS
#   0x1D50 (7504) + PID 0x615E (24926) = firmware ZMK
ioreg -p IOUSB -l -w 0 | grep -E "idVendor|idProduct"

# Le device expose-t-il reellement une interface HID clavier ?
# (c'est CA qui determine si les touches peuvent remonter)
hidutil list | grep -i 0x1d50

# La carte est-elle en bootloader ?
ls /Volumes/    # NICENANO monte = bootloader actif
```

**La distinction décisive :** un device peut être présent sur le bus USB (`ioreg`) **sans**
exposer d'interface HID (`hidutil`). Dans ce cas il est « vu » par le Mac mais aucune
frappe ne peut remonter. Vérifier les deux couches, pas seulement la première.

### Deux faux départs à ne pas refaire

**« Le Bluetooth est cassé, on verra ça après. »**
Le Corne n'apparaissait pas dans les appareils à proximité. Traité comme un problème
distinct à régler plus tard — c'était en fait le **même** symptôme que l'absence de HID :
le firmware ne présentait pas d'interface clavier. Deux symptômes simultanés après un
changement unique ont une cause unique jusqu'à preuve du contraire.

**« Pas de HID = firmware peripheral. »**
Hypothèse fausse : les deux moitiés ZMK compilent le support HID. L'absence de HID ne
renseigne pas sur le côté gauche/droite. Cette erreur a coûté un débranchement/rebranchement
et un flash inutile.
Ce qui a redressé le diagnostic : la moitié B, flashée avec un firmware **différent**,
s'est comportée **exactement** comme A. Deux firmwares différents, même symptôme
→ la cause est dans ce qu'ils partagent (`corne.conf`), pas dans ce qui les distingue.

### Cause réelle — CONFIRMÉE

**L'écart de version ZMK.** `west.yml` suivait `main` : le firmware fonctionnel datait de
mars 2024 (antérieur au tag v0.1.0), et on compilait contre un `main` de septembre 2026,
soit 18 mois d'évolution.

Résolu en épinglant **trois** références sur la même révision (`c3aa604` + `91d95ab`) :

| Fichier | Avant | Après |
|---|---|---|
| `config/west.yml` | `main` | `v0.3.0` |
| `build.yaml` | `nice_nano@2.0.0` | `nice_nano_v2` |
| `.github/workflows/build.yml` | `@main` | `@v0.3.0` |

**Le keymap n'a pas changé d'une ligne** entre le build cassé et le build fonctionnel.
Il n'a jamais été en cause.

> ⚠️ **Épingler le code sans épingler le workflow ne suffit pas.** Le premier essai a
> échoué sur `KeyError: 'qualifiers'` : le workflow resté en `@main` appelle
> `west boards --format {qualifiers}`, champ introduit par les hardware revisions et
> absent de v0.3.0. Les trois références doivent bouger ensemble.

### Deux hypothèses fausses émises en chemin

Consignées parce qu'elles ont coûté du temps et des manipulations inutiles :

1. **« Pas de service HID = firmware peripheral »** — faux, les deux moitiés compilent le
   HID. A conduit à un débranchement/rebranchement et à un flash inutile.
2. **« `CONFIG_ZMK_HID_CONSUMER_REPORT_USAGES_FULL` casse le descripteur »** — faux, le
   revert de `corne.conf` n'a rien changé. Plausible sur le papier, jamais vérifié.

Dans les deux cas l'hypothèse a été présentée avec trop d'assurance avant d'être testée.
Le fait qui a redressé le diagnostic : deux firmwares **différents** produisaient un
symptôme **identique** → chercher dans ce qu'ils partagent, pas dans ce qui les distingue.

### Leçon

**Ne pas mélanger correctifs et confort dans un même flash.** Les trois réglages
`corne.conf` (deep sleep, TX power, usages HID) n'avaient pas été demandés et ne
corrigeaient rien. Ils ont transformé un changement de keymap testable en panne totale,
et rendu le diagnostic beaucoup plus long : impossible de savoir si la panne venait du
keymap, de la conf ou du board.

Sur un firmware qu'on ne peut pas tester avant de le flasher, chaque flash ne devrait
porter **qu'une seule catégorie de changement**.

### Point pratique

Rien ne distingue physiquement les deux PCB d'un Corne — c'est le firmware qui décide du
côté. **Marquer la moitié gauche** (point de marqueur sous le PCB) évite de refaire ce
diagnostic à chaque flash.
