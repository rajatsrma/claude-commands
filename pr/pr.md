---
allowed-tools: Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh pr status:*), Bash(gh pr list:*), Bash(git log:*), Bash(find:*), Bash(grep:*)
argument-hint: "<PR_NUMBER>  e.g. `/pr-review 123`"
description: "AI-powered comprehensive PR review: analyzes correctness, security, performance, architecture. Provides categorized findings with inline comments and actionable recommendations."
---

## Initial Data Collection
```bash
# Gather comprehensive PR context
PR_DATA=$(gh pr view $ARGUMENTS --json title,body,author,baseRefName,headRefName,url,isDraft,mergeable,reviewDecision,additions,deletions,changedFiles)
PR_DIFF=$(gh pr diff $ARGUMENTS)
PR_STATUS=$(gh pr status $ARGUMENTS)
PR_FILES=$(gh pr view $ARGUMENTS --json files | jq -r '.files[].path')
```

## Context Analysis
!`gh pr view $ARGUMENTS --json title,body,author,baseRefName,headRefName,url,isDraft,mergeable,reviewDecision,additions,deletions,changedFiles`
!`gh pr diff $ARGUMENTS`
!`gh pr status $ARGUMENTS`

**Reference Materials:**
- Project guidelines: `CLAUDE.md` (root and `$HOME/.claude/CLAUDE.md`)
- Recent commits for context: `git log --oneline -10`
- Related files in the same directories as changes

---

## Enhanced Review Framework

### 🎯 **Impact Analysis**
1. **Blast Radius Assessment** - How many systems/users affected?
2. **Breaking Changes** - API changes, schema migrations, config updates
3. **Dependencies** - New packages, version bumps, potential conflicts
4. **Rollback Complexity** - How easily can this be reverted?

### 🔍 **Deep Code Analysis**

#### **Critical Security Review**
- **Injection Vulnerabilities**: SQL, NoSQL, Command, XSS, LDAP
- **Authentication/Authorization**: Proper token validation, role checks
- **Data Exposure**: Sensitive data in logs, responses, or client-side
- **Cryptography**: Secure algorithms, key management, entropy
- **Input Validation**: Sanitization, type checking, boundary validation

#### **Performance & Scalability**
- **Database Operations**: N+1 queries, missing indexes, bulk operations
- **Memory Management**: Leaks, excessive allocations, garbage collection
- **Caching Strategy**: Cache invalidation, TTL, cache stampede
- **Concurrency**: Race conditions, deadlocks, thread safety
- **Algorithm Complexity**: Big O analysis, optimization opportunities

#### **Reliability & Resilience**
- **Error Handling**: Graceful degradation, proper exception handling
- **Resource Management**: Connection pools, file handles, cleanup
- **Timeout Configuration**: Network calls, database queries, external APIs
- **Circuit Breakers**: Failure isolation, retry logic with backoff
- **Observability**: Logging, metrics, tracing for debugging

#### **Code Quality & Maintainability**
- **Architecture Patterns**: SOLID principles, design patterns, separation of concerns
- **Function Complexity**: Cyclomatic complexity, function length, parameter count
- **Test Coverage**: Unit tests, integration tests, edge cases
- **Documentation**: Code comments, API docs, README updates
- **Code Duplication**: DRY violations, shared utilities

---

## Severity Classification System

### 🚨 **CRITICAL** (Security/Data/System Impact)
- Security vulnerabilities (injection, auth bypass)
- Data loss or corruption risks
- System crashes or unavailability
- Breaking changes without proper migration
- Resource exhaustion (DoS potential)

### ⚠️ **HIGH** (Performance/Memory/Production Impact)
- Memory leaks or excessive memory usage
- Performance bottlenecks affecting user experience
- Missing resource cleanup (connections, files)
- Complex functions (>50 lines, >10 parameters)
- Missing error handling for critical paths

### 🔶 **MEDIUM** (Code Quality/Architecture)
- Code smells and maintainability issues
- Test coverage gaps for important features
- Architecture drift or pattern violations
- Missing documentation for public APIs
- Moderate performance inefficiencies

### 🔵 **LOW** (Style/Enhancement)
- Naming convention inconsistencies
- Formatting and style issues
- Minor refactoring opportunities
- Non-critical spelling mistakes
- Suggestion for future improvements

---

## Review Process Workflow

### Phase 1: Automated Analysis
```bash
echo "🔍 Analyzing PR #$ARGUMENTS..."
echo "📊 Files changed: $(echo "$PR_DATA" | jq -r '.changedFiles')"
echo "➕ Additions: $(echo "$PR_DATA" | jq -r '.additions')"
echo "➖ Deletions: $(echo "$PR_DATA" | jq -r '.deletions')"
```

### Phase 2: Comprehensive Review
For each finding, generate structured output:

```
🚨 CRITICAL | File: src/auth/login.js:42-45
Issue: SQL injection vulnerability in user authentication
Impact: Attackers can bypass authentication and access any user account
Code: 
```javascript
const query = `SELECT * FROM users WHERE email = '${email}'`;
```
Recommendation:
```javascript
const query = 'SELECT * FROM users WHERE email = ?';
const result = await db.query(query, [email]);
```
```

### Phase 3: Review Summary Generation
```markdown
## 📋 PR Review Summary for #$ARGUMENTS

### 🎯 **Purpose & Context**
[One-to-three sentence description of PR overall assessment]

### 📊 **Impact Assessment**
- **Blast Radius**: [Low/Medium/High/Critical]
- **Breaking Changes**: [Yes/No - with details if yes]
- **Rollback Complexity**: [Simple/Moderate/Complex]

### 🔍 **Issue Distribution**
Critical: X • High: Y • Medium: Z • Low: W

### 🎯 **Key Concerns**
[Top 3-5 most important issues that need immediate attention]

### ✅ **Positive Highlights**
[Good practices, well-implemented features, improvements]

### 🚦 **Final Decision**
**APPROVE** | **REQUEST CHANGES** | **REJECT**

[Optional: Overall recommendation and next steps]
```

---

## User Interaction & Workflow

### Review Presentation
1. **Display comprehensive summary** with categorized findings
2. **Show top 5 critical/high issues** for quick assessment
3. **Present final recommendation** with reasoning

### User Options
```
🔄 Available Actions:
• Type 'go' to submit review as-is
• Type 'edit' to modify findings or decision
• Type 'approve' to override decision to APPROVE
• Type 'reject' to override decision to REJECT  
• Type 'critical-only' to submit only Critical/High issues
• Type 'preview' to see what will be posted to GitHub
• Type 'cancel' to abort without posting
```

### Submission Process
```bash
# Determine review action based on severity
if [[ $CRITICAL_COUNT -gt 0 || $HIGH_COUNT -gt 3 ]]; then
    ACTION="--request-changes"
elif [[ $HIGH_COUNT -gt 0 || $MEDIUM_COUNT -gt 5 ]]; then
    ACTION="--comment"
else
    ACTION="--approve"
fi

# Override if user specified
[[ "$USER_DECISION" == "approve" ]] && ACTION="--approve"
[[ "$USER_DECISION" == "reject" ]] && ACTION="--request-changes"

# Submit main review
gh pr review $ARGUMENTS $ACTION --body "$SUMMARY_BODY"

# Add inline comments for each finding
for finding in "${FINDINGS[@]}"; do
    gh pr review $ARGUMENTS --comment "$finding.comment" \
        --path "$finding.file" --line "$finding.line"
done
```

---

## Advanced Analysis Features

### 🔗 **Cross-Reference Analysis**
- Check if changes align with recent related PRs
- Identify potential merge conflicts with other open PRs
- Verify consistency with established patterns in codebase

### 📈 **Metrics Integration**
- Estimate code coverage impact
- Calculate complexity metrics (cyclomatic, cognitive)
- Assess technical debt introduction/reduction

### 🔍 **Contextual Intelligence**
- Consider PR size and scope for review depth
- Adapt review criteria based on file types and frameworks
- Account for draft status and review urgency

### 🚀 **Post-Review Actions**
```bash
# After successful submission
echo "✅ Review posted successfully!"
echo "🔗 View PR: $(echo "$PR_DATA" | jq -r '.url')"
echo "📊 Posted $TOTAL_COMMENTS inline comments"
echo "🎯 Decision: $FINAL_DECISION"
```

---
