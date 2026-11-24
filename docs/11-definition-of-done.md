# 11 – Definition of Ready and Definition of Done

## Definition of Ready

A story can enter a sprint when:
- Business value and actor are clear.
- Acceptance criteria are testable.
- Dependencies are identified.
- API/event changes are understood.
- Database impact is identified.
- Security/privacy impact is considered.
- Error and recovery behavior is defined.
- Story is small enough to complete within a sprint or is split.

## Definition of Done

- [ ] Acceptance criteria implemented.
- [ ] Code follows project conventions.
- [ ] Database migration added and reviewed.
- [ ] API contract/OpenAPI updated.
- [ ] Kafka event contract updated where applicable.
- [ ] Validation and consistent error handling implemented.
- [ ] Authentication, authorization and ownership verified.
- [ ] Unit tests pass.
- [ ] Integration tests pass where relevant.
- [ ] Negative, boundary and concurrency cases covered where relevant.
- [ ] Logs, metrics and correlation IDs added for critical paths.
- [ ] No secrets or sensitive data in code/logs.
- [ ] No cross-service database access.
- [ ] README/runbook updated if setup or operations changed.
- [ ] Code review completed.
- [ ] CI pipeline passes.
- [ ] Known limitations documented.

## Pull request checklist

- What problem does this solve?
- Which story does it implement?
- What database/API/event changes are included?
- How was it tested?
- What happens on retry, duplicate delivery, timeout and restart?
- Are security and authorization covered?
- Are migrations backward-compatible?
- Are logs free of sensitive data?
