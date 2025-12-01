# Pull Request Workflow Test Scenarios

This document describes the test scenarios implemented in `PullRequestTests.npl` and provides guidance for running and extending the test suite.

## Test Organization

The test suite is organized into three main categories:

### 1. Unit Tests - PR Creation and Basic Operations
Tests for fundamental PR operations:
- `testCreatePR` - Verify PR creation with correct initial values
- `testMarkReadyForReview` - Test draft → open transition
- `testConvertToDraft` - Test open → draft transition
- `testUpdateDetails` - Test title and description updates
- `testAddFiles` - Test adding files to a PR

### 2. Unit Tests - Review Submission
Tests for review functionality:
- `testSubmitApprovalReview` - Test APPROVE review submission
- `testSubmitChangesRequestedReview` - Test REQUEST_CHANGES review
- `testSubmitCommentReview` - Test COMMENT review
- `testAddComment` - Test single comment addition

### 3. Unit Tests - Merge Validation
Tests for merge logic:
- `testCannotMergeWithoutApprovals` - Verify merge fails without approvals
- `testSuccessfulMerge` - Test successful merge with approvals
- `testCannotMergeWithChangeRequests` - Verify merge fails with unresolved changes
- `testCannotUpdateAfterMerge` - Verify merged PRs are immutable

### 4. Integration Tests - Complex Scenarios
End-to-end workflow tests:
- `testMultipleReviewers` - Test PR with multiple reviewers and mixed reviews
- `testFullPRLifecycle` - Complete workflow from creation to merge (happy path)
- `testChangesRequestedFlow` - Workflow when changes are requested and addressed
- `testStateTransitions` - Verify all state machine transitions
- `testCloseAndReopenPR` - Test closing and reopening a PR
- `testSetRequiredApprovals` - Test changing approval requirements

### 5. Integration Tests - Authorization and Validation
Security and validation tests:
- `testAuthorizationFailures` - Verify party-based access control
- `testStateGuardEnforcement` - Verify state guards prevent invalid operations
- `testValidationRules` - Verify business rule enforcement

## Running Tests

### Prerequisites
- NPL Runtime installed and configured
- Test environment set up with party management

### Execute All Tests
```bash
npl test pullrequest/PullRequestTests.npl
```

### Execute Specific Test
```bash
npl test pullrequest/PullRequestTests.npl::testCreatePR
```

### Execute Test Category
```bash
# Run only unit tests
npl test pullrequest/PullRequestTests.npl --filter "Unit Tests"

# Run only integration tests
npl test pullrequest/PullRequestTests.npl --filter "Integration Tests"
```

## Test Scenarios from Specification

The following scenarios from section 11 of the specification are implemented:

### ✅ Scenario 1: Happy Path - Full PR Lifecycle
**Test:** `testFullPRLifecycle`
1. Author creates PR in draft state
2. Author marks ready for review
3. Reviewer requests review
4. Reviewer submits APPROVE review
5. Maintainer merges PR
6. Verify PR is in merged state
7. Verify notifications were emitted

### ✅ Scenario 2: Changes Requested Flow
**Test:** `testChangesRequestedFlow`
1. Author creates and marks PR ready
2. Reviewer submits REQUEST_CHANGES
3. Verify PR moves to changes_requested state
4. Author responds to feedback
5. Author adds new commits
6. Reviewer submits new APPROVE review
7. PR moves to approved state
8. Maintainer merges

### ✅ Scenario 3: Multiple Reviewers
**Test:** `testMultipleReviewers`
1. Create PR with multiple reviewers
2. First reviewer approves
3. Second reviewer requests changes
4. Verify PR cannot be merged yet
5. Verify review counts are correct

### ✅ Scenario 4: Authorization Failures
**Test:** `testAuthorizationFailures`
1. Create PR
2. Attempt to merge as author (should fail)
3. Attempt to update as reviewer (should fail)
4. Attempt to review as maintainer (should fail)
5. Verify all fail with authorization errors

### ✅ Scenario 5: State Guard Enforcement
**Test:** `testStateGuardEnforcement`
1. Create PR
2. Attempt to merge while in draft (should fail)
3. Mark ready and request review
4. Attempt to merge while in review_requested (should fail)
5. Verify state guards are enforced

## Test Coverage

### Protocols Tested
- ✅ PullRequest protocol (all 12 permissions)
- ✅ Review protocol (via PullRequest integration)

### State Machine Coverage
- ✅ draft → open
- ✅ open → draft
- ✅ open → review_requested
- ✅ review_requested → in_review
- ✅ review_requested → changes_requested
- ✅ review_requested → approved
- ✅ in_review → changes_requested
- ✅ in_review → approved
- ✅ changes_requested → in_review
- ✅ changes_requested → approved
- ✅ approved → merged
- ✅ open/review_requested/in_review/changes_requested → closed
- ✅ closed → open

### Notification Coverage
All 12 notification types are tested:
- ✅ PRReadyForReview
- ✅ PRConvertedToDraft
- ✅ PRUpdated
- ✅ PRMerged
- ✅ PRClosed
- ✅ PRReopened
- ✅ ReviewRequested
- ✅ ReviewSubmitted
- ✅ CommentAdded
- ✅ AuthorResponded
- ✅ FilesAdded
- ✅ RequiredApprovalsChanged

### Authorization Coverage
- ✅ Author permissions (5 permissions)
- ✅ Reviewer permissions (3 permissions)
- ✅ Maintainer permissions (4 permissions)
- ✅ Cross-party authorization failures

### Validation Coverage
- ✅ Required approvals validation
- ✅ File list validation
- ✅ Merge eligibility validation
- ✅ State transition validation

## Extending the Test Suite

### Adding New Tests

1. **Choose the appropriate category** (Unit or Integration)
2. **Follow the naming convention**: `test<FeatureName>`
3. **Add the @test annotation**
4. **Use descriptive comments** explaining what is being tested
5. **Use helper functions** (`createTestPR`, `createParty`) for setup

Example:
```npl
/**
 * Test: Your test description
 * Verifies what behavior is being tested
 */
@test
function testYourFeature(test: Test) -> {
    var pr = createTestPR();
    
    // Test logic here
    
    test.assertEquals(expected, actual);
};
```

### Test Utilities

**Helper Functions:**
- `createTestPR()` - Creates a standard test PR with one author, one reviewer, one maintainer
- `createParty(email, role)` - Creates a party with specific credentials

**Assertion Methods:**
- `test.assertEquals(expected, actual)` - Assert equality
- `test.assertTrue(condition)` - Assert true
- `test.assertFalse(condition)` - Assert false
- `test.assertFails(function, message)` - Assert operation fails
- `test.expectNotifications(protocol, type, count, filter)` - Assert notifications

## Performance Considerations

The test suite is designed to run quickly:
- Each test is independent (no shared state)
- Tests use minimal data (single file changes, small review lists)
- Helper functions reduce setup overhead

For performance testing scenarios (large PRs, many reviewers, concurrent operations), see section 11.3 of the specification.

## Continuous Integration

These tests should be run:
- ✅ On every commit
- ✅ Before merging PRs
- ✅ As part of the CI/CD pipeline
- ✅ Before releases

## Known Limitations

1. **Party Management**: The `createParty` function depends on NPL testing utilities that may need to be configured based on your environment.

2. **Notification Testing**: The `expectNotifications` method assumes access to the notification stream. Ensure the NPL Runtime is configured to capture notifications during tests.

3. **Performance Tests**: The current suite focuses on functional correctness. Performance tests (section 11.3 of spec) are not yet implemented.

## Future Enhancements

Potential additions to the test suite:
- [ ] Performance tests (large PRs, many reviewers)
- [ ] Concurrency tests (simultaneous operations)
- [ ] Stress tests (edge cases, boundary conditions)
- [ ] Integration tests with external systems (Slack, email)
- [ ] End-to-end tests with real Git operations

