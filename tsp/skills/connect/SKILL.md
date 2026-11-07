---
name: connect
description: Connect this computer to TSP Build in one command by running the published TSP gateway's `connect` verb with npx. It pairs the computer after your browser consent, registers the current repository and starts the gateway login item. Run only when the user types /tsp:connect.
disable-model-invocation: true
argument-hint: "[folder]"
---

# tsp:connect

Sets up the TSP gateway on this computer so the TSP canvas can launch Claude Code and
Codex sessions here. The user runs this command; never start it on your own initiative.

## What to run

Run exactly this one command, in the session's current working directory (the
user's repository), with the user's folder argument appended only if they gave one:

```sh
npx -y @treestructuredplanning/gateway@0.1.2 connect $ARGUMENTS
```

The version is pinned so the code that runs is the published release this plugin
was built against. Do not change the version, add flags, chain other commands, or run
it in the background: pairing waits for the user in the browser.

Before running it, tell the user in two or three sentences what it does:

- It opens their browser to approve pairing with their TSP account, unless this
  computer is already paired. Nothing pairs without that approval.
- It registers this repository folder with the gateway.
- It installs a login item (a LaunchAgent on macOS, a user systemd unit on Linux, a
  Task Scheduler logon task on Windows) that starts the gateway at login. The gateway
  makes one outbound connection to TSP and accepts no inbound connections.

It writes the login item, the gateway config file
(`~/.config/tsp-gateway/config.json`, or `%APPDATA%\tsp-gateway` on Windows) and one
OS credential-store entry (`com.treestructuredplanning.gateway`). It is safe to run
again: an existing pairing, folder and login item are kept as they are.

## What never to do

- Never change the user's Claude Code permission settings, allowed tools, or
  `settings.json`, and never pass a permission-bypassing flag. Sessions TSP launches
  use the user's own defaults. Let the user approve the shell command through their
  normal permission prompt.
- Never approve the browser consent, sign in, or accept a trust prompt for the user.

## After it runs

Relay the command's summary lines. If it prints remaining steps, tell the user each
one plainly, as printed: each names its folder and what to do in it, such as signing
in, or running `claude` or `codex` once there and accepting its trust prompt. Only they
can do that. If the command fails, show its error message. A folder that is not a GitHub
clone, or a TSP it cannot reach, stops it before anything changes, and running it
again is always safe.

To undo everything, the user runs:

```sh
npx -y @treestructuredplanning/gateway@0.1.2 disconnect
```

That removes the login item and revokes the pairing in TSP. The full behaviour is
documented on the gateway's npm page,
https://www.npmjs.com/package/@treestructuredplanning/gateway.
