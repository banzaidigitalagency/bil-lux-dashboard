# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projet

Dashboard de suivi de campagne digitale pour BIL (Banque Internationale a Luxembourg) - Campagne Branding Belgique 2026. Fichier HTML unique avec donnees live Supabase.

## Memoire projet (a lire avant de travailler)

Le contexte accumule au fil des sessions est versionne dans `docs/memoire/` (index : `docs/memoire/README.md`). Ce CLAUDE.md decrit l'architecture ; `docs/memoire/` decrit le **pourquoi** (decisions, preferences de l'equipe, pieges connus). Trois notes non negociables :

- `docs/memoire/spend-billing-coefs.md` : **CRITIQUE**, ne jamais afficher le spend brut, toujours avec billing coefficients (le dashboard est vu par le client).
- `docs/memoire/budgets-source-de-verite.md` : les budgets viennent du Sheet "Matt - 26" via le sync vers `campaign_tracking`. Avant de toucher au code sur un budget faux, comparer Sheet vs `campaign_tracking`.
- `docs/memoire/design-approche.md` : pas de refonte, garder les sections et le violet BIL.

Quand une decision structurante est prise en session, l'ecrire dans `docs/memoire/` (versionne, partage) plutot que dans la memoire locale de Claude Code (par machine, non partagee).

## Architecture

Tout tient dans `index.html` : CSS inline, HTML, et JS (Chart.js + Supabase). Pas de build, pas de bundler, pas de framework.

- **Lock screen** : code d'acces `BIL2026` (constante `ACCESS_CODE`), session via `sessionStorage`
- **Multi-campagnes** : le dashboard suit plusieurs dispositifs pour un meme client, un onglet par campagne dans le header (commutateur `#scopeBar`). Voir la section **Perimetres** ci-dessous.
- **Supabase** : projet `mfqbhpxsuawujnfbcojr`, client code `billuxembourg` (constante `CLIENT_CODE` ; ex-`bil`, renomme pour anticiper un futur `bilsuisse`). Donnees chargees via 7 fonctions RPC appelees en parallele dans `loadDashboard`, plus `get_dashboard_scopes` appelee une fois au boot. **Toutes prennent `(p_client_code, p_scope)`**, `p_scope` valant `NULL` pour le perimetre par defaut du client :
  - `get_dashboard_kpis` - KPIs globaux avec coefs billing
  - `get_dashboard_by_platform` - breakdown par plateforme avec coefs
  - `get_dashboard_by_platform_daily` - spend/impr par plateforme et par jour (sert au filtre par phase via `aggregateByPhase`)
  - `get_dashboard_daily` - impressions par jour (chart timeline)
  - `get_dashboard_by_language_daily` - breakdown langue x canal x jour
  - `get_dashboard_video_views` - vues video DV360/YouTube
  - `get_dashboard_planning` - **planning budgets + objectifs impressions**, agrege depuis `campaign_tracking` par phase et par canal. Source unique des budgets/objectifs : `applyPlanning()` construit `BUDGETS`, `OBJECTIVES`, `PHASE_OBJECTIVES`, `PHASES_DATA.budgetMedia/imprObj`, `TOTAL_BUDGET`, `TOTAL_IMPR_OBJ` et le donut. **Aucun budget/objectif impressions n'est en dur dans le JS.** Mapping canal dans la RPC : `%CTV% -> ctv`, `Habillage% -> display_prog`, `pYoutube%/youtube -> google`, sinon `platform`. **Les phases ne sont plus en dur** : elles viennent de `dashboard_scopes.phases`, et le planning est filtre sur `bdc_number` du perimetre. Sans perimetre declare, repli sur `client_code = p_client_code OR client_name ILIKE clients.name` et une phase unique.
  - `get_dashboard_scopes(p_client_code)` - liste des onglets (cle, libelle, sous-titre, BDC, dates, phases, notes de canal). Appelee par `bootDashboard()`.
- **Export bilan** (voir `docs/memoire/export-bilan.md`) : deux boutons dans le header, xlsx multi-onglets (SheetJS charge a la demande depuis jsdelivr) et csv. Une 8e RPC, `get_dashboard_export(p_client_code, p_scope)`, appelee **seulement au clic** (pas dans `loadDashboard`), renvoie une ligne par **jour x canal x langue x crea x variante x format**, dimensions extraites du nommage par les helpers SQL `dashboard_norm_name` / `dashboard_lang_of` / `dashboard_crea_of` / `dashboard_variante_of` / `dashboard_format_of` (vocabulaire ferme). **Deux regles : le fichier est telecharge par le client, donc tout est en budget client (jamais d'investi reel) et rien ne descend au niveau line item** (pas de nom de campagne, de groupe d'annonces, d'annonce ni de ciblage). L'agregation est faite dans la RPC pour que ces noms ne transitent pas jusqu'au navigateur ; le JS ne fait plus que regrouper et mettre en forme. `exportRowsCache` est vide par `resetDashboardState()` a chaque changement de campagne.
- **Seules valeurs plan encore en dur** (absentes de `campaign_tracking`) : regroupees dans `PLAN_OVERRIDES`, **par campagne**. Pour `belgian` : objectifs de CLICS par phase, courbe mensuelle cumulee `cumTargets` (valeur validee), objectif de vues 100%. Une campagne sans entree dans `PLAN_OVERRIDES` (cas de `knokke`) affiche simplement "Sans objectif" sur ces KPIs, et sa courbe cible est **calculee** au prorata des jours reels de chaque ligne de planning (`computeCumTargets`).
- **Perimetres (onglets campagne)** : table `dashboard_scopes` (client_code, scope_key, label, subtitle, bdc_number, name_pattern, start_date, end_date, phases jsonb, channel_notes jsonb, is_default, sort_order). **Une ligne = un onglet. C'est le seul endroit a modifier pour ajouter une campagne** : ni le JS ni les RPC ne bougent. Deux perimetres au 2026-08-11 : `belgian` (BDC 202604417, 3 phases, defaut) et `knokke` (BDC 202607736, sans phase).
  - Helper `is_dashboard_scope(p_client_code, campaign_name, p_scope)` appele dans les 6 RPC qui lisent des insights : il ne garde que les campagnes dont le nom matche le `name_pattern` du perimetre. Fail-safe : client sans perimetre declare = aucun filtre, `p_scope NULL` = perimetre par defaut. Un renommage de code client ramene donc la pollution (visible) plutot qu'une perte de donnees silencieuse.
  - **Le rattachement se fait par NOM et pas par `campaign_tracking.platform_campaign_ids`** : cote Displayce ces IDs sont des IDs de lignes, pas le `platform_campaign_id` de la campagne. `BIL LUX - 2604 - BELGIAN` (714 691 impressions, tout le DOOH de phase 1) ne matche aucun ID de tracking, un filtre par IDs supprimerait le DOOH entier. A rouvrir si le sync Displayce est corrige.
  - Motif **inclusif** (`%BELGIAN%`, `%KNOKKE%`) et non exclusif : toute campagne hors dispositif est ecartee d'office, sans patcher la RPC a chaque fois.
  - Cote JS : `bootDashboard()` charge les perimetres, `selectScope(key)` recharge tout, `applyScopeMeta(scope)` reconstruit header, bandeau, funnel, donut, grille de phases et onglets de phase. Etat memorise dans l'URL (`#belgian` / `#knokke`) et dans `sessionStorage`. Les blocs sans donnee sont **masques** et non affiches vides : grille de phases si la campagne n'a pas de phases, carte langue s'il ne reste que du DOOH, lignes de canal absentes du plan.
- **Coefficients billing** : table `billing_coefficients`, le spend client = spend reel / coef. Fonction helper `get_billing_coef(client_id, platform::text)` avec fallback sur coefs par defaut (client_id IS NULL)
- **Charts** : Chart.js 4.x - un line chart (impressions cumulees) et un doughnut (repartition budget, alimente par `applyPlanning`)

## Couleurs BIL

Violet principal `#702F8A`. Les CSS variables vont de `--bil-dark` (#4A1D6B) a `--bil-lighter` (#F5EEF9). Les couleurs des canaux (LinkedIn bleu, Meta bleu, YouTube rouge, etc.) gardent leurs couleurs propres.

## Canaux actifs (8)

DOOH (Displayce), LinkedIn, Meta, DV360 Video, YouTube (Google), Teads, Prog. Display, CTV (Connected TV - YouTube). Pas de Presse (dashboard 100% digital).

## Mapping plateforme Supabase -> tableau

Le JS mappe les plateformes Supabase aux lignes du tableau HTML par index (`rowByPlatform` / `rowPlatformOrder`) :
- `displayce` -> row 0 (DOOH)
- `linkedin` -> row 1
- `meta` -> row 2
- `dv360` -> row 3 (DV360 Video)
- `google` -> row 4 (YouTube / pYoutube)
- `teads` -> row 5
- `display_prog` -> row 6 (Prog. Display / Habillage)
- `ctv` -> row 7 (CTV, couleur teal #00A8A8, sans objectif clics)

Deux canaux sont des sous-ensembles de plateformes mutualisees, scindes par **nom de campagne** (pas par code client, robuste aux renommages) :
- **Habillage** -> `display_prog` : campagne DV360 dont le nom contient `HABILLAGE`. Coef billing `habillage` (0.40) au lieu d'`outstream` (0.20).
- **CTV** -> `ctv` : campagnes Google Ads dont le nom contient `CTV` (ex `BIL - 2604 - BELGIAN - CTV - EN/FR`). Coef billing `ctv` (0.35, ligne ajoutee dans `billing_coefficients`, defaut client_id NULL). Cote planning, repere par `canal_label ILIKE '%CTV%'` (teste AVANT la regle `%youtube%`).

Le split est implemente dans les RPC de spend (`get_dashboard_by_platform`, `get_dashboard_by_platform_daily`, `get_dashboard_by_language_daily`) et dans `get_dashboard_planning` ; `get_dashboard_kpis` et `get_dashboard_daily` n'ont pas besoin du split CTV (la part CTV est sur platform `google` -> coef `pyoutube` 0.35, identique). `get_billing_coef` mappe `display_prog -> habillage` et `ctv -> ctv` (via la branche ELSE). Les 4 RPC sur `campaign_insights` renvoient un total Depenses identique (coherence verifiee).

Objectifs de CLICS (hardcode, `CLICK_OBJECTIVES_PHASE`) et courbe mensuelle (`CUM_TARGETS`) recalcules a la repart v2 (transfert +10k vers P1, Habillage P1 coupe, ligne CTV 3 346 EUR / 185 889 imp). CTV est sans objectif clics.

## Deploiement

Fichier statique, deployable sur Vercel/Cloudflare Pages. `open index.html` pour tester en local.

## Langue

Dashboard en francais. Commits en francais.
