# vmn-media-cloud — Claude context

## Wat is dit

Een gestripte fork van de Media Cloud WordPress plugin, gebouwd voor VMN.
Doel: alleen S3-opslag + Imgix CDN, zonder Freemius/video/vision/optimizer/wizard.

## Waar te beginnen

Zie `.planning/PROJECT.md` voor het volledige overzicht van milestones en taken.
Start met `/gsd-progress` om de huidige status te zien en de volgende stap te bepalen.

## Belangrijke beslissingen

- Freemius volledig verwijderen — ook uit `ilab-media-tools.php` en `classes/Utilities/LicensingManager.php`
- PHP minimumversie ophogen naar 8.1 (nu nog 7.4 in composer.json en plugin header)
- Plugin slug wordt `vmn-media-cloud`, text domain `vmn-media-cloud`
- `external/Freemius/` map volledig verwijderen
- `classes/Tools/Video/` en `classes/Tools/Vision/` volledig verwijderen
- Parallelle S3 uploads zijn een expliciete vereiste (OFFL-01)

## Codestructuur

```
classes/          PSR-4: MediaCloud\Plugin\
  Tools/          Toolmodules (Storage, Imgix, Tasks, ...)
  Utilities/      Helpers, logging, licensing (Freemius-refs hier ook)
config/           .config.php bestanden per tool
lib/              Ge-namespaced vendor libs (mcloud-aws, mcloud-imgix, ...)
external/         Freemius SDK — te verwijderen
views/            PHP/Twig templates voor admin UI
```
