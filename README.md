</center><img width="150" height="150" alt="riket" src="https://github.com/user-attachments/assets/ce619ca3-5c87-4997-8571-f87285128c20" /><center>


<h1>Riket Launcher</h1>


Desktop-launcher for Windows (Electron) for Riket Rollespill. Spillerne logger inn med Discord, ser om de har tilgang, og kommer inn på serveren med ett klikk. Launcheren henter alt innhold og utseende fra **riketrp.no**, så det du endrer i adminpanelet, vises i launcheren også.

## Hva launcheren gjør

| Fane / del | Innhold |
|---|---|
| **Hjem** | Toppbilde og tekst fra forsiden, «Spill nå», din tilgang (whitelistet / søknad under behandling / utestengt) og spillere online |
| **Min side** | Karakterene dine med jobb, kontanter, bank, lisenser, jobber/gjenger og kjøretøy – samme data som spillerportalen |
| **Regler** | Reglene fra /regler. Man må godta dem før man spiller første gang, og på nytt når de endres |
| **Nyheter** | De siste nyhetene fra nettsiden |
| **Snarveier** | Menyen på nettsiden, åpnes i nettleseren |
| **Innstillinger** (menyen under navnet) | Hva som skjer etter «Spill nå» (ingenting / minimer / lukk), start med Windows, sesongeffekter av/på, tøm FiveM-cache, logg ut, avinstaller |

I tillegg:
- **Utseende fra adminpanelet:** farger, sesongfarger, knappeform, kortstil, fonter, fanestil, logo og toppbilde. Q / E bytter fane, som på nettsiden.
- **Discord-knappen** viser hvor mange som er pålogget i Discord-serveren.
- **Sesongeffekter** (snø, konfetti, Halloween osv.) og sesongbanner, som på nettsiden.
- **Tøm FiveM-cache** sletter bare `cache`, `server-cache` og `server-cache-priv`. Spillfilene (`game-storage`) og innstillingene blir liggende, og den nekter hvis FiveM er åpent.

## Slik henger det sammen

### Innlogging

```
Launcher ──► Discord (i nettleseren)
                │  redirect
                ▼
     riketrp.no/launcher/auth/callback  ── bytter koden hos Discord (client secret ligger bare her)
                │                           og legger innloggingen klar i 5 minutter
                ▼
     riketrp.no/launcher/innlogget  («Du er logget inn»)

Launcher ──► POST /api/launcher/auth/poll  (hvert 2. sekund) ──► får en launcher-token (gyldig i 30 dager)
```

Launcheren lager en hemmelighet og sender bare sjekksummen (SHA-256) til Discord som `state`. Innloggingen kan bare hentes med selve hemmeligheten, så den som ser adressen i nettleseren, kan ikke ta den. Tokenen lagres kryptert på PC-en (Windows DPAPI via Electrons `safeStorage`).

Innloggingen bruker samme brukerregister som nettsiden: den vises under Brukere i adminpanelet, og sperrede brukere slipper ikke inn.

### Tilgang og «Spill nå»

Tilgangen sjekkes på nettsiden hver gang, med samme regler som spillerportalen:
- **Whitelistet** = godkjent søknad *eller* karakterer på serveren (eller admin). Er søknader slått av, slipper alle inn.
- **Stoppet** av ban i FiveM-databasen eller vedlikeholdsmodus.

FiveM godtar bare å bli startet fra nettleseren eller Windows, ikke av andre programmer. «Spill nå» åpner derfor **Koble til**-siden (`/status?spill=1`) i nettleseren. Den starter FiveM med en gang og blir stående åpen. Første gang spør nettleseren om den skal åpne FiveM, og krysser man av for «Tillat alltid», går det rett inn etter det.

> Launcheren er en ekstra vei inn. Serveren bør fortsatt sjekke whitelist selv (f.eks. i `playerConnecting`), ellers kan folk koble til direkte i FiveM.

## Filer

### Nettsiden (deployes med resten av riketrp.no)

| Fil | Hva |
|---|---|
| `server/utils/launcher.ts` | Token, tilgangssjekk, Discord-innlogging og henting av innlogging |
| `server/api/launcher/config.get.ts` | Tema, logo, toppbilde (fra Hero-blokken på forsiden), meny og Discord-klient-ID |
| `server/api/launcher/me.get.ts` | Brukeren og tilgangen |
| `server/api/launcher/launch.post.ts` | Sjekker tilgang før «Spill nå» og gir lenken til Koble til |
| `server/api/launcher/portal.get.ts` | Data til «Min side» |
| `server/api/launcher/news.get.ts` | Siste nyheter |
| `server/api/launcher/auth/poll.post.ts` | Launcheren henter innloggingen her |
| `server/api/launcher/auth/discord.post.ts` | Eldre innlogging via 127.0.0.1 (for launchere før 1.4) |
| `server/routes/launcher/auth/callback.get.ts` | Discord sender spilleren hit etter innlogging |
| `server/routes/launcher/connect.get.ts` | Sender videre til `/status?spill=1` (for eldre launchere) |
| `app/pages/launcher/innlogget.vue` | «Du er logget inn»-siden |
| `app/components/system/Status.vue` | Koble til-siden starter FiveM når `?spill=1` er med |

Launcheren bruker også de offentlige endepunktene som fantes fra før: `/api/status/server` (spillere online), `/api/content/rules` (regler), `/api/discord/server` (Discord) og `/api/settings` (sesong).

### Launcheren (`launcher/`)

| Fil | Hva |
|---|---|
| `src/config.js` | Nettsidens adresse og Discord-serverens ID – det eneste som må stilles inn |
| `src/main.js` | Vinduet, innlogging, kall til nettsiden, cache, innstillinger og avinstallering |
| `src/preload.js` | Det siden i vinduet får lov til å kalle (ingen direkte tilgang til Node) |
| `src/renderer/` | Selve grensesnittet (`index.html`, `app.js`, `launcher.css`) |
| `src/renderer/site.css` | Kopi av nettsidens `app/assets/css/main.css` |
| `src/renderer/icons.js` | Samme ikoner som nettsiden (`app/components/Icon.vue`) |
| `src/renderer/vendor/` | Sesonger og Markdown bygget fra nettsidens kode (`npm run vendor`) |
| `build/icon.ico` | Ikonet til programmet og installasjonsfilen |

## Godt å vite
- Installasjonsfilen er ikke kodesignert, så Windows viser «Windows beskyttet PCen din» første gang. Spillerne trykker «Mer info» → «Kjør likevel». Et kodesigneringssertifikat fjerner advarselen.
- Avinstallering (fra Innstillinger eller Windows) sletter også innloggingen og innstillingene til launcheren. FiveM og spillet blir ikke rørt.
- Innstillingene og innloggingen ligger i `%APPDATA%\Riket Rollespill`.
