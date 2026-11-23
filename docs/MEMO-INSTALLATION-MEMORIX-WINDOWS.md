# Installation de Memorix HTTP sur un autre PC Windows

Pour l’agent IA chargé de l’installation. Reproduire une instance Memorix par compte Windows, accessible aux clients locaux sur `http://127.0.0.1:3211/mcp` et démarrée par le Planificateur de tâches. Installer uniquement les intégrations demandées. Utiliser les chemins, le compte et les versions réellement présents sur le PC cible.

## 1. Vérifier le PC cible

Ouvrir PowerShell avec le compte qui utilisera les agents. Vérifier Node.js, npm, Git et la disponibilité du port. Si Memorix existe déjà, relever sa version, l’état de la tâche et du service, puis suivre la procédure de remplacement à la fin.

```powershell
node --version
npm --version
git --version
Get-Command memorix.cmd -ErrorAction SilentlyContinue
Get-ScheduledTask -TaskName 'Memorix HTTP (background ensure)' -ErrorAction SilentlyContinue
Get-NetTCPConnection -LocalPort 3211 -State Listen -ErrorAction SilentlyContinue
```

**PROUVÉ sur le PC de référence.** Le paquet déclare Node.js `>=22.18.0`. La configuration a été vérifiée avec Node.js `24.22.1`, npm `11.19.1`, Git `2.56.0.windows.3` et Memorix `1.10.5-local.1`. Si le port 3211 est occupé, identifier le propriétaire avant d’intervenir.

## 2. Installer le paquet avec les deux correctifs HTTP

Choisir l’une des deux méthodes. Exécuter les commandes dans l’ordre et traiter tout échec avant de poursuivre. La méthode B comporte un conflit de fusion prévu et documenté.

### A. Utiliser le paquet de référence

Transférer `paquets/memorix-1.10.5-local.1.tgz` depuis le dossier de référence. Adapter le chemin à son emplacement sur le PC cible. Le contrôle SHA-256 concerne cette archive précise.

```powershell
$paquet = (Resolve-Path -LiteralPath '.\memorix-1.10.5-local.1.tgz').Path
$attendu = '4646E2DF3147723DD4E734D72B0D6A767D675CDA1A7EECFBAE6289F82AB6FFC2'
if ((Get-FileHash -LiteralPath $paquet -Algorithm SHA256).Hash -ne $attendu) {
    throw 'Le SHA-256 du paquet diffère de celui de référence.'
}
npm install --global "$paquet"
if ($LASTEXITCODE -ne 0) { throw 'Échec de npm install.' }
memorix.cmd --version
```

### B. Reconstruire depuis le fork public

Dans un dossier de travail neuf, cloner le fork et combiner les deux commits épinglés. Git doit disposer d’une identité locale valide pour créer le commit de fusion. Installer les dépendances dans ce nouveau clone avec `npm ci`.

```powershell
git clone https://github.com/jfsema/memorix.git memorix-http
Set-Location -LiteralPath '.\memorix-http'
git switch --create local/integration 09f2a796cb19eaa3bff13797478fe52b4299745e
git merge --no-ff --no-commit 1b8d5840fe310fd0f6a87fc3e431f9ce5d2c1b46
git diff --name-only --diff-filter=U
```

**PROUVÉ avec `git merge-tree`.** Cette fusion signale un seul conflit, dans
`src/cli/commands/serve-http.ts`. Remplacer le bloc entre les marqueurs
`<<<<<<<` et `>>>>>>>` par ces deux imports. Si un autre conflit apparaît,
examiner l’écart avant de poursuivre.

```typescript
import {
  DEFAULT_SESSION_DISCONNECT_GRACE_MS,
  createSessionStreamWatch,
  parseSessionDisconnectGraceMs,
} from './http-session-stream-watch.js';
import { createCachedProjectDetector } from './project-detection-cache.js';
```

Préparer le fichier résolu et vérifier que l’arbre Git correspond à la fusion
de référence avant de créer le paquet.

```powershell
git add -- src/cli/commands/serve-http.ts
$tree = git write-tree
if ($LASTEXITCODE -ne 0 -or $tree -ne '93b518eb2f6d0efd2ea18dfb1baa2475d25d2631') {
    throw 'Arbre Git différent de la fusion de référence.'
}
git commit -m 'Intègre les deux correctifs HTTP de Memorix'
npm ci
npm version 1.10.5-local.1 --no-git-tag-version
npm run build
npm pack
npm install --global '.\memorix-1.10.5-local.1.tgz'
memorix.cmd --version
```

Résultat attendu pour les deux méthodes. `memorix.cmd --version` affiche `1.10.5-local.1`. Conserver la version, la provenance du paquet et les éventuels avertissements npm dans le compte rendu. Pour le build reconstruit, calculer sa propre empreinte. [Expiration des sessions SSE #408](https://github.com/AVIDS2/memorix/pull/408) · [Cache de détection #409](https://github.com/AVIDS2/memorix/pull/409)

## 3. Créer la tâche Windows et vérifier le service

Le service doit survivre à la fermeture du terminal de l’agent. Réserver les commandes `memorix background start|ensure|restart|stop` à la gestion prévue par le Planificateur et lancer le service depuis l’agent avec `schtasks /run`. Dans le script, `background start` est uniquement l’argument enregistré dans l’action de la tâche.

**PROUVÉ sur le PC de référence.** L’action utilise `conhost.exe --headless` et le chemin absolu de `memorix.cmd`. La tâche tourne avec le compte interactif, au niveau de privilège limité, à l’ouverture de session et toutes les cinq minutes. Le délai de deux minutes limite l’action planifiée de lancement. La tâche doit rester indépendante du terminal de l’agent.

Le bloc suivant crée la tâche uniquement si elle est absente. Si elle existe, inspecter son action, son compte et ses déclencheurs avant toute modification.

```powershell
$ErrorActionPreference = 'Stop'
$taskName = 'Memorix HTTP (background ensure)'
if (Get-ScheduledTask -TaskName $taskName -TaskPath '\' -ErrorAction SilentlyContinue) {
    throw 'Tâche existante. Vérifier sa configuration avant de la modifier.'
}

$account = [Security.Principal.WindowsIdentity]::GetCurrent().Name
$memorixCmd = (Get-Command memorix.cmd -ErrorAction Stop).Source
$memorixDir = Join-Path $env:USERPROFILE '.memorix'
New-Item -Path $memorixDir -ItemType Directory -Force | Out-Null

$action = New-ScheduledTaskAction `
    -Execute (Join-Path $env:SystemRoot 'System32\conhost.exe') `
    -Argument ('--headless cmd.exe /c ""{0}" background start"' -f $memorixCmd) `
    -WorkingDirectory $memorixDir
$logon = New-ScheduledTaskTrigger -AtLogOn -User $account
$periodic = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(5) `
    -RepetitionInterval (New-TimeSpan -Minutes 5)
$principal = New-ScheduledTaskPrincipal -UserId $account -LogonType Interactive -RunLevel Limited
$settings = New-ScheduledTaskSettingsSet -MultipleInstances IgnoreNew -StartWhenAvailable `
    -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
    -ExecutionTimeLimit (New-TimeSpan -Minutes 2)
Register-ScheduledTask -TaskName $taskName -TaskPath '\' -Action $action `
    -Trigger @($logon, $periodic) -Principal $principal -Settings $settings

schtasks /run /tn "$taskName"
if ($LASTEXITCODE -ne 0) { throw 'Échec du lancement de la tâche.' }
```

**DOC OFFICIELLE.** Une répétition sans durée se poursuit indéfiniment. `schtasks /run` demande un lancement immédiat avec le compte et l’action enregistrés. [Répétition du Planificateur](https://learn.microsoft.com/en-us/windows/win32/taskschd/repetitionpattern-duration) · [Commande schtasks run](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks-run)

Attendre environ 20 secondes après la première demande de lancement. Si le service initialise encore ses fichiers, réessayer jusqu’à 90 secondes après le lancement, puis examiner les journaux. Vérifier le service et le PID depuis un nouveau terminal si possible.

```powershell
$health = Invoke-RestMethod 'http://127.0.0.1:3211/health' -TimeoutSec 5
$health | Select-Object status, mode, port, pid
Get-NetTCPConnection -LocalPort 3211 -State Listen |
    Select-Object LocalAddress, LocalPort, OwningProcess
```

Attendu. `status = ok`, `mode = control-plane`, `port = 3211` et écoute sur `127.0.0.1`. Le propriétaire du port doit correspondre au PID renvoyé par `/health`. Le tableau de bord est à `http://127.0.0.1:3211/`.

## 4. Configurer les clients MCP

Pour un client pris en charge sans intégration existante, utiliser le CLI pour préparer l’intégration HTTP. Remplacer `cursor` par le client voulu. Les identifiants utiles sont `claude`, `codex`, `cursor`, `windsurf`, `opencode` et `dsh`.

```powershell
memorix.cmd setup --agent cursor --global --mcp http --dry-run
memorix.cmd setup --agent cursor --global --mcp http
```

Cette version de `setup` génère `http://localhost:3211/mcp`. Modifier seulement le point d’accès du serveur Memorix en `http://127.0.0.1:3211/mcp`. Pour une intégration déjà installée, effectuer ce changement dans l’entrée existante. Garder les autres serveurs, options et formats du client. Les chemins ci-dessous sont des repères sous `%USERPROFILE%`. Vérifier le fichier réellement lu sur le PC cible.

| Client | Fichier ou réglage à vérifier |
|---|---|
| Claude Code | Avec le plugin local, `.claude/plugins/marketplaces/memorix-local/plugins/memorix/.mcp.json` et `.claude/plugins/cache/memorix-local/memorix/<version>/.mcp.json`. Fusionner si nécessaire `"MCP_TIMEOUT": "60000"` dans l’objet `env` de `.claude/settings.json` pour reprendre le délai du PC de référence. |
| Codex | Avec `memorix@personal`, corriger la source `plugins/memorix/.mcp.json` et vérifier la copie active `.codex/plugins/cache/personal/memorix/<version>/.mcp.json`. Le fichier `.codex/config.toml` active le plugin dans cette configuration. |
| Cursor | `.cursor/mcp.json`. |
| Windsurf | `.codeium/windsurf/mcp_config.json`. |
| OpenCode | `.config/opencode/opencode.json` pour la configuration globale observée. |
| ZCode | `.zcode/cli/config.json`, entrée `mcp.servers.memorix`. Conserver `"protocolVersion": "legacy"` dans cette entrée. |
| DeepSeek Harness | `.dsh/cordis.patch.yml`, ou `cordis.patch.yml` dans le répertoire indiqué par `DSH_HOME`. |

Pour une installation ZCode neuve, ajouter cette entrée sous `mcp.servers`.

```json
{
  "memorix": {
    "type": "http",
    "url": "http://127.0.0.1:3211/mcp",
    "timeoutMs": 60000,
    "protocolVersion": "legacy"
  }
}
```

**PROUVÉ sur le PC de référence.** Avec Claude Code, les mesures ont donné une connexion moderne via `127.0.0.1` et un repli vers l’ancien protocole via `localhost`. Refaire les mesures sur le PC cible si la négociation du protocole reste lente.

Relancer les clients modifiés si nécessaire, puis ouvrir une nouvelle session dans un vrai dépôt Git. Dans Codex et Claude Code, vérifier les copies actives après toute réinstallation de leur plugin.

## 5. Valider une session par client

1. Depuis la racine du vrai dépôt, relever son chemin absolu avec `git rev-parse --show-toplevel`.
2. Appeler `memorix_session_start` avec ce chemin comme `projectRoot` et le nom du client comme `agent`. Vérifier que le résultat lie explicitement le projet attendu.
3. Appeler `memorix_project_context` avec la tâche réelle, puis `memorix_session_end` avec l’identifiant renvoyé. Répéter sur chaque client demandé. Dans les futures sessions, utiliser `memorix_search` pour consulter une décision passée et `memorix_store` pour les décisions et bugs non triviaux.
4. Dans le compte rendu, noter la version installée, le résultat de `/health` et les clients effectivement testés. Si seules les configurations ont été relues, préciser ce qui reste à tester dans les clients.

## 6. Dépanner et revenir à la version précédente

- **Service injoignable.** Vérifier le résultat de la tâche avec `Get-ScheduledTaskInfo -TaskName 'Memorix HTTP (background ensure)'`. Consulter `%USERPROFILE%\.memorix\background.log` et, s’il existe, `background.log.stdout`. Vérifier le chemin de `memorix.cmd`, l’accès à `node.exe` sous le compte de la tâche et le propriétaire du port. Utiliser ensuite `schtasks /run /tn "Memorix HTTP (background ensure)"` pour réessayer.
- **Service sain et client absent.** Vérifier l’URL de l’entrée Memorix dans la configuration active, la relance du client et la liaison explicite au projet. Si une copie en cache revient à l’ancienne adresse, retrouver et corriger aussi la source du plugin.
- **Remplacement ou retour arrière.** Relever la version et conserver le paquet précédent. Fermer les clients concernés, désactiver temporairement la tâche, puis identifier par PID et ligne de commande le processus Memorix qui écoute sur 3211. Arrêter ce processus et ses enfants Memorix identifiés, puis vérifier que le port est libre. Installer le paquet retenu. Pour revenir à la référence officielle de départ, utiliser `npm install --global memorix@1.10.4`. Contrôler le chemin du shim, réactiver la tâche, la lancer avec `schtasks /run`, puis refaire les contrôles de santé et de liaison. Préserver le dossier de données et les configurations de clients.
- **HYPOTHÈSES NON VÉRIFIÉES.** Le rôle d’ESET dans la lenteur de création des processus et l’origine des anciens hooks orphelins restent inconnus. Conserver les réglages de sécurité. Si un hook reste bloqué, relever son PID, son parent, sa ligne de commande et son heure de démarrage avant toute intervention.

La procédure installe une instance locale sur le PC cible. Traiter séparément tout transfert de données de mémoire existantes et de secrets.
