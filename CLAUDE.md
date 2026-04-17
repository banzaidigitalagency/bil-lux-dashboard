# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projet

Dashboard de suivi de campagne digitale pour BIL (Banque Internationale a Luxembourg) - Campagne Branding Belgique 2026. Fichier HTML unique avec donnees live Supabase.

## Architecture

Tout tient dans `index.html` : CSS inline, HTML, et JS (Chart.js + Supabase). Pas de build, pas de bundler, pas de framework.

- **Lock screen** : code d'acces `BIL2026` (constante `ACCESS_CODE`), session via `sessionStorage`
- **Supabase** : projet `mfqbhpxsuawujnfbcojr`, client code `bil`. Donnees chargees via 4 fonctions RPC :
  - `get_dashboard_kpis(p_client_code)` - KPIs globaux avec coefs billing
  - `get_dashboard_by_platform(p_client_code)` - breakdown par plateforme avec coefs
  - `get_dashboard_daily(p_client_code)` - impressions par jour (chart timeline)
  - `get_dashboard_video_views(p_client_code)` - vues video DV360/YouTube
- **Coefficients billing** : table `billing_coefficients`, le spend client = spend reel / coef. Fonction helper `get_billing_coef(client_id, platform::text)` avec fallback sur coefs par defaut (client_id IS NULL)
- **Charts** : Chart.js 4.x - un line chart (impressions cumulees) et un doughnut (repartition budget)

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
- row 6 (Prog. Display) n'a pas de donnees Supabase directes

## Deploiement

Fichier statique, deployable sur Vercel/Cloudflare Pages. `open index.html` pour tester en local.

## Langue

Dashboard en francais. Commits en francais.
