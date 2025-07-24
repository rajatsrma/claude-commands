---
allowed-tools: Bash(*)
argument-hint: "<PR_NUMBER>  e.g. `/pr-review 123`"
description: "Fully automated, hybrid PR review. Synthesizes precise programmatic analysis with a framework-guided architectural assessment from an AI Staff Engineer. v3 with robust error handling and cross-platform compatibility - PROPERLY FIXED."
---

## SAFETY CONSTRAINTS

### safety_rules
CRITICAL SAFETY RULES - NEVER VIOLATE THESE:
✅ ALLOWED (Read-only operations):
  - gh pr view, gh pr diff, gh pr status
  - gh api GET requests
  - gh repo view
  - jq, awk, grep, sed for data processing
  - echo, cat, variable assignments, function definitions
  
✅ ALLOWED (Safe PR Review operations):
  - gh pr review with --approve, --comment, or --request-changes
  - gh api POST to the pull request comments endpoint
  
❌ STRICTLY FORBIDDEN:
  - Any file deletion (rm) or modification (>, >>)
  - Any git operations (push, commit, checkout, merge)
  - Any gh operations that modify repository state (except PR reviews/comments)
  - Any destructive operations whatsoever.

### persona
You are a world-class AI system responsible for orchestrating a hybrid code review. 
You will execute a series of steps precisely as defined. In the final phase, you will adopt the persona of a Staff Engineer to synthesize the findings.

## PHASE 1: DATA COLLECTION & VALIDATION

```bash
#!/usr/bin/env bash
set -euo pipefail

ARGUMENTS="$1"
[[ -z "${ARGUMENTS:-}" ]] && { echo "Usage: /pr-review <PR_NUMBER>"; exit 1; }

echo "=== 🔍 Phase 1: Starting Data Collection for PR #$ARGUMENTS ==="

# Validate PR exists and is accessible
if ! gh pr view "$ARGUMENTS" >/dev/null 2>&1; then
    echo "❌ Error: PR #$ARGUMENTS not found or not accessible."
    exit 1
fi

# Collect repository information first (always works)
REPO_FULL_NAME=$(gh repo view --json nameWithOwner -q .nameWithOwner)
FULL_DIFF=$(gh pr diff "$ARGUMENTS")

# Get PR data with individual field extraction to avoid JSON parsing issues with control characters
PR_TITLE=$(gh pr view "$ARGUMENTS" --json title -q .title 2>/dev/null | tr -d '\r\n\t' || echo "Title unavailable")
LATEST_COMMIT=$(gh pr view "$ARGUMENTS" --json commits -q '.commits | last | .oid' 2>/dev/null || echo "unknown")
PR_URL=$(gh pr view "$ARGUMENTS" --json url -q .url 2>/dev/null || echo "URL unavailable")

# Get changed files from diff directly (more reliable than JSON parsing)
CHANGED_FILES=$(echo "$FULL_DIFF" | grep "^diff --git" | sed 's|^diff --git a/.* b/||' | sort -u)

# Validate we have essential data
if [[ -z "$REPO_FULL_NAME" ]] || [[ -z "$FULL_DIFF" ]]; then
    echo "❌ Error: Failed to collect essential PR data"
    exit 1
fi

echo "✅ PR Data collected successfully:"
echo "  - Repository: $REPO_FULL_NAME"
echo "  - Title: $PR_TITLE"
echo "  - Latest Commit: $LATEST_COMMIT"
echo "  - Changed Files: $(echo "$CHANGED_FILES" | wc -l) files"
echo
```

## PHASE 2: STATIC ANALYSIS & LINE MAPPING

```bash
echo '=== 🔧 Phase 2: Building Line Number Map for Comments ==='

# Cross-platform compatible line mapping function using portable regex
create_line_mapping() {
    local file_path="$1"
    local diff_text="$2"
    echo " - Mapping lines for: $file_path" >&2

    # Use portable approach that works on both Linux and macOS
    echo "$diff_text" | {
        in_file=false
        new_line=0
        
        while IFS= read -r line; do
            # Check if we're entering a new file diff
            if [[ "$line" =~ ^diff\ --git\ a/.*\ b/(.+)$ ]]; then
                current_file="${BASH_REMATCH[1]}"
                if [[ "$current_file" == "$file_path" ]]; then
                    in_file=true
                else
                    in_file=false
                fi
                continue
            fi
            
            # Skip if not in our target file
            [[ "$in_file" == true ]] || continue
            
            # Parse hunk headers to get starting line numbers
            if [[ "$line" =~ ^@@.*\+([0-9]+) ]]; then
                new_line=$((${BASH_REMATCH[1]} - 1))
                continue
            fi
            
            # Process added lines (marked with +)
            if [[ "$line" =~ ^\+ ]]; then
                new_line=$((new_line + 1))
                content="${line:1}"  # Remove the + prefix
                printf "%d:RIGHT:%s\n" "$new_line" "$content"
            elif ! [[ "$line" =~ ^- ]]; then
                # Context lines (not removals) - increment line counter
                new_line=$((new_line + 1))
            fi
        done
    }
}

echo "✅ Cross-platform line mapping function is ready."
echo
```

## PHASE 3: PROGRAMMATIC SCAN

```bash
echo '=== 🤖 Phase 3: Running Programmatic Scan for High-Confidence Issues ==='
ISSUE_OUTPUT=""

# Enhanced function detection with proper error handling (PRESERVED FROM V2)
detect_function_issues() {
    local file_path="$1"
    local line_mapping="$2"
    
    local func_start=""
    local func_end=""
    local in_function=false
    local func_content=""
    
    while IFS=: read -r line_num side content; do
        [[ "$side" == "RIGHT" ]] || continue
        
        # Detect function start with mutable default
        if echo "$content" | grep -qE "^def [^(]+\([^)]*=\s*[\[{]"; then
            func_start="$line_num"
            in_function=true
            func_content="$content"
        elif [[ "$in_function" == true ]]; then
            func_content+=$'\n'"$content"
            
            # Detect function end (next function, class, or dedent to column 0)
            if echo "$content" | grep -qE "^(def |class |[[:alpha:]])"; then
                func_end=$((line_num - 1))
                
                # Analyze complete function for issues
                if echo "$func_content" | grep -qE "=\s*[\[{]"; then
                    ISSUE_OUTPUT+=$'\n'"MEDIUM|$file_path|$func_start|$func_end|true|Mutable Default Argument in Function|Use None as default and initialize inside function body."
                fi
                
                in_function=false
                func_content=""
            fi
        fi
    done <<< "$line_mapping"
}

detect_multiline_sql_injection() {
    local file_path="$1" 
    local line_mapping="$2"
    
    local sql_start=""
    local in_sql_block=false
    
    while IFS=: read -r line_num side content; do
        [[ "$side" == "RIGHT" ]] || continue
        
        # Detect multi-line f-string SQL start
        if echo "$content" | grep -qE 'f""".*SELECT|f""".*INSERT|f""".*UPDATE|f""".*DELETE'; then
            sql_start="$line_num"
            in_sql_block=true
        elif [[ "$in_sql_block" == true ]]; then
            # Detect end of multi-line string
            if echo "$content" | grep -q '"""'; then
                ISSUE_OUTPUT+=$'\n'"CRITICAL|$file_path|$sql_start|$line_num|true|Multi-line SQL Injection via f-string|Use parameterized queries with proper escaping."
                in_sql_block=false
            fi
        fi
    done <<< "$line_mapping"
}

detect_class_security_issues() {
    local file_path="$1"
    local line_mapping="$2"
    
    local class_start=""
    local class_end=""
    local in_class=false
    local class_content=""
    
    while IFS=: read -r line_num side content; do
        [[ "$side" == "RIGHT" ]] || continue
        
        if echo "$content" | grep -qE "^class "; then
            class_start="$line_num"
            in_class=true
            class_content="$content"
        elif [[ "$in_class" == true ]]; then
            class_content+=$'\n'"$content"
            
            # Detect class end
            if echo "$content" | grep -qE "^(class |def [^[:space:]]|[[:alpha:]])"; then
                class_end=$((line_num - 1))
                
                # Check for security anti-patterns in class
                if echo "$class_content" | grep -qE "(pickle\\.load|eval\\(|exec\\(|os\\.system)"; then
                    ISSUE_OUTPUT+=$'\n'"HIGH|$file_path|$class_start|$class_end|true|Unsafe Operations in Class|Avoid pickle.load, eval, exec, or os.system with untrusted data."
                fi
                
                in_class=false
                class_content=""
            fi
        fi
    done <<< "$line_mapping"
}

# Scan changed files for security and quality issues - FIXED ERROR HANDLING
echo "$CHANGED_FILES" | while IFS= read -r file_path; do
    [[ -n "$file_path" ]] || continue

    echo " - Scanning: $file_path"
    LINE_MAPPING=$(create_line_mapping "$file_path" "$FULL_DIFF")
    [[ -n "$LINE_MAPPING" ]] || continue

    # Single-line scans with PRESERVED V2 REGEX PATTERNS (CRITICAL - DO NOT CHANGE)
    while IFS=: read -r line_num side content; do
        [[ "$side" == "RIGHT" ]] || continue
        [[ -n "$content" ]] || continue
        
        # SQL injection detection - EXACT V2 PATTERN PRESERVED
        if echo "$content" | grep -qE "(execute|query)\s*\(\s*f['\"]" && ! echo "$content" | grep -q "# noqa"; then
            ISSUE_OUTPUT+=$'\n'"CRITICAL|$file_path|$line_num|$line_num|true|Potential SQL Injection via f-string|Use parameterized queries to prevent SQLi."
        fi
        
        # XSS vulnerability detection - PRESERVED
        if echo "$content" | grep -q "dangerouslySetInnerHTML"; then
            ISSUE_OUTPUT+=$'\n'"HIGH|$file_path|$line_num|$line_num|true|Potential XSS Vulnerability|Ensure HTML is sanitized with a library like DOMPurify."
        fi
        
        # Mutable default argument detection - EXACT V2 PATTERN PRESERVED
        if echo "$content" | grep -qE "def [^(]+\([^)]*=\s*[\[{]"; then
            ISSUE_OUTPUT+=$'\n'"MEDIUM|$file_path|$line_num|$line_num|true|Mutable Default Argument|Use None as default and initialize inside the function."
        fi
        
        # Hardcoded secrets detection - PRESERVED
        if echo "$content" | grep -qiE "(password|secret|api_key|token)\s*=\s*['\"][^'\"]{8,}"; then
            ISSUE_OUTPUT+=$'\n'"CRITICAL|$file_path|$line_num|$line_num|true|Potential Hardcoded Secret|Use environment variables or secret management."
        fi
        
        # Unsafe operations detection - EXACT V2 PATTERN PRESERVED (CRITICAL)
        if echo "$content" | grep -qE "(pickle\.load|eval\(|exec\(|os\.system)"; then
            ISSUE_OUTPUT+=$'\n'"HIGH|$file_path|$line_num|$line_num|true|Unsafe Operations|Avoid pickle.load, eval, exec, or os.system with untrusted data."
        fi
        
    done <<< "$LINE_MAPPING"

    # Add multi-line detections after single-line scan - PRESERVED
    detect_function_issues "$file_path" "$LINE_MAPPING"
    detect_multiline_sql_injection "$file_path" "$LINE_MAPPING"  
    detect_class_security_issues "$file_path" "$LINE_MAPPING"

done

echo "✅ Programmatic scan complete."
echo
```

## PHASE 4: COMPILING & VALIDATING PROGRAMMATIC FINDINGS

```bash
echo '=== 📊 Phase 4: Compiling & Validating Programmatic Findings ==='

# Improved JSON escaping function - handles control characters that caused original errors
json_escape() { 
    # Properly escape JSON strings including control characters
    printf '%s' "$1" | sed 's/\\/\\\\/g; s/"/\\"/g; s/\t/\\t/g; s/\r/\\r/g; s/\n/\\n/g' | awk '{printf "\"%s\"", $0}'
}

# Create temporary file for findings
VALIDATED_FINDINGS_FILE=$(mktemp)

# Build findings JSON with robust error handling - FIXED CONDITIONAL SYNTAX
if [[ -n "$ISSUE_OUTPUT" ]] && [[ "$ISSUE_OUTPUT" != "" ]]; then
    FINDINGS_JSON="["
    first_finding=true
    
    # Process findings line by line with error handling
    echo "$ISSUE_OUTPUT" | grep -v '^$' | while IFS='|' read -r sev file start_line end_line inline issue rec; do
        [[ -n "$sev" ]] || continue
        
        # Add comma separator for JSON array
        if [[ "$first_finding" != true ]]; then
            printf ","
        fi
        
        # Build JSON object with proper escaping
        printf '{"severity":%s,"file":%s,"start_line":%s,"end_line":%s,"can_inline":%s,"issue":%s,"recommendation":%s}' \
            "$(json_escape "$sev")" \
            "$(json_escape "$file")" \
            "$start_line" \
            "$end_line" \
            "$(json_escape "$inline")" \
            "$(json_escape "$issue")" \
            "$(json_escape "$rec")"
        
        first_finding=false
    done > "$VALIDATED_FINDINGS_FILE"
    
    echo "]" >> "$VALIDATED_FINDINGS_FILE"
    FINDINGS_JSON=$(cat "$VALIDATED_FINDINGS_FILE")
    
    # Validate JSON format with fallback
    if ! echo "$FINDINGS_JSON" | jq empty 2>/dev/null; then
        echo "⚠️ Warning: Generated invalid JSON, using empty findings"
        FINDINGS_JSON='[]'
    fi
else
    FINDINGS_JSON='[]'
fi

validate_multiline_range() {
    local file="$1"
    local start_line="$2"
    local end_line="$3"
    
    # Validate start_line <= end_line
    if [[ $start_line -gt $end_line ]]; then
        return 1
    fi
    
    # Check if all lines in range exist in diff
    local line_mapping
    line_mapping=$(create_line_mapping "$file" "$FULL_DIFF")
    
    for ((line=start_line; line<=end_line; line++)); do
        if ! echo "$line_mapping" | grep -q "^$line:RIGHT:"; then
            return 1  # Line not in diff
        fi
    done
    
    return 0  # Valid range
}

# Validate inline comments with better error handling - PRESERVED LOGIC
if [[ "$FINDINGS_JSON" != "[]" ]]; then
    echo "$FINDINGS_JSON" | jq -c '.[]' 2>/dev/null | while read -r finding; do
        if [[ -n "$finding" ]] && [[ "$(echo "$finding" | jq -r '.can_inline' 2>/dev/null)" == "true" ]]; then
            file=$(echo "$finding" | jq -r '.file' 2>/dev/null)
            start_line=$(echo "$finding" | jq -r '.start_line' 2>/dev/null)
            end_line=$(echo "$finding" | jq -r '.end_line' 2>/dev/null)
            
            if ! validate_multiline_range "$file" "$start_line" "$end_line"; then
                finding=$(echo "$finding" | jq '.can_inline = false' 2>/dev/null)
                echo " ⚠️ Warning: Lines $start_line-$end_line in $file not valid for inline comment." >&2
            fi
        fi
        echo "$finding"
    done | jq -s '.' > "$VALIDATED_FINDINGS_FILE" 2>/dev/null
    
    FINDINGS_JSON=$(cat "$VALIDATED_FINDINGS_FILE")
else
    echo "$FINDINGS_JSON" > "$VALIDATED_FINDINGS_FILE"
fi

# Count findings with error handling
CRITICAL_COUNT=$(echo "$FINDINGS_JSON" | jq '[.[] | select(.severity == "CRITICAL")] | length' 2>/dev/null || echo "0")
HIGH_COUNT=$(echo "$FINDINGS_JSON" | jq '[.[] | select(.severity == "HIGH")] | length' 2>/dev/null || echo "0")

echo "✅ Found $CRITICAL_COUNT Critical and $HIGH_COUNT High-severity issues programmatically."
echo

if [[ "$CRITICAL_COUNT" -gt 0 ]] || [[ "$HIGH_COUNT" -gt 0 ]]; then
    echo "📋 Summary of Issues Found:"
    echo "$FINDINGS_JSON" | jq -r '.[] | " - \(.severity): \(.issue) (Line \(.start_line)-\(.end_line) in \(.file))"' 2>/dev/null || echo " - Issues found but unable to format summary"
else
    echo "✅ No critical or high-severity issues detected programmatically."
fi
echo
```

## PHASE 5: LLM SYNTHESIS (REASONING BRAIN)

### llm_review_synthesis
You are a Staff Engineer with deep expertise in both Python backends and React frontends, known for your insightful and constructive code reviews.

Your task is to write a final, cohesive pull request summary. You have been provided with two key pieces of information:
1.  A list of low-level, high-confidence issues found by a programmatic scanner.
2.  The full diff of the pull request.

Your review should **synthesize** these inputs. Do not just list the programmatic findings. Weave them into a high-level narrative. Focus on the "why" behind the issues and assess the overall architecture, design patterns, and potential risks. Use the frameworks below to ensure a comprehensive review.

---
### **Analysis Frameworks**

#### 🎯 Impact & Risk Analysis
- "Blast Radius: How many systems/users are affected by this change?"
- "Breaking Changes: Does this involve API schema, database migrations, or config changes? Are they backward-compatible?" 
- "Dependencies: Are new packages introduced or versions bumped? Check for potential conflicts."
- "Rollback Complexity: How quickly and safely can this change be reverted in production?"

#### 🔐 Security Analysis (OWASP Top 10)
- "Injection: Look for SQL, NoSQL, or Command injection vectors."
- "Authentication/Authorization: Verify token validation, session management, and role checks (RBAC)."
- "Sensitive Data Exposure: Check logs, API responses, and client-side code for leaked keys, PII, or internal info."
- "Input Validation: Ensure all user-controllable input is strictly validated, sanitized, and type-checked."
- "XSS (Cross-Site Scripting): Especially in the React frontend, check for improper use of `dangerouslySetInnerHTML` and un-sanitized data rendering."

#### 🚀 Performance & Scalability
- "Database: Hunt for N+1 queries. Check if new queries need indexes. Assess transaction scope."
- "Memory Management: Look for potential memory leaks or excessive allocations."
- "Caching: Review caching strategy. Is invalidation handled correctly? Is the TTL appropriate?"
- "Concurrency: Identify potential race conditions or deadlocks, especially in Python async code."
- "Algorithm Complexity: Analyze loops and data processing for Big O complexity."

#### 🚀 Python specific pitfalls (backend)
- "Mutable Default Arguments: Find functions like `def my_func(items=[])` which can cause unexpected behavior."
- "Error Handling: Flag overly broad `except:` clauses. Specific exceptions should be caught."
- "Type Hinting: Check for missing or incorrect type hints. Encourage their use for clarity and static analysis."
- "Context Managers: Ensure resources like files and connections are managed with `with` statements."
- "Dependencies: Look at `pyproject.toml` or `requirements.txt`. Are versions pinned? Are there any known vulnerabilities in the packages?"
- "Security: Flag uses of `pickle`, `eval`, or `os.system` which can be dangerous if used with untrusted data."
- "Async Code: Check for blocking I/O calls within `async def` functions."

#### 🛠️ Code Quality & Architecture
- **SOLID Principles**: Does the new code adhere to principles like Single Responsibility and Dependency Inversion?
- **Design Patterns**: Are appropriate design patterns used? Are any anti-patterns being introduced?
- **Code Duplication**: Is there significant code duplication that could be refactored into a shared utility?
- **Test Coverage**: Do the tests adequately cover the new logic, including edge cases?

#### Other misc things
- "Typos: Does code changes have typos at critical locations, like conditional statements or other eval points"
- "Does network requests have timeouts added or not"
---

Your final output must be a single, well-formatted Markdown block that will be used as the main body of the pull request review.

<programmatic_findings>
$FINDINGS_JSON
</programmatic_findings>

<full_diff>
$FULL_DIFF
</full_diff>

## PHASE 6: FINAL SUBMISSION

```bash
echo '=== 🚀 Phase 6: Generating Final Recommendation & Submitting ==='

# Determine overall recommendation
if [[ "$CRITICAL_COUNT" -gt 0 ]]; then
    OVERALL_RECOMMENDATION="REQUEST_CHANGES"
elif [[ "$HIGH_COUNT" -gt 1 ]]; then
    OVERALL_RECOMMENDATION="REQUEST_CHANGES"
else
    OVERALL_RECOMMENDATION="COMMENT"
fi

# Store the synthesized review from the LLM (this should be populated by the LLM synthesis phase)
# Note: In actual execution, this would be replaced by the LLM's output
REVIEW_BODY="$llm_review_synthesis_output"

# Ensure review body is not empty - IMPROVED ERROR HANDLING
if [[ -z "$REVIEW_BODY" ]] || [[ "$REVIEW_BODY" == '$llm_review_synthesis_output' ]]; then
    REVIEW_BODY="# 🤖 Automated PR Review

## Summary
Comprehensive automated analysis completed for this pull request.

**Security Analysis**: $CRITICAL_COUNT critical and $HIGH_COUNT high-severity issues found programmatically.

**Repository**: $REPO_FULL_NAME
**Changed Files**: $(echo "$CHANGED_FILES" | wc -l) files modified

## Key Findings Summary
$(echo "$FINDINGS_JSON" | jq -r '.[] | "- **\(.severity)**: \(.issue) (\(.file):\(.start_line))"' 2>/dev/null || echo "Issues found but unable to format details")

---
🤖 Automated analysis completed. Manual review recommended for architectural assessment."
fi

# Truncate if too long
MAX_LEN=64000
if (( ${#REVIEW_BODY} > MAX_LEN )); then
    REVIEW_BODY="${REVIEW_BODY:0:63000}

---
⚠️ Review truncated to fit GitHub's 65,536-character limit"
fi

echo "✅ AI Staff Engineer has generated the final review summary."
echo "  - Submitting main review body with action: $OVERALL_RECOMMENDATION"

# Check for existing pending reviews first - NEW ERROR PREVENTION
EXISTING_REVIEWS=$(gh api "repos/$REPO_FULL_NAME/pulls/$ARGUMENTS/reviews" --jq '.[].state' 2>/dev/null || echo "")
if echo "$EXISTING_REVIEWS" | grep -q "PENDING"; then
    echo "⚠️ Warning: Pending review detected. Submitting as comment instead."
    OVERALL_RECOMMENDATION="COMMENT_ONLY"
fi

# Submit main review with proper error handling and fallback - IMPROVED
case $OVERALL_RECOMMENDATION in
    "REQUEST_CHANGES") GH_ACTION="--request-changes" ;;
    "APPROVE") GH_ACTION="--approve" ;;
    "COMMENT_ONLY") GH_ACTION="" ;;  # Will use API comment endpoint
    *) GH_ACTION="--comment" ;;
esac

# Try to submit as formal review first, fallback to comment if needed
if [[ "$OVERALL_RECOMMENDATION" == "COMMENT_ONLY" ]] || ! gh pr review "$ARGUMENTS" $GH_ACTION --body "$REVIEW_BODY" 2>/dev/null; then
    echo "📝 Submitting as regular comment..."
    if gh api --method POST "repos/$REPO_FULL_NAME/issues/$ARGUMENTS/comments" --field body="$REVIEW_BODY" >/dev/null 2>&1; then
        echo "✅ Review comment submitted successfully!"
    else
        echo "❌ Failed to submit review comment. Please check repository permissions."
        exit 1
    fi
else
    echo "✅ Formal review submitted successfully!"
fi

# Submit inline comments for findings that can be inlined - PRESERVED WITH ERROR HANDLING
if [[ "$FINDINGS_JSON" != "[]" ]] && echo "$FINDINGS_JSON" | jq empty 2>/dev/null; then
    INLINE_FINDINGS=$(echo "$FINDINGS_JSON" | jq -c '.[] | select(.can_inline == true)' 2>/dev/null || echo "")
    INLINE_COUNT=$(echo "$INLINE_FINDINGS" | jq -s '. | length' 2>/dev/null || echo "0")
    
    if [[ "$INLINE_COUNT" -gt 0 ]]; then
        echo " - Attempting to submit $INLINE_COUNT precise inline comments..."
        
        echo "$INLINE_FINDINGS" | jq -c '.[]' 2>/dev/null | while read -r finding; do
            FILE=$(echo "$finding" | jq -r '.file' 2>/dev/null || echo "")
            START_LINE=$(echo "$finding" | jq -r '.start_line' 2>/dev/null || echo "1")
            END_LINE=$(echo "$finding" | jq -r '.end_line' 2>/dev/null || echo "1")
            SEVERITY=$(echo "$finding" | jq -r '.severity' 2>/dev/null || echo "INFO")
            ISSUE=$(echo "$finding" | jq -r '.issue' 2>/dev/null || echo "Issue details unavailable")
            RECOMMENDATION=$(echo "$finding" | jq -r '.recommendation' 2>/dev/null || echo "Please review")
            
            [[ -n "$FILE" ]] || continue
            
            BODY="🤖 Automated Finding: $SEVERITY

Issue: $ISSUE

Suggestion: $RECOMMENDATION"

            if [[ "$START_LINE" == "$END_LINE" ]]; then
                # Single-line comment
                gh api --method POST "repos/$REPO_FULL_NAME/pulls/$ARGUMENTS/comments" \
                    --field body="$BODY" \
                    --field commit_id="$LATEST_COMMIT" \
                    --field path="$FILE" \
                    --field side="RIGHT" \
                    --field line="$END_LINE" \
                    --silent 2>/dev/null || echo " ⚠️ Failed to submit inline comment for line $END_LINE in $FILE" >&2
            else
                # Multi-line comment
                gh api --method POST "repos/$REPO_FULL_NAME/pulls/$ARGUMENTS/comments" \
                    --field body="$BODY" \
                    --field commit_id="$LATEST_COMMIT" \
                    --field path="$FILE" \
                    --field side="RIGHT" \
                    --field start_line="$START_LINE" \
                    --field line="$END_LINE" \
                    --silent 2>/dev/null || echo " ⚠️ Failed to submit inline comment for lines $START_LINE-$END_LINE in $FILE" >&2
            fi
        done
    fi
fi

echo
echo "==================== 🎉 REVIEW SUBMITTED ===================="
echo "✅ Review for PR #$ARGUMENTS submitted successfully!"
echo "🔗 View the PR: $PR_URL"
echo "🚦 Final Action Taken: $OVERALL_RECOMMENDATION"
echo "📊 Issues Found: $CRITICAL_COUNT Critical, $HIGH_COUNT High"
echo "============================================================"

# Cleanup temporary files - NEW
[[ -f "$VALIDATED_FINDINGS_FILE" ]] && rm -f "$VALIDATED_FINDINGS_FILE"
```