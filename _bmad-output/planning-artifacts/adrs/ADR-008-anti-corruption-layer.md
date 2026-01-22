---
adr: ADR-008
title: "Anti-Corruption Layer (ACL) à la Frontière Mobile/Backend"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-008: Anti-Corruption Layer (ACL) à la Frontière Mobile/Backend

**Status:** ✅ ACCEPTÉ

**Context:** Déterminer où les ACL sont nécessaires dans l'architecture Pensine.

**Decision:** ACL obligatoire à la frontière mobile ↔ backend (API Sync Layer), mais pas entre bounded contexts backend.

---

## Rationale

### Pas d'ACL entre Bounded Contexts Backend (Internes)

- Contextes métier internes avec contrôle total
- Ubiquitous language cohérent et partagé
- Communication via Domain Events bien définis
- Aucun risque de corruption de modèle

### ACL OBLIGATOIRE à la frontière Mobile ↔ Backend

La structure des données mobile et backend PEUT diverger volontairement:

```typescript
// MOBILE (OP-SQLite - structure dénormalisée pour performance)
interface Capture {
  id: string;
  type: string;
  raw_content: string;  // JSON stringifié
  normalized_text: string;
  tags: string;  // JSON array stringifié
  metadata: string;  // JSON object stringifié
}

// BACKEND (DDD - structure normalisée)
class Capture {
  id: UUID
  type: CaptureType  // Value Object
  rawContent: AudioFile | TextContent | ImageFile  // Union type
  normalizedText?: NormalizedText  // Value Object
  tags: Tag[]  // Collection de Value Objects
  metadata: CaptureMetadata  // Value Object structuré
}
```

⚠️ **UPDATE (2026-01-22):** Structure mobile mise à jour pour OP-SQLite. Voir [ADR-018](./ADR-018-migration-watermelondb-opsqlite.md)

---

## L'API Sync Layer joue le rôle d'ACL et DOIT

### 1. VALIDATION (Protection Domaine)

```typescript
// Rejeter données invalides avant qu'elles n'atteignent le domaine
POST /sync/push
{
  changes: {
    captures: {
      created: [{
        id: "...",
        type: "audio",
        raw_content: "invalid_json"  // ❌ REJETÉ par ACL
      }]
    }
  }
}
→ 400 Bad Request (domaine backend jamais pollué)
```

### 2. TRANSFORMATION (Structure Mobile ↔ Backend)

```typescript
// Mobile → Backend (Pull response)
{
  raw_content: '{"audioUri": "file://..."}',  // JSON string
  tags: '["business", "idea"]'                 // JSON array string
}

// ACL transforme vers structure backend:
{
  rawContent: new AudioFile(uri),              // Value Object
  tags: [new Tag("business"), new Tag("idea")] // Collection VOs
}

// Backend → Mobile (Push)
{
  rawContent: audioFile.toJSON(),
  tags: tags.map(t => t.value)
}
→ Mobile reçoit format dénormalisé attendu
```

### 3. TRADUCTION VOCABULARY

```typescript
// Mobile utilise vocabulaire technique:
raw_content, normalized_text, audio_file_path

// Backend utilise vocabulaire métier:
rawContent, normalizedText, audioFile

// ACL traduit bidirectionnellement
```

---

## Avantages de cette approche

- **Mobile optimisé:** Structure plate, JSON strings pour performance, moins de joins
- **Backend protégé:** Validation stricte, types forts, Value Objects
- **Flexibilité:** Changer structure mobile sans impacter backend (et vice-versa)
- **Évolution indépendante:** Mobile et backend peuvent évoluer à des rythmes différents

---

## Consequences

**Requirements:**
- API Sync doit implémenter transformations bidirectionnelles
- Tests d'intégration critiques pour valider ACL
- Documentation des mappings mobile ↔ backend obligatoire

**Trade-offs acceptés:**
- Coût CPU léger pour transformations (acceptable pour MVP)
- Complexité ACL à maintenir (mais isolation précieuse)

**Impact:**
- Sync Layer = ACL implémenté dans Epic 6
- Schema mapping documenté dans architecture

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
