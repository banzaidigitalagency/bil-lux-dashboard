---
name: BIL Dashboard - Etat du projet
description: Dashboard de suivi campagne BIL Belgique 2026 - structure, donnees, decisions prises
type: project
maj: 2026-08-11
---

## Projet
Dashboard HTML statique de suivi de campagne digitale pour BIL (Banque Internationale a Luxembourg).
Campagne Branding Belgique - Annuel 2026, 3 phases (Avr-Juin, Juil-Aout, Sep-Nov).

## Repo
https://github.com/banzaidigitalagency/bil-lux-dashboard

## Stack
- Fichier HTML unique (index.html) avec CSS inline + Chart.js + Supabase JS
- Pas de framework - choix volontaire, suffisant pour le use case
- Supabase (projet mfqbhpxsuawujnfbcojr) pour les donnees live via les fonctions RPC
- Sync Supabase une fois par matin (pas de polling live, le user recharge la page)

## Couleurs BIL
- Violet principal : #702F8A
- Echelle --bil-dark (#4A1D6B) a --bil-lighter (#F5EEF9)
- Logo : bil-logo.png (violet sur fond transparent, affiche en blanc via filter invert)
- Favicon : SVG inline (carre violet BIL avec B blanc)

## Echelle typographique (variables CSS)
--fs-xxs 11px (labels courts, badges) / --fs-xs 12px (meta, progress values) / --fs-sm 13px (table, courant) / --fs-md 14px (titres card) / --fs-lg 18px / --fs-xl 24px / --fs-2xl 26px.
Tabular-nums active sur body pour alignement des chiffres.

## Donnees Supabase
- Client code : `billuxembourg` (ex-`bil`, renomme cote clients ET campaign_tracking le 2026-06-19 pour anticiper un futur `bilsuisse` ; constante `CLIENT_CODE` dans index.html). client_id inchange : `a5ca7085-5407-4abd-bada-87e9d51bb2a4`
- 6 ad_accounts : displayce (DOOH), linkedin, meta, dv360, google (YouTube), teads
- 7 RPC (cf CLAUDE.md, section a jour) : get_dashboard_kpis, _by_platform, _by_platform_daily, _daily, _by_language_daily, _video_views, _planning
- Coefficients billing dans `billing_coefficients` (spend client = spend reel / coef)
- Voir aussi [budgets-source-de-verite.md](budgets-source-de-verite.md) pour la source des budgets et le sync.

## Coefs billing (grille par defaut)
dooh 0.50 / outstream (dv360) 0.20 / pyoutube (google) 0.35 / teads 0.38 / habillage (display) 0.40 / linkedin 1.00 / meta 1.00

## Decisions prises
- Presse retiree (100% digital)
- 3 lignes DOOH fusionnees (Displayce renvoie un seul agregat)
- Ecran de verrouillage avec code d'acces BIL2026, session via sessionStorage
- Budget media : 112 985 EUR a l'origine, 112 988 EUR depuis la repart v2. Budget total HT : 152 933 EUR (HT non affiche sur dashboard)
- Objectif impressions : 10 370 000 au plan d'origine. **Valeur live au 2026-08-11 : 10 221 908** (somme `obj_impressions` de campaign_tracking, c'est elle qui pilote le dashboard). Vues 100% : 2 250 000 (toujours en dur, pas dans campaign_tracking)
- Onglets nav morts supprimes (Vue d'ensemble / Par canal / Budget / Exports etaient des div sans handler)
- Statut des phases dynamique base sur la date courante (data-phase-start/end sur chaque phase-card, updatePhases() au load)
- Progress bars objectifs impressions : couleurs de canal (DOOH orange, LinkedIn bleu, Meta bleu, DV360 vert, YouTube rouge, Teads dark, Display violet)
- Progress bars pacing budget : violet solide sobre (pas de gradient)

## Objectifs vues 100% (video completions)
Seulement 2 canaux portent cet objectif dans le plan BIL :
- **DV360 Video** : 65% de taux de vue vise = 975 000 vues / 1 500 000 impressions (taux plus bas car route vers inventaires tiers VAST/VPAID heterogenes malgre le "non-desactivable")
- **YouTube Instream** : 85% vise = 1 275 000 vues / 1 500 000 impressions (vraiment non-skip natif, quasi 100% en theorie, 85% compte les drop-offs techniques)
- **Total** : 2 250 000 vues
- Les autres canaux (DOOH, LinkedIn, Meta, Teads, Prog. Display) n'ont pas d'objectif de vue 100%. Leurs KPIs sont CTR/clics/CPM.

## Custom metrics dans campaign_insights (Supabase)
- **Google** stocke uniquement `video_completion_rate` (en %, ex. 89.81)
- **DV360** stocke `video_plays`, `video_completions`, `video_completion_rate`, `post_view_conversions`, `post_click_conversions`
- La RPC `get_dashboard_video_views` calcule correctement depuis le 2026-04-17 :
  - DV360 : somme directe de `video_completions`
  - Google : `impressions × (video_completion_rate / 100)` jour par jour
- Avant le fix, la RPC cherchait une cle inexistante `video_views_100` et renvoyait toujours 0.

## Pieges connus
- `checkAccess()` doit imperativement s'appeler APRES la declaration `const sb` et APRES `initCharts()`, sinon ReferenceError TDZ au reload avec session active (checkAccess -> unlock -> loadDashboard -> sb pas encore initialise)
- Toast d'erreur ne doit s'afficher que si TOUTES les RPC echouent (pas si une seule warn), sinon faux positifs

## Skills Claude Code versionnees
17 skills design dans .agents/skills/ (polish, critique, impeccable, typeset, etc.) + symlinks dans .claude/skills/ + skills-lock.json. Versionnees pour retrouver sur autre machine. settings.local.json reste ignore (permissions par machine).

## Etat design
Phase 1 (UI morte) + Phase 2 (de-IAiser) livrees au 2026-04-17. Dashboard ready-to-ship cote design.

## MAJ 2026-06-19 (session importante - voir CLAUDE.md, autorite a jour)
- **8 canaux** desormais (ajout CTV / Connected TV - YouTube, row 7, couleur teal #00A8A8, sans objectif clics). Cle `ctv`.
- **Budgets/objectifs impressions 100% live depuis Supabase** : RPC `get_dashboard_planning` agrege `campaign_tracking` par phase/canal. `applyPlanning()` construit BUDGETS, OBJECTIVES, PHASE_OBJECTIVES, PHASES_DATA, TOTAL_BUDGET, donut, montants HTML (ids #heroBudget/#kpiBudget). PLUS AUCUN budget en dur. Restent en dur (absents de campaign_tracking) : objectifs clics `CLICK_OBJECTIVES_PHASE` et courbe mensuelle `CUM_TARGETS`.
- **Habillage ET CTV scindes par NOM DE CAMPAGNE** (plus de guard `p_client_code='bil'` : casse au renommage). Habillage = DV360 name ILIKE '%HABILLAGE%' -> display_prog (coef 0.40). CTV = Google Ads name ILIKE '%CTV%' (BIL - 2604 - BELGIAN - CTV - EN/FR) -> ctv (coef 0.35, ligne ajoutee dans billing_coefficients). get_billing_coef mappe display_prog->habillage et ctv->ctv.
- **Bug corrige** : get_dashboard_kpis n'appliquait pas le coef Habillage (total Depenses surestime). Les 4 RPC sur campaign_insights donnent maintenant un total identique.
- Repart v2 (transfert +10k vers P1, Habillage P1 coupe a 2500, ligne CTV 3346 EUR/185889 imp). Total media inchange 112 988 EUR. Budgets par phase : P1 69088 / P2 7400 / P3 36500.
- CTV pas encore syncee dans `campaigns` (depense "-") tant que les campagnes Google Ads ne remontent pas. **Resolu : elle remonte depuis le 2026-06-19, cf MAJ 2026-08-11.**

## MAJ 2026-08-11 (etat des lieux, aucun changement de code)

Le code n'a pas bouge depuis le 2026-06-19 (dernier commit fonctionnel `45a16ac`). Ce qui a change, c'est la donnee.

### Phase en cours
Phase 1 terminee (statut `BILAN` dans campaign_tracking, toutes lignes closes au 30/06). **Phase 2 en cours** (2026-07-01 au 2026-08-31), en effectif reduit : seuls DV360, pYoutube, Meta et LinkedIn ont une ligne. DOOH, Teads, Habillage et CTV sont a l'arret jusqu'a la phase 3 (2026-09-01 au 2026-11-29, encore `A VENIR`, 0 campagne matchee).

### CTV : OK, syncee
Les deux campagnes `BIL - 2604 - BELGIAN - CTV - EN` et `- FR` remontent bien dans `campaigns` depuis le 2026-06-19. Bilan de la ligne : 583 379 impressions pour 3 347 EUR client, contre un objectif de 185 889 impressions / 3 346 EUR. Budget consomme au centime pres, impressions livrees a **3,1x l'objectif** (CPM tres en dessous du prevu). Le split par nom de campagne (`ILIKE '%CTV%'`) fonctionne comme prevu.

### PIEGE ACTUEL : la campagne Knokke pollue le dashboard
Depuis le 2026-07-28, un nouveau dispositif tourne sur le meme client Supabase : `2026 - BIL - BELGIQUE - KNOKKE - WEALTH MANAGEMENT` (Displayce, DV360, et une campagne LinkedIn pas encore livree). **Il n'a aucune ligne dans `campaign_tracking`** : ce n'est pas le BDC 202604417.

Or les RPC du dashboard agregent **toutes** les campagnes du client `billuxembourg`, sans filtre de BDC ni de nom. Knokke est donc compte dans les KPIs Belgian alors qu'il n'a pas d'objectif en face :
- environ 249 625 impressions injectees (54 383 Displayce + 195 242 DV360)
- environ 3 057 EUR de budget client (1 372 Displayce + 1 685 DV360)
- effet : la depense et les impressions montent sans contrepartie au planning, donc **le pacing affiche est fausse a la hausse**, surtout sur la ligne DOOH.

C'est le sujet a traiter en priorite si les chiffres sont questionnes. Deux pistes : filtrer les RPC sur les `platform_campaign_ids` de `campaign_tracking` (propre, aligne sur le planning), ou exclure par nom (`NOT ILIKE '%KNOKKE%'`, rapide mais fragile, meme piege que l'ancien guard `p_client_code='bil'`).

### Livraison au 2026-08-11 (BDC 202604417 + pollution Knokke)
- Depense client : 77 060 EUR sur 112 988 EUR de budget total (68%)
- Impressions : 13 341 544 sur 10 221 908 d'objectif (**130%**, objectif annuel deja depasse en phase 2)
- Clics 55 291, CTR moyen 0,41%, CPM moyen 5,78 EUR, reach 8 216 465
- Vues 100% : 3 802 124 pour un objectif de 2 250 000. Attention, ce total agrege desormais CTV et le DV360 Knokke, qui n'etaient pas dans le perimetre des 2,25 M d'origine (DV360 Instream + YouTube Instream seuls). Chiffre a ne pas presenter tel quel sans le retraiter.
- Par canal (budget client / impressions) : Displayce 22 544 / 769 074, Teads 11 692 / 3 418 534, DV360 11 101 / 1 065 747, Google YouTube 9 381 / 3 069 234, LinkedIn 8 247 / 505 333, Meta 8 239 / 3 768 134, CTV 3 347 / 583 379, Habillage 2 507 / 162 109

### Budgets par phase : confirmes
La RPC `get_dashboard_planning` renvoie toujours P1 69 088 / P2 7 400 / P3 36 500 = 112 988 EUR. La repart v2 est bien celle qui pilote le dashboard, `campaign_tracking` n'a pas derive du Sheet.

### Detail a connaitre sur campaign_tracking
La phase 3 contient des lignes **dupliquees** pour LinkedIn (2 x 2 639), Meta (2 639 + 2 639) et Teads (2 x 3 078). Ce sont deux lignes distinctes du Sheet, pas un bug de sync : la RPC les additionne correctement (LinkedIn P3 = 5 278). Ne pas "corriger" en dedupliquant.

## Habillage / Prog. Display (ajoute 2026-05-28 ; section partiellement obsolete depuis 2026-06-19, cf ci-dessus)
Row 6 (Prog. Display) est desormais alimente. Source : campagne DV360 dont le nom contient `HABILLAGE` (`2600 - BELGIAN - HABILLAGE`), distincte du DV360 video `INSTREAM`.
- Les 4 RPC dashboard (`get_dashboard_by_platform`, `_by_platform_daily`, `_by_language_daily`, `_kpis`) scindent l'habillage sous la cle platform `display_prog`, GUARDE a l'epoque par `p_client_code = 'bil'` (cegee et scc3fontaines ont aussi des campagnes DV360 habillage et ne doivent PAS etre impactes). Le guard a ete remplace par un split sur le nom de campagne le 2026-06-19.
- Coef billing : `get_billing_coef` mappe `display_prog -> habillage` (0.40), au lieu de `dv360 -> outstream` (0.20). Avant le fix, l'habillage etait noye dans le DV360 video avec le mauvais coef (sur-facturation x2).
- JS : cle `display_prog` partout (rowByPlatform:6, OBJECTIVES/BUDGETS/CLICK_OBJECTIVES/PHASE_OBJECTIVES). LANG_CHANNELS_ORDER et CHANNEL_META renommes de `programmatic_display` -> `display_prog` (l'ancien nom ne matchait jamais la cle RPC).
