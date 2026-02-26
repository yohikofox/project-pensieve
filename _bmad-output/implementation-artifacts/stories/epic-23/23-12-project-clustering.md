# Story 23.12 — Project Clustering

Status: backlog

## Story

As a **user with many captures**,
I want **Pensine to automatically group related captures into emerging projects**,
So that **I can see "8 captures about the truck this week" without manually organizing anything**.

## Context

Détection purement locale : similarité sémantique entre captures via embeddings LLM.
Aucune taxonomie prédéfinie — les projets émergent des patterns réels de l'utilisateur.

## Acceptance Criteria

### AC1 — Détection automatique de clusters
**Given** un historique de captures
**When** le clustering tourne en arrière-plan (expo-task-manager)
**Then** les captures sémantiquement proches sont regroupées en cluster nommé automatiquement par le LLM

### AC2 — Affichage dans le feed
**Given** un cluster détecté
**When** l'utilisateur ouvre le feed
**Then** une carte "Projet émergent : Camion (8 captures)" est affichée avec accès aux captures liées

### AC3 — Contrôle utilisateur
**Given** un cluster affiché
**When** l'utilisateur le consulte
**Then** il peut : confirmer le projet (le nommer), le rejeter, ou fusionner avec un projet existant

## References
- ADR-036 (local-first — embeddings on-device)
- ADR-019 (EventBus — events de clustering)
