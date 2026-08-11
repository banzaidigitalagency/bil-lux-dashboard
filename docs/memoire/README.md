# Mémoire projet BIL Dashboard

Contexte accumulé au fil des sessions, versionné dans le repo pour que toute l'équipe
(et Claude Code sur n'importe quelle machine) y ait accès.

`CLAUDE.md` à la racine reste l'autorité sur l'architecture technique du fichier `index.html`.
Ces notes-ci portent sur le **pourquoi** : décisions prises, préférences de l'équipe, pièges connus.

## Index

| Fichier | Contenu |
|---|---|
| [projet-etat.md](projet-etat.md) | État du projet : 8 canaux, stack, coefs billing, décisions, pièges connus. À jour au 2026-08-11 (dashboard multi-campagnes Belgian + Knokke, chiffres de livraison) |
| [budgets-source-de-verite.md](budgets-source-de-verite.md) | Chaîne Sheet "Matt - 26" → sync → `campaign_tracking` → RPC planning, piège de staleness |
| [design-approche.md](design-approche.md) | Comment itérer sur le design : garder les sections et le violet BIL, artisanat plutôt que refonte |
| [redaction-tirets-cadratins.md](redaction-tirets-cadratins.md) | Pas de tirets cadratins dans le contenu visible |
| [spend-billing-coefs.md](spend-billing-coefs.md) | CRITIQUE : jamais de spend brut affiché, toujours avec les billing coefficients |
| [export-bilan.md](export-bilan.md) | Export xlsx / csv : granularité, bouclage exact avec le dashboard, pièges de données |

## Convention

Une note = un sujet. Quand une décision structurante est prise en session, l'ajouter ici
plutôt que de la laisser dans la mémoire locale de Claude Code (qui n'est pas partagée).
Les notes de type "feedback" indiquent le **pourquoi** et le **comment l'appliquer**.

Les dates sont absolues (pas de "la semaine dernière"), pour rester lisibles dans six mois.
