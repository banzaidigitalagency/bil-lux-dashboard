---
name: bil-budgets-source-de-verite
description: "D'ou viennent les budgets/objectifs du dashboard BIL et comment ils se propagent (Sheet -> sync -> campaign_tracking -> RPC)"
type: project
maj: 2026-06-19
---

Chaine de verite des budgets et objectifs d'impressions du dashboard BIL :

1. **Source** : Google Sheet de suivi de Matt, onglet **"Matt - 26"**, id `1gKVxLx3ai_DLBt_Rfc4bnx0J2rFbiNTPMyYgp4AQd7c`. Le bloc BIL (BDC 202604417) ~ lignes 249+. Colonne **MONTANT CLIENT** = budget client par ligne (ce que le dashboard affiche). MONTANT A INVESTIR = part plateforme (client x coef).

2. **Sync** : un process gere par Matt lui-meme (dossier separe, hors de ce repo) lit le Sheet et upsert dans `campaign_tracking` (Supabase). C'est lui qui ecrit `client_code` (`billuxembourg` pour BIL) et les budgets/obj_impressions/dates. **Piege historique** : l'import a longtemps ete one-shot (pas d'`updated_at`), donc `campaign_tracking` pouvait etre fige (snapshot) et diverger du Sheet. Si un budget du dashboard semble faux, comparer Sheet vs campaign_tracking AVANT de toucher au code.

3. **Lecture** : la RPC `get_dashboard_planning(p_client_code)` agrege `campaign_tracking` par phase/canal, et le dashboard construit tout dynamiquement (cf [projet-etat.md](projet-etat.md) et CLAUDE.md). Donc : **modifier le Sheet + relancer le sync = dashboard a jour automatiquement**, zero code a toucher.

Regle absolue : le dashboard affiche le **budget client** (jamais le spend brut/investi). Voir [spend-billing-coefs.md](spend-billing-coefs.md).

Pour repartir d'une nouvelle repartition fournie en Excel (ex `BELGIAN_transfert_budget_avant-apres_v2.xlsx`), seules restent a recalculer a la main les donnees absentes de campaign_tracking : objectifs de CLICS (`CLICK_OBJECTIVES_PHASE`, clics ~ impressions a CTR constant) et courbe mensuelle (`CUM_TARGETS`, prorata des jours reels par ligne).
