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
    secrets: inherit
```

`secrets: inherit` reicht die drei Secrets durch, ohne sie einzeln aufzuzaehlen.

## Erforderliche Secrets im App-Repo

| Secret | Inhalt |
|---|---|
| `DOKKU_SSH_KEY` | privater ed25519-Key; oeffentlicher Teil liegt als `github-actions` in `dokku ssh-keys:list` |
| `DOKKU_HOST` | IP von empire |
| `DOKKU_HOST_KEY` | Ausgabe von `ssh-keyscan -t ed25519 <IP>` |

Setzen laesst sich das mit `devops/server/onboard-app.sh`, das die App auf
empire anlegt und die Secrets gleich mit schreibt.

## Eingaben

| Eingabe | Pflicht | Bedeutung |
|---|---|---|
| `app` | ja | Name der Dokku-App |
| `health_url` | nein | URL, die nach dem Deploy HTTP 200 liefern muss. Leer = keine Pruefung. |
| `health_attempts` | nein | Anzahl Versuche im Abstand von 10 s (Standard 10) |

## Warum zentral statt kopiert

Der Deploy-Weg aendert sich waehrend der Migration noch. Liegt er kopiert in 15
Repos, aendert man ihn 15-mal — und uebersieht drei. Hier ist es eine Datei.
