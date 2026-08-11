---
name: Ne jamais afficher le spend brut au client
description: "CRITIQUE : le dashboard BIL est vu par le client. Toujours appliquer les billing coefficients au spend avant affichage."
type: feedback
maj: 2026-04-21
---

**Regle absolue : le spend affiche dans le dashboard doit toujours etre avec les billing coefficients appliques.** Jamais le spend brut Supabase.

**Pourquoi :** Le dashboard est consulte par le client (BIL). Le spend brut represente le cout reel plateforme cote agence (avant marge). Si le client voit le brut, il voit une marge interne qu'il ne doit pas voir.

**Comment l'appliquer :**
- Toujours utiliser les RPC qui appliquent `get_billing_coef(client_id, platform)` au spend : `get_dashboard_kpis`, `get_dashboard_by_platform`, `get_dashboard_by_platform_daily`, `get_dashboard_daily` (modifiee le 2026-04-21) et `get_dashboard_export` (ajoutee le 2026-08-11)
- Vaut aussi pour ce qui sort du dashboard : le fichier d'export est telecharge par le client, voir [[export-bilan]] et [[ce-qui-sort-du-dashboard]] (qui couvre l'autre donnee a ne pas exposer : nos line items)
- Avant de creer ou modifier une RPC qui renvoie du spend : toujours ajouter `ci.spend / get_billing_coef(v_client_id, aa.platform::text)`
- Quand on agrege cote JS : utiliser `platformDailyData` (avec coefs) et non pas un champ brut
- Si une colonne "spend" vient d'une source non verifiee, suspecter le brut et verifier la RPC avant d'utiliser
- S'applique a tous les clients du projet Supabase mfqbhpxsuawujnfbcojr, pas juste BIL
