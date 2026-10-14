# test-manager-library-3145555-docs

> **Technical test · Sheet ADSO 3145555**
> Schedule: **Wednesday 14 October 2026, from 6:50 a. m. to 11:00 a. m.** (Colombia time). Delivery at **11:00 a. m.**

This repository contains the **spec** for *Manager Library*. It is the only one with content: the other five start empty.

## Instructions

Every trainee must **fork the six project repositories** and **solve the project against the stated spec**.

1. Fork, into their own GitHub account, each repository from the table at the end.
2. Read the spec in [`test-manager-library-3145555-docs`](https://github.com/code-sena/test-manager-library-3145555-docs).
   It is delivered in two versions: `spec-python/` and `spec-.net/`.
3. Develop on the forks.

## The challenge is built with React and PHP (Laravel)

The spec is written for Python and for .NET, but the challenge is **not** done in those languages:

| Layer | Technology of the challenge |
|---|---|
| Frontend | React |
| Backend | PHP with Laravel |

What the spec defines about the business — stories, acceptance criteria, rules, API contract, data model — is respected. What it defines about technology is translated to React and Laravel.

## The test is not about writing code

The main purpose is to assess **performance with SDD** (*Spec-Driven Development*): how a specification is read, interpreted and applied to carry it over to a different stack. The code is the means, not the end.

## Delivery

What counts is **what is pushed to the forks before 11:00 a. m.** Anything that arrives later is not reviewed.
There is no extension: push partial progress with small commits during the test.

## What this repository contains

| Folder | Content |
|---|---|
| [`spec-python/`](spec-python/) | The Manager Library spec written for a Python backend |
| [`spec-.net/`](spec-.net/) | The same system specified for a .NET backend |

The two versions describe the **same product** (the same stories, business rules and endpoints); they differ in the technology decisions. Neither of the two is the stack of the challenge.

## How to read the spec

In either of the two folders, in this order:

1. `constitution.md` — non-negotiable principles.
2. `spec.md` — what the system must do: actors, stories, acceptance criteria, business rules.
3. `plan.md`, `architecture.md` and `arquitectura-panoramica.md` — how it is built.
4. `data-model.md` and `api-contract.md` — the data model and the API contract.
5. `tasks.md` — what has to be done and with which evidence it is taken as done.
6. `adr/` — the architecture decisions, with their alternatives and consequences.

## The six repositories

| Repository | What goes there |
|---|---|
| [`test-manager-library-3145555-docs`](https://github.com/code-sena/test-manager-library-3145555-docs) | The spec: `spec-python/` and `spec-.net/` |
| [`test-manager-library-3145555-api`](https://github.com/code-sena/test-manager-library-3145555-api) | Backend in PHP (Laravel) |
| [`test-manager-library-3145555-app`](https://github.com/code-sena/test-manager-library-3145555-app) | Frontend in React |
| [`test-manager-library-3145555-page`](https://github.com/code-sena/test-manager-library-3145555-page) | Static public presentation site |
| [`test-manager-library-3145555-infra`](https://github.com/code-sena/test-manager-library-3145555-infra) | Containers, network, volumes and an empty database engine |
| [`test-manager-library-3145555-tool`](https://github.com/code-sena/test-manager-library-3145555-tool) | Utilities: demo data seeder |