# 🚗 Automatisation & Analyse de Données — Python + SQL Server + Power BI

Ce projet est parti d'un vrai problème : une concession automobile qui gérait encore
ses données manuellement dans des fichiers Excel éparpillés. Chaque rapport prenait
des heures à produire, les données n'étaient jamais à jour, et la direction prenait
des décisions sur des chiffres vieux de plusieurs semaines.

L'objectif : automatiser tout le pipeline — de l'extraction des données à la
mise à jour quotidienne du dashboard Power BI — sans intervention humaine.

---

## Architecture du système

```
  Sources de données
  (Excel, CSV, saisies manuelles)
          │
          │  Extraction + Nettoyage
          ▼
  ┌───────────────────────────┐
  │    Scripts Python         │
  │                           │
  │  - Chargement multi-      │
  │    sources (pandas)       │
  │  - Validation et          │
  │    normalisation          │
  │  - Conformité LGPD/GDPR   │
  │    (anonymisation)        │
  └────────────┬──────────────┘
               │  pyODBC / SQLAlchemy
               ▼
  ┌───────────────────────────┐
  │      SQL Server           │
  │                           │
  │  - Tables structurées     │
  │  - Vues analytiques       │
  │  - Procédures stockées    │
  │    pour agrégations       │
  └────────────┬──────────────┘
               │  Connexion DirectQuery
               ▼
  ┌───────────────────────────┐
  │       Power BI            │
  │                           │
  │  Dashboard temps réel     │
  │  pour la direction        │
  └───────────────────────────┘
               ▲
               │
  ┌────────────┴──────────────┐
  │   Python schedule         │
  │   (tâche nocturne)        │
  │                           │
  │  Mise à jour automatique  │
  │  chaque nuit à 23h00      │
  │  → toujours à jour        │
  │    le matin               │
  └───────────────────────────┘
```

---

## Ce que fait le pipeline automatisé

```python
import schedule
import time
from pipeline import run_full_pipeline

def nightly_job():
    print(f"[{datetime.now()}] Démarrage pipeline...")

    # 1. Extraction des nouvelles données
    raw_data = extract_all_sources()

    # 2. Nettoyage et validation
    clean_data = clean_and_validate(raw_data)

    # 3. Anonymisation LGPD (équivalent GDPR brésilien)
    anonymized = anonymize_personal_data(clean_data)

    # 4. Chargement dans SQL Server
    load_to_sqlserver(anonymized)

    print(f"[{datetime.now()}] Pipeline terminé — {len(clean_data)} lignes traitées")

# Planification : chaque nuit à 23h00
schedule.every().day.at("23:00").do(nightly_job)

while True:
    schedule.run_pending()
    time.sleep(60)
```

---

## Distribution sans Python installé

Une contrainte importante : les postes de la concession n'avaient pas Python.
Solution : PyInstaller pour transformer les scripts en `.exe` autonomes.

```bash
# Créer l'exécutable standalone
pyinstaller --onefile --noconsole pipeline.py

# Résultat : dist/pipeline.exe
# → Fonctionne sur n'importe quel PC Windows
# → Aucune installation Python requise
# → Planifiable via le Planificateur de tâches Windows
```

---

## Exemples de requêtes SQL analytiques

```sql
-- Ventes par commercial ce mois vs mois précédent
SELECT
    commercial,
    SUM(CASE WHEN MONTH(date_vente) = MONTH(GETDATE())
             THEN montant ELSE 0 END) AS ventes_ce_mois,
    SUM(CASE WHEN MONTH(date_vente) = MONTH(GETDATE()) - 1
             THEN montant ELSE 0 END) AS ventes_mois_precedent,
    ROUND(
        (SUM(CASE WHEN MONTH(date_vente) = MONTH(GETDATE())
                  THEN montant ELSE 0 END) /
         NULLIF(SUM(CASE WHEN MONTH(date_vente) = MONTH(GETDATE()) - 1
                         THEN montant ELSE 0 END), 0) - 1) * 100, 1
    ) AS evolution_pct
FROM ventes
GROUP BY commercial
ORDER BY ventes_ce_mois DESC;
```

---

## Ce que ce projet m'a appris

La conformité LGPD (équivalent brésilien du GDPR) était une contrainte que je n'avais
pas anticipée. Dès qu'on travaille avec des données clients réels — noms, adresses,
coordonnées — il faut anonymiser avant de stocker en base ou d'afficher en dashboard.
Implémenter le hachage des identifiants sensibles et les règles de rétention des données
m'a ouvert les yeux sur la dimension légale du travail data.

L'autre apprentissage : `schedule` est simple mais fragile. Si le PC s'éteint à 23h00,
la tâche ne s'exécute pas. Pour de la vraie production, le Planificateur de tâches Windows
(ou un cron Linux) est plus fiable car il gère les redémarrages manqués.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
