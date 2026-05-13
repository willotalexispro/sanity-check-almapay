# sanity-check-almapay

Pipeline de crawl SEO automatisé : Screaming Frog CLI + GitHub Actions + export vers Google Sheets (ou BigQuery).

Site crawle : **https://almapay.com/**
Declenchement : **tous les dimanches a 23h UTC (minuit/1h Paris)**

---

## Comment ca fonctionne

1. **Declenchement** : tous les dimanches a 23h UTC via cron GitHub Actions, ou manuellement via `workflow_dispatch`.
2. **Installation** : le runner Ubuntu telecharge et installe la derniere version du CLI Screaming Frog.
3. **Crawl** : SF crawle le site en mode headless avec la configuration `config/almapay.seospiderconfig` et produit deux exports :
   - `internal_all.csv` — rapport Internal:All (stocke dans `results/<date>/`)
   - `_custom_summary_report` — 513 colonnes de metriques agregees, envoye directement dans Google Drive (une ligne ajoutee par crawl)
4. **Upload Google Sheets** : l'onglet `Internal All` du Google Sheet cible est efface et remplace integralement a chaque crawl.
5. **Commit** : les fichiers CSV sont commites dans le repo pour garder un historique Git.

```
GitHub Actions (cron dimanche 23h UTC)
        |
        v
Install SF CLI + licence + OAuth Google Drive
        |
        v
Crawl du site (config almapay.seospiderconfig)
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
| `SF_LICENSE_USERNAME` | Username du compte Screaming Frog (pas l'email — voir commande SF Order) |
| `SF_LICENSE_KEY` | Cle de licence Screaming Frog — format `XXXX-XXXX-XXXX-XXXX`, disponible sur la page **Order** du compte SF. **Ne pas dériver de `licence.txt`** qui ne contient que le username. |
| `SF_LEASE_JSON` | Fichier `lease.json` encode en base64 (voir section Licence ci-dessous) |
| `SF_GOOGLE_DRIVE_STORED_CREDENTIAL` | Fichier `StoredCredential` encode en base64 (OAuth SF → Google Drive) |
| `SF_GOOGLE_DRIVE_ACCOUNT_CONFIG` | Fichier `AccountConfig` encode en base64 (OAuth SF → Google Drive) |
| `GCP_SA_KEY` | JSON du service account GCP (pour gspread → Google Sheets API) |
| `GOOGLE_SHEET_ID` | ID du Google Sheet cible (dans l'URL : `/spreadsheets/d/<ID>/`) |
| `SF_PSI_API_KEY` | Clé API Google PageSpeed Insights (pour les métriques PSI dans le crawl SF) |
| `SF_GSC_STORED_CREDENTIAL` | Fichier `StoredCredential` GSC encodé en base64 (OAuth SF → Google Search Console) |
| `SF_GSC_ACCOUNT_CONFIG` | Fichier `AccountConfig` GSC encodé en base64 (OAuth SF → Google Search Console) |
| `ANTHROPIC_API_KEY` | Clé API Anthropic (pour l'analyse Claude dans la notification Slack) |
| `SLACK_WEBHOOK_URL` | URL du webhook Slack entrant pour les notifications |

---

## Licence Screaming Frog — fonctionnement et solution

### Le probleme : licence liee a une machine physique

SF utilise un systeme de licence machine-bound. Lors de la premiere activation sur un Mac, SF genere un fichier `lease.json` signe cryptographiquement, qui contient :

```json
{
  "username": "willotalexis",
  "licence_key": "XXXX-XXXX-XXXX",
  "licence_type": "fixed-term",
  "machine_id": "8f6307e2-82bb-449b-953c-4487a780aaaf",
  "timestamp": 1776695719,
  "signature": "..."
}
```

Le runner GitHub Actions possede un `machine_id` different → SF retourne `Licence Status: Invalid` et limite le crawl a 500 URLs.

### La solution : usurper l'identite du Mac

Le workflow restaure deux fichiers avant de lancer SF :

1. **`lease.json`** (secret `SF_LEASE_JSON`) — le bail signe pour le Mac
2. **`machine-id.txt`** — hardcode avec le `machine_id` du Mac (`8f6307e2-82bb-449b-953c-4487a780aaaf`)

SF valide la signature du `lease.json` par rapport au `machine_id` present → la validation passe.

### Encoder le lease.json pour le secret

```bash
base64 -i ~/.ScreamingFrogSEOSpider/lease.json | pbcopy
```

Coller la valeur dans le secret GitHub `SF_LEASE_JSON`.

### Points d'attention

- **Expiration** : la licence expire le 30 Jul 2026. Apres renouvellement, un nouveau `lease.json` sera genere — mettre a jour le secret `SF_LEASE_JSON`.
- **Changement de Mac** : si SF est reactivee sur un nouveau Mac, le `machine_id` change. Il faut mettre a jour le secret `SF_LEASE_JSON` ET le `machine-id.txt` hardcode dans le workflow.
- **`SF_LICENSE_USERNAME`** : c'est le username (ex: `willotalexis`), pas l'email. Se trouve sur la page Order de ton compte SF.

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

## Configuration SF

Le fichier `config/almapay.seospiderconfig` contient le profil de configuration Screaming Frog pour ce site. Il est charge via le flag `--config` dans le workflow.

Pour le mettre a jour :
1. Ouvrir SF desktop et configurer selon tes besoins
2. `Configuration > Profiles > Save As...` → sauvegarder en `.seospiderconfig`
3. Remplacer `config/almapay.seospiderconfig` dans le repo

### Colonnes du Custom Summary

Le fichier `config/custom_summary_columns.txt` contient les 513 colonnes exportees par SF dans `_custom_summary_report`. Modifier ce fichier pour ajouter/supprimer des colonnes (liste separee par des virgules, sur une seule ligne).

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

## Architecture du workflow (4 jobs)

```
crawl (~40 min)
   ├── sheets (~1 min)  ─┐
   └── deck   (~2 min)  ─┴── slack (~30 sec)
```

- **crawl** : installation SF, crawl, traitement CSV, commit dans le repo (garde les 8 dernières semaines)
- **sheets** : upload des 3 onglets vers Google Sheets (Internal All, Crawl Overview, Issues Overview)
- **deck** : génération du PPTX avec comparaison W-o-W + upload vers le dossier Google Drive du client
- **slack** : analyse Claude + notification avec lien vers le deck

Si un job échoue, utiliser **"Re-run failed jobs"** dans GitHub Actions pour ne relancer que la partie concernée.

---

## Structure du repo

```
.
├── .github/
│   └── workflows/
│       └── screaming-frog-crawl.yml   # workflow principal (4 jobs)
├── config/
│   ├── almapay.seospiderconfig        # profil de configuration SF
│   └── custom_summary_columns.txt     # colonnes du custom summary SF
├── pipeline/
│   ├── generate.js                    # générateur de deck PPTX (pptxgenjs)
│   ├── parseCrawl.js                  # parser du CSV Crawl Overview SF
│   ├── package.json
│   └── configs/
│       └── almapay.json               # config client (nom, site, agence)
├── results/                           # 8 dernières semaines conservées
│   └── <date>/
│       ├── internal_all.csv
│       ├── internal_all.dated.csv
│       ├── crawl_overview.csv
│       └── issues_overview_report.csv
└── README.md
```
