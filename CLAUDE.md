# Instructions pour Claude — Duplication de ce pipeline

Quand l'utilisateur demande de dupliquer ce repo pour un nouveau client, suivre ce process exact :

## Étape 1 : Dupliquer le repo

1. Créer un nouveau repo GitHub privé `willotalexispro/sanity-check-<client>`
2. Copier tous les fichiers en adaptant :
   - L'URL crawlée dans le workflow
   - Le nom du fichier `.seospiderconfig` dans le workflow et dans `config/`
3. Copier les secrets qui existent en local (voir section ci-dessous)

## Étape 2 : Demander à l'utilisateur les éléments manquants

Après duplication, envoyer ce message à l'utilisateur (sans omettre aucun point) :

> **Repo dupliqué. Pour qu'il fonctionne, j'ai besoin de :**
>
> 1. **L'URL à crawler** — ex: `https://client.com/`
> 2. **Le fichier `.seospiderconfig`** — l'exporter depuis SF desktop : `Configuration > Profiles > Save As...`, puis me donner le chemin du fichier
> 3. **L'ID du Google Sheet** — dans l'URL du sheet : `/spreadsheets/d/<ID>/`
> 4. **La clé de licence SF** (`SF_LICENSE_KEY`) — format `XXXX-XXXX-XXXX-XXXX`, disponible sur la page **Order** du compte Screaming Frog (pas l'email, pas le username — la clé elle-même)

## Secrets copiés automatiquement depuis la machine locale

Ces secrets peuvent être re-encodés sans intervention de l'utilisateur :

| Secret | Source locale |
|--------|---------------|
| `SF_LICENSE_USERNAME` | `head -1 ~/.ScreamingFrogSEOSpider/licence.txt` |
| `SF_LEASE_JSON` | `~/.ScreamingFrogSEOSpider/lease.json` (base64) |
| `SF_GOOGLE_DRIVE_STORED_CREDENTIAL` | `~/.ScreamingFrogSEOSpider/google_drive/saveduser/<email>/StoredCredential` (base64) |
| `SF_GOOGLE_DRIVE_ACCOUNT_CONFIG` | `~/.ScreamingFrogSEOSpider/google_drive/saveduser/<email>/AccountConfig` (base64) |
| `GCP_SA_KEY` | `~/Desktop/Configurations_ScreamingFrog_Auto/githubactions-*.json` |

## ⚠️ Piège connu : SF_LICENSE_KEY

**`licence.txt` ne contient QUE le username** (`willotalexis`), pas la clé de licence.  
Ne jamais dériver `SF_LICENSE_KEY` depuis ce fichier — il sera vide et le crawl échouera avec `Licence invalid`.  
La clé doit être fournie manuellement par l'utilisateur (format `XXXX-XXXX-XXXX-XXXX`).
