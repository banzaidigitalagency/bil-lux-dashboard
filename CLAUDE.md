# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projet

Dashboard de suivi de campagne digitale pour BIL (Banque Internationale a Luxembourg) - Campagne Branding Belgique 2026. Fichier HTML unique avec donnees live Supabase.

## Architecture

Tout tient dans `index.html` : CSS inline, HTML, et JS (Chart.js + Supabase). Pas de build, pas de bundler, pas de framework.

- **Lock screen** : code d'acces `BIL2026` (constante `ACCESS_CODE`), session via `sessionStorage`
- **Supabase** : projet `mfqbhpxsuawujnfbcojr`, client code `billuxembourg` (constante `CLIENT_CODE` ; ex-`bil`, renomme pour anticiper un futur `bilsuisse`). Donnees chargees via 7 fonctions RPC (toutes appelees en parallele dans `loadDashboard`) :
  - `get_dashboard_kpis(p_client_code)` - KPIs globaux avec coefs billing
  - `get_dashboard_by_platform(p_client_code)` - breakdown par plateforme avec coefs
  - `get_dashboard_by_platform_daily(p_client_code)` - spend/impr par plateforme et par jour (sert au filtre par phase via `aggregateByPhase`)
  - `get_dashboard_daily(p_client_code)` - impressions par jour (chart timeline)
  - `get_dashboard_by_language_daily(p_client_code)` - breakdown langue x canal x jour
  - `get_dashboard_video_views(p_client_code)` - vues video DV360/YouTube
  - `get_dashboard_planning(p_client_code)` - **planning budgets + objectifs impressions**, agrege depuis `campaign_tracking` par phase et par canal. Source unique des budgets/objectifs : `applyPlanning()` construit `BUDGETS`, `OBJECTIVES`, `PHASE_OBJECTIVES`, `PHASES_DATA.budgetMedia/imprObj`, `TOTAL_BUDGET`, `TOTAL_IMPR_OBJ` et le donut. **Aucun budget/objectif impressions n'est en dur dans le JS.** Mapping canal dans la RPC : `Habillage% -> display_prog`, `pYoutube%/youtube -> google`, sinon `platform`. Bornes de phases (2026-06-30, 2026-08-31) en dur dans la RPC. La RPC matche `client_code = p_client_code OR client_name ILIKE clients.name` (fallback de securite conserve, meme si `campaign_tracking.client_code` vaut maintenant `billuxembourg`).
- **Seules valeurs plan encore en dur** (absentes de `campaign_tracking`) : objectifs de CLICS par phase (`CLICK_OBJECTIVES_PHASE`) et courbe mensuelle cumulee `CUM_TARGETS` (granularite mensuelle non stockee). Commentees dans le code.
- **Coefficients billing** : table `billing_coefficients`, le spend client = spend reel / coef. Fonction helper `get_billing_coef(client_id, platform::text)` avec fallback sur coefs par defaut (client_id IS NULL)
- **Charts** : Chart.js 4.x - un line chart (impressions cumulees) et un doughnut (repartition budget, alimente par `applyPlanning`)

## Couleurs BIL

Violet principal `#702F8A`. Les CSS variables vont de `--bil-dark` (#4A1D6B) a `--bil-lighter` (#F5EEF9). Les couleurs des canaux (LinkedIn bleu, Meta bleu, YouTube rouge, etc.) gardent leurs couleurs propres.

## Canaux actifs (7)

DOOH (Displayce), LinkedIn, Meta, DV360 Video, YouTube (Google), Teads, Prog. Display. Pas de Presse (dashboard 100% digital).

## Mapping plateforme Supabase -> tableau

Le JS mappe les plateformes Supabase aux lignes du tableau HTML par index :
- `displayce` -> row 0 (DOOH)
- `linkedin` -> row 1
- `meta` -> row 2
- `dv360` -> row 3
- `google` -> row 4 (YouTube)
- `teads` -> row 5
- `display_prog` -> row 6 (Prog. Display / Habillage)

Le canal Habillage est la campagne DV360 dont le nom contient `HABILLAGE` (`2600 - BELGIAN - HABILLAGE`). Les RPC dashboard la scindent du DV360 video sous la cle `display_prog` et lui appliquent le coef billing `habillage` (0.40) au lieu du coef `outstream` (0.20) du DV360 video. Le split se base **uniquement sur le nom de campagne** (`platform = 'dv360' AND name ILIKE '%HABILLAGE%'`), sans condition de code client (robuste aux renommages). Il est implemente dans `get_dashboard_by_platform`, `get_dashboard_by_platform_daily`, `get_dashboard_by_language_daily` et `get_dashboard_kpis` (total Depenses), plus le mapping `display_prog -> habillage` (inconditionnel) dans `get_billing_coef`.

## Deploiement

Fichier statique, deployable sur Vercel/Cloudflare Pages. `open index.html` pour tester en local.

## Langue

Dashboard en francais. Commits en francais.
