# Pensine Backend - Roadmap

## Future Improvements

### RGPD Compliance - Data Anonymization vs Deletion

**Current Implementation (MVP)**:
- Story 1.3 v2 implements full account deletion (user + audit logs)
- Compliant with RGPD Article 17 (right to erasure)

**Future Enhancement**:
- **Data Anonymization Strategy** instead of full deletion
- **Rationale**:
  - Enable statistics and analytics while respecting RGPD (Article 89)
  - Legal obligations for audio recordings platform (Article 17.3.b)
  - Judicial evidence preservation for potential investigations
  - Audit trail retention for legal compliance (3-5 years)

**Implementation Plan**:
1. **User Data Anonymization**:
   - Replace PII (email, name, etc.) with anonymized identifiers (`deleted_user_<hash>`)
   - Keep user ID for relational integrity
   - Flag account as `anonymized` instead of `deleted`

2. **Audit Logs Retention**:
   - Keep audit logs for legal obligations (3-5 years)
   - Remove CASCADE deletion on audit_logs
   - Link to anonymized user records

3. **Audio Files**:
   - Metadata retention for legal purposes
   - Encryption of sensitive content
   - Anonymized linkage to user records

4. **Statistics**:
   - Aggregated anonymized data for platform analytics
   - No personally identifiable information (PII)
   - Compliance with RGPD research exemptions

**Priority**: Post-MVP (before production launch with audio features)

**Legal Review Required**: Yes - consult with legal team on retention periods and anonymization strategy

---

## Other Future Tasks

_(Add future improvements and features here as they are identified)_
