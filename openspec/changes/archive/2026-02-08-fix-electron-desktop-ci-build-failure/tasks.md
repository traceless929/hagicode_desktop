# Tasks: Fix Electron Desktop CI Build Failure

## Overview

This document outlines the implementation tasks for fixing the CI build failure caused by a path mismatch between the smoke test validation and the actual main process entry point.

**Change ID**: `fix-electron-desktop-ci-build-failure`

**Status**: `pending`

---

## Phase 1: Preparation

### Task 1.1: Verify Current Build Behavior

**Description**: Run the build locally and confirm the exact output paths to validate the root cause analysis.

**Steps**:
1. Run `npm run build:all` to generate all build artifacts
2. Inspect the `dist/` directory structure
3. Confirm `dist/main/main.js` is generated (not `dist/main.js`)
4. Verify `package.json` points to `dist/main/main.js`

**Acceptance Criteria**:
- `dist/main/main.js` exists and is >1KB
- `dist/main.js` does NOT exist
- Documentation of actual vs expected paths

**Estimated Time**: 10 minutes

**Dependencies**: None

---

### Task 1.2: Review Smoke Test Code

**Description**: Thoroughly review the smoke test to identify all locations that need path updates.

**Steps**:
1. Read `scripts/smoke-test.js` completely
2. Identify all references to `main.js`
3. Document each location and its purpose
4. Confirm no other tests are affected

**Acceptance Criteria**:
- Complete list of lines requiring changes
- Understanding of each test's purpose
- No other tests affected by the path change

**Estimated Time**: 5 minutes

**Dependencies**: None

---

## Phase 2: Implementation

### Task 2.1: Update Smoke Test Path Checks

**Description**: Modify the smoke test to check for `dist/main/main.js` instead of `dist/main.js`.

**Files to Modify**:
- `scripts/smoke-test.js`

**Specific Changes**:

1. **Line 92-94** (verbose logging):
   ```javascript
   // Before:
   const mainJs = path.join(distPath, 'main.js');
   // After:
   const mainJs = path.join(distPath, 'main', 'main.js');
   ```

2. **Line 102-105** (main process test):
   ```javascript
   // Before:
   const mainJs = path.join(process.cwd(), 'dist', 'main.js');
   // After:
   const mainJs = path.join(process.cwd(), 'dist', 'main', 'main.js');
   ```

3. **Line 157** (package.json validation):
   ```javascript
   // Before:
   assert(pkg.main === 'dist/main.js', 'package.json main points to dist/main.js');
   // After:
   assert(pkg.main === 'dist/main/main.js', 'package.json main points to dist/main/main.js');
   ```

4. **Line 168** (main.js content test):
   ```javascript
   // Before:
   const mainJs = path.join(process.cwd(), 'dist', 'main.js');
   // After:
   const mainJs = path.join(process.cwd(), 'dist', 'main', 'main.js');
   ```

**Acceptance Criteria**:
- All four locations updated consistently
- Code style matches existing patterns
- Comments updated if necessary

**Estimated Time**: 10 minutes

**Dependencies**: Task 1.2

---

## Phase 3: Validation

### Task 3.1: Run Updated Smoke Test

**Description**: Execute the modified smoke test to verify all checks pass.

**Steps**:
1. Run `npm run smoke-test`
2. Verify all tests pass (10/10)
3. Run `npm run smoke-test:verbose` for detailed output
4. Review test output for any warnings

**Acceptance Criteria**:
- All 10 tests pass
- No test failures or skips
- Clean console output

**Estimated Time**: 5 minutes

**Dependencies**: Task 2.1

---

### Task 3.2: Verify Full Build Process

**Description**: Run the complete build pipeline to ensure no regressions.

**Steps**:
1. Clean build artifacts: `rm -rf dist pkg`
2. Run `npm run build:prod`
3. Verify smoke test passes as part of build
4. Check `dist/main/main.js` exists and is valid

**Acceptance Criteria**:
- `build:prod` completes successfully
- Smoke test passes during build
- `dist/main/main.js` exists and is >1KB
- No build errors or warnings

**Estimated Time**: 10 minutes

**Dependencies**: Task 3.1

---

### Task 3.3: Local Application Launch Test

**Description**: Launch the built application to ensure it works correctly.

**Steps**:
1. Run `npm run start` (builds and launches)
2. Verify main window opens
3. Check for any console errors in DevTools
4. Verify basic functionality works

**Acceptance Criteria**:
- Application launches successfully
- Main window renders correctly
- No console errors on startup
- Basic UI interactions work

**Estimated Time**: 10 minutes

**Dependencies**: Task 3.2

---

## Phase 4: Documentation

### Task 4.1: Update Change Proposal

**Description**: Update the proposal with actual implementation details and mark as completed.

**Steps**:
1. Update `proposal.md` with implementation notes
2. Add any lessons learned
3. Update status to `completed`

**Acceptance Criteria**:
- Proposal reflects actual changes made
- Status updated to `completed`

**Estimated Time**: 5 minutes

**Dependencies**: Task 3.3

---

## Phase 5: Verification (Optional)

### Task 5.1: CI Build Verification

**Description**: Push changes and verify CI build succeeds.

**Steps**:
1. Commit changes with descriptive message
2. Push to feature branch or main
3. Monitor GitHub Actions workflow
4. Verify all platform builds succeed (win, mac, linux)

**Acceptance Criteria**:
- All CI jobs pass
- Artifacts generated successfully
- No smoke test failures

**Estimated Time**: 15-30 minutes (depending on CI queue)

**Dependencies**: Task 4.1

---

## Summary

| Phase | Tasks | Total Estimate |
|-------|-------|----------------|
| Preparation | 2 | 15 min |
| Implementation | 1 | 10 min |
| Validation | 3 | 25 min |
| Documentation | 1 | 5 min |
| Verification (Optional) | 1 | 15-30 min |
| **Total** | **8** | **~1 hour** |

---

## Risk Mitigation

- **Low Risk**: Changes are isolated to test validation logic
- **Rollback Plan**: Revert smoke test changes if unexpected issues arise
- **Testing Strategy**: Incremental testing after each change

---

## Notes

- The application entry point in `package.json` is already correct (`dist/main/main.js`)
- Only the smoke test validation needs to be updated
- No changes to TypeScript, Vite, or application code required
