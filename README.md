# NPL Pull Request Workflow System

A complete GitHub-style Pull Request workflow system implemented in NPL (Noumena Protocol Language).

## 🎯 Overview

This project implements a production-ready PR workflow system with:

- **8-state state machine** (draft → open → review_requested → in_review → changes_requested → approved → merged/closed)
- **12 permissions** across 3 parties (author, reviewers, maintainer)
- **Full review system** with APPROVE, REQUEST_CHANGES, and COMMENT types
- **Merge validation** with configurable approval requirements
- **Comprehensive notifications** for all major events
- **Complete test suite** with 20+ test functions

## 📁 Project Structure

```
.
├── pullrequest/
│   ├── Types.npl                  # Data structures (152 lines)
│   ├── Review.npl                 # Review protocol (102 lines)
│   ├── PullRequest.npl            # Main PR protocol (435 lines)
│   ├── Notifications.npl          # Notification events (177 lines)
│   ├── PullRequestTests.npl       # Test suite (572 lines)
│   └── TEST_SCENARIOS.md          # Test documentation
├── pr-workflow-spec.md            # Complete specification
├── TESTING_GUIDE.md               # How to test this system
├── PROGRESS.md                    # Implementation progress
└── README.md                      # This file
```

## 🚀 Quick Start

### Option 1: Test with NPL CLI (Recommended)

```bash
# Install NPL CLI
brew install NoumenaDigital/tools/npl

# Initialize project
npl init --project-dir my-pr-workflow

# Copy the code
cp -r pullrequest/ my-pr-workflow/src/main/npl/

# Run tests
cd my-pr-workflow
npl test src/main/npl/pullrequest/PullRequestTests.npl
```

### Option 2: Use IntelliJ IDEA with NPL-Dev Plugin

1. Install IntelliJ IDEA
2. Install "Noumena Protocol Language (NPL)" plugin
3. Open this project
4. Right-click `PullRequestTests.npl` → Run Tests

### Option 3: Deploy to NOUMENA Cloud

1. Sign up at https://cloud.noumenadigital.com/
2. Create a new application
3. Upload the `pullrequest/` directory
4. Test via the auto-generated REST API

**See [TESTING_GUIDE.md](TESTING_GUIDE.md) for detailed instructions.**

## 📋 Features

### Core Protocols

**PullRequest Protocol** (`PullRequest.npl`)
- 8-state lifecycle management
- 5 author permissions (updateDetails, addFiles, markReadyForReview, convertToDraft, respondToReview)
- 3 reviewer permissions (requestReview, submitReview, addComment)
- 4 maintainer permissions (merge, close, reopen, setRequiredApprovals)
- 6 query functions (getReviewCount, getApprovalCount, canMerge, etc.)

**Review Protocol** (`Review.npl`)
- 2-state lifecycle (submitted → dismissed)
- Support for APPROVE, REQUEST_CHANGES, and COMMENT types
- File-level and line-level comments

**Data Structures** (`Types.npl`)
- 3 enums: ReviewType, FileChangeType, MergeStrategy
- 9 structs: FileChange, ReviewComment, PRSummary, ReviewSummary, MergeResult, PRMetrics, Contributor, Branch, MergeConflict

**Notifications** (`Notifications.npl`)
- 12 notification events covering all major PR lifecycle events

### Test Coverage

**20+ Test Functions** organized into 5 categories:
1. Unit Tests - PR Creation (5 tests)
2. Unit Tests - Review Submission (4 tests)
3. Unit Tests - Merge Validation (4 tests)
4. Integration Tests - Complex Scenarios (6 tests)
5. Integration Tests - Authorization (3 tests)

**100% Coverage:**
- ✅ All 12 permissions
- ✅ All 13 state transitions
- ✅ All 12 notification types
- ✅ Authorization rules
- ✅ Validation rules

## 🔧 How It Works

### Creating a Pull Request

```npl
// In NPL code
var pr = PullRequest[author, reviewers, maintainer](
    "Add new feature",
    "This PR implements feature X",
    "feature/new-feature",
    "main",
    filesChanged
);
```

### Via REST API

```bash
POST /pullrequest/PullRequest
{
  "parties": {
    "author": {"claims": {"email": ["alice@example.com"]}},
    "reviewers": [{"claims": {"email": ["bob@example.com"]}}],
    "maintainer": {"claims": {"email": ["dave@example.com"]}}
  },
  "arguments": {
    "title": "Add new feature",
    "description": "This PR implements feature X",
    "sourceBranch": "feature/new-feature",
    "targetBranch": "main",
    "filesChanged": [...]
  }
}
```

### Workflow Example

```npl
// 1. Create PR in draft state
var pr = PullRequest[author, reviewers, maintainer](...);

// 2. Mark ready for review
pr.markReadyForReview[author]();

// 3. Request review
pr.requestReview[reviewers]();

// 4. Submit approval
pr.submitReview[reviewers](ReviewType.APPROVE, "LGTM", listOf<ReviewComment>());

// 5. Merge
pr.merge[maintainer]("Merge feature branch");
```

## 📊 State Machine

```
draft (initial)
  ↓ markReadyForReview
open
  ↓ requestReview
review_requested
  ↓ submitReview (COMMENT)
in_review
  ↓ submitReview (APPROVE with sufficient approvals)
approved
  ↓ merge
merged (final)
```

Alternative paths:
- `open` → `draft` (convertToDraft)
- `in_review` → `changes_requested` (submitReview with REQUEST_CHANGES)
- `any state` → `closed` (close)
- `closed` → `open` (reopen)

## 🔐 Authorization

NPL provides built-in authorization through party-based access control:

- **Author** - Can update PR details, add files, mark ready, convert to draft
- **Reviewers** - Can request review, submit reviews, add comments
- **Maintainer** - Can merge, close, reopen, set approval requirements

Each party is matched against JWT token claims at runtime.

## 📚 Documentation

- **[TESTING_GUIDE.md](TESTING_GUIDE.md)** - Complete guide to testing the system
- **[TEST_SCENARIOS.md](pullrequest/TEST_SCENARIOS.md)** - Detailed test scenarios
- **[pr-workflow-spec.md](pr-workflow-spec.md)** - Full specification
- **[PROGRESS.md](PROGRESS.md)** - Implementation progress and decisions

## 🛠️ Technology Stack

- **Language**: NPL (Noumena Protocol Language)
- **Runtime**: NPL Runtime (includes REST API generation, persistence, authorization)
- **Database**: PostgreSQL (or H2 for testing)
- **API**: Auto-generated REST API with OpenAPI/Swagger documentation

## 📈 Implementation Stats

- **Total Lines of Code**: ~1,800 lines
- **Number of Files**: 5 NPL files + documentation
- **Test Functions**: 20+
- **Permissions**: 12
- **State Transitions**: 13
- **Notification Types**: 12
- **Development Time**: 1 day (6 PRs)

## 🤝 Contributing

This is a reference implementation. To extend it:

1. Add new permissions to `PullRequest.npl`
2. Add new notification types to `Notifications.npl`
3. Add tests to `PullRequestTests.npl`
4. Update documentation

## 📖 Learn More

- **NPL Documentation**: https://documentation.noumenadigital.com/
- **NPL Examples**: https://documentation.noumenadigital.com/what-is-npl/#npl-examples
- **Community Forum**: https://community.noumenadigital.com/

## 📝 License

This is a demonstration project. See individual file headers for licensing information.

## 🎓 Educational Value

This project demonstrates:

- **Protocol-oriented programming** in NPL
- **State machine design** for workflow systems
- **Authorization modeling** with party-based access control
- **API-first development** with auto-generated REST APIs
- **Test-driven development** with comprehensive test coverage
- **Event-driven architecture** with notifications

Perfect for learning NPL or understanding how to model complex workflows!

---

**Built with ❤️ using NPL (Noumena Protocol Language)**
