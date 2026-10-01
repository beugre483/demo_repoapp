# Pack de données : Ambatolampy II (démo AXIAN Energy)

Journée démo : **22/04/2026**, pas de **1 minute** (1 440 minutes), 77 onduleurs, 20,01 MWp DC.

## Principe : une seule source de vérité

Tout part d'un moteur physique minute par minute, onduleur par onduleur (`inverter_minute.csv`). Tous les autres fichiers en sont **dérivés par sommation**. Rien n'est écrit à la main, donc les sommes concordent par construction. `validate_pack.py` relit les CSV et vérifie 55 recoupements.

```
inverter_minute  ->  ptr_minute  ->  plant_minute  ->  compteur 63 kV
      |                  |
      +-- pertes par catégorie (waterfall) -- scenario_losses_minute -- anomalies
```

## Fichiers

| Fichier | Lignes | Contenu |
|---|---|---|
| `inverters.csv` | 77 | `inverter_id, ptr_id, rank_in_ptr, dc_kwp, ac_kw_nominal, strings_count` |
| `inverter_minute.csv` | 110 880 | par onduleur et par minute : `status, quality, power_dc_kw, voltage_dc_v, current_dc_a, power_ac_kw, energy_ac_kwh, energy_kwh, theoretical_kwh` |
| `ptr_minute.csv` | 7 200 | par poste et par minute : disponibilité, consigne réseau, théorique, 12 catégories de pertes, livré |
| `plant_minute.csv` | 1 440 | la centrale : météo, puissance, cumuls, PR, compteur 63 kV, disponibilité, salissure, anomalies ouvertes, statut |
| `anomalies.csv` | 7 | **une seule liste** ; colonne `detected_by` = Règles, ML ou Les deux |
| `scenarios.csv` | 9 | vérité terrain S01 à S09 + heures de détection réelles |
| `scenario_losses_minute.csv` | ~3 400 | énergie perdue attribuée à chaque scénario |
| `maintenance_events.csv` | 1 | S09, exclue de la disponibilité |
| `losses_waterfall_demo.csv` | 14 | waterfall du jour : théorique, 12 pertes, livré |
| `production_daily.csv` | 30 | avril 2026 : théorique, pertes, livré, PR, disponibilité |
| `reference_values.json` | | valeurs attendues à 10:30, 12:30 et 23:59, pour tester l'intégration |

## Définitions (identiques partout)

- **Énergie livrée** (`energy_kwh`, `delivered_kwh`) : énergie au compteur 63 kV = énergie AC onduleur × (1 − 0,8 % pertes transformateur/ligne).
- **Théorique** (`theoretical_kwh`) : puissance crête DC × irradiance / 1 000, sur la minute.
- **PR** = énergie livrée cumulée / théorique cumulée. Même formule pour une centrale, un poste ou un onduleur.
- **Waterfall** : théorique − température − salissure − câblage − défaut DC (S02) − dérive (S06) − rendement onduleur − écrêtage − déclassement (S03) − limitation réseau (S08) − arrêt (S01) − maintenance (S09) − transformateurs = livré. Fermé minute par minute.
- **Disponibilité** = onduleurs disponibles / onduleurs en périmètre. Hors périmètre : la nuit et les onduleurs en maintenance planifiée. Un onduleur en limitation ou en déclassement reste disponible.
- **Chaînes disponibles** = chaînes en périmètre − chaînes hors service (S02) − chaînes des onduleurs à l'arrêt (S01).
- **Statut centrale** : `Dégradé` si une anomalie Critique ou Majeure est ouverte, `Attention` si seule une anomalie mineure l'est, sinon `Normal`. Les anomalies de catégorie `externe` ne comptent pas.
- **Anomalies ouvertes** = anomalies (catégories `anomalie` et `qualité_données`) détectées et non résolues à cette minute.

## Scénarios de la journée

| ID | Quoi | Où / quand | Règles | ML |
|---|---|---|---|---|
| S01 | Arrêt onduleur | OND9_1, 09:40 → 12:05 | 09:44 | 09:44 |
| S02 | Défaut DC (5 chaînes sur 24) | OND7_9, dès 08:20 | 08:26 | 08:25 |
| S03 | Déclassement thermique à 70 % | OND8_6, 10:45 → 15:15 | 10:50 | 10:49 |
| S04 | Donnée douteuse (pas de communication) | OND10_7, 09:50 → 10:50 | alerte qualité 09:59 | rien |
| S05 | Dérive du capteur de température B (0 → +6 °C) | 07:00 → 17:00 | 13:00 | 11:22 |
| S06 | Dérive lente (jusqu'à 6,5 %) | OND7_2, 06:00 → 14:00 | **rien** (sous le seuil de 8 %) | **09:31** |
| S07 | Passage nuageux | toute la centrale, 11:30 → 11:45 | rien | rien |
| S08 | Limitation réseau à 87 % | PTR6 et PTR7, 12:00 → 13:00 | classée « externe » 12:04 | rien |
| S09 | Maintenance planifiée | OND8_9, 13:15 → 14:45 | exclue | exclue |

Les heures de détection viennent de **vrais algorithmes appliqués aux mesures** (écart au poste pour les règles ; modèle de comportement normal pour le ML), pas de valeurs saisies.

## Hypothèses à connaître

1. **DC simulé.** La vraie centrale ne fournit pas le DC (tension, courant, chaînes). Il est simulé pour que S02 et S03 se distinguent. Sur un site AC seul, S02 et S03 se ressembleraient.
2. **Puissances DC des onduleurs fictives** (autour de 260 kWp, total exact 20 010 kWp), sauf OND10_4, OND10_10, OND10_11, OND10_12 vus dans la démo. À remplacer par la vraie liste.
3. **Ratio DC/AC = 1,25** (AC nominal ≈ 208 kW), pour que l'écrêtage soit visible. À confirmer.
4. **24 chaînes par onduleur**, câblage + mismatch 2,2 %, pertes transformateur/ligne 0,8 % : hypothèses.
5. **Index compteur 63 kV fictif** : 38 412,500 MWh au 01/04/2026.
6. **Le « ML » est un modèle simple** (arbres de décision par gradient boosting, appris sur les minutes saines). Il illustre le principe, ce n'est pas un modèle de production.
7. **Énergie manquante (S04)** : `power_ac_kw` est vide pendant la coupure, mais `energy_kwh` est consolidée (estimée à partir des voisins). C'est ce qui garde le compteur et la somme des onduleurs cohérents.
8. **Les jours hors 22/04** (`production_daily.csv`) ont des événements aléatoires légers (6 jours) et une météo variée. Seul le 22/04 est détaillé à la minute. Si tu veux un rapport mensuel avec les scénarios répartis sur le mois, on l'enrichit.
9. **Écrêtage et dérive S06** : autour de midi, l'écrêtage masque une partie d'une perte faible. C'est physique, et c'est pourquoi le ML repère S06 le matin.

## Régénérer

```
python3 build_pack.py      # écrit data_pack/ (reproductible, graine fixe)
python3 validate_pack.py   # 55 contrôles, échoue au moindre écart
```
