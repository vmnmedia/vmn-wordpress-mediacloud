# Plan: VMN Media Offload Plugin (S3 + IMGix)

Een standalone WordPress plugin die de rol van Media Cloud Premium (`ilab-media-tools-premium`) overneemt: attachments uit de mediabibliotheek offloaden naar Amazon S3, serveren via IMGix, en daarbij **volledig backwards compatible** zijn met de bestaande postmeta die door MC op ~30 sites is achtergelaten.

Naam: **`vmn-media-offload`**, namespace **`VMN\MediaOffload`**, PSR-4 autoloading naar `src/`.

Doelplatform: **WordPress 6.9.x of hoger, PHP 8.3 of hoger**. Geen ondersteuning voor oudere versies — dit is een nieuwe codebase, geen migratiepad uit MC zelf.

> Dit document consolideert het oorspronkelijke offload-plan met de milestone-structuur uit `PROJECT.md` (concept van een collega dat uitging van een fork van Media Cloud — die aanpak is verworpen). Zie §18 voor de samengevoegde milestone-tracking met SCAF/STOR/OFFL/IMGX/COMP/ADMIN-prefixen.

---

## 1. Uitgangspunten en scope

- Volledig nieuwe codebase — **geen fork van Media Cloud, geen overgenomen code**. Het collega-plan in `PROJECT.md` ging uit van strippen van MC 4.6.4; die aanpak is verworpen omdat MC's architectuur (Freemius, tool-systeem, eigen task-tabellen, sleeping bugs) meer ruis brengt dan we besparen door te hergebruiken.
- Doelplatform: **WordPress 6.9.x of hoger, PHP 8.3 of hoger**. Single site (multisite-veilig waar dat zonder extra werk lukt; geen network-admin UI).
- Storage provider: alleen Amazon S3 (path-style URLs, SigV4).
- CDN/image transformer: alleen IMGix (public mode in v1, signing-hook voorzien voor later).
- Geen integratie in `vmn-base` — dit wordt een standalone plugin.
- Bestaande `VMN\WordPress\Offload_Adapter` (uit `vmn-base`, voor database-dumps) blijft zoals hij is en dient als referentie-implementatie voor het AWS SigV4 PUT-patroon. We knippen niet in vmn-base.
- Backwards compatible met door Media Cloud (Interfacelab / Media Cloud 4.x), MC 2.x én **WP Offload Media (Delicious Brains)** achtergelaten metadata. Dit is harde eis: zonder migratie moeten bestaande URL's blijven werken (zie §3, geverifieerd in §13).

Niet in scope (voor v1):
- Andere providers (GCS, R2, Backblaze, DO Spaces, Wasabi, …).
- Andere image-CDN's dan IMGix.
- Image editor / on-the-fly thumbnails server-side (IMGix doet dat).
- Multisite-netwerk admin instellingen (we lezen instellingen per-site).
- Automatische migratie van eerder geofloadde MC-bestanden naar onze eigen bucket/structuur (niet nodig — de remote files staan al op de juiste plek).
- Freemius / licentiebeheer / "premium tools" — wordt niet ingebouwd.
- Video transcoding, Vision API, Optimizer, Setup Wizard, eigen Task Manager als product-feature (we hebben een lichte progress-UI, geen MC-grade scheduler — zie §11.2).

---

## 2. Analyse van de bestaande situatie

### 2.1 Wat Media Cloud op de attachment achterlaat

Op basis van research van de [Interfacelab/ilab-media-tools](https://github.com/Interfacelab/ilab-media-tools) repository en de Media Cloud-documentatie is de canonieke opslag van offload-info **binnen `_wp_attachment_metadata`** (postmeta key `_wp_attachment_metadata`). Eventuele legacy 2.x-installaties hebben daarnaast losse `ilab_s3_info` / `ilab-s3-info-meta` postmeta — die komen we tijdens DB-inspectie nog tegen.

Schema dat MC schrijft op de top-level van `_wp_attachment_metadata`:

```php
$meta['s3'] = [
    'url'       => 'https://cdn.example.com/uploads/2024/03/photo.jpg',
    'bucket'    => 'my-bucket',
    'privacy'   => 'public-read',            // 'public-read' | 'private' | 'authenticated-read'
    'v'         => '4.0.2',                   // MEDIA_CLOUD_INFO_VERSION
    'key'       => 'uploads/2024/03/photo.jpg', // object key in de bucket
    'provider'  => 's3',                      // 's3' | 'google' | 'do' | 'minio' | ...
    'mime-type' => 'image/jpeg',
    'options'   => [ /* ACL, CacheControl, ... */ ],
];
```

En **identiek per size** onder `$meta['sizes'][$size_name]['s3']`. Dit is het belangrijkste: de canonical truth zit in `key` + `bucket` (en `provider`).

Filters die MC gebruikt voor URL-rewriting (en die wij óók moeten haken om compatible te zijn met content dat op `wp_get_attachment_url` leunt):
- `wp_get_attachment_url`
- `wp_get_attachment_image_src`
- `wp_calculate_image_srcset`
- `get_attached_file` (priority 10000) — zodat lokaal-ontbrekende files alsnog werken.
- `wp_update_attachment_metadata` (priority 1000) — hier schrijft MC z'n `s3`-subarrays.

Upload path bij MC: `uploads/@{date:Y/m}` (default), wat resulteert in `uploads/{year}/{month}/{filename}` — exact wat wij willen voortzetten.

Verwijderen: MC heeft twee settings, "Delete from Server" (lokaal verwijderen ná offload) en "Delete from Storage" (remote verwijderen bij delete uit medialibrary). Beide haken we in dezelfde manier in.

Cache-tabel: MC heeft een eigen tabel `wp_mcloud_post_map` voor URL→attachment-ID lookups. Die hebben wij **niet** nodig — we vertrouwen op postmeta. Tabel kan na migratie blijven staan (geen kwaad), of in een aparte cleanup-routine geleegd worden.

### 2.2 Wat we hergebruiken uit `Offload_Adapter`

De bestaande `VMN\WordPress\Offload_Adapter` in `vmn-base` doet al precies wat we voor S3 nodig hebben:
- AWS SigV4 ondertekening met de hand (geen aws-sdk-php nodig → geen Composer-dependency-hel).
- `PUT` via `wp_remote_request` met `UNSIGNED-PAYLOAD` content-hash (veilig over HTTPS).
- Optionele pre-signed GET-URLs.
- HMAC-SHA256 signing key-derivation (`AWS4` prefix).

We **forken het patroon** (niet de klasse zelf — we maken geen dependency op vmn-base) in de plugin als bv. `VMN\MediaOffload\Aws\S3_Client`. Verschillen:
- Credentials komen uit DB-settings i.p.v. constanten.
- `Content-Type` per file dynamisch (`wp_check_filetype`) i.p.v. hardcoded `application/octet-stream`.
- Optioneel `x-amz-acl: public-read` en `Cache-Control` headers ondersteunen.
- Stream-vriendelijke upload (`CURLOPT_INFILE`) voor grote files i.p.v. `file_get_contents` — anders kappen we op 200MB-attachments.

---

## 3. Backwards compatibility-strategie

**Vier** lagen — DB-inspectie (§13) heeft uitgewezen dat er drie historische offload-tools sporen hebben achtergelaten, niet twee. Volgorde per attachment-request:

1. **Onze eigen postmeta** (`_vmn_offload`, zie §5). Indien aanwezig en `status === 'success'` → IMGix-URL serveren.
2. **Media Cloud's `_wp_attachment_metadata['s3']`** (top-level én per-size onder `sizes[*]['s3']`). Bevestigd als de canonical locatie op vrijwel alle attachments (bv. dearchitect: 157.090/157.371). **We schrijven hier niet meer in** — read-only fallback.
3. **Legacy `ilab_s3_info` postmeta** (MC 2.x). Bevestigd aanwezig op alle 6 onderzochte sites (cmweb 35, dearchitect 4.201, amt 10.391, provada 486, pwnet 586, recycling_magazine 1). Shape: `a:2:{file, s3:{url,bucket,privacy,key,provider,v,optimized,options,formats,mime-type}}` — feitelijk dezelfde `s3`-substructuur als laag 2. Read-only fallback.
4. **Delicious Brains "WP Offload Media" legacy** (`amazonS3_info` postmeta). Nieuw ontdekt: op dearchitect (128.186), amt (44.055) en pwnet (6.217). Significant: dearchitect heeft **2.796 attachments die ALLEEN deze laag hebben** (geen MC-meta erbij), dus zonder deze fallback breken hun URL's. Shape: `a:3:{bucket, key, region}` — eenvoudiger maar bevat alle info die we nodig hebben. Keys hebben altijd `uploads/`-prefix (geverifieerd: 128.186/128.186 op dearchitect).
5. **Niets** → lokaal serveren (default WP-gedrag).

Op die manier:
- Bestaande sites blijven onmiddellijk werken na een swap (deactivate MC, activate ons).
- Nieuw geüploade attachments krijgen onze metadata-structuur.
- Een optionele "normalize" wp-cli command kan eventueel MC-metadata kopiëren naar onze sleutels (zie §11), maar het is geen vereiste — lagen 2–4 blijven fungeren.

### 3.1 Variatie in key-format (kritisch)

DB-inspectie heeft aangetoond dat er **drie key-formats** in productie staan en we daar niets aan kunnen aannemen — we moeten altijd de `key` letterlijk uit metadata lezen, nooit reconstrueren uit lokaal pad.

| Site | Voorbeeldkey | Patroon |
|---|---|---|
| dearchitect, amt, pwnet, recycling_magazine | `uploads/2018/08/photo-150x150.jpg` | mét `uploads/`-prefix |
| provada | `2025/02/photo-300x174.png` | zónder `uploads/`-prefix |
| cmweb | `2015/02/XWmFJ8PG-photo-pdf-150x150.jpg` | zónder `uploads/`, mét random 8-char filename-prefix (MC anti-enumeration feature) |

Voor IMGix maakt het niet uit (IMGix origin = de bucket, dus de IMGix-URL is `{imgix_domain}/{key_exact}`). Maar het maakt onze code wel afhankelijk van **literal-key-respect**: geen path-rewriting, geen suffix-stripping.

Voor **nieuwe uploads** kiezen wij één consistent format. Voorstel: `uploads/{year}/{month}/{filename}` (matched 4 van 6 sites en is identiek aan lokaal WP-pad — minste ruis).

---

## 4. Plugin-architectuur

### 4.1 Directorystructuur

```
vmn-media-offload/
├── vmn-media-offload.php              # Plugin bootstrap (header, autoloader, init)
├── composer.json                       # Alleen PSR-4 autoloader, geen runtime deps
├── README.md
├── uninstall.php                       # Opruimen van eigen options bij verwijdering (niet postmeta)
├── languages/
├── assets/
│   ├── admin.css
│   └── admin.js                        # Bulk progress, retry-knoppen
└── src/
    ├── Plugin.php                      # Bootstrap, dependency wiring
    ├── Config.php                      # Centrale plek voor "vaste waardes"
    ├── Settings/
    │   ├── Settings.php                # Settings API registratie + Settings page
    │   └── Settings_Page.php
    ├── Aws/
    │   └── S3_Client.php               # SigV4 PUT/DELETE/HEAD/presign (gebaseerd op Offload_Adapter)
    ├── Storage/
    │   ├── Offload_Service.php         # Orchestratie: lokaal → S3 → metadata schrijven
    │   ├── Attachment_Meta.php         # Lees/schrijf onze postmeta + MC-fallback
    │   └── Path_Builder.php            # uploads/{year}/{month}/{file} bepalen
    ├── Url/
    │   ├── Url_Rewriter.php            # Filters wp_get_attachment_url etc. + IMGix-URL bouwer
    │   └── Content_Filter.php          # Optionele the_content filter
    ├── Hooks/
    │   ├── Upload_Hooks.php            # Na wp_generate_attachment_metadata → offload
    │   ├── Delete_Hooks.php            # delete_attachment → remote delete
    │   └── Media_Library_Hooks.php     # Mediagrid kolom + per-attachment retry UI
    ├── Bulk/
    │   ├── Bulk_Offloader.php          # Admin page + AJAX/REST handlers + progress
    │   └── Content_Url_Replacer.php    # the_content S&R bij bulk run
    ├── Cli/
    │   └── Cli_Commands.php            # `wp vmn-offload ...` (offload-id, bulk, replace-urls, doctor)
    └── Support/
        ├── Logger.php                  # Errors per attachment + globaal
        └── Mime.php                    # mime/content-type helpers
```

### 4.2 Bootstrap

- Hooked op `plugins_loaded`.
- Lazy: filters/hooks alleen registreren als `Settings::is_configured()` true is (anders is de plugin "uit" en doet niets — voorkomt halve offloads bij missende credentials).

---

## 5. Onze eigen attachment-metadata

We slaan onze offload-status op in **één** postmeta-rij per attachment, key `_vmn_offload`. Bewust **niet** binnen `_wp_attachment_metadata`, om te voorkomen dat WP-core of andere plugins het per ongeluk vertrappen. Wel **mirrorren we** naar `_wp_attachment_metadata['s3']` (en `sizes[*]['s3']`) in MC-compatible shape, zodat third-party code die op die structuur leunt blijft werken.

Shape van `_vmn_offload`:

```php
[
    'status'        => 'success',          // 'pending'|'success'|'failed'|'partial'
    'provider'      => 's3',
    'bucket'        => 'my-bucket',
    'region'        => 'eu-west-1',
    'key'           => 'uploads/2026/05/photo.jpg',
    'url'           => 'https://my-bucket.s3.eu-west-1.amazonaws.com/uploads/2026/05/photo.jpg',
    'imgix_url'     => 'https://my-source.imgix.net/uploads/2026/05/photo.jpg',
    'mime_type'     => 'image/jpeg',
    'privacy'       => 'public-read',
    'old_url'       => 'https://site.tld/wp-content/uploads/2026/05/photo.jpg',
    'offloaded_at'  => 1715731200,         // unix ts
    'plugin_version'=> '1.0.0',
    'sizes'         => [
        'thumbnail' => [
            'key'       => 'uploads/2026/05/photo-150x150.jpg',
            'url'       => '…',
            'imgix_url' => '…',
            'status'    => 'success',
        ],
        // …
    ],
    'errors'        => [                    // historie voor diagnose
        // [ 'at' => 1715731199, 'message' => 'AWS HTTP 403: SignatureDoesNotMatch', 'size' => 'medium' ],
    ],
    'attempts'      => 1,
]
```

Tegelijk schrijven we de MC-compatible mirror:

```php
$meta['s3'] = [
    'url'       => $url,
    'bucket'    => $bucket,
    'privacy'   => 'public-read',
    'v'         => '1.0.0',                 // onze plugin-versie, zodat MC-readers herkennen
    'key'       => $key,
    'provider'  => 's3',
    'mime-type' => $mime,
    'options'   => [],
];
$meta['sizes'][$size]['s3'] = [ … gelijke shape per size … ];
```

Dat geeft drie voordelen:
1. Onze interne logica leest snel en typed uit `_vmn_offload`.
2. Externe code die op MC-shape vertrouwt (custom blokken, oude exports) blijft werken.
3. Bij eventuele rollback kunnen we zonder data-verlies terug — de MC-shape blijft de bron-van-waarheid voor *anderen*.

---

## 6. Settings

### 6.1 Beheerbare instellingen (Settings → Media Offload)

| Sleutel (option) | Type | Beschrijving |
|---|---|---|
| `vmn_offload_access_key` | string | AWS Access Key ID |
| `vmn_offload_secret_key` | string (write-only, masked) | AWS Secret Access Key |
| `vmn_offload_bucket` | string | S3 bucket-naam |
| `vmn_offload_region` | string | AWS region (bv. `eu-west-1`) |
| `vmn_offload_imgix_domain` | string | bv. `mysite.imgix.net` |
| `vmn_offload_delete_local_after` | bool | Lokaal verwijderen na succesvolle offload? |
| `vmn_offload_delete_remote_on_delete` | bool | Remote verwijderen wanneer attachment uit MB wordt gegooid? **Default off** — buckets hebben momenteel géén versioning (bevestigd, zie §19.4), dus delete is destructief en moet bewust aan gezet worden. |
| `vmn_offload_serve_via_imgix` | bool | Master-toggle voor IMGix. Bij `false` worden URL's direct naar de S3-bucket-URL gerouteerd (fallback bij IMGix-storing of voor sites die geen image-CDN nodig hebben). |
| `vmn_offload_concurrent_uploads` | int (default 6, max 12) | Aantal `curl_multi`-handles per attachment-offload. |

Opgeslagen als **één** option (`vmn_offload_settings`, array) i.p.v. losse rijen → minder DB-roundtrips en makkelijker te exporteren/dupliceren tussen sites.

Secret key opslag: optioneel encrypted-at-rest via `AUTH_KEY` / `SECURE_AUTH_KEY` (eenvoudige `openssl_encrypt`). Bij voorkeur lezen we credentials uit constanten zodra `VMN_OFFLOAD_ACCESS_KEY` / `VMN_OFFLOAD_SECRET_KEY` etc. defined zijn — DB-fallback alleen als constanten ontbreken. Dat sluit aan op hoe `Offload_Adapter` werkt en past bij onze deploys.

Settings page heeft een **"Test connection"** knop: doet een `HEAD` op `https://{bucket}.s3.{region}.amazonaws.com/?max-keys=0` met SigV4, meldt 200/403/404 in begrijpelijke taal.

### 6.2 Vaste waarden (in `Config.php`, niet beheerbaar)

In `src/Config.php` als publieke constanten — zodat we ze later met minimaal werk naar settings kunnen promoveren.

```php
final class Config {
    public const PATH_TEMPLATE        = 'uploads/{year}/{month}';   // letterlijk identiek aan WP-lokaal
    public const FORCE_HTTPS          = true;                        // altijd https
    public const FORCE_PATH_STYLE_URL = true;                        // alle MC-sites draaien met use-path-style-endpoint (buckets bevatten dots in de naam: vmn-cmweb.nl etc.)
    public const DEFAULT_ACL          = 'public-read';
    public const DEFAULT_CACHE_CTL    = 'public, max-age=31536000, immutable';
    public const SIGNATURE_VERSION    = 'v4';
    public const STORAGE_CLASS        = 'STANDARD';
    public const IMGIX_AUTO           = 'compress,format';           // ?auto=compress,format
    public const IMGIX_DEFAULT_Q      = 75;                          // verhoogd t.o.v. MC-instelling (was 50), zie §19.8
    public const IMGIX_USE_HTTPS      = true;
    public const META_KEY             = '_vmn_offload';
    public const META_KEY_LEGACY      = '_wp_attachment_metadata';   // mirror-target
    public const RETRY_MAX            = 3;                            // bij upload-poging
    public const UPLOAD_TIMEOUT_S     = 300;
    public const BIG_SIZE_THRESHOLD   = 2560;                        // mcloud-storage-big-size-threshold
    public const BIG_SIZE_PRIVACY     = 'private';                   // originals > threshold krijgen private ACL (matches MC-config)
}
```

> Antwoorden uit DB-inspectie (zie §13):
> - **ACL's**: alle sites `public-read` voor regular files. **Big-size originals (>2560px) zijn `private`** — bv. cmweb heeft 576 private attachments. Onze plugin moet die ACL respecteren én voor private files via IMGix gaan (`mcloud-imgix-serve-private-images: on` is overal aan).
> - **IMGix params**: `auto-format` en `auto-compress` overal aan, `default-quality=50` overal — we moeten de `q=50` standaard meesturen.
> - **Path-prefix**: gemengd. `mcloud-storage-prefix` is overal `uploads/@{date:Y/m}` maar in werkelijkheid hebben cmweb en provada keys zónder `uploads/`-prefix (zie §3.1). PATH_TEMPLATE is dus alleen relevant voor nieuwe uploads — voor bestaande gebruiken we de stored key letterlijk.
> - **Random filename-prefix**: cmweb heeft een MC-feature aan staan die 8-char random prefixes toevoegt aan filenames (`XWmFJ8PG-photo.jpg`). Andere sites niet. We hoeven dit voor nieuwe uploads niet over te nemen, mits we bestaande keys letterlijk respecteren.

---

## 7. Upload flow

### 7.1 Hook-volgorde voor afbeeldingen

WordPress' eigen volgorde bij een upload is: file → metadata stub → thumbnails genereren → `wp_generate_attachment_metadata` → `wp_update_attachment_metadata` → `add_attachment`. We willen offloaden **nadat alle sizes lokaal zijn**, zodat we ze in één run kunnen pakken.

```
add_filter( 'wp_generate_attachment_metadata', [ Upload_Hooks::class, 'after_metadata_generated' ], 9999, 2 );
```

Returnt de (gemuteerde) `$metadata` array — met daarin onze mirror onder `['s3']` en `['sizes'][*]['s3']`. We schrijven ook `_vmn_offload` via `update_post_meta()`.

Voor non-images valt WP terug op `wp_generate_attachment_metadata` met een lege of minimal metadata array. Dezelfde hook werkt — we slaan alleen de single-file offload op (geen `sizes`).

### 7.2 Algoritme

```
function offload_attachment( $attachment_id ):
    $files = collect_files( $attachment_id )            # master + alle sizes
    $results = []
    foreach $files as $size_name => $local_path:
        $key   = build_key( $local_path )               # uploads/{year}/{month}/{basename}
        $ok    = $s3->put( $local_path, $key, $mime, $acl, $cache_ctl )
        $results[$size_name] = [ 'key' => $key, 'ok' => $ok, 'error' => $err ?? null ]
    end foreach

    $any_failed = any( $results, fn($r) => !$r['ok'] )

    if all_succeeded:
        write_meta( $attachment_id, status: 'success', sizes: $results )
        mirror_into_wp_attachment_metadata( $attachment_id, $results )
        if settings.delete_local_after:
            delete_local_files( $files )                 # ALLEEN bij volledige success
    elseif any_succeeded:
        write_meta( $attachment_id, status: 'partial', sizes: $results, errors: ... )
        # geen lokale delete
    else:
        write_meta( $attachment_id, status: 'failed', errors: ... )
        # geen lokale delete
    end if
end
```

### 7.3 Hard requirement: nooit lokaal verwijderen bij failure

`delete_local_files()` wordt **uitsluitend** aangeroepen wanneer `status === 'success'` (alle sizes ok). Bij `partial` of `failed` blijft alles lokaal staan. Dit is afgedicht via:
- Unit test op `Offload_Service::should_delete_local_after()`.
- Code review checklist.

### 7.4 Performance / async / concurrency

Voor v1 doen we de upload **synchroon** in de upload-request (geen externe queue). Dat is gedrag dat MC ook had en voorkomt complexiteit met cron/AS-dependencies. Wel met twee belangrijke verbeteringen t.o.v. MC:

- **Stream-based PUT** via `CURLOPT_INFILE` (geen `file_get_contents`). MC laadde grote PDF's/MP4's volledig in geheugen — onze plugin niet.
- **Concurrent multi-size upload** via `curl_multi_*` (OFFL-01 in §18). Een attachment heeft typisch 8–15 sizes (incl. custom `vmn_*`-sizes). MC uploadt deze strikt serieel; per size kost dat ~150–400ms latency. Met `curl_multi_init` paralleliseren we tot bv. 6 in flight → een 12-size attachment gaat van ~3s naar ~600ms. Foutafhandeling per handle blijft individueel — als 1 size faalt, gaat `status` naar `partial` zoals in §7.2.

Bulk-runs (§11) gebruiken dezelfde service per attachment; parallelle uploads van *meerdere* attachments tegelijk wordt v1.1 (Action Scheduler-pad).

---

## 8. URL serving (IMGix + S3)

### 8.1 Filterketen

```
add_filter( 'wp_get_attachment_url',        [ Url_Rewriter::class, 'rewrite_url' ],        10, 2 );
add_filter( 'wp_get_attachment_image_src',  [ Url_Rewriter::class, 'rewrite_image_src' ],  10, 4 );
add_filter( 'wp_calculate_image_srcset',    [ Url_Rewriter::class, 'rewrite_srcset' ],     10, 5 );
add_filter( 'get_attached_file',            [ Url_Rewriter::class, 'rewrite_attached_file'],10000, 2 );
```

### 8.2 Resolve-logica

```
function resolve( $attachment_id, $size = 'full' ):
    $meta = read_vmn_meta( $attachment_id )
    if $meta && $meta['status'] === 'success':
        return imgix_url( $meta, $size )

    $wp_meta = wp_get_attachment_metadata( $attachment_id )
    if isset $wp_meta['s3']:                                       # MC-compatible fallback
        $key = ($size === 'full' || empty $wp_meta['sizes'][$size]['s3'])
             ? $wp_meta['s3']['key']
             : $wp_meta['sizes'][$size]['s3']['key']
        return imgix_url_from_key( $key )

    return null                                                    # caller serveert lokaal
end
```

IMGix-URL bouwen: `https://{imgix_domain}/{key}?auto=compress,format` (de path is letterlijk de S3 key, omdat IMGix gevoed wordt vanuit de bucket). Geen aparte parameter signing nodig zolang we IMGix in public-mode draaien.

### 8.3 Niet-image attachments

Voor PDF/zip/MP4/etc.: direct de S3 public URL serveren, géén IMGix (IMGix is alleen voor images). De resolver detecteert dit op `mime_type`.

---

## 9. Verwijderen

### 9.1 Lokaal na offload

Zie §7. Setting `vmn_offload_delete_local_after` schakelt dit per site.

### 9.2 Remote bij verwijderen uit medialibrary

```
add_action( 'delete_attachment', [ Delete_Hooks::class, 'on_delete' ], 10, 2 );
```

Bij delete:
- Lees `_vmn_offload` én MC-mirror.
- Verzamel **alle keys** (master + sizes, ook MC-keys voor sites met legacy data).
- Roep `S3_Client::delete($key)` aan per key (`DELETE` request, SigV4).
- Log per key naar `Logger`.

Setting `vmn_offload_delete_remote_on_delete` schakelt dit per site uit/aan.

> Veiligheidsnet: bij `false` voor `delete_remote_on_delete` doen we **niets** remote. Wel ruimen we onze postmeta op (zodat de listing klopt). Geen "soft delete" of versionering in v1 — als de bucket versioning aan heeft, vangt AWS dat zelf op.

---

## 10. Per-attachment UI: retry + foutmelding

### 10.1 Mediagrid-kolom

Extra kolom "Offload" in `upload.php` (list view) met badge:
- `Local` (geen offload-poging gedaan)
- `Offloaded` (success)
- `Partial` (sommige sizes mislukt)
- `Failed` (alles mislukt)

Klikbaar → opens een dialog met details.

### 10.2 Attachment edit screen

Custom meta box "Media Offload" op de attachment-bewerkpagina:
- Toont status, S3 key, IMGix URL, datum van laatste poging, attempt count.
- Toont laatste **N error messages** uit `errors` array (timestamp + message + size).
- Knoppen:
    - **Retry offload** (alleen actief bij `failed` / `partial` / als status leeg is)
    - **Force re-offload** (overschrijft remote — confirmatie nodig)
    - **Delete remote only** (verwijdert in S3, behoudt lokaal — verborgen tenzij de admin power-user is)

Backend via REST endpoint `POST /wp-json/vmn-offload/v1/attachments/{id}/retry`. Permission check: `upload_files` capability minimum, of `manage_options` voor force-actions.

---

## 11. Bulk offloader

### 11.1 Admin pagina "Media → Bulk Offload"

Deze pagina vervult de rol van MC's "Task Manager" voor onze use cases (OFFL-02/03 uit PROJECT.md). Layout:
- Bovenaan: tellingen (Total / Offloaded / Failed / Local-only).
- Filters: alleen `failed`, alleen `local-only`, alleen `partial`, of *alles wat nog niet `success` is*.
- Checkbox: **"Vervang URL's in `the_content` na offload"** (default uit).
- Start-knop → kicks off de job.
- Tijdens een lopende run: live progress-bar (polling op `bulk/status`), met onder de bar een lijst van **per-file fouten** (laatste 50, met attachment-link en kort foutbericht — geen full stack).

### 11.2 Job-engine

Twee implementatiepaden — we kiezen **B** maar staan A toe als fallback:

**A. AJAX-loop** (eenvoudig, robuust voor 30 sites met hooguit duizenden attachments):
- Frontend roept `POST /wp-json/vmn-offload/v1/bulk/tick` aan met een batchgrootte van bv. 5.
- Backend pakt 5 attachment-id's, offload ze, retourneert progress.
- Frontend updatet voortgangsbalk, herhaalt tot done. Bij refresh: cursor in transient `_vmn_offload_bulk_cursor`.

**B. Action Scheduler-based** (geschikt voor sites met 50k+ attachments):
- Bij start: queue alle target-id's als `vmn_offload_attachment` actions.
- AS verwerkt 1 per cron-tick. UI poll `bulk/status`.
- Action Scheduler is GPL en kan vendored worden (zoals WooCommerce doet).

Voor v1 implementeren we A (eenvoudiger), met een schone interface zodat we B later kunnen toevoegen zonder UI-breuk.

### 11.3 URL-replacement in `the_content` tijdens bulk

Wanneer de checkbox aan staat:
- Na elk succesvol offloaded attachment: vind alle `wp_posts` waarvan `post_content` de oude lokale URL bevat (en eventuele varianten met size suffixes zoals `-300x200`).
- Vervang door de **IMGix-URL** (niet de S3-URL — we willen door IMGix gaan voor transformaties).
- Werkt ook met de bekende WP image size-suffixen: `name-WxH.ext` → `name.ext` (size = querystring via IMGix `?w=…&h=…`, optioneel).

WP-CLI equivalent: `wp vmn-offload bulk --replace-urls --filter=failed`.

> **Belangrijk:** dit S&R is destructief op `post_content`. We slaan **vóór elke vervanging** de oude URL ook in `_vmn_offload['old_url']` op, en bieden een `wp vmn-offload revert-content` command voor noodgevallen (lookup op `_vmn_offload['old_url']` per attachment → zet terug).

### 11.4 Alternatief: `the_content` filter (geen DB-mutatie)

Als alternatief op de DB-S&R bieden we een filter:

```
add_filter( 'the_content', [ Content_Filter::class, 'rewrite_urls' ], 9999 );
```

Implementatie:
- Regex over `wp-content/uploads/{Y}/{m}/{filename}(?:-WxH)?\.(ext)`.
- Per match: lookup attachment via `attachment_url_to_postid()` (gecached in een memory-cache per request én een persistent transient).
- Vervang door IMGix-URL.

Setting `vmn_offload_content_filter_mode`:
- `none` — niets aanraken (default voor bestaande sites die al S&R hebben gedaan).
- `filter` — runtime filter (geen DB-write).
- `database` — runtime filter UIT, vertrouwt op uitgevoerde S&R.

Zo kan een site bewust kiezen: of een eenmalige bulk replace doen, of leunen op de filter.

---

## 12. WP-CLI commands

Compleet pakket onder `wp vmn-offload …`:

- `wp vmn-offload status [--id=<id>]` — toont config + per-attachment status.
- `wp vmn-offload offload <id> [--force]` — single attachment (re-)offload.
- `wp vmn-offload bulk [--filter=<all|failed|local|partial>] [--replace-urls] [--batch=50]` — bulk run.
- `wp vmn-offload replace-urls [--dry-run]` — alleen de DB-S&R draaien.
- `wp vmn-offload revert-content [--id=<id>]` — terugzetten van `old_url`.
- `wp vmn-offload doctor` — sanity-check: credentials, bucket access, IMGix domain reachable.
- `wp vmn-offload normalize-mc-meta [--dry-run]` — kopieer MC's `s3`-meta naar onze `_vmn_offload` zodat alle attachments dezelfde shape hebben (puur cosmetisch — fallback blijft werken).

---

## 13. Database-inspectie — uitgevoerd

Uitgevoerd op 2026-05-15 vanuit de lokale dev-omgeving op zes lokale DB's (`vmn_cmweb`, `vmn_dearchitect`, `vmn_amt`, `vmn_provada`, `vmn_pwnet`, `vmn_recycling_magazine`) via `mysql -uroot -h127.0.0.1`.

### 13.1 Gevonden postmeta-keys

| Key | Schrijver | Aanwezig op |
|---|---|---|
| `_wp_attachment_metadata` met `['s3']` | Media Cloud 4.x | overal (dearchitect 157.179/157.371 = 99.9%) |
| `ilab_s3_info` | Media Cloud 2.x | overal (1 t/m 10.391 rijen per site) |
| `amazonS3_info` + `amazonS3_cache` + `as3cf_filesize_total` | Delicious Brains "WP Offload Media" | dearchitect, amt, pwnet |

Op dearchitect: **2.796 attachments hebben alleen `amazonS3_info` zonder MC-meta** — fallback voor as3cf is dus harde eis, niet optioneel.

### 13.2 Bevestigde MC-config (consistent over alle 6 sites)

```
provider                          : s3
region                            : eu-north-1
use-path-style-endpoint           : on        # buckets bevatten dots → SNI-veilig pad
privacy (default)                 : public-read
big-size-threshold                : 2560
big-size-original-privacy         : private   # !! niet alles is public
delete-from-server                : on
filter-content                    : on        # ook replace-all-image-urls=on
imgix-signing-key                 : (leeg)    # → public mode
imgix-domains                     : vmn-{site}.imgix.net
imgix-auto-format / auto-compress : on
imgix-default-quality             : 50        # !! niet meegenomen in oorspronkelijk PLAN
imgix-serve-private-images        : on
imgix-do-not-urlencode            : on
imgix-remove-extra-variables      : on
storage-prefix (setting)          : uploads/@{date:Y/m}  # maar zie §3.1 voor afwijking in actuele keys
```

Buckets: `vmn-cmweb.nl`, `vmn-dearchitect.nl`, `vmn-amt.nl`, `vmn-provada.nl`, `vmn-pwnet.nl`, `vmn-recycling-magazine.com` (laatste de uitzondering — `.com` i.p.v. `.nl`).

### 13.3 MC-eigen tabellen (aanwezig op cmweb)

```
wp_mcloud_task
wp_mcloud_task_data
wp_mcloud_task_schedule
wp_mcloud_task_token
```

Dit is MC's eigen queue (Action-Scheduler-achtig). Mag blijven staan na deactivatie (geen referentiële integriteit met core). **Uninstall.php raakt deze NIET** — alleen MC-uninstall mag dat doen.

### 13.4 Voorbeeld-shapes (geverifieerd)

MC 4.x in `_wp_attachment_metadata` (per size én top-level):
```php
'sizes' => [
    'thumbnail' => [
        'file' => 'photo-150x150.jpg',
        'width' => 150, 'height' => 150,
        'mime-type' => 'image/jpeg',
        's3' => [
            'url'       => 'https://s3.eu-north-1.amazonaws.com/vmn-dearchitect.nl/uploads/2018/08/photo-150x150.jpg',
            'provider'  => 's3',
            'bucket'    => 'vmn-dearchitect.nl',
            'key'       => 'uploads/2018/08/photo-150x150.jpg',
            'privacy'   => 'public-read',
            'v'         => '4.0.2',
            'mime-type' => 'image/jpeg',
            'region'    => 'eu-north-1',         // soms aanwezig, soms niet
        ],
    ],
],
```

ILab 2.x in `ilab_s3_info`:
```php
[
    'file' => '2019/10/file.docx',
    's3' => [ /* zelfde shape als hierboven, plus 'optimized', 'options', 'formats' */ ],
]
```

AS3CF (Delicious Brains) in `amazonS3_info`:
```php
[
    'bucket' => 'vmn-dearchitect.nl',
    'key'    => 'uploads/2017/01/photo.jpg',
    'region' => 'eu-north-1',
]
```
Geen per-size info — as3cf rekent erop dat sizes via dezelfde `key`-conventie reconstrueerbaar zijn (en dat is op deze sites consistent het geval).

### 13.5 Te verwerken in code

1. `Config.php` → `IMGIX_DEFAULT_Q=50`, `BIG_SIZE_THRESHOLD=2560`, `BIG_SIZE_PRIVACY='private'`, `FORCE_PATH_STYLE_URL=true` (zie §6.2).
2. `Attachment_Meta::read_legacy()` → drie leeslagen (MC, ilab_s3_info, amazonS3_info).
3. `Url_Rewriter` → key altijd letterlijk overnemen, geen `uploads/`-injectie.
4. `S3_Client` → path-style URL bouwen: `https://s3.{region}.amazonaws.com/{bucket}/{key}` (niet `{bucket}.s3.{region}…`) — anders crashen we op buckets met dots.
5. `Offload_Service` → bij big-size original (>2560px width/height) ACL `private`.

---

## 14. Tests & QA

### 14.1 Unit (PHPUnit + Brain Monkey)
- `Path_Builder` → correct `uploads/{year}/{month}/{file}` ongeacht datum/timezone.
- `Aws\S3_Client` → SigV4-canonical request testen tegen bekende AWS test vectors.
- `Attachment_Meta::read_legacy()` → MC-shape fixture in, onze shape uit.
- `Offload_Service` → bij gesimuleerde 4xx op één size: status = `partial`, lokaal niet verwijderd.

### 14.2 Integration
- Echte upload naar een test-bucket, met de echte WP-upload flow (via `wp_handle_upload` test harness).
- Bulk job met 100+ test attachments, controle voortgang en correcte status-eindstand.

### 14.3 Acceptance (op staging van één site)
- Switch van MC → ons, zonder DB-migratie: alle bestaande URLs blijven IMGix-served.
- Nieuw upload → eindigt op IMGix-URL in `the_content` (na publish).
- Failed upload (credentials kapot): lokaal staat alles, status `failed`, retry-knop werkt.

---

## 15. Migratie & rollout

### 15.1 Activatie-flow per site
1. Installeer onze plugin (deactiveer MC nog **niet**).
2. Configureer settings (zelfde bucket / region / IMGix-domain als MC nu gebruikt).
3. **Test connection** in de admin → moet groen worden.
4. `wp vmn-offload doctor` om te bevestigen.
5. Steekproef: open een bestaande post → URLs moeten nog identiek geserveerd worden (omdat onze fallback de MC-meta leest).
6. Deactiveer MC.
7. Upload één testimage → verifieer dat hij eindigt in `_vmn_offload['status'] === 'success'`.
8. (Optioneel) `wp vmn-offload normalize-mc-meta` om alle bestaande attachments óók in onze meta-shape te hebben.

### 15.2 Rollback
Als er iets stuk gaat: deactiveer onze plugin en activeer MC weer. Omdat we de MC-shape mirroren in `_wp_attachment_metadata['s3']`, blijft MC zonder data-verlies functioneren. Eventuele nieuw geüploade attachments tijdens onze run hebben óók een geldige MC-mirror.

### 15.3 Uninstall
`uninstall.php` verwijdert *alleen*: onze options (`vmn_offload_settings`), eigen transients, eigen DB-tabel (indien aanwezig). **Niet** de `_vmn_offload` postmeta — die laten we staan ter informatie. **Nooit** de `_wp_attachment_metadata` aanraken bij uninstall.

---

## 16. Beveiliging

- AWS credentials nooit loggen, ook niet in `errors[]`. Logger sanitizet `X-Amz-*`-headers.
- Settings page: secret veld is `type=password`, opslag via `sanitize_text_field` + optionele `openssl_encrypt` met `AUTH_KEY`-derived sleutel.
- Capabilities: instellingen → `manage_options`. Per-attachment retry → `upload_files`. Force-actions → `manage_options`.
- Nonces op alle admin POST/REST endpoints. REST endpoints altijd `permission_callback`.
- Bij `delete_attachment` controleren dat `force_delete === true` (geen "naar prullenbak"-delete triggert remote delete).
- IMGix-URL signing in v1 niet nodig (public-mode); wel hook `vmn_offload_imgix_signing_token` voorzien zodat het later toe te voegen is.

---

## 17. Hooks voor uitbreiding

Public API zodat sites custom-gedrag kunnen koppelen zonder fork.

| Filter / Action | Args | Doel |
|---|---|---|
| `vmn_offload_should_offload` | `bool $yes, int $attachment_id` | Skip bv. specifieke mime types of users. |
| `vmn_offload_build_key` | `string $key, int $id, string $size` | Custom remote key. |
| `vmn_offload_s3_put_args` | `array $args, int $id` | Headers/ACL aanpassen. |
| `vmn_offload_imgix_url` | `string $url, array $meta, string $size` | Custom IMGix-params. |
| `vmn_offload_content_replace_url` | `string $new, string $old` | Override per-URL replacement. |
| `vmn_offload_after_success` | (action) `int $id, array $meta` | Eigen logging/notifications. |
| `vmn_offload_after_failure` | (action) `int $id, WP_Error $err` | Alerting. |

---

## 18. Gefaseerde oplevering & milestones

Twee niveaus: **fasen** (rollout-mijlpalen, wanneer iets bij wie live gaat) en **milestones met task-codes** (per-feature werkpakketten, geleend uit `PROJECT.md` van collega).

### 18.1 Fasen (rollout)

| Fase | Inhoud | Doel-mijlpaal |
|---|---|---|
| **0** | DB-inspectie (§13) ✓ | Definitieve `Config`-waarden vastgelegd |
| **1 (MVP)** | M1 + M2 + M5 — plugin scaffold, S3-client, upload-hook, Url_Rewriter (incl. legacy-fallbacks) | Eén testsite: nieuwe uploads gaan via IMGix, bestaande blijven werken |
| **2** | M6 + M3 (UI deel) — admin settings page, per-attachment UI, bulk offloader, delete-hooks | Productieklare basis voor 1 pilotsite |
| **3** | M3 (worker) + M4 — concurrent upload-fix, DB S&R-tool, WP-CLI, srcset-rewriting | Rollout naar 5 sites |
| **4** | Stabilisatie + rollout naar alle 30 sites, MC deactiveren | Volledige migratie afgerond |
| **5 (v1.1)** | Action Scheduler-pad voor bulk over álle sites, optionele IMGix-signing, async upload-queue | Schalen voor de grootste sites |

### 18.2 Milestones (per-feature werkpakketten)

Gebaseerd op de structuur uit `PROJECT.md`, met onze omgezette scope (nieuwe codebase, geen fork). Task-codes blijven bruikbaar voor ticketing/PR-titels.

#### M1 — Scaffold & opschonen

| Code | Omschrijving | Sectie | Status |
|---|---|---|---|
| SCAF-01 | Plugin-scaffold `vmn-media-offload` (header, autoloader, plugin-class, activation-hook) | §4 | todo |
| SCAF-02 | Geen Freemius — geen analytics, geen "premium upsell", geen licentiecheck | §1 | todo |
| SCAF-03 | Minimale composer-deps (alleen PSR-4 autoloader; geen aws-sdk-php) | §4 | todo |
| SCAF-04 | Geen video/vision/optimizer/wizard modules — die zaten in MC, wij bouwen ze niet | §1 | todo |

#### M2 — S3 storage (kern)

| Code | Omschrijving | Sectie | Status |
|---|---|---|---|
| STOR-01 | Auto-offload bij WP media upload (`wp_generate_attachment_metadata`-hook) | §7 | todo |
| STOR-02 | S3 credentials in admin (DB-fallback, constant-override) — `S3_Client` met SigV4 PUT/DELETE/HEAD | §6, §16 | todo |
| STOR-03 | Attachment URL's herschrijven naar IMGix (en S3 als IMGix off) — `Url_Rewriter` | §8 | todo |
| STOR-04 | Lokale kopie bewaren/verwijderen instelbaar; **nooit** verwijderen bij `partial`/`failed` | §7.3, §9 | todo |
| STOR-05 | Legacy-fallback laag 2/3/4 (`_wp_attachment_metadata['s3']` + `ilab_s3_info` + `amazonS3_info`) | §3 | todo |
| STOR-06 | Path-style URL-builder voor buckets met dots in de naam | §6.2, §13.5 | todo |

#### M3 — Offload worker & monitoring

| Code | Omschrijving | Sectie | Status |
|---|---|---|---|
| OFFL-01 | **Concurrent multi-size uploads** via `curl_multi_*` (de "fix" t.o.v. MC's strikt seriële uploads) | §7.4 | todo |
| OFFL-02 | Realtime voortgang op de bulk-pagina (per-batch progress + ETA) | §11.1 | todo |
| OFFL-03 | Fouten per bestand zichtbaar (laatste-50 lijst op bulk-pagina + volledige history in `_vmn_offload['errors']`) | §10, §11.1 | todo |
| OFFL-04 | Big-size original (>2560px) krijgt `private` ACL; via IMGix met presigned source-URL | §6.2, §8.3 | todo |
| OFFL-05 | Retry-logica per size (`Config::RETRY_MAX=3` met exponential backoff op 5xx) | §6.2 | todo |

#### M4 — IMGix integratie

| Code | Omschrijving | Sectie | Status |
|---|---|---|---|
| IMGX-01 | Alle image-URLs via IMGix met `auto=compress,format` en `q=75` | §6.2, §8 | todo |
| IMGX-02 | IMGix domain instelbaar in admin (één per site, `*.imgix.net` of custom) | §6.1 | todo |
| IMGX-03 | `wp_calculate_image_srcset` filter: alle srcset-entries krijgen IMGix-URL met `w`-param | §8.1 | todo |
| IMGX-04 | IMGix uitschakelbaar via `vmn_offload_serve_via_imgix` (fallback naar S3-public-URL) | §6.1 | todo |
| IMGX-05 | Signing-hook `vmn_offload_imgix_signing_token` voorbereid (geen v1-implementatie) | §16 | todo |

#### M5 — Compatibiliteit & techniek

| Code | Omschrijving | Sectie | Status |
|---|---|---|---|
| COMP-01 | PHP **8.3+** syntaxis door hele codebase (readonly props, typed enums, `never` return-types waar passend) | §1 | todo |
| COMP-02 | WordPress **6.9.x+** — geen polyfills, geen back-compat shims onder die versie | §1 | todo |
| COMP-03 | Geen deprecated WP-hooks/functies; geen `wp_make_content_images_responsive` etc. die in 6.9 weg zijn | §1 | todo |
| COMP-04 | PHPStan level 8 op `src/` (geen mixed-types laten lopen) | §14 | todo |
| COMP-05 | PHPUnit 11 + Brain Monkey; minimaal 60% coverage op `Storage/`, `Aws/`, `Url/` | §14.1 | todo |

#### M6 — Admin UI

| Code | Omschrijving | Sectie | Status |
|---|---|---|---|
| ADMIN-01 | Settings-pagina "Settings → Media Offload" met S3- en IMGix-tabs + **Test connection** | §6 | todo |
| ADMIN-02 | Per-attachment meta-box op `attachment.php` (status, retry, force re-offload, error log) | §10.2 | todo |
| ADMIN-03 | Bulk-pagina "Media → Bulk Offload" met live voortgang en error-lijst | §11.1 | todo |
| ADMIN-04 | Mediagrid kolom met badge (`Local`/`Offloaded`/`Partial`/`Failed`) | §10.1 | todo |

### 18.3 Volgorde van aanpak

1. **M1** — scaffold/opschonen (geen feature-werk, alleen plugin-skelet).
2. **M2** + **M6 (ADMIN-01)** parallel — kern + minimale settings-UI.
3. **M3** — offload-worker (bouwt op M2's `S3_Client`).
4. **M4** — IMGix (bouwt op M2 + M3).
5. **M6 (overige)** — UI-componenten naast worker.
6. **M5** — compat/test-pass loopt mee vanaf M2; finaliseren na M4.

---

## 19. Open punten — status

Status van de openstaande vragen na DB-inspectie op 2026-05-15 en afstemming met Floris:

1. ~~**DB-export** van postmeta van één representatieve productiesite.~~ **Opgelost** — 6 lokale DB's geïnspecteerd (zie §13). Bevindingen verwerkt in §3, §6.2, §13.
2. ~~**Bevestiging IMGix-mode**: public, of signed?~~ **Public mode** — `mcloud-imgix-signing-key` is leeg op alle onderzochte sites. Signing in v1 niet nodig; wel hook voorzien (§16, IMGX-05).
3. ~~**Privacy / ACL**: zijn alle huidige offloaded files `public-read`?~~ Default `public-read`, **maar** big-size originals (>2560px width/height) krijgen `private`. Bv. cmweb 576 private files. `mcloud-imgix-serve-private-images: on` betekent dat private originals tóch via IMGix worden geserveerd (vermoedelijk met S3-presign door IMGix zelf — verifiëren bij implementatie van laag 2 in §8). → OFFL-04 in §18.
4. ~~**Bucket-policy / versioning**.~~ **Versioning staat NIET aan** (bevestigd door Floris). Conclusie: `delete_remote_on_delete` blijft default OFF en is destructief — gebruiker moet bewust opt-in. Geen "soft delete"-vangnet van AWS. Documenteren in admin-warning bij die setting (§6.1, ADMIN-01).
5. ~~**Path-prefix variatie**.~~ **Gemengd** — zie §3.1. Hard requirement: stored `key` altijd letterlijk gebruiken.
6. ~~**Naming**.~~ **Bevestigd**: package `vmn-media-offload`, namespace `VMN\MediaOffload`.
7. ~~**AS3CF (Delicious Brains) als derde legacy laag.**~~ Verwerkt in §3 laag 4 (STOR-05).
8. ~~**IMGix default quality.**~~ **Bevestigd**: van 50 → **75** voor nieuwe uploads. `Config::IMGIX_DEFAULT_Q = 75`. Bestaande URL's blijven q=50 omdat IMGix die niet bewaart in de key — als de oude `the_content` `q=50` hardgecodeerd heeft, geldt dat alleen daar.
9. ~~**MC-eigen tabellen (`wp_mcloud_task*`).**~~ Worden in onze uninstall met rust gelaten. Geen actie nodig.

Alle blokkerende open punten zijn opgelost. **Groen licht voor Fase 1 / Milestone 1 (SCAF-01..04).**

### Nog te bevestigen tijdens bouw (niet blokkerend)

- **Private files via IMGix** (§8): klopt het dat IMGix met "serve private images on" zelf S3-presign doet, of moeten wij een presigned source-URL voeren? Pas te testen zodra we een dev-bucket hebben.
- **Custom srcset-sizes** (`vmn_16-9_l`, `vmn_square_big` etc.): aantal en naamgeving uit `wp_get_attachment_metadata` halen — geen hardcoded lijst.
- **`-scaled` originals**: WP-core schrijft `-scaled.jpg` voor images > big-image-size-threshold; verifiëren dat onze offload deze als de "full"-key behandelt (niet als een aparte size).
