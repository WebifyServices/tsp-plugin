# Security Policy

## Reporting a vulnerability

Please do not open a public issue for security problems. Report them privately to **security@webifyservices.com** with enough detail to reproduce, and we'll acknowledge and work with you on a fix and disclosure timeline.

## Scope

This repository contains the installable client plugin: marketplace manifests and the seven skill behavior specs. It carries no server code and no credentials. The plugin connects to the hosted TSP MCP server and authenticates through OAuth 2.1 + PKCE in your browser; no static token is ever stored in the plugin.

Vulnerabilities in the hosted service itself (the MCP server, the API, the canvas) are in scope for the same contact address even though that code does not live in this repo.
