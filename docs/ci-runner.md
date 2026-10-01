<!--
  - SPDX-FileCopyrightText: 2026 FINDER e.V.
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Selbst gehostete GitHub-Runner auf dem FINDER-Host

Diese Anleitung ist die zentrale Beschreibung für alle privaten Repositories des Kontos `glazperle`,
die ihre CI auf dem FINDER-Host statt bei GitHub laufen lassen. Die einzelnen Repositories verweisen
hierher und beschreiben bei sich nur den Schalter. Entscheidung: Max, 2026-10-01. Vorlage in der
Umsetzung ist `glazperle/finder-board` (PR #928, ADR 0299, `docs/betrieb/ci-runner.md`).

Dieses Repository (`glazperle/server`) ist ein öffentlicher Abzweig von Nextcloud. Es hält nur diese
Anleitung. Seine eigenen Workflows laufen weiter bei GitHub und werden nicht umgestellt.

## 1. Zweck und Aufbau

Zweck: GitHub-Minuten sparen und schwere Läufe (Tests, Builds, Browser-Tests) auf der vorhandenen
Hardware ausführen, ohne die Produktion zu gefährden.

| Eigenschaft          | Wert                                                                                                                           |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Host                 | FINDER-Host, Ubuntu 22.04, 32 Kerne                                                                                             |
| Benutzer             | `ghrunner`, ohne Login, nicht in der Gruppe `docker`                                                                            |
| Docker               | eigenes **rootloses** Docker des Benutzers, `DOCKER_HOST=unix:///run/user/982/docker.sock` (in der Unit gesetzt)               |
| Sichtbarkeit         | Die Produktionscontainer sind für den Benutzer nicht erreichbar (anderer Docker-Daemon, anderer Socket)                         |
| Platz                | 150 GB, eigener Bereich unter `/opt/gh-runner`                                                                                  |
| Grenzen              | gemeinsam 12 Kerne und 24 GB RAM in `user-982.slice`, mit niedriger CPU- und IO-Priorität                                       |
| Instanzen            | systemd-User-Units `gh-runner@<repo>-<n>`                                                                                       |
| Etikett              | `finder-host` (dazu das übliche `self-hosted`)                                                                                  |
| Bindung              | Bei einem persönlichen Konto ohne Organisation ist **jede Instanz an genau ein Repository** gebunden                            |
| Aufräumen            | nächtlich um 03:30, nur im rootlosen Docker                                                                                     |
| Netzwerk             | Kein offener Port. Der Runner verbindet sich ausgehend zu GitHub (Long Polling über HTTPS)                                      |

### Schalter in den Repositories

Jeder umgestellte Job trägt

```yaml
runs-on: ${{ fromJSON(vars.CI_RUNNER_LABELS || '"ubuntu-latest"') }}
```

| Repository-Variable | Wirkung                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| nicht gesetzt       | Alle Jobs laufen auf `ubuntu-latest` bei GitHub (Ausgangszustand, keine Änderung).   |
| `CI_RUNNER_LABELS`  | JSON-Liste, hier `["self-hosted","finder-host"]`: die Jobs laufen auf dem FINDER-Host. |

Die Variable wird je Repository gesetzt (Einstellungen, Secrets and variables, Actions, Variables)
oder per Befehl:

```bash
gh variable set CI_RUNNER_LABELS -R glazperle/<repo> --body '["self-hosted","finder-host"]'
```

Jobs, die nicht auf den geteilten Host gehören, tragen weiter `runs-on: ubuntu-latest` mit einem
Kommentar, warum. Das sind in der Regel:

- Deploy- und Veröffentlichungsjobs (SSH auf den Host, Registry-Push, PyPI, Releases): Sie halten
  Secrets, und ein Deploy darf nicht von der Verfügbarkeit des Runners auf demselben Host abhängen.
- Jobs mit festen Ports, festen Container-Namen oder `docker compose` mit Produktionsnamen: Sie
  würden mit parallelen Läufen kollidieren.
- Jobs mit `sudo apt-get` oder `playwright install --with-deps`: Der Benutzer `ghrunner` hat kein `sudo`.
- Der externe Gesundheitscheck der Produktion: Er soll von außen messen.
- Zeitgesteuerte Kleinstjobs ohne Nutzen für die Auslastung.

## 2. Die Skripte auf dem Host

Beide liegen unter `/root/` und werden als `root` ausgeführt. Sie stehen nicht in diesem Repository.
Die Beschreibung gibt den vereinbarten Umfang wieder; weicht ein Skript davon ab, gilt das Skript, und
diese Anleitung wird angepasst.

**`/root/runner-basis.sh`** richtet die gemeinsame Grundlage ein (einmalig, wiederholbar):

- legt den Benutzer `ghrunner` an (ohne Login, ohne Gruppe `docker`), das Verzeichnis `/opt/gh-runner`
  und die systemd-User-Konfiguration samt Linger;
- installiert rootloses Docker für diesen Benutzer, mit Socket unter `/run/user/982/docker.sock`;
- lädt das Runner-Paket von GitHub nach `/opt/gh-runner` und installiert die Unit-Vorlage
  `gh-runner@.service` (mit gesetztem `DOCKER_HOST`);
- setzt die Grenzen in `user-982.slice` (12 Kerne, 24 GB RAM, niedrige CPU- und IO-Gewichtung);
- richtet das nächtliche Aufräumen um 03:30 ein.

**`/root/runner-instanz.sh`** legt eine einzelne Instanz an. Es erwartet die Umgebungsvariablen
`REPO` und `NR` und das **Registrierungs-Token auf stdin**:

- legt das Arbeitsverzeichnis `/opt/gh-runner/<repo>-<n>` an;
- registriert den Runner bei `https://github.com/glazperle/<repo>` mit dem Namen `<repo>-<n>` und den
  Etiketten `self-hosted,finder-host`;
- aktiviert und startet die Unit `gh-runner@<repo>-<n>`.

Das Token wird nie in einer Datei, im Shell-Verlauf oder in einem Kommandozeilenargument abgelegt,
sondern nur durch die Pipe gereicht.

## 3. Neue Instanz einrichten

Voraussetzung: Das Repository ist **privat** und kein Fork eines fremden Projekts. Die Jobs sind
umgestellt (siehe Abschnitt 8).

1. Registrierungs-Token holen (gilt eine Stunde, einmal benutzbar). Von einem Rechner mit `gh` und
   Adminrechten auf dem Repository:

   ```bash
   gh api -X POST repos/glazperle/<repo>/actions/runners/registration-token -q .token
   ```

2. Token über stdin an das Instanzskript auf dem Host geben (als `root`):

   ```bash
   gh api -X POST repos/glazperle/<repo>/actions/runners/registration-token -q .token \
     | ssh root@<host> 'REPO=<repo> NR=1 /root/runner-instanz.sh'
   ```

   Für eine zweite Instanz desselben Repositories `NR=2` verwenden. Als Faustregel genügt eine
   Instanz je Repository, bei schwerer CI (Browser-Tests, große Matrix) zwei.

3. Prüfen, dass die Instanz online ist:

   ```bash
   gh api repos/glazperle/<repo>/actions/runners -q '.runners[] | [.name,.status,.busy] | @tsv'
   ```

4. Erst danach die Variable setzen:

   ```bash
   gh variable set CI_RUNNER_LABELS -R glazperle/<repo> --body '["self-hosted","finder-host"]'
   ```

5. Einen Pull Request oder `workflow_dispatch` auslösen und im Lauf prüfen, dass der Job auf dem
   Runner `<repo>-<n>` lief. Danach das Repository in die Liste in Abschnitt 8 eintragen.

Die Reihenfolge ist wichtig: Wird die Variable gesetzt, bevor eine Instanz online ist, bleiben alle
Jobs in der Warteschlange.

## 4. Pausieren, Abmelden, Entfernen

**Pausieren eines Repositories (schnell, ohne Zugang zum Host):** die Variable löschen. Neue Jobs
laufen sofort wieder bei GitHub.

```bash
gh variable delete CI_RUNNER_LABELS -R glazperle/<repo>
```

Bereits wartende Jobs behalten ihr Etikett; sie werden abgebrochen und neu gestartet.

**Alles anhalten (auf dem Host, als `ghrunner`):**

```bash
systemctl --user stop 'gh-runner@*'
```

Zusätzlich die Variablen löschen, sonst bleiben neue Jobs wartend.

**Eine Instanz abmelden und entfernen:**

1. Unit stoppen und deaktivieren (als `ghrunner`):

   ```bash
   systemctl --user disable --now gh-runner@<repo>-<n>
   ```

2. Runner bei GitHub abmelden. Mit einem Entfernen-Token:

   ```bash
   gh api -X POST repos/glazperle/<repo>/actions/runners/remove-token -q .token
   ```

   Dann im Arbeitsverzeichnis `/opt/gh-runner/<repo>-<n>` als `ghrunner` `./config.sh remove --token <token>`
   ausführen (Token über die Umgebung oder die Eingabe, nicht in Dateien). Alternativ die Instanz in
   den Einstellungen des Repositories (Actions, Runners) entfernen oder per
   `gh api -X DELETE repos/glazperle/<repo>/actions/runners/<id>`.
3. Arbeitsverzeichnis `/opt/gh-runner/<repo>-<n>` löschen.
4. Die Variable `CI_RUNNER_LABELS` des Repositories löschen und das Repository aus Abschnitt 8
   streichen.

## 5. Aufräumen und Platz

- Nächtlich um 03:30 räumt ein Timer **nur im rootlosen Docker** auf (nicht benutzte Container,
  Abbilder, Netze, Build-Cache). Die Produktions-Docker-Instanz bleibt unberührt.
- Die Arbeitsverzeichnisse der Instanzen wachsen mit Abhängigkeiten und Caches. Insgesamt stehen
  150 GB unter `/opt/gh-runner` zur Verfügung.
- Platz prüfen (als `ghrunner`):

  ```bash
  df -h /opt/gh-runner
  du -sh /opt/gh-runner/* | sort -h | tail
  docker system df
  ```

- Sofort aufräumen, wenn der Platz knapp wird (als `ghrunner`, mit gesetztem `DOCKER_HOST`):

  ```bash
  docker system prune --all --force --volumes
  ```

  Das trifft nur den rootlosen Docker, kein Produktionssystem.
- Läufer-Arbeitsverzeichnisse (`_work`) einer Instanz dürfen bei gestoppter Unit gelöscht werden.

## 6. Grenzen ändern

Die Obergrenze gilt für alle Instanzen zusammen und liegt in `user-982.slice` (CPU, Speicher, IO).

```bash
# als root auf dem Host
systemctl set-property user-982.slice CPUQuota=1200% MemoryMax=24G
systemctl show user-982.slice -p CPUQuotaPerSecUSec -p MemoryMax
```

- `CPUQuota=1200%` entspricht 12 Kernen, `MemoryMax=24G` den 24 GB RAM. Zum Ändern andere Werte setzen.
- Die niedrige Priorität kommt aus `CPUWeight` und `IOWeight` (kleine Werte, etwa 20 und 20).
- Bei Speichermangel beendet der Kernel Prozesse innerhalb der Grenze, die Produktion ist davon nicht
  betroffen.
- Mehr Platz: Bereich `/opt/gh-runner` erweitern bzw. das Verzeichnis auf größeren Datenträger legen;
  danach `runner-basis.sh` und die Instanzen prüfen.
- Mehr Parallelität erreicht man nicht über höhere Grenzen, sondern über weitere Instanzen eines
  Repositories (`NR=2`, `NR=3`). Die Grenze bleibt gemeinsam.

## 7. Sicherheitsmodell und Restrisiko

Was schützt:

- **Kein offener Port.** Der Runner baut die Verbindung zu GitHub nur ausgehend auf.
- **Ein Repository je Instanz.** Der Runner nimmt nur Aufträge des einen privaten Repositories an, an
  das er bei der Registrierung gebunden wurde.
- **Unprivilegierter Benutzer.** `ghrunner` hat kein Login, kein `sudo`, keine Gruppe `docker` und
  keinen Zugriff auf die `.env` der Produktion.
- **Eigenes rootloses Docker.** Aufträge sehen die Produktionscontainer nicht und können sie nicht
  steuern.
- **Begrenzte Mittel.** 12 Kerne, 24 GB, niedrige Priorität, 150 GB Platz.
- **Secrets nur dort, wo nötig.** Jobs mit Deploy-Schlüsseln oder Registry-Zugängen laufen bei GitHub.

Restrisiko: **Wer in ein angebundenes Repository pushen oder einen Pull Request im Repository
auslösen darf, führt Code auf dem Host aus**, als unprivilegierter Benutzer und begrenzt. Ein
Workflow kann das Netzwerk des Hosts erreichen und andere Dateien des Benutzers `ghrunner` lesen,
etwa die Arbeitsverzeichnisse der anderen Instanzen. Deshalb gilt:

- Push-Rechte nur an Personen, denen man auch Serverzugang gäbe.
- **Keine öffentlichen Repositories** und **keine Forks fremder Projekte** an diesen Runner. Dort
  könnte jeder über einen Pull Request Code einschleusen.
- Den Workflow-Ordner `.github/workflows` von Dritten nicht ungeprüft übernehmen.
- Dependabot- und andere Bot-Pull-Requests laufen ebenfalls dort; sie stammen aus dem Repository selbst.
- Secrets der Repositories (Actions-Secrets) sind für Workflows auf dem Runner lesbar, soweit der
  Job sie anfordert. Deshalb keine Deploy-Secrets in Jobs, die auf dem Runner laufen.

## 8. Angebundene Repositories

Stand 2026-10-01. Die Runner-Instanzen registriert Max; die Spalte „Instanzen" ist der Vorschlag.
Eingetragen wird ein Repository erst, wenn Instanz und Variable gesetzt sind.

| Repository            | Instanzen | Umgestellt                                          | Bleibt bei GitHub                               |
| --------------------- | --------- | --------------------------------------------------- | ----------------------------------------------- |
| `finder-board`        | nach Bedarf | Vorlage, siehe dessen `docs/betrieb/ci-runner.md` und ADR 0299 | siehe dort |
| `ktw-forum`           | 1         | `build.yml`: `sync`, `theme`                         | `keycloak` (fester Name und Port), `discourse-version` |
| `ctc-survey`          | 1         | `ci.yml`: `frontend-checks`; `security-audit.yml`    | `backend-tests` (fester Port, `sudo`), `notify-analytik` (Token) |
| `proxi-app`           | 1         | `ci.yml`: `app`                                      | -                                               |
| `rebound`             | 2         | `test.yml`, `guards`, `i18n-check`, `mail-relay-test`, `typografie` | `e2e`, `e2e-full` (`--with-deps`, Compose, Ports) |
| `denkometer`          | 1         | `ci.yml`: alle Jobs                                  | -                                               |
| `analytikplattform`   | 2         | `ci.yml`: alle Jobs                                  | `deploy` (SSH, Secrets), `health-monitor`       |
| `rallly-finder`       | 1         | `ci.yml`: alle Jobs                                  | -                                               |
| `woop`                | 1         | `lint`, `test`, `build`, `security`, `type-check`    | `e2e` (`--with-deps`, fester Port)              |
| `rp`                  | 1         | `ci.yml`: `app`                                      | -                                               |
| `unplugged`           | 1         | `ci.yml`: `app`                                      | -                                               |
| `einfachwirksam`      | 1         | `ci.yml`: `app`                                      | -                                               |
| `sexuologie`          | 1         | `ci.yml`: `test`, `image`                            | -                                               |
| `odoo-mcp`            | 1         | `publish.yml`: `test`, `build`                       | Veröffentlichungsjobs, `mcp-inspector` (Compose), `docker-publish`, `mcp-registry`, `pages` |
| `bssh`                | 1         | `ci.yml`: `lint`                                     | `phpunit` (fester Port 33061)                   |
| `3vk`                 | 1         | `ci.yml`: alle Jobs                                  | -                                               |
| `cal-docker`          | 1         | `docs-consistency`                                   | Image-Builds und Cron (Registry-Secrets, Compose, Ports) |
| `kimai-mcp-msal`      | 1         | `ci.yml`: `test`                                     | `docker-publish`, `publish` (Registry, PyPI)    |
| `bookstack-mcp`       | 1         | `ci.yml`: `quality`, `docker`                        | -                                               |
| `leitfaden-praevention` | 1       | `ci.yml`: `build`                                    | -                                               |

Bleiben bewusst bei GitHub, ohne Umstellung:

| Repository         | Grund                                                                                   |
| ------------------ | --------------------------------------------------------------------------------------- |
| `server`           | öffentlicher Abzweig von Nextcloud, bleibt auf GitHub                                    |
| `cal.com`, `act`   | Abzweige fremder Projekte, bleiben auf GitHub                                            |
| `kimai_mcp`        | öffentlich, bleibt auf GitHub                                                            |
| `betterbahn-mcp`   | nur ein täglicher Zeitplanjob, kein Nutzen                                               |
| `fulltext-mcp`     | ein einziger Job, der auch Releases mit Token veröffentlicht                             |

## 9. Fehlersuche

Befehle auf dem Host als `ghrunner` (die Dienste laufen im Benutzerkontext, nicht als `root`).
Benutzer-Umgebung bei `su`/`sudo -u`: `export XDG_RUNTIME_DIR=/run/user/982`.

| Symptom                                   | Prüfung                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Jobs bleiben „Queued"                     | Ist eine Instanz online? `gh api repos/glazperle/<repo>/actions/runners`. Läuft die Unit? Stimmt das Etikett? Ist die Variable richtig geschrieben (gültiges JSON)? |
| Unit läuft nicht                          | `systemctl --user status gh-runner@<repo>-<n>`                                                                     |
| Protokoll der Instanz                     | `journalctl --user -u gh-runner@<repo>-<n> -f` (mit `--since "1 hour ago"` einschränken)                           |
| Alle Instanzen                            | `systemctl --user list-units 'gh-runner@*'`                                                                        |
| „Cannot connect to the Docker daemon"     | `echo $DOCKER_HOST` (erwartet `unix:///run/user/982/docker.sock`); `systemctl --user status docker`; `docker info`; Socket vorhanden? `ls -l /run/user/982/docker.sock` |
| `services:`/`docker run` schlägt fehl     | Port belegt (fester Port in einem anderen Lauf)? Job umstellen auf freien Port oder bei GitHub lassen                |
| „No space left on device"                 | `df -h /opt/gh-runner`, `docker system df`, dann Abschnitt 5                                                        |
| `sudo: command not found` / `apt-get` verweigert | Der Benutzer hat kein `sudo`. Job bei GitHub lassen oder die Abhängigkeit einmalig als `root` auf dem Host installieren |
| Playwright meldet fehlende Bibliotheken   | Einmalig als `root`: `npx playwright install-deps chromium`; Browser lädt der erste Lauf nach `~/.cache/ms-playwright` |
| Runner „offline" nach Neustart des Hosts  | `loginctl show-user ghrunner -p Linger` muss `yes` sein, sonst starten User-Units nicht ohne Login                  |
| Registrierung schlägt fehl                | Token abgelaufen (eine Stunde) oder schon benutzt: ein neues holen                                                  |
| Lauf ist langsam                          | Gewollt niedrige Priorität. `systemctl status user-982.slice` zeigt die Auslastung; bei Dauerlast weitere Instanzen nur, wenn Kerne frei sind |

Im Zweifel: Variable `CI_RUNNER_LABELS` löschen. Dann läuft alles wieder bei GitHub, und die Ursache
lässt sich in Ruhe suchen.
