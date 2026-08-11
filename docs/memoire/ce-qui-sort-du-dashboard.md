---
name: Tout ce qui sort du dashboard est vu par le client
description: "Le dashboard et ses exports sont consultés par BIL : ni investi média réel, ni structure interne (line items, ciblages, nommage). Vaut pour toute nouvelle section ou tout nouvel export."
type: feedback
maj: 2026-08-11
---

**Avant d'ajouter une donnée au dashboard ou à un fichier téléchargeable, se demander : est-ce que
le client doit voir ça ?** Deux familles de données ne doivent jamais sortir.

## 1. L'investi média réel

Le spend brut Supabase est le coût plateforme avant marge. Toujours passer par
`get_billing_coef`. Détail et cas d'application dans [[spend-billing-coefs]].

## 2. La structure interne de la campagne

Demande explicite de Matthieu le 2026-08-11, à propos de l'export bilan : *"le client aura accès
et je ne veux pas qu'il voit nos LI"*. Concerne surtout DV360 et Teads, où le line item porte
notre montage média.

Ne sortent pas : les noms de campagne, de groupe d'annonces / line item et d'annonce, les
ciblages et segments (SirData, Equativ, whitelists, Smart Targeting, Finance, Broad), les
environnements d'achat DOOH, et plus généralement notre nommage interne.

Sortent : ce que le client reconnaît de sa propre campagne, c'est-à-dire le canal, la langue,
la créa, la variante de texte et le format.

**Comment l'appliquer :** filtrer côté RPC, pas côté affichage. Masquer une colonne dans la page
ne suffit pas, la réponse Supabase reste lisible dans l'onglet réseau du navigateur pour
quiconque a le code d'accès. Quand on doit lire du nommage pour en déduire une dimension, extraire
un **vocabulaire fermé** (voir les helpers `dashboard_*_of` décrits dans [[export-bilan]]) : un
libellé non prévu ne fuit alors jamais.
