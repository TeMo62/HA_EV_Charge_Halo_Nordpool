# EV Charging Planner Assumptions

## Scope

This document contains items that are not verified design decisions in the current project model. The items are separated from verified implementation and design facts.

## Assumption: Configuration Helpers Are the Persistent Source of Truth

### Description

The project may follow a principle in which input helpers contain the authoritative configuration and runtime values are temporary derived state.

### Why It Is Uncertain

This principle appears in project history as a proposed principle. It is not explicitly established as a verified project decision in Phase 2.

### How It Can Be Verified

Confirm with the project owner whether configuration helpers are the intended authoritative source for settings and whether runtime entities are to be treated only as temporary state.

## Assumption: Preference for Simple YAML and Avoidance of Python and Node-RED

### Description

The project may prefer simple YAML-based Home Assistant configuration and avoid Python and Node-RED.

### Why It Is Uncertain

These statements appear as advice in project history. Phase 2 does not establish them as verified project decisions.

### How It Can Be Verified

Confirm with the project owner whether these are mandatory project constraints or historical implementation advice.
