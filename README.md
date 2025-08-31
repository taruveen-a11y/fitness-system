# Fittoria (v1.0)

## 1. Purpose & Scope
This document defines the end‑to‑end lifecycle, roles, states, data artifacts, automations, metrics, and phased roadmap for a platform coordinating medically-informed fitness programming between Patient, Doctor, Trainer, and Lab Technician. It supports safe exercise prescription, adherence tracking, clinical oversight, and progressive optimization.

---

## 2. Core Roles & High-Level Responsibilities

| Role | Key Responsibilities | Access Summary (MVP) |
|------|----------------------|----------------------|
| Patient (P) | Create account, complete HealthHistory, upload prior reports, grant consents, perform sessions, log adherence, review plans, act on lab orders | Own data (except doctor private notes) |
| Doctor (D) | Review history & reports, perform DoctorAssessment, set Restrictions, set Clearance, refer to Trainer, order labs, produce summaries, modify restrictions | Full clinical data, progress aggregates |
| Trainer (T) | Accept referral, conduct TrainingIntake, build TrainingPlan within Restrictions, deliver sessions, adjust sessions, produce ProgressSnapshots, flag concerns | Restrictions, clearance status, adherence aggregates (+ limited history if consent) |
| Lab Technician (L) | Fulfill LabOrders, upload LabResults | LabOrder minimal identifiers |
| System (S) | Orchestrate workflow, enforce permissions & restrictions, generate Focus Cards, maintain audit/version history, trigger notifications, validate plans | N/A (governing layer) |
| (Future) Nutritionist / Mental Health Coach | Parallel domain-specific support under consent | Scoped per future consent categories |

---

## 3. Lifecycle Phases Overview

| Phase | Name | Primary Objective | Key Artifacts |
|-------|------|-------------------|---------------|
| 0 | Patient Onboarding & Existing Report Collection | Capture baseline history & external documents | HealthHistory, ExternalReports |
| 1 | Doctor Clinical Assessment & Restrictions | Establish medical clearance & constraints | DoctorAssessment, Restrictions, ClearanceStatus |
| 2 | Trainer Intake & Program Design | Translate restrictions into actionable training plan | TrainingIntake, TrainingPlan (v1) |
| 3 | Training Delivery & Progress Tracking | Execute sessions & capture adherence + safety signals | SessionRecord, ProgressSnapshot |
| 4 | Monitoring & Re‑assessment (Labs & Summaries) | Update clinical context and modify plan/restrictions | LabOrder, LabResult, DoctorSummary |
| 5 | Ongoing Optimization / Graduation | Longitudinal optimization or program closure | OutcomeReport, Updated Restrictions, CaseState changes |

---

## 4. Detailed Phase Flows

### Phase 0: Onboarding & Report Collection
0.1 P: Create account & complete HealthHistory (conditions, medications, injuries)  
0.2 P: Upload existing medical reports (PDF/images)  
0.3 S: (Optional) OCR + metadata extraction (tag: lab / imaging / physician note)  
0.4 P: Grant consent: Doctor (required), Trainer (optional pre‑clearance)  
0.5 S: Case Status → "Awaiting Doctor Review"  

### Phase 1: Doctor Assessment & Clearance
1.1 D: Review HealthHistory + ExternalReports  
1.2 D: Create DoctorAssessment (structured: primary issues, risk factors, baseline metrics)  
1.3 D: Define Restrictions (e.g., “No high‑impact plyo 4 wks”, “Max HR < 160”, “Avoid overhead pressing left arm”)  
1.4 D: Set Clearance Status: Pending → Conditionally Cleared → Cleared  
1.5 D: Create Referral to Trainer (links Restrictions object)  
1.6 S: Notify P & T; Focus Card: “Complete Trainer Intake”  
1.7 (Optional) D: Order baseline LabOrder(s) (A1c, Lipids, etc.)  

### Phase 2: Trainer Intake & Program Design
2.1 T: Accept referral (timestamp logged)  
2.2 T: View restricted HealthHistory fields (if consent granted)  
2.3 T: Conduct TrainingIntakeForm (movement screen, fitness level, schedule, equipment, preferences)  
2.4 S: Validate proposed modalities vs active Restrictions (hard fail or warning)  
2.5 T: Build TrainingPlan (macro structure + Week 1 schedule)  
2.6 S: Publish plan version v1 → sessions surfaced to Patient  
2.7 P: Review & Accept (or Request Changes → revision loop)  
2.8 CaseState: "Planning" → "ActiveTraining" upon acceptance  

### Phase 3: Training Delivery & Daily Loop
3.1 S (Daily): Generate Focus Card (e.g., “Complete Session A (Low Impact)”, “Hydration Habit”)  
3.2 P: Execute session; log sets/reps / completion / RPE  
3.3 T: (Optional) Live adjustments (store modifications + reason)  
3.4 S: Detect restriction violations (e.g., HR > cap) → flag  
3.5 T: Weekly ProgressSnapshot (adherence %, volume trends, notes on constraints)  
3.6 S: If low adherence or anomalies → suggest Doctor Review  

### Phase 4: Lab & Reassessment Cycle
4.1 D: Place LabOrder (reason: plateau, symptom, scheduled reassessment)  
4.2 S: Notify P with instructions  
4.3 L: Collect specimen, upload LabResults → status Completed  
4.4 S: Normalize units, flag out-of-range (disclaimer: informational only)  
4.5 D: Review results, create/append DoctorSummary (update Restrictions if needed)  
4.6 If Restrictions change → S: Notify P & T; plan revision prompt  
4.7 T: Adjust TrainingPlan (new version v2+)  
4.8 S: Maintain version diff & audit trail  

### Phase 5: Ongoing Optimization / Completion
5.1 S: Quarterly (configurable) OutcomeReport (vs baseline metrics)  
5.2 D: Reassessment; may graduate (clear restrictions)  
5.3 T: Transition plan to maintenance or performance focus  
5.4 P: Continue subscription or exit (archival)  
5.5 CaseState: "ActiveTraining" → "Maintenance" or "Closed"  

---

## 5. State Machine

Primary CaseState values:
- IntakePending
- DoctorReview
- AwaitingTrainerIntake
- Planning
- ActiveTraining
- PendingLabResults (substate flag layered on ActiveTraining)
- Reassessment
- Maintenance
- Closed

Allowed Transitions (non-exhaustive):
- IntakePending → DoctorReview (health history submitted)
- DoctorReview → AwaitingTrainerIntake (doctor clearance + referral issued)
- AwaitingTrainerIntake → Planning (trainer accepts)
- Planning → ActiveTraining (patient approves plan)
- ActiveTraining → PendingLabResults (lab(s) ordered & outstanding)
- PendingLabResults → ActiveTraining (labs completed; no plan change)
- PendingLabResults → Reassessment (doctor summary prompts revision)
- Reassessment → ActiveTraining OR → Maintenance
- Any → Closed (graduation or voluntary exit)

Guardrails:
- No TrainingPlan publishing before ClearanceStatus ∈ {Conditionally Cleared, Cleared}
- Reassessment required upon severe violation or adverse event before returning to ActiveTraining

---

## 6. Permissions & Guardrails (MVP)

| Data Artifact | Doctor | Trainer | Patient | Lab Tech | Notes |
|---------------|--------|---------|---------|----------|-------|
| HealthHistory (full) | R/W | R* (consent) | R/W | – | Trainer sees limited subset only |
| ExternalReports | R | R* (consent) | R/W | – | |
| DoctorAssessment | R/W | R (summary only) | R (shared fields) | – | Private vs shared note segmentation |
| Restrictions | R/W | R | R | – | Versioned, enforced |
| TrainingIntake | R | R/W | R (summary) | – | |
| TrainingPlan Versions | R | R/W | R | – | Published versions immutable |
| SessionRecords | Aggregate R | R/W | R/W (own) | – | Doctor sees aggregates (not raw per-set unless escalated) |
| ProgressSnapshots | R | R/W | R (high-level) | – | |
| LabOrders | R/W | Status (basic) | R | R/W (fulfill) | Trainer excludes result values |
| LabResults | R/W | – | R (interpretable view) | W (upload) | Optional patient-level obfuscation of reference ranges disclaimers |
| DoctorSummary | R/W | R (actionable deltas) | R (shared) | – | |
| OutcomeReport | R/W | R | R | – | |
| Audit Logs | R (admin) | – | – | – | Internal governance |

System-Level Enforcement:
- Plan validation rejects any exercise modality flagged as disallowed by active RestrictionVersion.
- Data access requests logged with timestamp & principal.

---

## 7. Automation & Intelligence Roadmap

| Phase | Automations |
|-------|-------------|
| MVP | Plan validation (restriction compliance); Focus Card aggregation (sessions, pending labs, active restrictions); Basic adherence % computation |
| Phase 2 | Adaptive load recommendations (RPE + restriction boundaries); Early warning: “Adherence drop 2 weeks + outstanding lab” |
| Phase 3 | Condition-specific plan modifiers (e.g., PCOS metabolic emphasis); AI doctor summarizer (“12 sessions, avg RPE 6.2, adherence 80%, weight −1.2%”); Advanced analytics dashboards |

---

## 8. Progress & Analytics Metrics

1. Funnel: account_created → health_history_submitted → doctor_assessment_completed → trainer_intake_completed → training_plan_published  
2. Cycle Times:  
   - T1: HealthHistory → DoctorAssessment (<48h goal)  
   - T2: Referral → Trainer Acceptance (<24h)  
   - T3: Trainer Acceptance → First Session Delivered (<72h)  
3. Engagement: sessions_per_week, adherence %, habit completion rate, plan_revision_frequency  
4. Clinical Coordination: restrictions_active_count, lab_order_to_result_time, restriction_violation_rate  
5. Outcome Proxies: session volume growth (total lifted kg), RPE trend stabilization, circumference / weight trend, metabolic markers (Phase 2+)  
6. Safety: violation_alerts_per_100_sessions (target <2), unauthorized_access_attempts (target 0)  

---

## 9. Exceptions / Edge Case Handling

| Code | Scenario | Trigger | System Action |
|------|----------|---------|---------------|
| A | Doctor Assessment Delay | >X days after HealthHistory | Reminder → escalate to admin |
| B | Trainer Rejects Referral | Referral rejection | Case → DoctorReview; capture reason |
| C | Restriction Violation | Real-time metric breach | Flag session; if severe → Doctor notification & potential plan suspension |
| D | Lab Results Delayed | LabOrder age > threshold | Patient reminder; second reminder → doctor escalation |
| E | Consent Revoked | Patient revokes trainer consent | Trainer loses detailed history; prompts patient re-consent or doctor minimal update pathway |
| F | Mid-Session Adverse Event | Trainer logs event | Immediate doctor alert; auto-suspend plan until new clearance |
| G | Inactivity | No sessions >14 days | Mark “At Risk”; notify trainer (+ doctor if clinical flags present) |

---

## 10. Key Data Model Relationships (Conceptual)

- SessionRecord → TrainingPlanVersionId + RestrictionVersionId (ensures historical context)
- TrainingPlanVersion (immutable content + metadata)  
- RestrictionVersion (change log for safety reasoning)  
- DoctorSummary → snapshot hash of LabResults set used  
- LabOrder (status: Ordered | Collected | Completed | Canceled)  
- LabResult (normalized values, reference ranges, flags)  

Versioning Rationale:
Preserves justification trail (e.g., “Exercise allowed because RestrictionVersion v3 removed overhead limitation”).

---

## 11. Notifications (Event → Recipient → Template)

| Event Key | Recipient(s) | Message (Example) |
|-----------|--------------|-------------------|
| doctor.assessment.created | Patient | “Your doctor set your fitness clearance.” |
| referral.created | Trainer | “New patient referral awaiting acceptance.” |
| referral.accepted | Patient | “Trainer accepted. Complete intake questionnaire.” |
| training_plan.version.published | Patient | “Your training plan (Week 1) is ready.” |
| lab.order.created | Patient | “Lab tests ordered—please schedule.” |
| lab.result.received | Doctor | “New lab results available for review.” |
| restrictions.updated | Trainer + Patient | “Updated safety guidelines—review before next session.” |
| violation_alert.flag | Trainer (+ Doctor if severe) | “Restriction exceeded during Session A.” |

Delivery Channels (MVP): In-app notification center + email fallback (configurable).

---

## 12. MVP vs Phased Enhancements

| Layer | MVP | Phase 2 | Phase 3 |
|-------|-----|---------|---------|
| Clinical Docs | Report upload (manual) | Structured LabResult ingestion | AI summarization |
| Training Plan | Static validation | Adaptive suggestions | Condition-specific templates |
| Safety | Basic restriction enforcement | Automated violation detection (RPE/HR) | Predictive risk scoring |
| Analytics | Adherence + volume summaries | Lab trend integrations | Advanced dashboards & cohort analytics |
| Focus Card | Sessions + labs + restrictions | Early warnings & churn risk | Personalized optimization insights |
| Consent | Binary per role | Granular categories (labs, notes) | Dynamic consent negotiation prompts |

---

## 13. Sample Patient Timeline (Illustrative)

| Day | Event |
|-----|-------|
| 0 | Account + HealthHistory + upload prior A1c report |
| 1 | Doctor review & clearance with “No HIIT 2 weeks” restriction → referral to trainer |
| 1 (later) | Trainer accepts; intake scheduled |
| 2 | Trainer Intake + Plan v1 published (3 sessions + 2 low-intensity walks) |
| 3–9 | Patient completes 5/6 scheduled actions (adherence 83%) |
| 10 | Doctor orders follow-up A1c labs |
| 12 | Labs collected |
| 14 | Lab results → DoctorSummary relaxes HIIT restriction → Plan v2 updated |

---

## 14. Governance & Compliance

Principles:
- Role-Based Access + explicit Patient consent gating sensitive fields.
- Separation of clinical vs fitness schemas (minimize PHI in non-clinical tables).
- Immutable audit logs: view + modify events for DoctorAssessment, Restrictions, LabResults.
- Disclaimers: “Informational only—consult your physician for medical decisions.”

Audit Log Fields (minimum):
- actor_id, role, action_type, resource_type, resource_id, timestamp (UTC), previous_hash (chain), reason (optional), diff_summary.

---

## 15. Backlog Epics (Derived)

1. Health History & Report Upload  
2. Doctor Assessment & Restrictions  
3. Referral & Acceptance Flow  
4. Trainer Intake & Plan Authoring  
5. Session Logging & Adherence Metrics  
6. Lab Order Metadata (Phase 1)  
7. Lab Result Ingestion (Phase 2)  
8. Restriction Enforcement & Violation Flags  
9. Focus Card Integration with Clinical Context  
10. Notifications & Escalation Rules  
11. Consent & Audit Framework  
12. Progress Snapshot & Doctor Summary  
13. Outcome Reporting & Version History  

---

## 16. Risk & Mitigation (Selected)

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Delayed doctor assessments | Onboarding stalls | SLA reminders + escalation workflow |
| Incomplete restrictions clarity | Safety gaps | Structured restriction taxonomy (movement category, intensity, duration, conditional notes) |
| Over-permission creep | Privacy compliance risk | Consent matrix + periodic access review |
| Version confusion | Misaligned training decisions | Immutable versioning + clear “active” badges |
| False violation alerts | Alert fatigue | Threshold tuning + severity tiers |

---

## 17. Glossary

| Term | Definition |
|------|------------|
| HealthHistory | Structured patient-entered clinical & injury baseline data |
| DoctorAssessment | Doctor-authored structured evaluation object |
| Restrictions | Active constraint set (movement, intensity, modality, duration) |
| Clearance Status | Enum controlling whether training may proceed |
| TrainingIntake | Trainer-collected functional & contextual inputs |
| TrainingPlan Version | Immutable structured weekly schedule snapshot |
| SessionRecord | Executed or planned session instance with outcomes |
| ProgressSnapshot | Weekly roll-up of adherence & key metrics |
| LabOrder / LabResult | Ordered diagnostic test request and its resulting data |
| DoctorSummary | Periodic clinical narrative + actionable changes |
| OutcomeReport | Longitudinal comparison vs baseline metrics |
| Focus Card | Context-aware daily prioritized action item |
| CaseState | Lifecycle stage of patient program |

---

## 18. Implementation Priorities (First Pass Ordering)

1. Core Data Models & Auth (Users, Roles, Consent, CaseState)  
2. HealthHistory + Report Upload  
3. DoctorAssessment + Restrictions + Clearance  
4. Referral Flow (Doctor → Trainer)  
5. Trainer Intake + Plan v1 creation + Patient approval  
6. Session Logging + Adherence metrics + Focus Card (basic)  
7. Notifications (core events)  
8. ProgressSnapshot (weekly)  
9. LabOrder (metadata) (Phase 1 complete)  
10. Versioning & Audit baseline  

---

## 19. Non-Functional Considerations

| Concern | Baseline Approach |
|---------|------------------|
| Security | Role-scoped API tokens, encrypted PHI at rest, audit hash chaining |
| Performance | Caching for read-heavy artifacts (Restrictions, active plan) |
| Reliability | Event-driven notifications (idempotent) |
| Extensibility | Versioned schema for Restrictions & Plan enabling future granular modifiers |
| Privacy | Consent gating before join queries exposing clinical data to Trainers |

---

## 20. Summary

This specification establishes a structured, auditable, and safety-first framework for integrating clinical oversight with personalized training. It phases sophistication (automation, analytics, AI summarization) while ensuring an MVP foundation of correctness, compliance, and user clarity.

---  
Document Status: Draft v1.0  
Next Review: Align with data modeling workshop & consent matrix refinement.
