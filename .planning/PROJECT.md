# vmn-media-cloud

## Doel

Een gestripte, onderhoudbare fork van de Media Cloud plugin, specifiek voor VMN WordPress.
Alle overbodige modules (Freemius, video, vision, optimizer, wizard) worden verwijderd.
Focus: S3-opslag + Imgix CDN, met een schone PHP 8.1+ codebase.

## Startpunt

De repo bevat de originele Media Cloud 4.6.4 plugin als basis. Nog niets is gestript of herschreven.

## Milestones

### Milestone 1 — Scaffold & opschonen

| Taak    | Omschrijving                              | Status |
|---------|-------------------------------------------|--------|
| SCAF-01 | Plugin scaffold vmn-media-cloud           | todo   |
| SCAF-02 | Geen Freemius code, nergens               | todo   |
| SCAF-03 | Alleen benodigde composer deps            | todo   |
| SCAF-04 | Video/vision/optimizer/wizard weg         | todo   |

### Milestone 2 — S3 Storage

| Taak    | Omschrijving                              | Status |
|---------|-------------------------------------------|--------|
| STOR-01 | Auto-offload bij WP media upload          | todo   |
| STOR-02 | S3 credentials instelbaar in admin        | todo   |
| STOR-03 | Attachment URLs herschreven naar S3/Imgix | todo   |
| STOR-04 | Lokale kopie bewaren/verwijderen instelbaar | todo |

### Milestone 3 — Offload worker

| Taak    | Omschrijving                              | Status |
|---------|-------------------------------------------|--------|
| OFFL-01 | Parallelle concurrent S3 uploads (de fix) | todo   |
| OFFL-02 | Realtime voortgang in Task Manager        | todo   |
| OFFL-03 | Fouten per bestand zichtbaar in Task Manager | todo |

### Milestone 4 — Imgix integratie

| Taak    | Omschrijving                              | Status |
|---------|-------------------------------------------|--------|
| IMGX-01 | Alle media via Imgix SDK URL              | todo   |
| IMGX-02 | Imgix domain + token instelbaar           | todo   |
| IMGX-03 | srcset gebruikt Imgix URLs                | todo   |
| IMGX-04 | Imgix uitschakelbaar (fallback naar S3)   | todo   |

### Milestone 5 — Compatibiliteit

| Taak    | Omschrijving                              | Status |
|---------|-------------------------------------------|--------|
| COMP-01 | PHP 8.1+ syntaxis door hele codebase      | todo   |
| COMP-02 | Werkt op WP 6.x zonder fouten             | todo   |
| COMP-03 | Geen deprecated WP7 hooks/functies        | todo   |

### Milestone 6 — Admin UI

| Taak    | Omschrijving                              | Status |
|---------|-------------------------------------------|--------|
| ADMIN-01 | S3 instellingenpagina                    | todo   |
| ADMIN-02 | Imgix instellingenpagina                 | todo   |
| ADMIN-03 | Task Manager pagina met live voortgang   | todo   |

## Volgorde van aanpak

1. Begin met Milestone 1 (scaffold/opschonen) — dit is de basis voor alles
2. Dan Milestone 2 (S3 storage) + Milestone 6 (admin UI) parallel
3. Dan Milestone 3 (offload worker) — bouwt op M2
4. Dan Milestone 4 (Imgix) — bouwt op M2 + M3
5. Tot slot Milestone 5 (compatibiliteit) — door de hele codebase

## Technische context

- WordPress plugin, PSR-4 autoloading (`MediaCloud\Plugin\` → `classes/`)
- Composer-gebaseerd, `lib/` bevat ge-namespaced vendor deps
- Huidige PHP-eis: 7.4 — verhogen naar 8.1+
- Freemius SDK staat in `external/Freemius/` — volledig verwijderen
- Video tools: `classes/Tools/Video/` — volledig verwijderen
- Vision tools: `classes/Tools/Vision/` — volledig verwijderen
