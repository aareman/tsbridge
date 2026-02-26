# Docker Service Discovery Enhancement: Define Services on Tsbridge Container

## TL;DR

> **Quick Summary**: Add ability to define services on the tsbridge container labels in Docker mode, enabling auto-discovery of matching containers even when they don't have `tsbridge.enabled=true`. This supports dynamically created containers like Nextcloud AIO.
> 
> **Deliverables**:
> - New label parsing for services defined on tsbridge container
> - Modified service discovery to find containers by name from tsbridge container definitions
> - Merge logic to combine with existing service container discovery
> - Unit tests for new functionality
> 
> **Estimated Effort**: Medium
> **Parallel Execution**: YES - 2 waves
> **Critical Path**: Parse tsbridge container services → Find matching containers → Merge with existing → Tests

---

## Context

### Original Request
User wants to add a new feature in Docker mode: specify labels on the tsbridge container to define services that will be auto-discovered when matching containers exist. This is needed for dynamically created containers (like Nextcloud AIO) where you can't add labels to the service container itself.

### Interview Summary
**Key Discussions**:
- Service definition format: Inline style (`tsbridge.service.nextcloud.port=80`)
- Backend resolution: Match by container name, use exposed ports
- Missing container: Skip until exists
- Auto-discover: Yes - when container appears, create proxy even without `tsbridge.enabled=true`
- Testing: TDD approach

**Research Findings**:
- Current implementation in `internal/docker/docker.go` and `internal/docker/labels.go`
- `findServiceContainers` queries Docker for containers with `tsbridge.enabled=true`
- Polling fallback exists (default 1 minute interval)

### Metis Review
**Identified Gaps** (to address in plan):
- Edge case: What if service defined on tsbridge container but container name conflicts with another service?
- Edge case: What if container has `tsbridge.enabled=false` but is in tsbridge container's service list?
- Validation: Should we warn if a service is defined both ways?

---

## Work Objectives

### Core Objective
Enable defining services on the tsbridge container labels that get auto-discovered when matching containers appear, in addition to the existing `tsbridge.enabled=true` discovery mechanism.

### Concrete Deliverables
- [ ] New function to parse service definitions from tsbridge container labels
- [ ] Modified container discovery to find containers matching service names from tsbridge container
- [ ] Merge logic to combine services from both discovery mechanisms (deduplication)
- [ ] Unit tests for new parsing and discovery functions
- [ ] Documentation updates for new label format

### Definition of Done
- [ ] Services can be defined on tsbridge container via labels
- [ ] Matching containers are discovered even without `tsbridge.enabled=true`
- [ ] Existing behavior (tsbridge.enabled=true on containers) still works
- [ ] No duplicate services when same service is discovered both ways
- [ ] All new code has unit tests
- [ ] Documentation updated

### Must Have
- Backward compatible with existing Docker label discovery
- Services defined on tsbridge container work without requiring labels on service containers
- Polling continues to work as safety net

### Must NOT Have (Guardrails)
- No changes to TOML file provider
- No breaking changes to existing Docker label behavior
- No auto-removal of services (only remove when container stops)

---

## Verification Strategy

### Test Decision
- **Infrastructure exists**: YES (Go testing with testify)
- **Automated tests**: TDD (Tests first)
- **Framework**: Go's built-in testing + testify assertions

### QA Policy
Every task includes agent-executed QA scenarios. Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Foundation):
├── Task 1: Parse service definitions from tsbridge container labels [quick]
├── Task 2: Find containers by name from tsbridge container service list [quick]
├── Task 3: Merge services from both discovery mechanisms [quick]
└── Task 4: Unit tests for new parsing function [quick]

Wave 2 (Integration + Tests):
├── Task 5: Unit tests for container matching [quick]
├── Task 6: Unit tests for merge logic [quick]
├── Task 7: Integration test with Docker (if available) [deep]
└── Task 8: Update documentation [writing]
```

## TODOs

- [x] 1. Parse service definitions from tsbridge container labels

  **What to do**:
  - Add new function to parse service names from tsbridge container labels
  - Look for labels matching pattern `tsbridge.service.{name}.*`
  - Extract service names from the label keys
  - Return list of service names defined on tsbridge container
  - Use existing labelParser pattern from labels.go

  **Must NOT do**:
  - Don't change existing parseGlobalConfig behavior
  - Don't modify service container label parsing

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Understanding existing label patterns and extending them
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3, 4)
  - **Blocks**: Task 2
  - **Blocked By**: None

  **References**:
  - `internal/docker/labels.go:47-52` - labelParser struct and newLabelParser
  - `internal/docker/labels.go:142-188` - parseGlobalConfig pattern
  - `internal/docker/docker.go:524-559` - findSelfContainer (how tsbridge container is found)

  **Acceptance Criteria**:
  - [ ] Function accepts container labels and label prefix
  - [ ] Returns slice of service names defined on container
  - [ ] Handles empty labels gracefully
  - [ ] Unit test created
  - [ ] go test ./internal/docker/... → PASS

  **QA Scenarios**:
  ```
  Scenario: Parse service names from tsbridge container labels
    Tool: Bash
    Preconditions: None
    Steps:
      1. Create test labels map with tsbridge.service.nextcloud.port=80, tsbridge.service.redis.port=6379
      2. Call new parseServiceNames function
      3. Assert returned slice contains ["nextcloud", "redis"]
    Expected Result: Function returns correct service names
    Evidence: .sisyphus/evidence/task-1-parse-service-names.txt
  ```

- [x] 2. Find containers by name from tsbridge container service list

  **What to do**:
  - Add function to find containers matching service names from tsbridge container
  - Query Docker for containers by name (not by label)
  - Reuse existing container matching logic
  - Return containers that match any service name from tsbridge container

  **Must NOT do**:
  - Don't change existing findServiceContainers behavior
  - Don't require tsbridge.enabled=true on matched containers

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Needs to understand Docker API and container listing
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3, 4)
  - **Blocks**: Task 5
  - **Blocked By**: Task 1

  **References**:
  - `internal/docker/docker.go:561-607` - findServiceContainers (existing pattern)
  - `internal/docker/docker.go:609-654` - getContainerByID (container lookup)

  **Acceptance Criteria**:
  - [ ] Function finds containers by name from service list
  - [ ] Works without tsbridge.enabled=true on containers
  - [ ] Returns empty slice if no matching containers
  - [ ] Unit test created

  **QA Scenarios**:
  ```
  Scenario: Find container by name from service list
    Tool: Bash
    Preconditions: Docker container "myapp" exists
    Steps:
      1. Call findContainersByNames(["myapp"])
      2. Assert returned containers include "myapp"
    Expected Result: Container found by name
    Evidence: .sisyphus/evidence/task-2-find-by-name.txt
  ```

- [x] 3. Merge services from both discovery mechanisms

  **What to do**:
  - Modify Load() function to combine services from:
    1. Traditional: containers with tsbridge.enabled=true
    2. New: containers matching service names from tsbridge container
  - Deduplicate by service name (prefer traditional discovery)
  - Log when service is discovered via new mechanism

  **Must NOT do**:
  - Don't create duplicate services
  - Don't break existing behavior

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Integration work requiring understanding of both discovery paths
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 4)
  - **Blocks**: Task 7
  - **Blocked By**: Tasks 1, 2

  **References**:
  - `internal/docker/docker.go:167-219` - Load() function
  - `internal/docker/docker.go:720-751` - configEqual (deduplication helper)

  **Acceptance Criteria**:
  - [ ] Both discovery mechanisms work together
  - [ ] No duplicate services in final config
  - [ ] Traditional tsbridge.enabled=true takes precedence if conflict
  - [ ] Integration test passes

- [x] 4. Unit tests for service definition parsing

  **What to do**:
  - Write unit tests for new parseServiceNames function
  - Test edge cases: empty labels, no matching pattern, multiple services
  - Follow existing test patterns in internal/docker/

  **Must NOT do**:
  - Don't modify existing tests
  - Don't break existing test coverage

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Writing unit tests following existing patterns
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 3)
  - **Blocked By**: Task 1

  **References**:
  - Look for existing test files in internal/docker/*_test.go

  **Acceptance Criteria**:
  - [ ] Tests cover: empty labels, no matches, single service, multiple services
  - [ ] go test ./internal/docker/... → PASS
  - [ ] Coverage maintained or improved

---

## Wave 2

- [x] 5. Unit tests for container matching

  **What to do**:
  - Write unit tests for findContainersByNames function
  - Mock Docker client for testing
  - Test: found, not found, multiple matches

  **Acceptance Criteria**:
  - [ ] go test ./internal/docker/... → PASS

- [x] 6. Unit tests for merge logic

  **What to do**:
  - Write unit tests for merge logic in Load()
  - Test deduplication behavior
  - Test precedence when same service discovered both ways

  **Acceptance Criteria**:
  - [ ] go test ./internal/docker/... → PASS

- [x] 7. Integration test with Docker (if available)

  **What to do**:
  - Create integration test that verifies end-to-end behavior
  - Start test containers, verify discovery works
  - Use existing integration test patterns

  **Acceptance Criteria**:
  - [ ] Integration test passes if Docker available
  - [ ] Graceful skip if Docker not available

- [x] 8. Update documentation

  **What to do**:
  - Update docs/docker-labels.md with new feature
  - Document: how to define services on tsbridge container
  - Provide example for Nextcloud AIO use case

  **Must NOT do**:
  - Don't change existing documentation structure
  - Don't remove existing examples

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: Documentation update
  - **Skills**: []

  **References**:
  - `docs/docker-labels.md` - existing documentation

  **Acceptance Criteria**:
  - [ ] New section explaining service definitions on tsbridge container
  - [ ] Example YAML shown
  - [ ] Clear distinction from existing tsbridge.enabled=true approach

---


## Final Verification Wave (MANDATORY — after ALL implementation tasks)

> 4 review agents run in PARALLEL. ALL must APPROVE. Rejection → fix → re-run.

- [ ] F1. **Plan Compliance Audit** — `oracle`
- [ ] F2. **Code Quality Review** — `unspecified-high`
- [ ] F3. **Real Manual QA** — `unspecified-high`
- [ ] F4. **Scope Fidelity Check** — `deep`

---

## Commit Strategy

- **1**: `feat(docker): parse service definitions from tsbridge container labels`
- **2**: `feat(docker): discover containers by name from tsbridge container`
- **3**: `feat(docker): merge services from both discovery mechanisms`
- **4**: `test(docker): add unit tests for service definition parsing`
- **5**: `test(docker): add unit tests for container matching and merge`
- **6**: `docs(docker): document service definition on tsbridge container`

---

## Success Criteria

### Verification Commands
```bash
go test ./internal/docker/... -v
go build ./...
```

### Final Checklist
- [ ] All "Must Have" present
- [ ] All "Must NOT Have" absent
- [ ] All tests pass
