# L'image de test de diverse-odoo

`test-image/Dockerfile` construit l'image de job du gate `tests.yml` de
[diverse-cite-com/diverse-odoo](https://github.com/diverse-cite-com/diverse-odoo) :
`ghcr.io/xtremxpert/odoo-ci:20.0`.

Elle n'a aucun lien avec le `Dockerfile` à la racine de ce dépôt, l'image de job
de la chaîne GitLab de Bemade (`odoo-ci-dind.yaml`).

> **Historique.** Jusqu'au 2026-10-12, ce Dockerfile vivait dans diverse-odoo
> (`ci/Dockerfile`, workflow `ci-image-20.yml`) et l'image était publiée sous
> `ghcr.io/diverse-cite-com/odoo-ci`. Le jeton d'un dépôt de diverse-cite-com ne
> peut pas publier chez XtremXpert : la construction a donc déménagé ici avec
> le nom de l'image. L'historique git du fichier est resté dans diverse-odoo.

## Publication

Le workflow [`test-image.yml`](../.github/workflows/test-image.yml) construit et
publie l'image :

- à chaque push sur `main` qui touche `test-image/` ou le workflow ;
- à la main (`workflow_dispatch`).

Il tourne sur l'ensemble de runners `arc-odoo-ci` (Docker-in-Docker, cluster
maison ; valeurs dans `hermes-guardrails/agent-deploy/maison/arc-odoo-ci-values.yaml`)
et publie avec le `GITHUB_TOKEN` du dépôt : aucun secret. Deux tags :
`:20.0`, que tire le gate, et `:20.0-<sha>`, pour retrouver l'image d'un push
ancien.

## Le paquet doit être public

`tests.yml` s'exécute **dans** cette image (`container:`). Le runner de
diverse-cite-com la tire avec le jeton de son propre dépôt, qui ne peut pas lire
un paquet privé de XtremXpert. Le paquet `odoo-ci` doit donc être **public**.

GitHub crée un nouveau paquet en privé. Au premier build, il faut le basculer à
la main : `https://github.com/users/XtremXpert/packages/container/odoo-ci/settings`
→ « Change visibility » → Public. Vérifier ensuite sans s'authentifier :

```bash
docker logout ghcr.io
docker pull ghcr.io/xtremxpert/odoo-ci:20.0
```

Le rendre public n'expose rien : le Dockerfile ne contient aucune directive
`COPY` ni `ADD` (`grep -nE '^(COPY|ADD)' test-image/Dockerfile` ne renvoie
rien), donc l'image ne porte aucun fichier d'aucun dépôt.

## Ce que l'image contient, et ce qu'elle ne contient pas

Le détail et les sources sont commentés dans `test-image/Dockerfile`.

| Bloc | Contenu | Origine |
|---|---|---|
| 0 | garde-fou d'architecture (amd64 seulement) | — |
| 1 | `git`, `expect-dev` (`unbuffer`), `jq`, `unzip`, les bibliothèques du navigateur | liste du step « Provision » d'origine + `libnspr4`, `libnss3`, `libcups2t64` |
| 2 | Chrome for Testing, milestone 139, `linux64` | `googlechromelabs.github.io` |
| 3 | `manifestoo>=1.1`, `checklog-odoo`, `coverage`, `pyOpenSSL>=24.1` | PyPI |
| 4 | `oca_wait_for_postgres`, `oca_checklog_odoo` | `raw.githubusercontent.com/OCA/oca-ci`, épinglés par commit **et par sha256** |
| 5 | vérification : `command -v` sur les outils, `google-chrome --version`, import de `coverage` | — |

**Elle ne contient aucun code testé.** `addons/` et `vendored/` viennent du
checkout de la PR de diverse-odoo, que `tests.yml` copie dans
`/mnt/extra-addons`.

Conséquence à connaître : **une PR qui modifie `test-image/Dockerfile` n'est
validée par rien.** Le changement ne prend effet qu'au push sur `main`, qui
republie `:20.0`, et le prochain gate de diverse-odoo l'utilise. C'est la
contrepartie d'une image d'outillage partagée par toutes les PR.

## Modifier l'image

- Les versions se changent dans `test-image/Dockerfile`, aux endroits commentés :
  `CHROME_MILESTONE`, `OCA_CI_SHA` (et les deux sha256 juste en dessous, à
  recalculer avec `sha256sum` sur les fichiers récupérés), la liste de paquets.
- Chrome n'est pas figé au build : l'index est interrogé à la construction, donc
  `:20.0` peut changer de build de Chrome d'un push à l'autre à milestone
  constant.
- Construire en local, pour la même architecture que le runner (amd64 ; depuis
  un Mac Apple Silicon, par émulation) :

  ```bash
  docker build --platform linux/amd64 --file test-image/Dockerfile test-image
  ```

## Pourquoi trois paquets de plus que la liste d'origine

La liste apt d'origine (`docker-odoo-enterprise/Dockerfile:251-262`) ne suffit pas
à faire démarrer Chrome 139 sur `odoo:20.0` : `google-chrome --no-sandbox
--version` sort en 127 avec « `libnspr4.so: cannot open shared object file` », et
`ldd` liste cinq bibliothèques absentes (`libnspr4.so`, `libnss3.so`,
`libnssutil3.so`, `libsmime3.so`, `libcups.so.2`). D'où `libnspr4`, `libnss3` et
`libcups2t64`.

Aucun paquet `-dev` n'est installé, volontairement : l'image de base ajoute le
dépôt PGDG, dont `libpq5` est en 18.x, alors que le `libpq-dev` d'Ubuntu 24.04
exige `libpq5 = 16.x`. Installer un `-dev` échoue en « held broken packages ».
