# GitHub-Style Pull Request Workflow System - NPL Implementation Specification

## Executive Summary

This specification defines a comprehensive GitHub-style Pull Request (PR) workflow system implemented in NPL (Noumena Protocol Language). The system demonstrates NPL's authorization model, state machines, and multi-party coordination capabilities while creating a familiar developer workflow that showcases Augment Code's ability to assist developers in learning and building with entirely new programming languages.

**Target Audience:** Developers familiar with Git workflows but new to NPL
**Complexity Level:** Intermediate - showcases NPL's core features without overwhelming complexity
**Estimated Build Time:** 4-6 hours with AI assistance
**Key NPL Features Demonstrated:** 
- Party-based authorization
- State machine transitions
- Protocol composition
- Structs and enums
- Notifications
- Permission guards
- Protocol-level and global functions

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Domain Model](#domain-model)
3. [Protocol Architecture](#protocol-architecture)
4. [Detailed Protocol Specifications](#detailed-protocol-specifications)
5. [State Machines](#state-machines)
6. [Authorization Model](#authorization-model)
7. [Data Structures](#data-structures)
8. [API Surface](#api-surface)
9. [Business Rules and Validation](#business-rules-and-validation)
10. [Notification System](#notification-system)
11. [Test Scenarios](#test-scenarios)
12. [Implementation Roadmap](#implementation-roadmap)

---

## 1. System Overview

### 1.1 Purpose

The PR Workflow System models a code review process where:
- **Authors** create pull requests proposing code changes
- **Reviewers** review code and provide feedback
- **Maintainers** have final approval authority and can merge changes
- **The system** enforces workflow rules, tracks history, and maintains audit trails

### 1.2 Core Concepts

**Pull Request Lifecycle:**
```
draft → open → review_requested → changes_requested ⟷ in_review → approved → merged
                                                                              ↓
                                                                           closed
```

**Key Invariants:**
- Only authors can create and update PRs
- Only designated reviewers can review PRs
- Only maintainers can merge PRs
- Merged PRs cannot be modified
- All state transitions are auditable
- Authorization is enforced at every step

### 1.3 Why This Demo Is Effective

1. **Familiarity:** Every developer understands PR workflows
2. **Complexity:** Rich enough to show NPL's power, simple enough to understand quickly
3. **Authorization Showcase:** Demonstrates NPL's party-based security model naturally
4. **State Machines:** Clear state transitions that map to real workflows
5. **Multi-Protocol:** Shows protocol composition and relationships
6. **Real-World:** Solves an actual problem developers face

---

## 2. Domain Model

### 2.1 Entities

#### PullRequest (Primary Protocol)
Represents a proposed code change with full lifecycle management.

**Parties:**
- `author` - Creates the PR, can update it, responds to feedback
- `reviewers` - List of users who can review and approve/reject
- `maintainer` - Repository owner/admin who can merge or close

**Core Data:**
- `title: Text` - PR title (max 200 chars)
- `description: Text` - Detailed description of changes
- `sourceBranch: Text` - Branch name containing changes
- `targetBranch: Text` - Branch to merge into (usually main/master)
- `filesChanged: List<FileChange>` - List of files affected
- `createdAt: DateTime` - When PR was created
- `updatedAt: DateTime` - Last update timestamp

**Derived/Computed Data:**
- `reviewCount: Number` - Total reviews submitted
- `approvalCount: Number` - Number of approved reviews
- `changesRequestedCount: Number` - Number of reviews requesting changes
- `commentCount: Number` - Total comments across all reviews
- `isApproved: Boolean` - Whether PR has sufficient approvals
- `canMerge: Boolean` - Whether PR is ready to merge

#### Review (Nested Protocol)
Represents a reviewer's assessment of the PR.

**Parties:**
- `reviewer` - Person conducting the review

**Core Data:**
- `reviewType: ReviewType` - APPROVE, REQUEST_CHANGES, COMMENT
- `summary: Text` - Overall review feedback
- `comments: List<ReviewComment>` - Line-by-line comments
- `submittedAt: DateTime` - When review was submitted

#### ReviewComment (Struct)
Individual feedback on specific code sections.

**Fields:**
- `filePath: Text` - File being commented on
- `lineNumber: Number` - Line number in the file
- `commentText: Text` - The actual comment
- `createdAt: DateTime` - When comment was made
- `commentId: Text` - Unique identifier

### 2.2 Entity Relationships

```
Repository (1) ──┬──> (N) PullRequest
                 │
                 └──> (N) User (as maintainer)

PullRequest (1) ──> (1) author
                ├──> (N) reviewers
                ├──> (1) maintainer
                └──> (N) Review

Review (1) ──> (N) ReviewComment
```

---

## 3. Protocol Architecture

### 3.1 Package Structure

```
pullrequest/
├── PullRequest.npl       # Main PR protocol
├── Review.npl            # Review sub-protocol
├── Types.npl             # Enums and structs
├── Notifications.npl     # Event notifications
└── Helpers.npl           # Utility functions
```

### 3.2 Protocol Hierarchy

**Primary Protocol: PullRequest**
- Manages the entire PR lifecycle
- Coordinates reviews
- Enforces merge rules
- Emits notifications

**Secondary Protocol: Review**
- Created by PullRequest
- Scoped to individual reviewer
- Immutable once submitted

### 3.3 Design Principles

1. **Single Responsibility:** Each protocol has one clear purpose
2. **Immutability:** Reviews cannot be edited once submitted (mimics GitHub)
3. **Authorization First:** Every action checks party membership
4. **State Guards:** Permissions are restricted by current state
5. **Audit Trail:** NPL's built-in versioning tracks all changes
6. **Explicit Transitions:** State changes are deliberate and documented

---

## 4. Detailed Protocol Specifications

### 4.1 PullRequest Protocol

```npl
package pullrequest;

@api
protocol[author, reviewers, maintainer] PullRequest(
    var title: Text,
    var description: Text,
    var sourceBranch: Text,
    var targetBranch: Text,
    var filesChanged: List<FileChange>
) {
    // Validation on construction
    require(!title.isEmpty(), "PR title cannot be empty");
    require(title.length() <= 200, "PR title must be 200 characters or less");
    require(!sourceBranch.isEmpty(), "Source branch must be specified");
    require(!targetBranch.isEmpty(), "Target branch must be specified");
    require(sourceBranch != targetBranch, "Source and target branches must be different");
    require(!filesChanged.isEmpty(), "PR must contain at least one file change");
    
    // State machine definition
    initial state draft;
    state open;
    state review_requested;
    state in_review;
    state changes_requested;
    state approved;
    final state merged;
    final state closed;
    
    // Core data fields
    private var reviews: List<Review> = listOf<Review>();
    private var requiredApprovals: Number = 2;
    var isDraft: Boolean = true;
    var mergeCommitSha: Optional<Text> = optionalOf<Text>();
    var closedAt: Optional<DateTime> = optionalOf<DateTime>();
    var mergedAt: Optional<DateTime> = optionalOf<DateTime>();
    var mergedBy: Optional<Party> = optionalOf<Party>();
    
    // Timestamps
    var createdAt: DateTime = now();
    var updatedAt: DateTime = now();
    
    // === AUTHOR PERMISSIONS ===
    
    @api
    permission[author] updateDetails(
        newTitle: Text,
        newDescription: Text
    ) | draft, open, review_requested, changes_requested {
        require(!newTitle.isEmpty(), "Title cannot be empty");
        require(newTitle.length() <= 200, "Title too long");
        
        title = newTitle;
        description = newDescription;
        updatedAt = now();
        
        notify PRUpdated(title, getUsername(author));
    };
    
    @api
    permission[author] addFiles(
        newFiles: List<FileChange>
    ) | draft, open, changes_requested {
        require(!newFiles.isEmpty(), "Must provide at least one file");
        
        filesChanged = filesChanged.concat(newFiles);
        updatedAt = now();
        
        notify FilesAdded(newFiles.size(), title);
    };
    
    @api
    permission[author] markReadyForReview() | draft {
        require(filesChanged.size() > 0, "Cannot mark PR ready without files");
        
        isDraft = false;
        become open;
        updatedAt = now();
        
        notify PRReadyForReview(title, getUsername(author));
    };
    
    @api
    permission[author] convertToDraft() | open, review_requested {
        isDraft = true;
        become draft;
        updatedAt = now();
        
        notify PRConvertedToDraft(title);
    };
    
    @api
    permission[author] respondToReview(
        reviewId: Text,
        response: Text
    ) | changes_requested, in_review {
        require(!response.isEmpty(), "Response cannot be empty");
        
        // In a real system, this would update review comments
        // For demo purposes, we'll just update timestamp
        updatedAt = now();
        
        notify AuthorResponded(title, getUsername(author), reviewId);
    };
    
    // === REVIEWER PERMISSIONS ===
    
    @api
    permission[reviewers] requestReview() | open {
        become review_requested;
        updatedAt = now();
        
        notify ReviewRequested(title, getUsername(reviewers));
    };
    
    @api
    permission[reviewers] submitReview(
        reviewType: ReviewType,
        summary: Text,
        comments: List<ReviewComment>
    ) | review_requested, in_review, changes_requested returns Review {
        require(!summary.isEmpty(), "Review summary cannot be empty");
        
        // Create the review
        var review = Review[reviewers](reviewType, summary, comments);
        reviews = reviews.concat(listOf<Review>(review));
        
        // Update state based on review type and current reviews
        if (reviewType == ReviewType.REQUEST_CHANGES) {
            become changes_requested;
        } else if (reviewType == ReviewType.APPROVE) {
            if (getApprovalCount() >= requiredApprovals) {
                become approved;
            } else {
                become in_review;
            }
        } else {
            // COMMENT type
            if (getCurrentState() != in_review && getCurrentState() != changes_requested) {
                become in_review;
            }
        }
        
        updatedAt = now();
        
        notify ReviewSubmitted(
            title,
            getUsername(reviewers),
            reviewType,
            comments.size()
        );
        
        return review;
    };
    
    @api
    permission[reviewers] addComment(
        filePath: Text,
        lineNumber: Number,
        commentText: Text
    ) | open, review_requested, in_review, changes_requested {
        require(!commentText.isEmpty(), "Comment cannot be empty");
        require(lineNumber > 0, "Line number must be positive");
        
        var comment = ReviewComment(
            filePath,
            lineNumber,
            commentText,
            now(),
            generateCommentId()
        );
        
        // Add to lightweight comments (not full review)
        updatedAt = now();
        
        notify CommentAdded(title, getUsername(reviewers), filePath, lineNumber);
    };
    
    // === MAINTAINER PERMISSIONS ===
    
    @api
    permission[maintainer] setRequiredApprovals(
        count: Number
    ) | draft, open, review_requested {
        require(count > 0, "Required approvals must be positive");
        require(count <= 10, "Required approvals cannot exceed 10");
        
        requiredApprovals = count;
        updatedAt = now();
        
        notify RequiredApprovalsChanged(title, count);
    };
    
    @api
    permission[maintainer] merge(
        commitMessage: Text
    ) | approved returns MergeResult {
        require(!commitMessage.isEmpty(), "Commit message required");
        require(getApprovalCount() >= requiredApprovals, "Insufficient approvals");
        require(!hasUnresolvedChangeRequests(), "Cannot merge with unresolved change requests");
        
        var sha = generateMergeCommitSha();
        mergeCommitSha = optionalOf<Text>(sha);
        mergedAt = optionalOf<DateTime>(now());
        mergedBy = optionalOf<Party>(maintainer);
        
        become merged;
        updatedAt = now();
        
        notify PRMerged(
            title,
            sourceBranch,
            targetBranch,
            sha,
            getUsername(maintainer)
        );
        
        return MergeResult(true, sha, "Successfully merged");
    };
    
    @api
    permission[maintainer] close(
        reason: Text
    ) | open, review_requested, in_review, changes_requested, approved {
        require(!reason.isEmpty(), "Close reason required");
        
        closedAt = optionalOf<DateTime>(now());
        become closed;
        updatedAt = now();
        
        notify PRClosed(title, reason, getUsername(maintainer));
    };
    
    @api
    permission[maintainer] reopen() | closed {
        closedAt = optionalOf<DateTime>();
        become open;
        updatedAt = now();
        
        notify PRReopened(title, getUsername(maintainer));
    };
    
    // === QUERY FUNCTIONS (available to all parties) ===
    
    @api
    function getReviewCount() returns Number -> reviews.size();
    
    @api
    function getApprovalCount() returns Number -> {
        return reviews
            .filter(function(r: Review) returns Boolean -> r.reviewType == ReviewType.APPROVE)
            .size();
    };
    
    @api
    function getChangesRequestedCount() returns Number -> {
        return reviews
            .filter(function(r: Review) returns Boolean -> r.reviewType == ReviewType.REQUEST_CHANGES)
            .size();
    };
    
    @api
    function getCommentCount() returns Number -> {
        return reviews
            .map(function(r: Review) returns Number -> r.comments.size())
            .sum();
    };
    
    @api
    function canMerge() returns Boolean -> {
        return getCurrentState() == approved 
            && getApprovalCount() >= requiredApprovals
            && !hasUnresolvedChangeRequests();
    };
    
    @api
    function getReviewers() returns List<Text> -> {
        return getPartyUsernames(reviewers);
    };
    
    @api
    function getSummary() returns PRSummary -> {
        return PRSummary(
            title,
            getCurrentState(),
            getReviewCount(),
            getApprovalCount(),
            getChangesRequestedCount(),
            canMerge(),
            createdAt,
            updatedAt
        );
    };
    
    // === PRIVATE HELPER FUNCTIONS ===
    
    private function hasUnresolvedChangeRequests() returns Boolean -> {
        // Check if there are any REQUEST_CHANGES reviews
        // without subsequent APPROVE from same reviewer
        return getChangesRequestedCount() > 0;
    };
    
    private function getCurrentState() returns State -> {
        // This would return the current state
        // Implementation detail handled by NPL runtime
        return draft; // Placeholder
    };
    
    private function generateCommentId() returns Text -> {
        return "comment_" + now().toText();
    };
    
    private function generateMergeCommitSha() returns Text -> {
        return "sha_" + now().toMillis().toText();
    };
};
```

### 4.2 Review Protocol

```npl
package pullrequest;

protocol[reviewer] Review(
    var reviewType: ReviewType,
    var summary: Text,
    var comments: List<ReviewComment>
) {
    require(!summary.isEmpty(), "Review summary cannot be empty");
    
    var submittedAt: DateTime = now();
    var reviewId: Text = generateReviewId();
    
    initial state submitted;
    final state acknowledged;
    
    @api
    permission[reviewer] acknowledge() | submitted {
        become acknowledged;
    };
    
    @api
    function getCommentCount() returns Number -> comments.size();
    
    @api
    function getReviewSummary() returns ReviewSummary -> {
        return ReviewSummary(
            reviewId,
            reviewType,
            summary,
            comments.size(),
            submittedAt
        );
    };
    
    private function generateReviewId() returns Text -> {
        return "review_" + now().toMillis().toText();
    };
};
```

---

## 5. State Machines

### 5.1 PullRequest State Machine

**States:**

| State | Description | Entry Conditions | Exit Conditions |
|-------|-------------|------------------|-----------------|
| `draft` | PR is being prepared | Created as draft | Author marks ready OR converted from open |
| `open` | PR is ready but not yet reviewed | Marked ready from draft | Reviewer requests review OR converted to draft |
| `review_requested` | Review has been formally requested | Reviewer action from open | Review submitted OR converted to draft |
| `in_review` | Actively being reviewed | Comment or approval submitted | All approvals received OR changes requested |
| `changes_requested` | Reviewer requested modifications | REQUEST_CHANGES review submitted | New review submitted |
| `approved` | PR has required approvals | Sufficient APPROVE reviews | Merged or closed |
| `merged` | PR has been merged (final) | Maintainer merge action | N/A (final state) |
| `closed` | PR was closed without merging (final) | Maintainer close action | Can reopen to `open` |

**Transition Matrix:**

```
From/To         | draft | open | review_req | in_review | changes_req | approved | merged | closed
----------------|-------|------|------------|-----------|-------------|----------|--------|--------
draft           |   -   |  ✓   |     -      |     -     |      -      |    -     |   -    |   -
open            |   ✓   |  -   |     ✓      |     -     |      -      |    -     |   -    |   ✓
review_req      |   ✓   |  -   |     -      |     ✓     |      ✓      |    ✓     |   -    |   ✓
in_review       |   -   |  -   |     -      |     -     |      ✓      |    ✓     |   -    |   ✓
changes_req     |   -   |  -   |     ✓      |     ✓     |      -      |    ✓     |   -    |   ✓
approved        |   -   |  -   |     -      |     -     |      -      |    -     |   ✓    |   ✓
merged          |   -   |  -   |     -      |     -     |      -      |    -     |   -    |   -
closed          |   -   |  ✓   |     -      |     -     |      -      |    -     |   -    |   -
```

### 5.2 Review State Machine

**States:**

| State | Description | Entry | Exit |
|-------|-------------|-------|------|
| `submitted` | Review has been submitted | Created | Acknowledged |
| `acknowledged` | Review has been seen (final) | Acknowledged | N/A (final) |

This is intentionally simple as reviews are immutable records.

---

## 6. Authorization Model

### 6.1 Party Definitions

**Author Party:**
- Role: Creator and owner of the PR
- Claims: `{ "user_id": ["alice@example.com"], "role": ["developer"] }`
- Permissions:
  - Update PR details
  - Add files
  - Mark ready for review
  - Convert to draft
  - Respond to reviews

**Reviewers Party (List):**
- Role: Team members authorized to review code
- Claims: `{ "user_id": ["bob@example.com", "carol@example.com"], "role": ["developer", "senior-dev"] }`
- Permissions:
  - Request review
  - Submit reviews
  - Add comments
  - Approve or request changes

**Maintainer Party:**
- Role: Repository administrator
- Claims: `{ "user_id": ["dave@example.com"], "role": ["admin", "maintainer"] }`
- Permissions:
  - All reviewer permissions
  - Set approval requirements
  - Merge PRs
  - Close/reopen PRs
  - Override review states (if needed)

### 6.2 Authorization Patterns

**Pattern 1: Single Party Access**
```npl
permission[author] updateDetails(...) {
    // Only the author can update
}
```

**Pattern 2: Multi-Party Access**
```npl
permission[reviewers] submitReview(...) {
    // Any reviewer in the list can submit
}
```

**Pattern 3: State-Guarded Access**
```npl
permission[maintainer] merge(...) | approved {
    // Only in 'approved' state AND only maintainer
}
```

**Pattern 4: Conditional Access**
```npl
permission[reviewers] requestReview() | open {
    require(!isDraft, "Cannot review draft PR");
    // Additional business rules
}
```

### 6.3 Party Assignment at Creation

```npl
// Creating a PR with specific parties
function createPR(
    authorParty: Party,
    reviewerParties: List<Party>,
    maintainerParty: Party,
    title: Text,
    description: Text,
    source: Text,
    target: Text,
    files: List<FileChange>
) returns PullRequest -> {
    return PullRequest[authorParty, reviewerParties, maintainerParty](
        title,
        description,
        source,
        target,
        files
    );
};
```

### 6.4 JWT Token Structure

Example token for author "Alice":
```json
{
  "sub": "alice@example.com",
  "email": "alice@example.com",
  "preferred_username": "alice",
  "role": ["developer"],
  "team": ["platform"],
  "iat": 1234567890,
  "exp": 1234571490
}
```

NPL matches these claims against party definitions to authorize actions.

---

## 7. Data Structures

### 7.1 Enums

```npl
package pullrequest;

enum ReviewType {
    APPROVE,
    REQUEST_CHANGES,
    COMMENT
};

enum FileChangeType {
    ADDED,
    MODIFIED,
    DELETED,
    RENAMED
};

enum MergeStrategy {
    MERGE_COMMIT,
    SQUASH,
    REBASE
};
```

### 7.2 Structs

```npl
package pullrequest;

struct FileChange {
    filePath: Text,
    changeType: FileChangeType,
    linesAdded: Number,
    linesDeleted: Number,
    oldPath: Optional<Text>  // For renames
};

struct ReviewComment {
    filePath: Text,
    lineNumber: Number,
    commentText: Text,
    createdAt: DateTime,
    commentId: Text
};

struct PRSummary {
    title: Text,
    state: Text,  // Would be State enum in full implementation
    reviewCount: Number,
    approvalCount: Number,
    changesRequestedCount: Number,
    canMerge: Boolean,
    createdAt: DateTime,
    updatedAt: DateTime
};

struct ReviewSummary {
    reviewId: Text,
    reviewType: ReviewType,
    summary: Text,
    commentCount: Number,
    submittedAt: DateTime
};

struct MergeResult {
    success: Boolean,
    commitSha: Text,
    message: Text
};

struct PRMetrics {
    timeToFirstReview: Optional<Number>,  // milliseconds
    timeToMerge: Optional<Number>,
    totalReviews: Number,
    totalComments: Number,
    filesChangedCount: Number,
    totalLinesChanged: Number
};
```

### 7.3 Helper Data Structures

```npl
package pullrequest;

struct Contributor {
    username: Text,
    email: Text,
    role: Text
};

struct Branch {
    name: Text,
    lastCommitSha: Text,
    lastUpdated: DateTime
};

struct MergeConflict {
    filePath: Text,
    conflictType: Text,
    description: Text
};
```

---

## 8. API Surface

### 8.1 Generated REST Endpoints

NPL automatically generates REST endpoints for `@api` annotated permissions and functions.

**Base URL:** `http://localhost:12000/api/pullrequest`

#### Protocol Instantiation

**Create Pull Request**
```
POST /PullRequest
Content-Type: application/json
Authorization: Bearer <jwt_token>

{
  "title": "Add user authentication",
  "description": "Implements JWT-based auth...",
  "sourceBranch": "feature/auth",
  "targetBranch": "main",
  "filesChanged": [
    {
      "filePath": "src/auth/jwt.ts",
      "changeType": "ADDED",
      "linesAdded": 150,
      "linesDeleted": 0,
      "oldPath": null
    }
  ],
  "parties": {
    "author": {
      "claims": {
        "user_id": ["alice@example.com"],
        "role": ["developer"]
      }
    },
    "reviewers": [
      {
        "claims": {
          "user_id": ["bob@example.com"],
          "role": ["senior-dev"]
        }
      }
    ],
    "maintainer": {
      "claims": {
        "user_id": ["dave@example.com"],
        "role": ["admin"]
      }
    }
  }
}

Response 201 Created:
{
  "protocolId": "pr_abc123",
  "state": "draft",
  "createdAt": "2024-12-01T10:00:00Z"
}
```

#### Author Actions

**Update PR Details**
```
POST /PullRequest/{protocolId}/updateDetails
Content-Type: application/json
Authorization: Bearer <author_jwt>

{
  "newTitle": "Add JWT authentication with refresh tokens",
  "newDescription": "Updated description..."
}

Response 200 OK:
{
  "success": true,
  "updatedAt": "2024-12-01T10:15:00Z"
}
```

**Mark Ready for Review**
```
POST /PullRequest/{protocolId}/markReadyForReview
Authorization: Bearer <author_jwt>

Response 200 OK:
{
  "success": true,
  "newState": "open",
  "notification": "PR ready for review"
}
```

**Add Files**
```
POST /PullRequest/{protocolId}/addFiles
Content-Type: application/json
Authorization: Bearer <author_jwt>

{
  "newFiles": [
    {
      "filePath": "src/auth/refresh.ts",
      "changeType": "ADDED",
      "linesAdded": 75,
      "linesDeleted": 0
    }
  ]
}

Response 200 OK:
{
  "success": true,
  "totalFiles": 2
}
```

#### Reviewer Actions

**Request Review**
```
POST /PullRequest/{protocolId}/requestReview
Authorization: Bearer <reviewer_jwt>

Response 200 OK:
{
  "success": true,
  "newState": "review_requested"
}
```

**Submit Review**
```
POST /PullRequest/{protocolId}/submitReview
Content-Type: application/json
Authorization: Bearer <reviewer_jwt>

{
  "reviewType": "APPROVE",
  "summary": "LGTM! Great implementation of JWT auth.",
  "comments": [
    {
      "filePath": "src/auth/jwt.ts",
      "lineNumber": 45,
      "commentText": "Consider adding rate limiting here",
      "createdAt": "2024-12-01T11:00:00Z",
      "commentId": "comment_xyz789"
    }
  ]
}

Response 200 OK:
{
  "reviewId": "review_def456",
  "newState": "in_review",
  "approvalCount": 1
}
```

**Add Comment**
```
POST /PullRequest/{protocolId}/addComment
Content-Type: application/json
Authorization: Bearer <reviewer_jwt>

{
  "filePath": "src/auth/jwt.ts",
  "lineNumber": 67,
  "commentText": "Nice error handling!"
}

Response 200 OK:
{
  "success": true,
  "commentId": "comment_abc999"
}
```

#### Maintainer Actions

**Set Required Approvals**
```
POST /PullRequest/{protocolId}/setRequiredApprovals
Content-Type: application/json
Authorization: Bearer <maintainer_jwt>

{
  "count": 2
}

Response 200 OK:
{
  "success": true,
  "requiredApprovals": 2
}
```

**Merge Pull Request**
```
POST /PullRequest/{protocolId}/merge
Content-Type: application/json
Authorization: Bearer <maintainer_jwt>

{
  "commitMessage": "Merge pull request #123: Add JWT authentication"
}

Response 200 OK:
{
  "success": true,
  "commitSha": "sha_abc123def456",
  "message": "Successfully merged",
  "newState": "merged",
  "mergedAt": "2024-12-01T14:00:00Z"
}
```

**Close Pull Request**
```
POST /PullRequest/{protocolId}/close
Content-Type: application/json
Authorization: Bearer <maintainer_jwt>

{
  "reason": "Duplicate of PR #120"
}

Response 200 OK:
{
  "success": true,
  "newState": "closed",
  "closedAt": "2024-12-01T14:30:00Z"
}
```

#### Query Functions

**Get Review Count**
```
GET /PullRequest/{protocolId}/getReviewCount
Authorization: Bearer <any_party_jwt>

Response 200 OK:
{
  "count": 3
}
```

**Get PR Summary**
```
GET /PullRequest/{protocolId}/getSummary
Authorization: Bearer <any_party_jwt>

Response 200 OK:
{
  "title": "Add JWT authentication with refresh tokens",
  "state": "approved",
  "reviewCount": 3,
  "approvalCount": 2,
  "changesRequestedCount": 0,
  "canMerge": true,
  "createdAt": "2024-12-01T10:00:00Z",
  "updatedAt": "2024-12-01T13:45:00Z"
}
```

**Check Can Merge**
```
GET /PullRequest/{protocolId}/canMerge
Authorization: Bearer <any_party_jwt>

Response 200 OK:
{
  "canMerge": true,
  "reason": "All approval requirements met"
}
```

### 8.2 OpenAPI Specification

NPL automatically generates OpenAPI specs for all protocols. Available at:
```
http://localhost:12000/swagger-ui/
```

The Swagger UI provides:
- Interactive API documentation
- Request/response examples
- Authentication testing
- Schema definitions

---

## 9. Business Rules and Validation

### 9.1 Creation Rules

| Rule | Enforcement | Error Message |
|------|-------------|---------------|
| Title required | `require(!title.isEmpty())` | "PR title cannot be empty" |
| Title max length | `require(title.length() <= 200)` | "PR title must be 200 characters or less" |
| Source branch required | `require(!sourceBranch.isEmpty())` | "Source branch must be specified" |
| Target branch required | `require(!targetBranch.isEmpty())` | "Target branch must be specified" |
| Different branches | `require(sourceBranch != targetBranch)` | "Source and target branches must be different" |
| Files required | `require(!filesChanged.isEmpty())` | "PR must contain at least one file change" |

### 9.2 Update Rules

| Rule | Enforcement | State Restriction |
|------|-------------|-------------------|
| Draft updates allowed | State guard | `draft, open, changes_requested` |
| Cannot update after merge | State guard | NOT `merged` |
| Cannot update when closed | State guard | NOT `closed` |
| Title still required | Permission validation | All update states |
| Author only can update | Party check | `[author]` permission |

### 9.3 Review Rules

| Rule | Enforcement | Description |
|------|-------------|-------------|
| Summary required | `require(!summary.isEmpty())` | Reviews must have text |
| Comment must have text | Validation in addComment | No empty comments |
| Line numbers positive | `require(lineNumber > 0)` | Valid file references |
| Review after open | State guard: `open, review_requested` | Can't review drafts |
| One review per reviewer | Business logic | Track reviewer submissions |

### 9.4 Merge Rules

| Rule | Enforcement | Description |
|------|-------------|-------------|
| Must be approved | State guard: `approved` | Cannot merge without approval |
| Sufficient approvals | `require(getApprovalCount() >= requiredApprovals)` | Meet threshold |
| No unresolved changes | `require(!hasUnresolvedChangeRequests())` | Address all feedback |
| Commit message required | `require(!commitMessage.isEmpty())` | Document merge |
| Maintainer only | Party check: `[maintainer]` | Authorization |

### 9.5 State Transition Rules

```npl
// Example: Smart state transitions based on reviews
if (reviewType == ReviewType.REQUEST_CHANGES) {
    // Any REQUEST_CHANGES review moves to changes_requested
    become changes_requested;
} else if (reviewType == ReviewType.APPROVE) {
    if (getApprovalCount() >= requiredApprovals) {
        // Sufficient approvals → move to approved
        become approved;
    } else {
        // Still need more approvals → in_review
        become in_review;
    }
} else {
    // COMMENT type keeps current review state
    if (getCurrentState() != in_review && getCurrentState() != changes_requested) {
        become in_review;
    }
}
```

### 9.6 Data Validation Patterns

**Pattern: Non-empty text**
```npl
require(!text.isEmpty(), "Field cannot be empty");
```

**Pattern: Bounded numbers**
```npl
require(count > 0, "Count must be positive");
require(count <= 10, "Count cannot exceed 10");
```

**Pattern: Valid references**
```npl
require(lineNumber > 0, "Line number must be positive");
require(filesChanged.any(function(f) -> f.filePath == targetFile), "File not in PR");
```

**Pattern: Conditional validation**
```npl
if (isDraft) {
    require(false, "Cannot perform action on draft PR");
}
```

---

## 10. Notification System

### 10.1 Notification Definitions

```npl
package pullrequest;

// PR lifecycle events
notification PRReadyForReview(title: Text, author: Text) returns Unit;
notification PRConvertedToDraft(title: Text) returns Unit;
notification PRUpdated(title: Text, author: Text) returns Unit;
notification PRMerged(
    title: Text,
    sourceBranch: Text,
    targetBranch: Text,
    commitSha: Text,
    mergedBy: Text
) returns Unit;
notification PRClosed(title: Text, reason: Text, closedBy: Text) returns Unit;
notification PRReopened(title: Text, reopenedBy: Text) returns Unit;

// Review events
notification ReviewRequested(prTitle: Text, requestedBy: Text) returns Unit;
notification ReviewSubmitted(
    prTitle: Text,
    reviewer: Text,
    reviewType: ReviewType,
    commentCount: Number
) returns Unit;
notification CommentAdded(
    prTitle: Text,
    commenter: Text,
    filePath: Text,
    lineNumber: Number
) returns Unit;
notification AuthorResponded(
    prTitle: Text,
    author: Text,
    reviewId: Text
) returns Unit;

// File events
notification FilesAdded(fileCount: Number, prTitle: Text) returns Unit;

// Configuration events
notification RequiredApprovalsChanged(prTitle: Text, newCount: Number) returns Unit;
```

### 10.2 Notification Usage Patterns

**Pattern 1: Simple notifications**
```npl
permission[author] markReadyForReview() | draft {
    isDraft = false;
    become open;
    notify PRReadyForReview(title, getUsername(author));
};
```

**Pattern 2: Data-rich notifications**
```npl
permission[maintainer] merge(commitMessage: Text) | approved {
    var sha = generateMergeCommitSha();
    notify PRMerged(
        title,
        sourceBranch,
        targetBranch,
        sha,
        getUsername(maintainer)
    );
    become merged;
};
```

**Pattern 3: Conditional notifications**
```npl
permission[reviewers] submitReview(...) {
    // Logic...
    
    if (reviewType == ReviewType.APPROVE && getApprovalCount() >= requiredApprovals) {
        notify PRReadyToMerge(title, requiredApprovals);
    }
    
    notify ReviewSubmitted(title, getUsername(reviewers), reviewType, comments.size());
};
```

### 10.3 Notification Stream

The NPL Runtime exposes a notification stream that external systems can subscribe to:

```
ws://localhost:12000/notifications
```

Example event payload:
```json
{
  "notificationType": "PRMerged",
  "timestamp": "2024-12-01T14:00:00Z",
  "protocolId": "pr_abc123",
  "data": {
    "title": "Add JWT authentication",
    "sourceBranch": "feature/auth",
    "targetBranch": "main",
    "commitSha": "sha_abc123def456",
    "mergedBy": "dave@example.com"
  }
}
```

### 10.4 Integration Scenarios

**Slack Integration:**
```javascript
// Pseudocode for Slack bot
notificationStream.on('PRReadyForReview', (event) => {
  slack.postMessage({
    channel: '#code-review',
    text: `🔔 New PR ready: ${event.data.title} by ${event.data.author}`
  });
});
```

**Email Integration:**
```javascript
notificationStream.on('ReviewSubmitted', (event) => {
  if (event.data.reviewType === 'REQUEST_CHANGES') {
    email.send({
      to: event.data.author,
      subject: `Changes requested on PR: ${event.data.prTitle}`,
      body: `${event.data.reviewer} has requested changes...`
    });
  }
});
```

**Metrics Integration:**
```javascript
notificationStream.on('PRMerged', (event) => {
  metrics.recordMerge({
    pr: event.protocolId,
    duration: event.timestamp - event.data.createdAt,
    reviewCount: event.data.reviewCount
  });
});
```

---

## 11. Test Scenarios

### 11.1 Unit Test Structure

```npl
package pullrequest;

@test
function testCreatePR(test: Test) -> {
    var authorParty = createParty("alice@example.com", "developer");
    var reviewerParty = createParty("bob@example.com", "reviewer");
    var maintainerParty = createParty("dave@example.com", "maintainer");
    
    var files = listOf<FileChange>(
        FileChange("src/test.ts", FileChangeType.ADDED, 100, 0, optionalOf<Text>())
    );
    
    var pr = PullRequest[authorParty, listOf<Party>(reviewerParty), maintainerParty](
        "Test PR",
        "Test description",
        "feature/test",
        "main",
        files
    );
    
    test.assertEquals("Test PR", pr.title);
    test.assertEquals(true, pr.isDraft);
    test.assertEquals(1, pr.filesChanged.size());
};

@test
function testMarkReadyForReview(test: Test) -> {
    var pr = createTestPR();
    
    test.assertEquals(true, pr.isDraft);
    
    pr.markReadyForReview[pr.author]();
    
    test.assertEquals(false, pr.isDraft);
    test.expectNotifications(pr, PRReadyForReview, 1, function() returns Boolean -> true);
};

@test
function testSubmitApprovalReview(test: Test) -> {
    var pr = createTestPR();
    pr.markReadyForReview[pr.author]();
    pr.requestReview[pr.reviewers]();
    
    var review = pr.submitReview[pr.reviewers](
        ReviewType.APPROVE,
        "Looks good!",
        listOf<ReviewComment>()
    );
    
    test.assertEquals(1, pr.getReviewCount());
    test.assertEquals(1, pr.getApprovalCount());
};

@test
function testCannotMergeWithoutApprovals(test: Test) -> {
    var pr = createTestPR();
    pr.markReadyForReview[pr.author]();
    
    var result = test.assertFails(
        function() -> pr.merge[pr.maintainer]("Merge commit"),
        "Should fail without approvals"
    );
    
    test.assertTrue(result.contains("Insufficient approvals"));
};

@test
function testSuccessfulMerge(test: Test) -> {
    var pr = createTestPR();
    pr.markReadyForReview[pr.author]();
    pr.requestReview[pr.reviewers]();
    
    // Submit enough approvals
    pr.submitReview[pr.reviewers](ReviewType.APPROVE, "LGTM", listOf<ReviewComment>());
    
    // Assuming requiredApprovals = 1
    test.assertTrue(pr.canMerge());
    
    var result = pr.merge[pr.maintainer]("Merge feature");
    
    test.assertTrue(result.success);
    test.assertFalse(result.commitSha.isEmpty());
    test.expectNotifications(pr, PRMerged, 1, function() returns Boolean -> true);
};

@test
function testCannotUpdateAfterMerge(test: Test) -> {
    var pr = createTestPR();
    // ... merge the PR ...
    
    var result = test.assertFails(
        function() -> pr.updateDetails[pr.author]("New title", "New desc"),
        "Should not allow updates after merge"
    );
};

@test
function testMultipleReviewers(test: Test) -> {
    var reviewer1 = createParty("bob@example.com", "reviewer");
    var reviewer2 = createParty("carol@example.com", "reviewer");
    
    var pr = PullRequest[author, listOf<Party>(reviewer1, reviewer2), maintainer](
        "Multi-reviewer PR",
        "Testing multiple reviewers",
        "feature/multi",
        "main",
        testFiles
    );
    
    pr.markReadyForReview[author]();
    pr.requestReview[reviewer1]();
    
    pr.submitReview[reviewer1](ReviewType.APPROVE, "Good", listOf<ReviewComment>());
    pr.submitReview[reviewer2](ReviewType.REQUEST_CHANGES, "Needs work", listOf<ReviewComment>());
    
    test.assertEquals(2, pr.getReviewCount());
    test.assertEquals(1, pr.getApprovalCount());
    test.assertEquals(1, pr.getChangesRequestedCount());
    test.assertFalse(pr.canMerge());
};

@test
function testStateTransitions(test: Test) -> {
    var pr = createTestPR();
    
    // draft → open
    pr.markReadyForReview[pr.author]();
    test.assertEquals("open", pr.getCurrentState());
    
    // open → review_requested
    pr.requestReview[pr.reviewers]();
    test.assertEquals("review_requested", pr.getCurrentState());
    
    // review_requested → in_review
    pr.submitReview[pr.reviewers](ReviewType.COMMENT, "Suggestion", listOf<ReviewComment>());
    test.assertEquals("in_review", pr.getCurrentState());
    
    // in_review → approved
    pr.submitReview[pr.reviewers](ReviewType.APPROVE, "LGTM", listOf<ReviewComment>());
    test.assertEquals("approved", pr.getCurrentState());
    
    // approved → merged
    pr.merge[pr.maintainer]("Merge it");
    test.assertEquals("merged", pr.getCurrentState());
};

// Helper function for tests
function createTestPR() returns PullRequest -> {
    var author = createParty("alice@example.com", "developer");
    var reviewer = createParty("bob@example.com", "reviewer");
    var maintainer = createParty("dave@example.com", "maintainer");
    
    var files = listOf<FileChange>(
        FileChange("src/test.ts", FileChangeType.ADDED, 50, 0, optionalOf<Text>())
    );
    
    return PullRequest[author, listOf<Party>(reviewer), maintainer](
        "Test PR",
        "Description",
        "feature/test",
        "main",
        files
    );
};

function createParty(email: Text, role: Text) returns Party -> {
    // Creates a party with specific claims
    // Implementation depends on NPL testing utilities
};
```

### 11.2 Integration Test Scenarios

**Scenario 1: Happy Path - Full PR Lifecycle**
```
1. Author creates PR in draft state
2. Author adds files
3. Author marks ready for review
4. Reviewer requests review
5. Reviewer submits APPROVE review
6. Maintainer merges PR
7. Verify PR is in merged state
8. Verify notifications were emitted
9. Verify audit trail is complete
```

**Scenario 2: Changes Requested Flow**
```
1. Author creates and marks PR ready
2. Reviewer submits REQUEST_CHANGES
3. Verify PR moves to changes_requested state
4. Author responds to feedback
5. Author adds new commits
6. Reviewer submits new APPROVE review
7. PR moves to approved state
8. Maintainer merges
```

**Scenario 3: Multiple Reviewers**
```
1. Create PR with 3 reviewers
2. Set required approvals to 2
3. First reviewer approves
4. Second reviewer requests changes
5. Verify PR cannot be merged yet
6. Author addresses feedback
7. Second reviewer approves
8. Verify PR can now be merged
9. Maintainer merges successfully
```

**Scenario 4: Authorization Failures**
```
1. Create PR
2. Attempt to merge as author (should fail)
3. Attempt to review as author (should fail)
4. Attempt to update as reviewer (should fail)
5. Verify all fail with authorization errors
```

**Scenario 5: State Guard Enforcement**
```
1. Create PR
2. Attempt to merge while in draft (should fail)
3. Mark ready and request review
4. Attempt to merge while in review_requested (should fail)
5. Verify state guards are enforced
```

### 11.3 Performance Test Cases

**Test 1: Large PR Creation**
```
- Create PR with 100+ file changes
- Measure creation time
- Verify all files are stored correctly
- Target: < 500ms
```

**Test 2: Many Reviews**
```
- Create PR with 20 reviewers
- Have each submit a review
- Measure time to process all reviews
- Target: < 2s total
```

**Test 3: Concurrent Updates**
```
- Multiple reviewers submit reviews simultaneously
- Verify all reviews are captured
- Verify no race conditions
- Verify state transitions are correct
```

---

## 12. Implementation Roadmap

### 12.1 Phase 1: Core Protocols (2 hours)

**Tasks:**
1. Set up NPL project structure
2. Define all enums and structs in Types.npl
3. Implement PullRequest protocol skeleton
4. Implement Review protocol
5. Add basic state machines
6. Test protocol creation and basic transitions

**Deliverables:**
- Working protocol definitions
- Can create PRs and Reviews
- Basic state transitions work
- Unit tests pass

### 12.2 Phase 2: Permissions and Authorization (1.5 hours)

**Tasks:**
1. Implement all author permissions
2. Implement all reviewer permissions
3. Implement all maintainer permissions
4. Add party-based access controls
5. Add state guards to permissions
6. Test authorization rules

**Deliverables:**
- All permissions implemented
- Authorization working correctly
- State guards enforced
- Authorization tests pass

### 12.3 Phase 3: Business Logic (1 hour)

**Tasks:**
1. Add validation rules (require statements)
2. Implement query functions
3. Add computed properties
4. Implement helper functions
5. Test business rules

**Deliverables:**
- All validation rules enforced
- Query functions working
- Business logic tests pass

### 12.4 Phase 4: Notifications and Integration (1 hour)

**Tasks:**
1. Define all notifications
2. Add notify calls to permissions
3. Test notification emission
4. Document notification payloads

**Deliverables:**
- Notifications defined
- Notifications emitted correctly
- Integration examples documented

### 12.5 Phase 5: Testing and Documentation (0.5 hours)

**Tasks:**
1. Write comprehensive unit tests
2. Create integration test scenarios
3. Generate API documentation
4. Create usage examples

**Deliverables:**
- Full test coverage
- API documentation
- Usage guide
- Demo script

---

## 13. Demo Script

### 13.1 Preparation

**Setup:**
```bash
# Install NPL CLI
brew install NoumenaDigital/tools/npl

# Create project
npl init --projectDir pr-workflow-demo

# Copy in protocol files
cp *.npl pr-workflow-demo/src/

# Start NPL Runtime
cd pr-workflow-demo
docker-compose up -d

# Deploy code
npl deploy --target local --clear
```

**Create test users:**
```bash
# In development mode, NPL has embedded OIDC
# Create users: alice (author), bob (reviewer), dave (maintainer)
# Users are pre-seeded in dev mode
```

### 13.2 Live Demo Flow

**Act 1: Creating and Preparing a PR**

```bash
# 1. Alice creates a PR (as draft)
curl -X POST http://localhost:12000/api/pullrequest/PullRequest \
  -H "Authorization: Bearer <alice_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Add user authentication system",
    "description": "Implements JWT-based authentication with refresh tokens",
    "sourceBranch": "feature/auth",
    "targetBranch": "main",
    "filesChanged": [
      {
        "filePath": "src/auth/jwt.ts",
        "changeType": "ADDED",
        "linesAdded": 150,
        "linesDeleted": 0
      }
    ],
    ...parties...
  }'

# Response: { "protocolId": "pr_123", "state": "draft" }

# 2. Alice adds more files
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/addFiles \
  -H "Authorization: Bearer <alice_token>" \
  -d '{
    "newFiles": [
      {
        "filePath": "src/auth/refresh.ts",
        "changeType": "ADDED",
        "linesAdded": 75,
        "linesDeleted": 0
      }
    ]
  }'

# 3. Alice marks it ready for review
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/markReadyForReview \
  -H "Authorization: Bearer <alice_token>"

# Response: { "newState": "open" }
# Notification: PRReadyForReview emitted
```

**Act 2: Code Review Process**

```bash
# 4. Bob (reviewer) requests review
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/requestReview \
  -H "Authorization: Bearer <bob_token>"

# Response: { "newState": "review_requested" }

# 5. Bob submits a review with comments
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/submitReview \
  -H "Authorization: Bearer <bob_token>" \
  -d '{
    "reviewType": "REQUEST_CHANGES",
    "summary": "Great start! A few security concerns to address.",
    "comments": [
      {
        "filePath": "src/auth/jwt.ts",
        "lineNumber": 45,
        "commentText": "Should add rate limiting here to prevent token generation abuse",
        "createdAt": "2024-12-01T11:00:00Z",
        "commentId": "c1"
      },
      {
        "filePath": "src/auth/refresh.ts",
        "lineNumber": 22,
        "commentText": "Consider adding expiration validation for refresh tokens",
        "createdAt": "2024-12-01T11:01:00Z",
        "commentId": "c2"
      }
    ]
  }'

# Response: { "reviewId": "rev_456", "newState": "changes_requested" }
# Notification: ReviewSubmitted with REQUEST_CHANGES

# 6. Check PR status
curl -X GET http://localhost:12000/api/pullrequest/PullRequest/pr_123/getSummary \
  -H "Authorization: Bearer <alice_token>"

# Response shows: state="changes_requested", reviewCount=1, changesRequestedCount=1

# 7. Alice responds to feedback
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/respondToReview \
  -H "Authorization: Bearer <alice_token>" \
  -d '{
    "reviewId": "rev_456",
    "response": "Great catches! Ive addressed both issues in the latest commits."
  }'

# 8. Bob reviews again and approves
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/submitReview \
  -H "Authorization: Bearer <bob_token>" \
  -d '{
    "reviewType": "APPROVE",
    "summary": "Perfect! All concerns addressed. LGTM! 🚀",
    "comments": []
  }'

# Response: { "newState": "approved", "approvalCount": 1 }
```

**Act 3: Merging**

```bash
# 9. Check if PR can be merged
curl -X GET http://localhost:12000/api/pullrequest/PullRequest/pr_123/canMerge \
  -H "Authorization: Bearer <dave_token>"

# Response: { "canMerge": true }

# 10. Dave (maintainer) merges the PR
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/merge \
  -H "Authorization: Bearer <dave_token>" \
  -d '{
    "commitMessage": "Merge pull request #123: Add JWT authentication system"
  }'

# Response: {
#   "success": true,
#   "commitSha": "sha_abc123def456",
#   "message": "Successfully merged",
#   "newState": "merged"
# }
# Notification: PRMerged emitted

# 11. Verify final state
curl -X GET http://localhost:12000/api/pullrequest/PullRequest/pr_123/getSummary \
  -H "Authorization: Bearer <dave_token>"

# Shows: state="merged", all metrics, merge timestamp
```

**Act 4: Authorization Demo (Shows NPL's Security)**

```bash
# Try to merge as author (should fail)
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/merge \
  -H "Authorization: Bearer <alice_token>" \
  -d '{ "commitMessage": "Trying to merge myself" }'

# Response: 403 Forbidden - "User does not match party 'maintainer'"

# Try to review as author (should fail)
curl -X POST http://localhost:12000/api/pullrequest/PullRequest/pr_123/submitReview \
  -H "Authorization: Bearer <alice_token>" \
  -d '{ "reviewType": "APPROVE", "summary": "Approving my own PR", "comments": [] }'

# Response: 403 Forbidden - "User does not match party 'reviewers'"
```

### 13.3 Key Talking Points

**While demoing, highlight:**

1. **NPL's Unique Features:**
   - "Notice how authorization is built into the language - we define parties in the protocol signature"
   - "State transitions are explicit with `become` keyword - NPL tracks these automatically"
   - "The `@api` annotation auto-generates REST endpoints - no boilerplate needed"

2. **Authorization Model:**
   - "Each permission specifies WHO can call it with `permission[party]`"
   - "The NPL Runtime enforces this using JWT claims matching"
   - "State guards with `| state` restrict WHEN actions can happen"

3. **Developer Experience:**
   - "Coming from TypeScript/Python/Java, NPL's syntax is familiar but domain-specific"
   - "Augment Code helped me understand NPL's party model by referencing the documentation"
   - "The compiler catches authorization and state errors before runtime"

4. **Built-in Features:**
   - "NPL automatically persists all data - no ORM needed"
   - "Every change is versioned for complete audit trail"
   - "Notifications provide integration points for external systems"

5. **Real-World Value:**
   - "This 300-line NPL program replaces thousands of lines of traditional backend code"
   - "Authorization logic is co-located with business logic - easier to maintain"
   - "State machines prevent invalid transitions - impossible to reach bad states"

---

## 14. Success Metrics

### 14.1 Demo Effectiveness

**Measures:**
- Time to understand NPL basics: < 30 minutes
- Time to modify working code: < 15 minutes
- Audience can explain party-based auth: 80%+
- Audience understands state machines: 90%+
- Questions about Augment Code's role: Expected

### 14.2 Technical Quality

**Code Quality:**
- All state transitions valid: 100%
- Authorization rules enforced: 100%
- No runtime errors: 100%
- Test coverage: 80%+

**Performance:**
- PR creation: < 200ms
- Review submission: < 150ms
- State transitions: < 50ms
- API response time: < 100ms

### 14.3 Learning Outcomes

**For the audience:**
- Understands NPL's authorization model
- Sees value of state machines for workflows
- Appreciates language-level security
- Interested in trying NPL themselves
- Impressed with Augment Code's capability

**For Augment Code:**
- Demonstrates capability with new languages
- Shows value of documentation-driven assistance
- Highlights code generation and suggestions
- Proves adaptability to domain-specific languages

---

## 15. Extensions and Future Work

### 15.1 Additional Features (Optional)

**If time permits or for follow-up demos:**

1. **PR Labels and Milestones**
```npl
enum PRLabel {
    BUG_FIX,
    FEATURE,
    DOCUMENTATION,
    BREAKING_CHANGE,
    HOT_FIX
};

protocol[...] PullRequest(...) {
    var labels: List<PRLabel> = listOf<PRLabel>();
    var milestone: Optional<Text> = optionalOf<Text>();
    
    permission[author, maintainer] addLabel(label: PRLabel) {
        labels = labels.concat(listOf<PRLabel>(label));
    };
};
```

2. **Merge Conflict Detection**
```npl
struct MergeCheck {
    hasConflicts: Boolean,
    conflictingFiles: List<Text>,
    canAutoMerge: Boolean
};

function checkMergeability() returns MergeCheck {
    // Logic to detect conflicts
};
```

3. **Review Assignment**
```npl
permission[maintainer] assignReviewer(newReviewer: Party) {
    // Add to reviewers party
    reviewers = reviewers.concat(listOf<Party>(newReviewer));
    notify ReviewerAssigned(title, getUsername(newReviewer));
};
```

4. **Draft Comments**
```npl
struct DraftComment {
    filePath: Text,
    lineNumber: Number,
    commentText: Text,
    savedAt: DateTime
};

var draftComments: List<DraftComment> = listOf<DraftComment>();

permission[reviewers] saveDraftComment(...) {
    // Save without submitting
};
```

5. **Approval Policies**
```npl
enum ApprovalPolicy {
    REQUIRE_ALL,
    REQUIRE_MAJORITY,
    REQUIRE_MINIMUM
};

var approvalPolicy: ApprovalPolicy = ApprovalPolicy.REQUIRE_MINIMUM;
```

### 15.2 Integration Ideas

**CI/CD Integration:**
```npl
struct BuildStatus {
    passed: Boolean,
    buildUrl: Text,
    timestamp: DateTime
};

var lastBuildStatus: Optional<BuildStatus> = optionalOf<BuildStatus>();

permission[*ciSystem] updateBuildStatus(status: BuildStatus) {
    lastBuildStatus = optionalOf<BuildStatus>(status);
    
    if (!status.passed) {
        // Prevent merge on failed build
        notify BuildFailed(title, status.buildUrl);
    }
};
```

**Jira Integration:**
```npl
var jiraTicket: Optional<Text> = optionalOf<Text>();

permission[author] linkJiraTicket(ticketId: Text) {
    require(ticketId.startsWith("PROJ-"), "Invalid Jira ticket format");
    jiraTicket = optionalOf<Text>(ticketId);
};
```

**Code Coverage Tracking:**
```npl
struct CoverageReport {
    percentage: Number,
    linesAdded: Number,
    linesCovered: Number,
    reportUrl: Text
};

var coverageReport: Optional<CoverageReport> = optionalOf<CoverageReport>();
```

### 15.3 Advanced NPL Features

**Pattern 1: Protocol Composition**
```npl
// A repository protocol that creates PRs
protocol[owner] Repository(var name: Text) {
    var pullRequests: List<PullRequest> = listOf<PullRequest>();
    
    permission[owner] createPR(...) returns PullRequest {
        var pr = PullRequest[author, reviewers, owner](...);
        pullRequests = pullRequests.concat(listOf<PullRequest>(pr));
        return pr;
    };
};
```

**Pattern 2: Obligations (Future NPL Feature)**
```npl
// If NPL adds obligations (like permissions but required)
obligation[author] addressFeedback() | changes_requested {
    // Author must respond within timeframe
};
```

**Pattern 3: Complex State Guards**
```npl
permission[maintainer] forceMerge() | in_review, changes_requested {
    require(hasEmergencyOverride(), "Emergency override required");
    become merged;
};
```

---

## 16. Troubleshooting Guide

### 16.1 Common Issues

**Issue: "User does not match party"**
- **Cause:** JWT token claims don't match party definitions
- **Fix:** Verify JWT contains correct role/user_id claims
- **Example:** Party expects `role: ["developer"]` but JWT has `role: ["engineer"]`

**Issue: "Permission not allowed in current state"**
- **Cause:** Trying to call permission with state guard from wrong state
- **Fix:** Check current state with `getSummary()`, transition to allowed state first
- **Example:** Trying to `merge()` from `in_review` state (requires `approved`)

**Issue: "require clause failed"**
- **Cause:** Validation rule in permission failed
- **Fix:** Check require error message, provide valid input
- **Example:** Empty title, source == target branch, etc.

**Issue: NPL compilation errors**
- **Cause:** Syntax errors in NPL code
- **Fix:** Check NPL language documentation, use IDE plugin
- **Common:** Missing semicolons, incorrect type annotations, undefined functions

### 16.2 Debugging Tips

**Enable verbose logging:**
```yaml
# In docker-compose.yml
environment:
  - LOG_LEVEL=DEBUG
```

**Check protocol state:**
```bash
# Get full protocol instance
curl http://localhost:12000/api/pullrequest/PullRequest/{id}
```

**View notification stream:**
```bash
# Listen to all events
wscat -c ws://localhost:12000/notifications
```

**Check party claims:**
```bash
# Decode JWT token
echo $TOKEN | jwt decode -
```

### 16.3 Performance Optimization

**For large PRs:**
- Batch file additions instead of one-by-one
- Limit comment count per review
- Use pagination for query results

**For many reviewers:**
- Consider minimum approval count instead of all
- Use review request assignments instead of all reviewers

**Database tuning:**
```yaml
# PostgreSQL configuration for NPL Runtime
POSTGRES_MAX_CONNECTIONS=100
POSTGRES_SHARED_BUFFERS=256MB
```

---

## 17. Conclusion

This specification provides a comprehensive blueprint for building a GitHub-style Pull Request workflow system in NPL. The system demonstrates:

✅ **NPL's Core Strengths:**
- Party-based authorization model
- State machine workflows
- Auto-generated REST APIs
- Built-in persistence and audit trails

✅ **Developer-Friendly Design:**
- Familiar domain (code review)
- Clear state transitions
- Comprehensive validation
- Rich notification system

✅ **Augment Code Showcase:**
- Working with unfamiliar language
- Documentation-driven development
- Code generation assistance
- Understanding domain-specific constructs

✅ **Production-Ready Features:**
- Complete authorization model
- Full CRUD operations
- Integration hooks
- Comprehensive testing

**Next Steps:**
1. Review this specification
2. Load NPL documentation into Augment Code context
3. Begin implementation following the roadmap
4. Build incrementally with AI assistance
5. Test thoroughly at each phase
6. Prepare demo script and talking points
7. Practice demo flow
8. Share with the community!

---

## Appendix A: Quick Reference

### NPL Syntax Cheatsheet

```npl
// Protocol definition
protocol[party1, party2] ProtocolName(params) { ... }

// State definition
initial state stateName;
final state stateName;

// Permission
permission[party] actionName(params) | stateGuard { ... }

// State transition
become newState;

// Validation
require(condition, "error message");

// Notification
notify NotificationName(params);

// Function
function functionName(params) returns ReturnType -> expression;

// Struct
struct StructName {
    field1: Type,
    field2: Type
};

// Enum
enum EnumName {
    VARIANT1,
    VARIANT2
};
```

### Common NPL Patterns

```npl
// Optional values
var optional: Optional<Text> = optionalOf<Text>();
optional = optionalOf<Text>("value");
var value = optional.getOrFail();

// Lists
var list: List<Number> = listOf<Number>();
list = list.concat(listOf<Number>(1, 2, 3));
var size = list.size();
var filtered = list.filter(function(n) -> n > 0);

// DateTime
var now = now();
var timestamp = now.toMillis();

// Party operations
var username = getUsername(party);
var claims = party.claims();
```

### API Patterns

```bash
# Create protocol
POST /api/package/Protocol

# Call permission
POST /api/package/Protocol/{id}/permissionName

# Call function
GET /api/package/Protocol/{id}/functionName

# Get all instances
GET /api/package/Protocol

# Get single instance
GET /api/package/Protocol/{id}
```

---

**Document Version:** 1.0  
**Last Updated:** 2024-12-01  
**Status:** Ready for Implementation  
**Estimated LOC:** ~800 lines of NPL code  
**Complexity:** Intermediate  
**Prerequisites:** NPL CLI, Docker, Basic REST API knowledge
