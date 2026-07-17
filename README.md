# Server Bouncer

**ASA cluster login, identity, ban, and duplicate-survivor protection.**

Server Bouncer is a binary-only ASA Server API plugin for server owners who want one shared source of truth across an ARK: Survival Ascended cluster. It tracks players by EOS ID, blocks conflicting logins and duplicate survivors, supports cluster-wide bans, protects legitimate transfers, audits profile files, and sends useful Discord webhook events.

## What It Does

- Prevents the same EOS ID from being active on two servers at once.
- Optionally enforces one registered survivor per EOS ID across the cluster.
- Allows legitimate cluster transfers by creating and consuming transfer markers.
- Stores timed and permanent bans in MySQL so every server enforces them.
- Displays the configured ban or duplicate-character reason when ASA completes the kick.
- Supports manual allow-list entries and optional Permissions-plugin checks.
- Audits profile locations and reports duplicate profile files.
- Can quarantine duplicate profiles when explicitly enabled.
- Sends categorized Discord webhooks for commands, bans, kicks, disconnects, expirations, duplicate logins, and profile audits.

## Included Files

```text
ServerBouncer.dll
config.json
PluginInfo.json
README.md
LICENSE
```

This release does not include source code, project files, headers, or development binaries.

Current release: `1.1`

`ServerBouncer.dll` SHA-256:

```text
3280D1E7DC2F684761FB08E9CFA1AA02BBDA994CA4A8A1354D2BE815E0F03931
```

## Requirements

- Windows ARK: Survival Ascended dedicated server
- A compatible ASA Server API installation
- MySQL or MariaDB reachable by every protected server
- The same Server Bouncer version and database settings on every server
- The Permissions plugin only when using permission-group features

## Quick Start

### 1. Prepare MySQL

Create one database and account shared by the entire cluster. Run the following as a MySQL administrator after replacing the example password:

```sql
CREATE DATABASE server_bouncer CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'server_bouncer'@'%' IDENTIFIED BY 'replace_with_a_strong_password';
GRANT ALL PRIVILEGES ON server_bouncer.* TO 'server_bouncer'@'%';
FLUSH PRIVILEGES;
```

Use a more restrictive host than `%` when your database setup allows it. The database and user must exist before loading the plugin. Server Bouncer creates and updates its own tables.

### 2. Install the Plugin

Stop the ASA server. Create this directory:

```text
ShooterGame\Binaries\Win64\ArkApi\Plugins\ServerBouncer
```

Place these three files inside it:

```text
ServerBouncer.dll
config.json
PluginInfo.json
```

Do this on every server in the cluster. Do not run another copy of the plugin from an older folder name at the same time.

### 3. Configure the Database

Edit the `Mysql` section in `config.json`:

```json
"Mysql": {
  "MysqlHost": "127.0.0.1",
  "MysqlPort": 3306,
  "MysqlUser": "server_bouncer",
  "MysqlPass": "replace_with_a_strong_password",
  "MysqlDB": "server_bouncer"
}
```

Every server must point to the same database. Never publish a configured file containing your real database password or Discord webhook URLs.

### 4. Name Each Server

Set a different top-level `ServerName` on each server:

```json
"ServerName": "Cluster - The Island"
```

This name appears in kick reasons, database location records, RCON results, and Discord webhooks. If omitted, Server Bouncer uses the session name from `GameUserSettings.ini`, then falls back to the map name.

### 5. Start and Verify

Start the server and look for a line similar to:

```text
[ServerBouncer][info] ServerBouncer loaded
```

Then run this through RCON:

```text
ServerBouncer.Reload
```

A successful reply confirms that the plugin can find and parse its configuration.

## Recommended First Test

1. Install Server Bouncer on two test servers using the same MySQL database.
2. Give each server a unique `ServerName`.
3. Join server one with a survivor and disconnect normally.
4. Run `ServerBouncer.LastSeen <eosid>` and verify the recorded server.
5. Join server one, then attempt to connect to server two with the same account. The second login should be rejected as already active.
6. Perform a legitimate in-game cluster transfer from server one to server two. The transfer marker should allow the destination login and then be consumed.
7. Test a short ban, verify its kick message, then remove it with `ServerBouncer.UnBan <eosid>`.
8. Configure a Discord webhook and verify command, ban, and kick events.

Use test servers and backups before enabling profile quarantine.

## Configuration Guide

`config.json` must contain valid JSON. JSON comments are not supported. After editing it, restart the server or run `ServerBouncer.Reload`.

### Duplicate Login and Survivor Protection

```json
"BlockMultipleCharacters": true
```

When enabled, the first accepted survivor becomes the registered survivor for that EOS ID. Server Bouncer stores its player-data ID, character name, level, and last server. A conflicting survivor is kicked unless a valid transfer marker proves that the player is moving between servers.

The active-session check is independent of duplicate-survivor enforcement. It prevents the same EOS ID from remaining logged into multiple protected servers.

### Transfers

Server Bouncer automatically marks transfers when ASA uploads the current survivor and inventory through the normal transfer flow. The destination server consumes the marker after accepting the matching survivor.

```json
"TransferGraceSeconds": 0
```

- `0`: the marker does not expire; it remains until successfully consumed.
- A positive value: the marker expires after that many seconds.

An RCON operator can also create a marker manually with `ServerBouncer.MarkTransfer` when testing or recovering from a failed transfer.

### Bans

Bans are stored in MySQL and apply to every connected server. Ban duration is entered in hours. Use `0` for a permanent ban.

The default player-facing template supports these placeholders:

- `{Reason}`
- `{date banned}`
- `{Expiration}`

### Discord Webhooks

Use `Webhook` as a fallback URL, or assign separate URLs under `Webhooks`:

```json
"Webhook": "",
"Webhooks": {
  "Commands": "",
  "Bans": "",
  "Disconnections": "",
  "Kicks": "",
  "BanExpirations": "",
  "DupeCharactersAndLogins": "",
  "ProfileAudit": ""
}
```

If a category is blank, Server Bouncer uses the fallback `Webhook`. Leave both blank to disable that category.

### Exclusive Join

Exclusive-join enforcement runs only when the ASA server is started with:

```text
-exclusivejoin
```

Manual allow-list entries use:

```text
ServerBouncer.ExclusiveJoin <eosid>
ServerBouncer.UnExclusiveJoin <eosid>
```

Permission-group access is optional:

```json
"ExclusiveJoinPermissions": {
  "Enabled": true,
  "Groups": ["WhiteListGroup", "Admins"]
}
```

The groups are checked only when the Permissions plugin is loaded. Manual EOS entries continue to work without it.

### Permission-Gated Servers or Maps

```json
"JoinRequiresPermission": {
  "Enabled": false,
  "Maps": {
    "TheIsland_WP": "",
    "premium-map": "Admins,VIP"
  }
}
```

Start a server with `-serverkey=premium-map` to use a stable key instead of its technical map name. A comma-separated value allows any listed permission.

### Profile Audit

Profile audit scans configured server save folders and shared cluster-transfer folders for duplicate EOS profile locations.

The safe default is report-only:

```json
"Enabled": false,
"QuarantineEnabled": false,
"QuarantineOnScheduledAudit": false,
"QuarantineOnDuplicateProfileBlock": false
```

Configure paths with forward slashes:

```json
"Servers": [
  { "Name": "Island", "RootPath": "C:/ARK/Server1" },
  { "Name": "Center", "RootPath": "D:/ARK/Server2" }
],
"ClusterPaths": [
  { "Name": "Shared Cluster", "Path": "C:/ARK/SharedCluster/clusters/mycluster" }
]
```

`RootPath` points to the server root. You may use `SavedArksPath` instead when you want to point directly at a `ShooterGame/Saved/SavedArks` directory.

Important behavior:

- `QuarantineEnabled=false` never moves profile files.
- `ScanProfiles` always performs a report.
- `QuarantineProfiles` moves selected duplicates only when quarantine is enabled.
- `QuarantinePreferRegisteredProfile=true` prefers the survivor registered in MySQL.
- `ProtectActiveClusterTransfers=true` avoids quarantining an active transfer file.
- Backups go to `QuarantineBackupPath` or `ServerBouncerProfileBackups` beneath the affected save location.
- Keep current backups before enabling automated quarantine.

The plugin identifies profile ownership from filenames. It does not derive a survivor's level by parsing raw `.arkprofile` data.

### Kick Timing

The `Advanced` section controls warning and kick timing. Recommended defaults are already provided.

- `PreKickWarningEnabled`: displays an in-game warning before the native kick.
- `RequireResolvedPlayerDataForKick`: waits for ASA to resolve the survivor before login-time enforcement.
- `StabilizeLoginKicksBeforeNativeKick`: allows the connection to settle before kicking.
- `LoginKickStabilizeSeconds`: settle time for login-time kicks.
- `ConnectedBanKickDelaySeconds`: delay when banning a player already in game.
- `BanLoginKickDelaySeconds`: delay when a banned player connects.
- `DuplicateCharacterKickDelaySeconds`: delay before kicking a resolved duplicate survivor.
- `UnresolvedDuplicateCharacterKickDelaySeconds`: delay when the incoming player-data ID is unresolved.
- `BanWarningRepeatSeconds` and `DuplicateCharacterWarningRepeatSeconds`: warning repeat intervals.

Leave `AggressiveKickMessageDebug=false` during normal operation.

## RCON Commands

| Command | Purpose |
| --- | --- |
| `ServerBouncer.Reload` | Reload `config.json` without restarting. |
| `ServerBouncer.LastSeen <eosid>` | Show the registered survivor's last server. |
| `ServerBouncer.Ban <eosid> <hours> <reason>` | Create or replace a cluster-wide ban. Use `0` hours for permanent. |
| `ServerBouncer.UnBan <eosid>` | Remove a ban. |
| `ServerBouncer.ExclusiveJoin <eosid>` | Add a manual exclusive-join entry. |
| `ServerBouncer.UnExclusiveJoin <eosid>` | Remove a manual exclusive-join entry. |
| `ServerBouncer.MarkTransfer <eosid> [minutes]` | Create a transfer marker. Use `0` minutes for no expiration. |
| `ServerBouncer.ScanProfiles` | Run an immediate read-only duplicate-profile report. |
| `ServerBouncer.QuarantineProfiles` | Run a scan and quarantine selected duplicates only when enabled. |

Examples:

```text
ServerBouncer.Ban 00000000000000000000000000000000 4 repeated rule violations
ServerBouncer.Ban 00000000000000000000000000000000 0 permanent ban reason
ServerBouncer.UnBan 00000000000000000000000000000000
ServerBouncer.MarkTransfer 00000000000000000000000000000000 15
ServerBouncer.ScanProfiles
```

`ServerBouncer.Reload` is also registered as a server-console command.

## Updating or Renaming an Existing Installation

1. Back up the current plugin folder and database.
2. Stop every protected server.
3. Create the new `ArkApi\Plugins\ServerBouncer` folder.
4. Copy in the new DLL and metadata.
5. Move your existing settings into the new `config.json` and preserve the current MySQL connection values.
6. Remove or disable the previous plugin folder so only one copy loads.
7. Start one server first and confirm successful database connection and config load.
8. Start the remaining servers and test RCON, login, transfer, and ban behavior.

Existing database table names are retained internally for upgrade compatibility. Your registered survivors, bans, transfer markers, and active-session data remain available when the same database is used.

## Troubleshooting

### Plugin does not load

- Confirm `ServerBouncer.dll` is inside `ArkApi\Plugins\ServerBouncer`.
- Confirm ASA Server API itself loads successfully.
- Check the server console and `ArkApi\Logs` for `ServerBouncer` errors.
- Confirm the installed ASA Server API version is compatible with the release.

### Config cannot be opened

- Confirm the file is named exactly `config.json`.
- Confirm it is in the same `ServerBouncer` folder as the DLL.
- Validate the JSON and remove comments or trailing commas.

### MySQL connection fails

- Confirm host, port, username, password, and database name.
- Confirm the MySQL user can connect from the server machine.
- Confirm Windows Firewall and database bind settings allow the connection.
- Confirm every server points at the same database.

### Players are incorrectly reported as active

- Confirm all servers have unique `ServerName` values.
- Confirm clocks are reasonably synchronized.
- Review `SessionTimeoutSeconds` and `HeartbeatSeconds`.
- Make sure an old plugin copy is not loading from another folder.

### Legitimate transfers are blocked

- Confirm both servers share the same database.
- Confirm the source server loaded Server Bouncer before the upload occurred.
- Check the transfer webhook and logs for marker creation and consumption.
- Use `ServerBouncer.MarkTransfer` only as a controlled recovery or test tool.

### Webhooks do not arrive

- Confirm the URL is complete and has not been revoked.
- Check the category URL and fallback `Webhook` value.
- Confirm the server can make outbound HTTPS connections.

## Security Notes

- Treat `config.json` as a secret after adding database credentials or webhooks.
- Use a dedicated MySQL user rather than a root account.
- Back up the database and player saves before major updates.
- Test quarantine and enforcement changes outside production first.

## Ownership and Support

Server Bouncer remains the property of Michael Henderson / Safe Mode Software. Downloading or using the plugin does not transfer ownership. See `LICENSE` for the permitted use and restrictions.
