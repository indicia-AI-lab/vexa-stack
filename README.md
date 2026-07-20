# vexa-stack

Capture-only deploy van [Vexa](https://github.com/Vexa-ai/vexa)
(Apache-2.0) voor **Vector**: een bot die als anonieme gast Teams-meetings
in gaat en de ruwe audio als artifact in MinIO bewaart. Transcriptie en
analyse gebeuren **niet** hier maar in Vector's eigen pipeline — zie
**ADR-0004** in de vector-repo (`docs/adr/0004-teams-bot-vexa.md`) voor
het besluit en de afwegingen.

Draait op de ecom/AI Dokploy-VPS (TransIP) als Compose-project in de
Dokploy-organisatie **Indicia AI** — de gedocumenteerde uitzondering op
de één-Application-per-service-conventie: dit is andermans stack die we
als geheel pinnen en bijwerken.

## Wat draait er

| Service | Poort (127.0.0.1) | Rol |
|-|-|-|
| gateway | 18056 | publieke API-voordeur (`X-API-Key`) — Vector praat hiertegen |
| admin-api | 18057 | users + API-keys minten (`X-Admin-API-Key`) |
| meeting-api | 18080 | meeting-lifecycle, schrijft opnames naar MinIO |
| runtime | 18090 | spawnt bot-containers via de docker-socket |
| postgres | 5458 | staat van meetings/users |
| minio | 9000 / 9001 (console) | opslag van de audio-artifacts (bucket `vexa`) |
| redis | — | intern (command bus) |

De **bot** is geen compose-service: runtime spawnt per meeting een
`vexaai/vexa-bot`-container (~3,6 GB image, ±1–2 GB RAM per draaiende
bot) op het vaste docker-netwerk `vexa-stack`. Weggelaten t.o.v.
upstream: transcriptie, agent-api/agent-worker, mcp, terminal, dashboard.

Alles bindt op 127.0.0.1 — er is bewust géén publieke ingang; Vector
draait op dezelfde host en bereikt de gateway via localhost.

## Deploy (Dokploy Compose + CI/CD)

De stack draait als **Compose**-service in de Dokploy-organisatie
**Indicia AI**, met de standaard tag-pipeline (kennisbank
`dokploy-deploy`) in aangepaste vorm — er wordt niets gebouwd:

```
git tag X.Y.Z (op main) → checks (compose-validatie, tag-guard)
  → mirror (gepinde upstream-images → ghcr.io/indicia-ai-lab/vexa-stack/*)
  → deploy (ecom-runner → Dokploy-webhook → compose-redeploy)
```

- **Versie bumpen**: pas de `${IMAGE_TAG:-…}`-defaults in
  `docker-compose.yml` aan (alle tegelijk), commit op main, tag `X.Y.Z`.
  De mirror-job kopieert manifest-voor-manifest (`docker buildx
  imagetools create`), dus geen rebuild en identieke digests.
- **GHCR-packages moeten public staan** (eenmalig na de eerste
  mirror-run, per package: Package settings → Danger zone → public).
  Niet alleen voor Dokploy: het **bot-image wordt door de runtime via
  de host-docker-socket gepulld**, buiten Dokploy's registry-auth om —
  een private package breekt dus pas bij de eerste bot-spawn, niet bij
  de deploy.
- **Panel-env** (Dokploy → service → Environment): de waarden uit
  `.env.example` — secrets, poorten, `DOCKER_GID` (986 op de ecom-VPS).
  Géén `IMAGE_TAG`/`BROWSER_IMAGE` in het panel zetten; de versie is
  in de compose gepind zodat CI en runtime nooit uiteenlopen.
- Zet Dokploy's **auto-deploy on push uit** — deploys zijn tag-gedreven
  via de webhook (repo-secret `DOKPLOY_WEBHOOK_URL`, uit het
  webhook-veld van de Compose-service).

Na de eerste succesvolle deploy, op de VPS:

1. API-key voor Vector minten (idempotent, max 3 gelijktijdige bots
   conform ADR-0004):

   ```sh
   ADMIN_TOKEN=<panel-env> EMAIL=vector@indicia.nl MAX_BOTS=3 bin/provision-token
   ```

   De output (`vxa_…`) is de `X-API-Key` voor Vector's env
   (`VEXA_API_KEY`). Token kwijt? Opnieuw draaien — tokens zijn additief.
2. Rooktest zonder echte meeting: `curl -s http://127.0.0.1:18056/health`
   en `curl -s -H "X-API-Key: vxa_…" http://127.0.0.1:18056/bots`
   (lege lijst = goed).

Lokaal draaien kan met `cp .env.example .env` (secrets invullen, evt.
`VEXA_REGISTRY=docker.io/vexaai` om rechtstreeks upstream te pullen) en
`docker compose up -d`.

## Gebruik (de API die Vector aanroept)

```sh
# Bot een meeting in sturen (link volstaat; platform/id/passcode worden afgeleid)
curl -X POST http://127.0.0.1:18056/bots \
  -H "X-API-Key: vxa_…" -H "Content-Type: application/json" \
  -d '{"meeting_url": "https://teams.microsoft.com/meet/<id>?p=<passcode>", "bot_name": "Indicia Notulist"}'

# Bot stoppen
curl -X DELETE http://127.0.0.1:18056/bots/teams/<native_meeting_id> -H "X-API-Key: vxa_…"

# Meetings + opnames
curl http://127.0.0.1:18056/bots -H "X-API-Key: vxa_…"
curl http://127.0.0.1:18056/recordings -H "X-API-Key: vxa_…"
# Audio ophalen: /recordings/{id}/media/{mediafile_id}/raw (bytes) of /download (signed URL)
```

Boven de per-user botlimiet antwoordt de API `429 MaxBotsExceeded`.

## Teams-vereisten

- De tenant moet **anonieme/gast-join** toestaan; de host laat de bot
  toe vanuit de lobby. (Blokkeert de tenant dat, dan bestaat upstream
  een authenticated-bot-mode — `BOT_AUTHENTICATED` — die een ingelogde
  browsersessie gebruikt; nu bewust uit.)
- Bots joinen de **web**client headless en lezen de UI (Playwright).
  Sprekerlabels van Vexa zijn DOM-scraping — Vector negeert ze;
  diarisatie komt uit de audio (ADR-0004, besluit 4).

## Agenda-sync (F-8.4-optie, nog niet in gebruik)

Vexa v0.12 heeft **native ICS-agenda-sync + auto-join**: meeting-api
sweept gekoppelde ICS-feeds (via admin-api `/internal/calendar-configs`)
en stuurt automatisch bots naar gevonden meetings. Dat kan de basis van
F-8.4 zijn (Outlook publiceert per gebruiker een ICS-URL) zónder eigen
Graph-OAuth-koppeling. Nog te onderzoeken vóór we hierop leunen: hoe
configs per user worden aangemaakt, sweep-frequentie, en of de
join-URL-dedupe uit ADR-0004 hier al inzit. Tot die tijd stuurt Vector
zelf bots aan via `POST /bots`.

## Upgraden

1. Upstream release bekijken (changelog + issues rond Teams-join).
2. `${IMAGE_TAG:-…}`-defaults in `docker-compose.yml` naar de nieuwe
   tag, commit op main, `git tag X.Y.Z && git push origin X.Y.Z`.
3. Rooktest (zie Deploy) + één echte proefmeeting.
4. Rollback = de vorige defaults terugzetten en opnieuw taggen.

**Pin altijd een `vX.Y.Z`-tag, nooit de rolling `v012`-tag** — die
schuift stilletjes mee met upstream. Bekend breekpatroon: Microsoft
wijzigt de Teams-web-UI en de bot komt niet meer binnen (blijft in
`joining`/failed). Dat is het moment om upstream te bumpen, niet om
zelf te patchen.

## Beheer & risico's

- **Docker-socket**: runtime mount `/var/run/docker.sock` en is daarmee
  root-equivalent op de host. Geen extra services aan deze stack hangen;
  API's blijven localhost-only.
- **Bot-containers vallen buiten Dokploy-zicht**: runtime spawnt ze los
  van compose. `docker ps --filter ancestor=vexaai/vexa-bot:<tag>`
  toont ze; een hangende bot mag gewoon gestopt worden.
- **Schijf**: opnames blijven in MinIO staan (bucket `vexa`, console op
  127.0.0.1:9001). Vector kopieert het artifact naar zijn eigen
  `RECORDINGS_DIR`; periodiek oude artifacts opruimen hoort nog
  ingericht te worden (retention is een open punt).
- **Capaciteit**: max 3 gelijktijdige bots (per-user limiet, gezet bij
  het minten). VPS: 8 vCPU / 16 GB, gedeeld met de ecom-klantapps.

## Bestanden

- `docker-compose.yml` — de stack (afgeleid van upstream v0.12.15; de
  afwijkingen staan in de kopcommentaar); hier is óók de versie gepind
- `.env.example` — invulsjabloon voor panel-env / lokaal; `.env` staat
  in `.gitignore`
- `.github/workflows/ci.yml` — compose-validatie op elke push/PR
- `.github/workflows/deploy.yml` — tag-pipeline: checks → GHCR-mirror →
  Dokploy-webhook
- `bin/provision-token` — gevendorde upstream-helper (v0.12.15) om
  idempotent een user + API-key te minten
