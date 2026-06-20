# Test Plan Template (ATP)

> Feature/module-level test plan. Fill during `/sprint-testing` for epic or module scope.

```yaml
Feature: <feature-name>
Module: <module-path>
Epic: EPIC-BK-XXX
Author: <name>
Date: <YYYY-MM-DD>
```

## Scope

- **In scope**: <list of features/modules to test>
- **Out of scope**: <list of exclusions>
- **Test types**: Manual / API / E2E / Integration

## Test Environment

- **Environment**: <staging / local>
- **URL**: <web + api>
- **Credentials**: <test user>

## Test Cases

| TC# | Priority | Type | Description | ATC Ref | Automation Status |
|-----|----------|------|-------------|---------|-------------------|
| 1 | High | Manual | <desc> | ATC-XXX | Candidate |
| 2 | Medium | API | <desc> | — | Candidate |

## Regression Impact

- <features that might break>
- <modules to include in regression>

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| <desc> | <H/M/L> | <action> |

## Exit Criteria

- [ ] All P1 test cases passed
- [ ] No P1/P2 bugs open
- [ ] Regression suite green
