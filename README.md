# screaming-frog-auto

Pipeline de crawl SEO automatisé : Screaming Frog CLI + GitHub Actions + export vers Google Sheets (ou BigQuery).

---

## Comment ca fonctionne

1. **Declenchement** : tous les lundis a 7h UTC (9h Paris) via cron GitHub Actions, ou manuellement via `workflow_dispatch`.
2. **Installation** : le runner Ubuntu telecharge et installe la derniere version du CLI Screaming Frog.
3. **Crawl** : SF crawle le site cible en mode headless et produit deux exports :
   - `internal_all.csv` — rapport Internal:All (stocke dans `results/<date>/`)
   - `_custom_summary_report` — 513 colonnes de metriques agregees, envoye directement dans Google Drive (une ligne ajoutee par crawl)
4. **Upload Google Sheets** : l'onglet `Internal All` du Google Sheet cible est efface et remplace integralement a chaque crawl.
5. **Commit** : les fichiers CSV sont commites dans le repo pour garder un historique Git.

```
GitHub Actions (cron lundi 7h UTC)
        |
        v
Install SF CLI + licence + OAuth Google Drive
        |
        v
Crawl du site
        |-- internal_all.csv --> traitement --> Google Sheets (onglet "Internal All")
        |-- _custom_summary_report --> Google Drive (Sheet natif SF, 1 ligne par crawl)
        |
        v
Commit results/ dans le repo Git
```

---

## Secrets GitHub requis

| Secret | Description |
|--------|-------------|
| `SF_LICENSE_USERNAME` | Email du compte Screaming Frog |
| `SF_LICENSE_KEY` | Cle de licence Screaming Frog |
| `SF_GOOGLE_DRIVE_STORED_CREDENTIAL` | Fichier `StoredCredential` encode en base64 (OAuth SF → Google Drive) |
| `SF_GOOGLE_DRIVE_ACCOUNT_CONFIG` | Fichier `AccountConfig` encode en base64 (OAuth SF → Google Drive) |
| `GCP_SA_KEY` | JSON du service account GCP (pour gspread → Google Sheets API) |
| `GOOGLE_SHEET_ID` | ID du Google Sheet cible (dans l'URL : `/spreadsheets/d/<ID>/`) |

### Ou trouver les fichiers OAuth SF

Apres avoir connecte Google Drive dans l'app Screaming Frog desktop, les fichiers se trouvent sur Mac a :
```
~/.ScreamingFrogSEOSpider/google_drive/saveduser/<email>/StoredCredential
~/.ScreamingFrogSEOSpider/google_drive/saveduser/<email>/AccountConfig
```

Les encoder en base64 pour les ajouter comme secrets :
```bash
base64 -i StoredCredential | pbcopy   # copie dans le presse-papier
base64 -i AccountConfig | pbcopy
```

---

## Configuration

### Site a crawler

Dans `.github/workflows/screaming-frog-crawl.yml`, modifier :
```yaml
--crawl http://spilou.com/
```

### Colonnes du Custom Summary

Le fichier `config/custom_summary_columns.txt` contient les 513 colonnes exportees par SF dans `_custom_summary_report`. Modifier ce fichier pour ajouter/supprimer des colonnes (liste separee par des virgules, sur une seule ligne).

---

## Dupliquer pour un nouveau client

1. Creer un nouveau repo GitHub a partir de celui-ci (bouton "Use this template" ou fork + renommage).
2. Configurer les secrets GitHub (voir tableau ci-dessus).
3. Changer l'URL du site dans le workflow.
4. Creer un Google Sheet vide et partager le avec l'email du service account GCP.
5. Ajouter l'ID du sheet dans le secret `GOOGLE_SHEET_ID`.

---

## Destinations d'export alternatives

### BigQuery (non active dans ce projet)

Pour exporter `internal_all` vers BigQuery au lieu de Google Sheets, remplacer le step "Upload Internal:All to Google Sheets" par :

```yaml
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets.GCP_SA_KEY }}

- name: Setup gcloud
  uses: google-github-actions/setup-gcloud@v2

- name: Upload Internal:All to BigQuery
  env:
    BQ_PROJECT: mon-projet-gcp
    BQ_DATASET: screaming_frog
    BQ_TABLE: internal_all
  run: |
    bq load \
      --source_format=CSV \
      --skip_leading_rows=1 \
      --replace \
      "${BQ_PROJECT}:${BQ_DATASET}.${BQ_TABLE}" \
      "results/${{ env.CRAWL_DATE }}/internal_all.dated.csv"
```

**Prerequis BigQuery :**
- Dataset cree dans le projet GCP
- Service account avec role `BigQuery Data Editor` sur le dataset
- Schema de la table aligne avec les colonnes de `internal_all.dated.csv`

Pour le Custom Summary, SF n'exporte pas nativement vers BigQuery. Il faudrait lire le Google Sheet `_custom_summary_report` via l'API Sheets puis le charger dans BigQuery via `bq load`.

---

## Structure du repo

```
.
├── .github/
│   └── workflows/
│       └── screaming-frog-crawl.yml   # workflow principal
├── config/
│   └── custom_summary_columns.txt     # 513 colonnes du custom summary SF
├── results/
│   └── <date>/
│       ├── internal_all.csv           # export brut SF
│       └── internal_all.dated.csv     # avec colonne crawl_date
└── README.md
```
