---
name: BIL Dashboard - Etat du projet
description: Dashboard de suivi campagne BIL Belgique 2026 - structure, donnees, decisions prises
type: project
maj: 2026-06-19
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
- Budget media : 112 985 EUR, budget total HT : 152 933 EUR (HT non affiche sur dashboard)
- Objectif impressions : 10 370 000, vues 100% : 2 250 000
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
- CTV pas encore syncee dans `campaigns` (depense "-") tant que les campagnes Google Ads ne remontent pas.

## Habillage / Prog. Display (ajoute 2026-05-28 ; section partiellement obsolete depuis 2026-06-19, cf ci-dessus)
Row 6 (Prog. Display) est desormais alimente. Source : campagne DV360 dont le nom contient `HABILLAGE` (`2600 - BELGIAN - HABILLAGE`), distincte du DV360 video `INSTREAM`.
- Les 4 RPC dashboard (`get_dashboard_by_platform`, `_by_platform_daily`, `_by_language_daily`, `_kpis`) scindent l'habillage sous la cle platform `display_prog`, GUARDE a l'epoque par `p_client_code = 'bil'` (cegee et scc3fontaines ont aussi des campagnes DV360 habillage et ne doivent PAS etre impactes). Le guard a ete remplace par un split sur le nom de campagne le 2026-06-19.
- Coef billing : `get_billing_coef` mappe `display_prog -> habillage` (0.40), au lieu de `dv360 -> outstream` (0.20). Avant le fix, l'habillage etait noye dans le DV360 video avec le mauvais coef (sur-facturation x2).
- JS : cle `display_prog` partout (rowByPlatform:6, OBJECTIVES/BUDGETS/CLICK_OBJECTIVES/PHASE_OBJECTIVES). LANG_CHANNELS_ORDER et CHANNEL_META renommes de `programmatic_display` -> `display_prog` (l'ancien nom ne matchait jamais la cle RPC).
