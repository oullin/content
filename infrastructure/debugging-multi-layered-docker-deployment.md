---
title: "Debugging a Multi-Layered Docker & Caddy Deployment"
excerpt: "Troubleshooting process for deploying a Go API with PostgreSQL on a VPS with Caddy and Docker."
slug: "2025-11-15-debugging-multi-layered-docker-deployment"
published_at: 2025-11-15
author: "gocanto"
categories: "infrastructure"
tags: ["infrastructure", "Docker", "Caddy", "PostgreSQL", "Go", "tls", "proxy"]
---
![reverse-proxy](https://github.com/user-attachments/assets/da7d8035-c151-42c3-9db8-1495e14a95d5)

## Debugging a Multi-Layered Docker + Caddy + mTLS Deployment
> **TL;DR**

There is a built “relay” path on the web tier that forwards signed requests to an internal **mTLS** gateway on the API tier. 
Along the way we hit:

- **421 Misdirected** Request loops caused by strict **SNI/Host** checks on the **mTLS** listener,
- a few **Caddy file** gotchas (handle_path ordering, named matcher scope), 
- a **CA** sync issue between web and **API** containers (client cert rejected),
- and a stray **404** from path rewriting.

The final config delivers 200s end-to-end with mTLS, and we added guards (**405** on GET) and **CI** validation.

## Glossary of Terms
| Term                                       | Definition                                                                                               |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------|
| **Makefile**                               | A file used by `make` to define automation rules for building and deployment.                            |
| **Docker Compose**                         | A tool for defining and running multi-container Docker applications.                                     |
| **SNI (Server name indication)**           | It is an addition to the TLS encryption protocol that enables a client device to specify the domain name |
| **Security Profile**                       | A set of rules in AppArmor that defines what actions a process can perform.                              |
| **TLS (Transport Layer Security)**         | Protocol for encrypting data between servers and clients.                                                |
| **mTLS** (Mutual Transport Layer Security) | mTLS is a security protocol that provides two-way authentication and encryption for network connections. |
| **CA** (A Certificate Authority)           | It a trusted entity that issues digital certificates to verify the identity of websites.                 |
| **Docker Secret**                          | A secure way to store sensitive data (like usernames or passwords) in Docker.                            |
| **Entrypoint Script**                      | The startup script that runs when a container is launched.                                               |
| **`security_opt`**                         | A Docker Compose option that modifies container security settings.                                       |
| **VPS (Virtual Private Server)**           | A virtualised server instance hosted by a provider.                                                      |
| **`docker exec`**                          | A Docker command used to run a command inside a running container.                                       |
