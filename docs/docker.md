# agentlink in een container

Zo zet je een agent in een Docker-container en koppel je hem aan een gesprek in de app. Dit is voor een gebruiker
van een app die agentlink gebruikt, en voor een agent die het leest.

De bestanden staan in [`docker/`](../docker): de [Dockerfile](../docker/Dockerfile), het startscript
[`agentlink-start`](../docker/agentlink-start) en een voorbeeld voor [compose](../docker/compose.yaml).

## Wanneer een container

- **Een dienstagent:** een agent die de leden van een rol bedient, met één taak. Wat hij kan, moet vastliggen voordat
  iemand met hem praat. In een container staat dat in het image: alleen agentlink, één CLI en het gereedschap van
  zijn taak.
- **Een agent die altijd aan moet staan,** op een server zonder bureaublad.
- **Een persoonlijke agent die je afschermt van de rest van je computer:** hij ziet alleen wat in de container staat
  en de map die je hem geeft.

Werkt je agent in je eigen projectmap, met je eigen gereedschap, dan heb je geen container nodig: gebruik dan het
installatiecommando uit de [README](../README.md).

## Persoonlijke agent of dienstagent

| | Persoonlijke agent | Dienstagent |
|---|---|---|
| Wie hij bedient | zijn eigenaar, en wie hem in een gesprek leent | de leden van een rol |
| De uitnodiging | een gewone plek | een plek met een bedoeling (`agentlink brief <link>` toont die) |
| Koppelen | `agentlink join <link>` | `agentlink join <link> --service-agent --posture "<hoe je draait>"` |
| In de container | `AGENTLINK_JOIN_ARGS` leeg | `AGENTLINK_JOIN_ARGS="--service-agent --tools off"` en `AGENTLINK_POSTURE` |

`--tools` is het plafond van wat de agent mag (zie de README). Standaard is dat `full`: alles, zonder vragen. In een
container is de container de grens, dus dat kan voor een persoonlijke agent. Een dienstagent koppel je met
`--tools off`, en wat hij wél mag, zet je in zijn image (hieronder, "Een eigen image").

## Het image bouwen

Er is (nog) geen kant-en-klaar image in een registry. Je bouwt het zelf uit de Dockerfile van een release:

```bash
docker build -t agentlink https://github.com/appfabriek/agentlink.git#v<versie>:docker
```

Met `#main:docker` bouw je de nieuwste. Het image bevat:

- Debian slim, met `tini` als init (PID 1);
- de gebruiker `agent` (uid 10001), geen root;
- agentlink uit de release van die versie, gecontroleerd tegen `SHA256SUMS`;
- Claude Code, vastgepind op een versie en gecontroleerd met de SHA-256 uit het manifest van die versie.

Meer niet: geen `git`, geen `gh`, geen compiler. Heeft je agent gereedschap nodig, bouw dan een eigen image erop
(hieronder, "Een eigen image"). Het werkt op `linux/amd64` en `linux/arm64`. Voor beide tegelijk,
naar je eigen registry:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t <registry>/agentlink:<versie> --push \
  https://github.com/appfabriek/agentlink.git#v<versie>:docker
```

Andere vormen, met dezelfde Dockerfile:

| | Hoe |
|---|---|
| Zonder CLI (agentlink en het startscript), om op te bouwen | `--target base` |
| Een andere versie van agentlink | `--build-arg AGENTLINK_VERSION=<versie>` |
| Een eigen build van agentlink | `--build-arg AGENTLINK_SOURCE=local --build-context agentlink-local=<map met de binary agentlink>` |

## Starten

Je hebt drie dingen nodig: het image, een uitnodigingslink uit de app (**Koppel een agent**) en een eigen
Anthropic-sleutel in een bestand.

### Met `docker run`

```bash
docker volume create agentlink-home
docker run -d --name agentlink --restart unless-stopped \
  --read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop ALL --security-opt no-new-privileges:true \
  -v agentlink-home:/home/agent \
  -v /pad/naar/sleutel:/run/secrets/anthropic-api-key:ro \
  -e INVITE_LINK='<uitnodigingslink>' \
  agentlink
docker logs -f agentlink
```

De eerste start koppelt de agent. Hij meldt zich daarna in het gesprek, en `agentlink run` draait als hoofdproces.

Een persoonlijke agent die in een projectmap werkt, krijgt die map als werkmap: `-v /pad/naar/project:/home/agent/work`.
De map moet schrijfbaar zijn voor uid 10001.

De link blijft zo in de config van de container staan. Hij werkt maar één keer, dus dat kan geen kwaad. Wil je hem
er niet in, koppel dan eerst apart en start daarna zonder link:

```bash
docker run --rm -v agentlink-home:/home/agent -v /pad/naar/sleutel:/run/secrets/anthropic-api-key:ro \
  -e INVITE_LINK='<uitnodigingslink>' agentlink true
```

Een dienstagent leest eerst de briefing van de plek, zonder te koppelen:

```bash
docker run --rm --entrypoint agentlink agentlink brief '<uitnodigingslink>'
```

Daarna koppelt hij met zijn houding erbij, met deze twee regels in het `docker run`-commando hierboven:

```bash
  -e AGENTLINK_JOIN_ARGS='--service-agent --tools off' \
  -e AGENTLINK_POSTURE='Dienstagent in een Linux-container zonder root; alleen <gereedschap>.' \
```

### Met compose

[`docker/compose.yaml`](../docker/compose.yaml) doet hetzelfde, met de afscherming erbij. Zet in de map `docker/`
een bestand `.env` met het pad naar de sleutel (alleen het pad, niet de sleutel). Elk `docker compose`-commando
leest het daar:

```bash
echo 'ANTHROPIC_API_KEY_FILE=/pad/naar/sleutel' > .env
INVITE_LINK='<uitnodigingslink>' docker compose up -d --build
```

Start later zonder `INVITE_LINK`: `docker compose up -d`. `AGENTLINK_JOIN_ARGS` en `AGENTLINK_POSTURE` geef je op
dezelfde manier mee; zet je ze niet, dan houdt een image dat op dit image bouwt zijn eigen.

### Wat het startscript doet

Bij elke start, in deze volgorde:

1. Is `ANTHROPIC_API_KEY_FILE` gezet, dan moet dat bestand leesbaar zijn en meer dan witruimte bevatten. Anders stopt
   het met exitcode 2.
2. Staat er geen `agent.json` in `AGENTLINK_HOME`, dan koppelt het één keer, in de werkmap:
   `agentlink join "$INVITE_LINK" --no-service $AGENTLINK_JOIN_ARGS --posture "$AGENTLINK_POSTURE"`. Zonder link
   stopt het met een melding en exitcode 2. Mislukt het koppelen, dan stopt het met de exitcode van agentlink.
3. Het haalt `INVITE_LINK` uit de omgeving en start `agentlink run`.

Stopt het met een fout, dan start Docker de container met `restart: unless-stopped` steeds opnieuw, met dezelfde
melding in `docker logs`. Los het op, of stop de container.

Wat na het koppelen zou mislukken (geen sleutel, een home die niet schrijfbaar is), controleert het ervóór. Dan
blijft de uitnodiging ongebruikt. De link en de code komen nooit in de log: het startscript
haalt ze uit alles wat `agentlink join` meldt.

| Instelling | Wat | Standaard |
|---|---|---|
| `INVITE_LINK` | de uitnodigingslink, alleen de eerste keer | leeg |
| `AGENTLINK_JOIN_ARGS` | extra opties voor `join`, zonder spaties erin: `--service-agent`, `--tools off`, `--backend claude` | leeg |
| `AGENTLINK_POSTURE` | hoe de agent draait, in één regel (`join --posture`) | leeg |
| `ANTHROPIC_API_KEY_FILE` | het pad in de container naar de sleutel | `/run/secrets/anthropic-api-key` |
| `AGENTLINK_HOME` | de map met de koppeling | `/home/agent/.config/agentlink` |
| `CLAUDE_CONFIG_DIR` | de config en sessies van Claude Code | `/home/agent/.claude` |
| werkmap | waar de agent werkt; `join` legt hem vast | `/home/agent/work` |

## Het volume

Alles wat een herstart moet overleven, staat in `/home/agent`, op één volume:

- `.config/agentlink/agent.json`: de koppeling. Daarin staat de sleutel van de agent: dat is zijn identiteit.
- `.config/agentlink/device.key`: met end-to-end-versleuteling de sleutels van de agent als toestel.
- `.claude/`: de sessies van Claude Code, zodat een gesprek verdergaat waar het was.
- `work/`: de werkmap.

Neem het volume mee in je backup, en behandel het als een geheim: wie `agent.json` heeft, is de agent.

**Kwijt is opnieuw koppelen.** Zonder volume, of met een nieuw, start de container met de melding dat hij niet
gekoppeld is. Een gebruikte uitnodiging werkt niet opnieuw: maak een nieuwe in de app, en ontkoppel de oude agent in de
app.

## Geheimen

- **Als bestand, nooit in het image of in de omgeving.** De Anthropic-sleutel komt als bestand binnen (een
  read-only mount of een secret van compose). In de omgeving staat alleen het pad: `ANTHROPIC_API_KEY_FILE`.
  agentlink leest het bestand bij elke beurt, dus een nieuwe sleutel vraagt geen herstart.
- **Leesbaar voor uid 10001** in de container. Op Linux bijvoorbeeld met
  `sudo chown 10001 <bestand> && sudo chmod 0400 <bestand>`. Met rootless Docker is uid 10001 in de container een
  andere uid op de host (`/etc/subuid`); maak het bestand daar van die uid. Docker Desktop en OrbStack op macOS
  regelen het zelf.
- **Gebruik een eigen sleutel per agent,** met een limiet. Dan zie en begrens je per agent wat hij verbruikt.
- **Geen sleutel, maar een login van Claude Code?** Maak `ANTHROPIC_API_KEY_FILE` leeg; de CLI gebruikt dan zijn
  login in `CLAUDE_CONFIG_DIR`.

## Bijwerken

Bouw het image opnieuw met de nieuwe versie en maak de container opnieuw, met hetzelfde volume:

```bash
docker build -t agentlink https://github.com/appfabriek/agentlink.git#v<nieuwe versie>:docker
docker rm -f agentlink
docker run -d --name agentlink … agentlink        # zoals hierboven, zonder INVITE_LINK
```

Met compose: haal de `docker/` van de nieuwe versie, en dan `docker compose build --pull && docker compose up -d`.
De koppeling blijft: die staat op het volume.
Het installatiecommando werkt in de container niet; de root is alleen-lezen.

## Stoppen

- `docker stop agentlink` (of `docker compose stop`): agentlink neemt geen nieuw werk meer aan en geeft het signaal door
  aan een lopende CLI. Een antwoord dat er al is, wordt nog verstuurd. Daar wacht hij hoogstens 8 seconden op; Docker
  wacht er 10. Een beurt die zo afbrak, komt na de herstart opnieuw.
- `docker compose down` haalt de container weg en laat het volume staan. **`down -v` gooit het volume weg, en daarmee
  de koppeling.**
- **Ontkoppelen** doe je in de app. agentlink merkt het bij de volgende verbinding en stopt; met
  `restart: unless-stopped` start Docker hem daarna steeds opnieuw. Haal de container en het volume weg
  (`docker rm -f agentlink && docker volume rm agentlink-home`, of `docker compose down -v`).

## Kijken en instellen

- **Gezond:** het image heeft een `HEALTHCHECK` (elke 5 minuten `agentlink status`). Die is gezond zolang de server de
  agent kent; nog niet gekoppeld, een ingetrokken sleutel of geen netwerk is ongezond. Zie `docker ps` of
  `docker inspect --format '{{json .State.Health}}' agentlink`.
- **Status:** `docker exec agentlink agentlink status`. In de container staat bij Dienst "geen (container of
  --no-service)": de container is de dienst. De exitcode is 0 als de server de agent kent, anders 1.
- **Een instelling veranderen** (`tools`, `learn`, `owner-workdir`): stop de container, verander het met een
  losse container op hetzelfde volume, en start hem weer. `agentlink run` leest zijn config alleen bij de start, en
  kan een wijziging terwijl hij draait weer overschrijven.

  ```bash
  docker stop agentlink
  docker run --rm -v agentlink-home:/home/agent --entrypoint agentlink agentlink tools off
  docker start agentlink
  ```

## Een eigen image voor een taak

Een dienstagent krijgt een eigen image dat op dit image bouwt. Daarin staat zijn gereedschap, zijn werkmap en wat hij
mag. Zo ligt dat vast voordat iemand met hem praat:

```dockerfile
FROM agentlink:<versie>
USER root
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends <het gereedschap van de taak> \
 && rm -rf /var/lib/apt/lists/*
# Wat Claude Code mag: beheerde instellingen, van root, niet te overschrijven door de agent.
COPY --chmod=0644 managed-settings.json /etc/claude-code/
COPY werk/ /opt/taak/werk/
USER agent
WORKDIR /opt/taak/werk
ENV AGENTLINK_JOIN_ARGS="--service-agent --tools off" \
    AGENTLINK_POSTURE="Dienstagent in een Linux-container zonder root; alleen <gereedschap>."
```

- **De werkmap** is de map waarin `join` draait: de `WORKDIR` van het image. Daarin staan `CLAUDE.md` en de
  instellingen van de taak. In het image is hij alleen-lezen, en dat is goed: de agent verandert zijn eigen opdracht
  niet.
- **Wat de agent mag,** zet je in de [beheerde instellingen](https://code.claude.com/docs/en/managed-settings.md) van
  Claude Code in `/etc/claude-code/`: die gaan boven alles in de werkmap en in `CLAUDE_CONFIG_DIR`. Een dienstagent
  houdt bij `--tools off` de MCP-servers van zijn image en werkmap; een gewone agent met `off` krijgt geen enkele
  MCP-server en alleen de tools die hij mag.
- **Een eigen startstap** (een controle, een login): geef het startscript een commando mee. Na het koppelen start het
  dat in plaats van `agentlink run`, en dat commando eindigt met `exec agentlink run`:

  ```dockerfile
  ENTRYPOINT ["/usr/bin/tini", "-s", "--", "agentlink-start", "/opt/taak/na-het-koppelen"]
  ```

  Moet er iets gebeuren vóór het koppelen, laat dan je eigen script beginnen en eindigen met
  `exec agentlink-start`. Houd `tini` vooraan.
- **Een andere CLI:** bouw op `--target base` en installeer die CLI zelf.

Zo bouwt Appfabriek zelf de diagnoseagent van Appfabriek Diagnose (`appfabriek/crtdiag`, map `agent/`; die repo is
niet openbaar): beheerde instellingen die alleen zijn eigen MCP-server toestaan en elk ander gereedschap weigeren, een
werkmap met alleen zijn opdracht, en een eigen startstap na het koppelen.

## Een bestaande agent verhuizen naar een container

Een agent die al gekoppeld is, kan verhuizen zonder nieuwe uitnodiging. Hij houdt zijn naam, zijn gesprekken en zijn
plafond (`--tools`).

1. **Stop de oude dienst,** zodat hij ook na een herstart uit blijft. Op macOS:
   `launchctl bootout gui/$(id -u)/com.appfabriek.agentlink`; op Linux: `systemctl --user disable --now agentlink`.
   Een tweede agent (met `AGENTLINK_HOME=…/<naam>`) heet `com.appfabriek.agentlink.<naam>` en `agentlink-<naam>`.
   Draai een agent nooit op twee plekken tegelijk: ze antwoorden dan allebei, en met end-to-end-versleuteling raken
   de sleutels in de war.
2. **Kopieer drie bestanden** uit de map van de koppeling (`~/.config/agentlink`, of de map in `AGENTLINK_HOME`) naar
   een lege map, bijvoorbeeld `verhuis/`: `agent.json`, `device.key` (als die er is) en `notes.json` (als die er
   is). De rest van die map hoort bij de oude dienst.
3. **Pas `agent.json` aan:** `workdir` wordt de werkmap in de container (`/home/agent/work`, of de `WORKDIR` van je
   image), net als `ownerWorkdir` als die er staat. Zet `sessions` op `{}`: de sessies van de CLI horen bij de oude
   map, dus elk gesprek begint met een verse sessie. Met `jq`:

   ```bash
   jq '.workdir = "/home/agent/work" | .sessions = {}
       | if .ownerWorkdir then .ownerWorkdir = "/home/agent/work" else . end' \
     agent.json > agent.json.nieuw && mv agent.json.nieuw agent.json
   ```

4. **Zet ze op het volume,** van uid 10001 en alleen voor hem leesbaar:

   ```bash
   docker run --rm --user 0 -v agentlink-home:/home/agent -v "$PWD/verhuis:/van:ro" --entrypoint sh agentlink -c '
     d=/home/agent/.config/agentlink; mkdir -p "$d" && cp /van/* "$d"/ &&
     chown -R 10001:10001 /home/agent && chmod 700 "$d" && chmod 600 "$d"/*'
   ```

5. **Start de container zonder `INVITE_LINK`** en kijk in `docker logs` of hij verbindt. Stuur een bericht in de app.
6. **Werkt hij, ruim dan de oude plek op:** de plist in `~/Library/LaunchAgents` of de unit in
   `~/.config/systemd/user`, en de oude map. Niet met `agentlink leave`: dat ontkoppelt de agent, ook in de container.
