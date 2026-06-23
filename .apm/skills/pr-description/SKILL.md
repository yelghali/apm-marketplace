---
name: pr-description
description: >-
  Activate when the user asks for a pull-request description, a summary of
  uncommitted or staged changes, or release notes. Use when preparing to open
  a PR or when the user says "draft a PR description for me".
---
# PR Description Skill

Produce a pull-request description with these sections, in order.

## Summary

One sentence: what changes and why. No file lists, no implementation detail.

## Motivation

Two to four sentences describing the problem this solves or the capability it
adds. Link to the issue or design doc if one exists.

## Changes

Bullet list grouped by area (e.g. "API", "Tests", "Docs"). One bullet per
logical change, not per file.

## Risk and rollback

Note any breaking changes, migrations required, or feature flags. Mention how
to revert if something breaks.

## Testing

How the change was verified: commands run, environments tested.
