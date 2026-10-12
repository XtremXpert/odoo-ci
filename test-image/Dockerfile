# Image de TEST de diverse-odoo — l'image de job du gate `tests.yml` de
# diverse-cite-com/diverse-odoo.
#
# Construite par `.github/workflows/test-image.yml` de ce dépôt et publiée sous
# `ghcr.io/xtremxpert/odoo-ci:20.0`. Elle vivait dans diverse-odoo (`ci/`,
# publiée sous `ghcr.io/diverse-cite-com/odoo-ci`) jusqu'au 2026-10-12. Ce
# fichier n'a aucun lien avec le `Dockerfile` à la racine de ce dépôt (l'image
# de job de la chaîne GitLab de Bemade). Aucune dépendance à une infrastructure
# de build externe :
#   - base      : image officielle Odoo (Docker Hub, publique), le même tag que
#                 l'image de production de diverse-odoo ;
#   - navigateur: Chrome for Testing, index public googlechromelabs.github.io ;
#   - outillage : PyPI (manifestoo, checklog-odoo, coverage) ;
#   - scripts   : dépôt public github.com/OCA/oca-ci, épinglé par commit.
#
# POURQUOI CETTE IMAGE EXISTE. `odoo:20.0` est l'image de PRODUCTION : elle ne
# contient ni git, ni navigateur, ni outillage de test. Le gate installait donc
# tout cela à chaque exécution : ~5 minutes avant le premier test, à chaque PR,
# et une dépendance à apt, PyPI et googlechromelabs qui n'a rien à voir avec le
# code testé. Ce fichier construit tout cela une fois.
#
# PAQUET PUBLIC. `tests.yml` s'exécute DANS cette image (`container:`) : c'est
# le runner ARC de diverse-cite-com qui la tire, et le jeton d'un dépôt de
# diverse-cite-com ne peut pas lire un paquet privé de XtremXpert. Le paquet
# `odoo-ci` de XtremXpert doit donc rester public. Ça n'expose rien : ce
# fichier ne contient AUCUNE directive COPY ni ADD
# (`grep -nE '^(COPY|ADD)' test-image/Dockerfile` ne renvoie rien), donc l'image
# ne porte aucun fichier d'aucun dépôt.
#
# CE QUE CETTE IMAGE NE CONTIENT PAS : le code testé. `addons/` et `vendored/`
# viennent du checkout de la PR de diverse-odoo — `tests.yml` les copie à plat
# dans /mnt/extra-addons, le chemin et la disposition que leur donne le
# Dockerfile de production de diverse-odoo. Une image de test qui embarquerait
# du code testerait le code d'hier ; c'est le contenu de la PR qui doit être
# testé. Seul `test-image/Dockerfile` justifie donc de reconstruire cette image.
#
# ARCHITECTURE. Chrome for Testing n'est récupéré qu'en `linux64`, l'artefact
# x86_64 : le runner ARC (dind) est amd64. Un build sur une autre architecture
# produirait une image dont le navigateur ne s'exécute pas — et un gate vert
# dont tous les tours ont été skippés. Le garde-fou TARGETARCH plus bas
# transforme ce piège en échec de build.
#
# Prérequis d'amorçage : cette image doit avoir été publiée AU MOINS UNE FOIS
# avant que `tests.yml` puisse la tirer. Voir test-image/README.md.

# Un `ARG` déclaré avant `FROM` ne vaut que pour la ligne `FROM` (voir la note
# du Dockerfile de production, diverse-odoo/Dockerfile:20-23) : il n'est pas redéclaré ici
# parce que cette image ne se sert de `${ODOO_VERSION}` nulle part ailleurs.
ARG ODOO_VERSION=20.0
FROM odoo:${ODOO_VERSION}

# Contrairement à l'image de production (dernière ligne : `USER odoo`), cette
# image reste root, et ce n'est pas de la commodité : les steps du gate écrivent
# hors de /home/odoo. `pip install -r requirements.txt` modifie le python du
# système (ubuntu:noble, PEP 668) et `cp -RL addons/. /mnt/extra-addons/`
# écrit dans le volume des addons. Le conteneur est jetable, lancé une fois par
# job ; `tests.yml` demande d'ailleurs explicitement `--user root`.
USER root

# `odoo:20.0` hérite de `SHELL ["/bin/bash", "-xo", "pipefail", "-c"]`
# (odoo/docker 20.0/Dockerfile, ligne 4) : les RUN ci-dessous sont donc exécutés
# par bash, pipefail compris. `set -euo pipefail` est tout de même répété dans
# chaque bloc pour que le comportement ne dépende pas d'une ligne héritée que
# personne ne relit ici.

# Milestone Chrome : même défaut que le provisionnement au runtime
# (`CHROME_MILESTONE: "139"` dans tests.yml, lui-même le défaut de la CI
# historique, docker-odoo-enterprise/Dockerfile:232). L'index
# googlechromelabs est interrogé AU BUILD et donne la dernière build stable de
# ce milestone : le numéro exact n'est pas figé ici, pas plus qu'il ne l'était
# au runtime — voir « Not verified » dans test-image/README.md.
ARG CHROME_MILESTONE=139

# --- 0. Garde-fou d'architecture ---------------------------------------------
# L'artefact Chrome récupéré à l'étape 2 est x86_64. Le sélectionner sur une
# autre architecture donnerait une image publiable et inutilisable : Odoo
# marque chaque test de navigateur comme skippé (il attend le port devtools et
# abandonne), le gate de skip échoue alors pour une raison qui n'a rien à voir
# avec le code. Mieux vaut une image qui ne se construit pas.
# TARGETARCH est fourni par BuildKit ; repli sur l'architecture de l'hôte si le
# builder ne le définit pas, comme le fait l'image de base elle-même
# (odoo/docker 20.0/Dockerfile, `if [ -z "${TARGETARCH}" ]`).
ARG TARGETARCH
RUN set -euo pipefail \
    && ARCH="${TARGETARCH:-$(dpkg --print-architecture)}" \
    && if [ "$ARCH" != "amd64" ]; then \
         echo "ERROR: test-image/Dockerfile ne supporte que linux/amd64 (cible = ${ARCH})." >&2; \
         echo "       Chrome for Testing est téléchargé en linux64 (x86_64) à l'étape 2." >&2; \
         echo "       Construire avec --platform linux/amd64, ou aligner le runner." >&2; \
         exit 1; \
       fi \
    && echo "architecture OK: ${ARCH}"

# --- 1. Paquets système ------------------------------------------------------
# Liste identique à celle du step « Provision » d'origine, dans le même ordre :
#   * les bibliothèques du navigateur (docker-odoo-enterprise/Dockerfile:251-262,
#     la liste de la CI d'origine) — `libasound2t64` est le nom que cette
#     bibliothèque porte sur noble ;
#   * `git`, parce que `actions/checkout` exécute git DANS ce conteneur et que
#     `odoo:20.0` n'en installe pas ;
#   * `expect-dev`, qui fournit `unbuffer` — la commande de test transcrite
#     passe par lui (tests.yml, « Install and test the addons ») ;
#   * `jq` et `unzip`, utilisés par l'étape 2 ci-dessous.
#
# TROIS PAQUETS DE PLUS QUE LA LISTE D'ORIGINE, pour une raison MESURÉE et non
# déduite : la liste d'origine ne suffit pas à faire démarrer Chrome 139 dans
# l'image `odoo:20.0`. Construite telle quelle, l'étape 2 échoue —
# `google-chrome --no-sandbox --version` sort en 127 avec « libnspr4.so: cannot
# open shared object file » — et `ldd /opt/chrome-linux64/chrome` liste cinq
# bibliothèques absentes : libnspr4.so, libnss3.so, libnssutil3.so,
# libsmime3.so, libcups.so.2. Elles viennent de `libnspr4`, `libnss3` et
# `libcups2t64` (`libcups2t64` est le nom que libcups2 porte sur noble, comme
# libasound2t64 juste au-dessus). Le step « Provision » d'origine ne les
# installait pas non plus : le gate n'aurait pas passé sa PREMIÈRE étape sur le
# runner. La correction vit donc ici ; si ce step est un jour restauré, elle
# doit y être reportée.
#
# AUCUN PAQUET `-dev` ICI, délibérément — même raison que l'image de production
# (diverse-odoo/Dockerfile:37-42). L'image de base ajoute le dépôt PostgreSQL (PGDG), dont
# `libpq5` est en 18.x alors que le `libpq-dev` d'Ubuntu 24.04 exige
# `libpq5 = 16.15` : installer un paquet `-dev` échoue en « held broken
# packages » (exit 100, constaté en construisant cette image). Elle n'a besoin
# d'aucun en-tête de compilation — les dépendances du projet s'installent en
# roues binaires — et `expect-dev` n'est pas concerné : c'est le paquet
# d'`expect`, il fournit `unbuffer`, pas des en-têtes.
# `pg_isready` (postgresql-client) est déjà présent dans l'image de base
# (odoo/docker 20.0/Dockerfile, « install latest postgresql-client ») :
# `tests.yml` en dépend via `oca_wait_for_postgres`, et l'assertion finale de ce
# fichier le vérifie plutôt que de le réinstaller.
RUN set -euo pipefail \
    && apt-get update -qq \
    && apt-get install -y -qq --no-install-recommends \
         git \
         ffmpeg \
         expect-dev \
         tzdata-legacy \
         jq \
         unzip \
         libatk-bridge2.0-0 \
         libatk1.0-0 \
         libdrm2 \
         libxcomposite1 \
         libxdamage1 \
         libxfixes3 \
         libxrandr2 \
         libgbm1 \
         libxkbcommon0 \
         libpango-1.0-0 \
         libcairo2 \
         libasound2t64 \
         libnspr4 \
         libnss3 \
         libcups2t64 \
    && rm -rf /var/lib/apt/lists/*

# --- 2. Chrome for Testing ---------------------------------------------------
# Verbatim de l'étape d'origine : même index, même artefact `linux64`, même lien
# symbolique. Odoo cherche `google-chrome` sur le PATH
# (odoo/tests/common.py:2127), et `--no-sandbox` est ce qu'Odoo passe lui-même
# au navigateur (odoo/tests/common.py:1444) : Chrome refuse de démarrer en root
# sans ce drapeau, et ce conteneur est root. Le `--version` final est le même
# contrôle qu'au runtime — il échoue ici, au build, plutôt que dans un job de PR.
RUN set -euo pipefail \
    && curl -sSL "$(curl -s https://googlechromelabs.github.io/chrome-for-testing/latest-versions-per-milestone-with-downloads.json \
        | jq -r ".milestones.\"${CHROME_MILESTONE}\".downloads.chrome | .[] | select(.platform == \"linux64\") .url")" \
         -o /tmp/chrome.zip \
    && unzip -q /tmp/chrome.zip -d /opt \
    && ln -snf /opt/chrome-linux64/chrome /usr/bin/google-chrome \
    && rm -f /tmp/chrome.zip \
    && /usr/bin/google-chrome --no-sandbox --version

# --- 3. Outillage Python -----------------------------------------------------
# `odoo:20.0` n'a ni venv ni uv, et son python3.12 est marqué
# EXTERNALLY-MANAGED (PEP 668, ubuntu:noble) : `--break-system-packages` est
# obligatoire — le même échappement que l'image de production (diverse-odoo/Dockerfile:80) et
# le job Playwright du pipeline GitLab (.gitlab-ci.yml:56,
# PIP_BREAK_SYSTEM_PACKAGES=1). `--ignore-installed` en revanche n'est PAS
# nécessaire ici : ces trois paquets ne sont installés par aucun paquet Debian de
# l'image de base, donc pip n'a jamais à en désinstaller un (c'est ce qui
# l'oblige côté production, voir diverse-odoo/Dockerfile:66-71).
#   manifestoo    : liste les addons et leurs dépendances (tests.yml,
#                   `manifestoo --select-addons-dir … list` / `list-depends`) ;
#   checklog-odoo : lu par l'enveloppe OCA de l'étape 4 ;
#   coverage      : l'instrumentation du rapport (tests.yml, `coverage run`,
#                   `coverage report|xml|html`).
# Versions identiques à celles du step d'origine : `manifestoo>=1.1`, les deux
# autres sans borne. Rien de ce fichier n'est installé ici : `requirements.txt`
# est celui de la PR, et le gate l'installe à chaque run (c'est justement ce qui
# ne peut pas être figé dans une image).
RUN set -euo pipefail \
    && python3 -m pip install --no-cache-dir --break-system-packages \
         "manifestoo>=1.1" \
         checklog-odoo \
         coverage

# --- 3bis. Le couple cryptography / pyOpenSSL, sans quoi le gate ne démarre
#           même pas Odoo ------------------------------------------------------
# Ce n'est PAS un extra. L'étape « Install this project's python requirements »
# de tests.yml installe `requirements.txt` avec `--ignore-installed`, ce qui fait
# monter cryptography en 50.x (tirée par les dépendances de web-push). Le
# pyOpenSSL 23 livré par Debian, lui, reste en place — et il référence des
# constantes disparues depuis cryptography 43. Odoo en meurt à l'import :
#
#   File "/usr/lib/python3/dist-packages/OpenSSL/crypto.py", line 846
#   AttributeError: module 'lib' has no attribute 'GEN_EMAIL'
#
# Mesuré sur une initialisation de base réelle avec l'image de production, qui
# présentait exactement ce couple ; le gate y est exposé à l'identique, puisque
# c'est le même `--ignore-installed` sur la même image de base.
#
# `--ignore-installed` est donc obligatoire ICI, contrairement au §3 :
# pyOpenSSL EST fourni par Debian dans cette image, donc pip ne peut pas le
# désinstaller pour installer sa version. Installer les deux dans /usr/local
# règle le problème pour de bon : ce pyOpenSSL n'est pas dans requirements.txt,
# donc le gate ne le réinstallera pas et le couple restera cohérent.
RUN set -euo pipefail \
    && python3 -m pip install --no-cache-dir --break-system-packages --ignore-installed \
         "pyOpenSSL>=24.1"

# --- 4. Scripts d'aide OCA ---------------------------------------------------
# `oca_wait_for_postgres` (interroge `pg_isready` jusqu'à ce que postgres
# réponde) et `oca_checklog_odoo` (un `cat` tant que OCA_ENABLE_CHECKLOG_ODOO
# n'est pas positionné — l'état dans lequel la CI d'origine les laissait,
# docker-odoo-enterprise/Dockerfile:309) viennent du dépôt public OCA/oca-ci,
# exactement comme dans le step d'origine. OCA/oca-ci ne publie aucun tag :
# l'épinglage est un commit, le MÊME que celui du step d'origine.
# Les deux sommes figent en plus le CONTENU servi pour ce commit (calculées sur
# ce que ce commit sert aujourd'hui, reproductible par `sha256sum` sur le
# fichier de /usr/local/bin) : un déplacement de tag, une réécriture du dépôt ou
# un proxy qui répond autre chose font échouer le build au lieu de publier une
# image avec un script que personne n'a relu. Pour mettre à jour : changer le
# SHA, relancer le fetch, remplacer les deux sommes.
ARG OCA_CI_SHA=75ea09c037640ee38fed9d929b07e348cd42061a
RUN set -euo pipefail \
    && OCA_CI_BIN="https://raw.githubusercontent.com/OCA/oca-ci/${OCA_CI_SHA}/bin" \
    && curl -fsSL "${OCA_CI_BIN}/oca_wait_for_postgres" -o /usr/local/bin/oca_wait_for_postgres \
    && curl -fsSL "${OCA_CI_BIN}/oca_checklog_odoo"     -o /usr/local/bin/oca_checklog_odoo \
    && chmod 755 /usr/local/bin/oca_wait_for_postgres /usr/local/bin/oca_checklog_odoo \
    && echo "d7234cbe32d7492603b9ee0a30691473fb65a32914f4ce2e791d4bac033bd7a9  /usr/local/bin/oca_wait_for_postgres" \
         | sha256sum -c - \
    && echo "6d3f5e9e52566811633287886e05efa8250e89b0a5d2db831c2d7419ccd2ee58  /usr/local/bin/oca_checklog_odoo" \
         | sha256sum -c -

# --- 5. Vérification de l'image ----------------------------------------------
# L'assertion finale du step d'origine, exécutée ici à la construction : mieux
# vaut une image qui ne se construit pas qu'un gate qui échoue vingt minutes plus
# tard, dans Odoo, sur un outil absent. `command -v` sort non-zéro dès qu'un des
# noms manque. L'import de `coverage` est l'équivalent du canari d'ABI de la CI
# d'origine (`Dockerfile`:53-55 à la racine de ce dépôt) : il prouve que la couche pip est
# importable, pas seulement que le point d'entrée existe.
RUN set -euo pipefail \
    && command -v git manifestoo unbuffer pg_isready \
         oca_wait_for_postgres oca_checklog_odoo google-chrome \
    && python3 -c "import coverage; print('coverage', coverage.__version__)"
