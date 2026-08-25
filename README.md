# dokku-deploy

Wiederverwendbarer GitHub-Actions-Workflow, der eine App auf **empire**
(Hetzner, Dokku) deployt. Wird von den App-Repos per `uses:` eingebunden.

Hintergrund, Serverzustand und Handgriffe: `devops/docs/empire-runbook.md`.

## Einbinden

In der `ci.yml` des App-Repos, **hinter** den bestehenden Test-Job:

```yaml
jobs:
  test:
    # ... die vorhandene Testsuite, unveraendert ...

  deploy:
    needs: test                       # <- die Bedingung "nur wenn CI gruen"
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    uses: Lionizers/dokku-deploy/.github/workflows/deploy.yml@main
    with:
      app: meine-app
      health_url: https://meine-app.lionizers.com/health
    secrets: inherit                  # nur INNERHALB von Lionizers/* -- siehe unten
```

`secrets: inherit` reicht die drei Secrets durch, ohne sie einzeln aufzuzaehlen.

## ⚠️ Ausserhalb der Lionizers-Org: Secrets einzeln durchreichen

Die GitHub-Doku beschraenkt `inherit` ausdruecklich auf denselben Owner:
*"Workflows that call reusable workflows **in the same organization or
enterprise** can use the `inherit` keyword to implicitly pass the secrets."*

Ein Repo unter einem persoenlichen Account (z.B. `NilsLoewe/coaching`) ist ein
anderer Owner. Dort bricht der Job nach zwei Sekunden ab mit:

```
Secret DOKKU_SSH_KEY is required, but not provided while calling.
```

**Die Meldung zeigt auf die Secrets, die Ursache ist die Owner-Grenze.** Alle
drei Secrets liegen im Repo und sind korrekt -- man sucht sonst genau an der
Stelle, die funktioniert. Belegt am 25.08.2026 beim Umzug von `coaching`.

```yaml
    secrets:
      DOKKU_SSH_KEY: ${{ secrets.DOKKU_SSH_KEY }}
      DOKKU_HOST: ${{ secrets.DOKKU_HOST }}
      DOKKU_HOST_KEY: ${{ secrets.DOKKU_HOST_KEY }}
```

Aus demselben Grund ist dieses Repo seit dem 25.08.2026 **public**: ein privater
Reusable Workflow laesst sich org-uebergreifend gar nicht erst aufrufen. Er
enthaelt keine Geheimnisse -- nur Secret-*Namen*, den Ablauf und Kommentare;
die Server-IP steht ohnehin oeffentlich im DNS.

## Erforderliche Secrets im App-Repo

| Secret | Inhalt |
|---|---|
| `DOKKU_SSH_KEY` | privater ed25519-Key; oeffentlicher Teil liegt als `github-actions` in `dokku ssh-keys:list` |
| `DOKKU_HOST` | IP von empire |
| `DOKKU_HOST_KEY` | Ausgabe von `ssh-keyscan -t ed25519 <IP>` |

Setzen laesst sich das mit `devops/server/onboard-app.sh`, das die App auf
empire anlegt und die Secrets gleich mit schreibt.

Zieht ein **MCP-Server** um, gilt zusaetzlich
`devops/docs/mcp-server-nach-empire.md` -- der OAuth-State in Redis muss
mitwandern, und `/mcp` taugt nicht als `health_url` (401 ohne Token).

## Eingaben

| Eingabe | Pflicht | Bedeutung |
|---|---|---|
| `app` | ja | Name der Dokku-App |
| `health_url` | nein | URL, die nach dem Deploy HTTP 200 liefern muss. Leer = keine Pruefung. |
| `health_attempts` | nein | Anzahl Versuche im Abstand von 10 s (Standard 10) |

## Warum zentral statt kopiert

Der Deploy-Weg aendert sich waehrend der Migration noch. Liegt er kopiert in 15
Repos, aendert man ihn 15-mal — und uebersieht drei. Hier ist es eine Datei.
