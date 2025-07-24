# Pull Request Review Commands

This directory contains specialized commands for automated pull request review and analysis using Claude Code.

## Available Commands

### `/pr` - Comprehensive Analysis
**File:** `pr.md`

Standard comprehensive PR review with multi-phase analysis framework.

**Usage:**
```bash  
/pr 456
```

**Features:**
- **Severity Classification**: Critical/High/Medium/Low issue categorization
- **Multi-Phase Process**:
  1. Automated data collection
  2. Deep code analysis  
  3. Review summary generation
  4. User interaction workflow
- **Advanced Analysis**:
  - Cross-reference analysis with recent PRs
  - Metrics integration (coverage, complexity)
  - Contextual intelligence based on PR scope
- **Flexible Submission**: Multiple approval options and overrides

### `/pr-review-multi-line` - Hybrid Automated Review
**File:** `pr-review-multi-line-v3-fixed.md`

Advanced hybrid review combining programmatic static analysis with AI synthesis.

**Usage:**
```bash
/pr-review-multi-line 789
```

**Features:**
- **6-Phase Analysis Process**:
  1. Data collection & validation
  2. Static analysis & line mapping
  3. Programmatic security scanning
  4. Finding compilation & validation
  5. LLM synthesis with Staff Engineer persona
  6. Automated GitHub submission
- **Static Code Analysis**:
  - SQL injection detection
  - XSS vulnerability scanning  
  - Hardcoded secrets identification
  - Unsafe operations flagging
  - Multi-line issue detection
- **Robust Error Handling**: Cross-platform compatibility with fallback mechanisms
- **Automated Submission**: Direct GitHub API integration with inline comments

## Security Focus

All PR review commands emphasize defensive security practices:

### 🔐 Security Checks
- **Injection Vulnerabilities**: SQL, NoSQL, Command, XSS detection
- **Authentication Issues**: Token validation, session management
- **Data Exposure**: Sensitive information in logs, responses, client-side code
- **Input Validation**: Sanitization and type checking analysis
- **Cryptography**: Secure algorithms and key management review

### 🐍 Python-Specific Security
- Dangerous functions: `pickle.load`, `eval`, `exec`, `os.system`
- Mutable default arguments
- Broad exception handling
- Missing context managers
- Async/await patterns

### ⚛️ React-Specific Security  
- XSS via `dangerouslySetInnerHTML`
- Component security patterns
- State management vulnerabilities
- Client-side data exposure

## Prerequisites

### Required Tools
- **GitHub CLI (`gh`)**: Must be installed and authenticated
  ```bash
  gh auth login
  gh auth status  # Verify authentication
  ```
- **Repository Access**: PR review permissions on target repository
- **Network Access**: Connectivity to GitHub API

### Command Dependencies
- `jq` - JSON processing (usually pre-installed)
- `bash` - Shell execution environment
- Standard Unix tools: `grep`, `sed`, `awk`

## Usage Patterns

### Quick Review
```bash
# Basic review for small PRs
/pr-review 123
```

### Comprehensive Analysis
```bash  
# Full analysis for complex PRs
/pr-review-multi-line 456
```

### Standard Workflow
```bash
# Balanced approach
/pr 789
```

## Command Workflow

1. **Data Collection**: Fetch PR details, diff, and metadata
2. **Analysis**: Apply security, performance, and quality frameworks
3. **Finding Generation**: Create structured, actionable feedback
4. **User Interaction**: Present findings with submission options
5. **GitHub Integration**: Submit review with inline comments

## Customization

### Modifying Analysis Frameworks
Each command contains configurable analysis sections:
- Security rules and patterns
- Performance benchmarks  
- Code quality standards
- Language-specific checks

### Adding New Patterns
To add new detection patterns:
1. Update the programmatic scan section
2. Add corresponding severity classification
3. Include recommendation templates
4. Test with sample code

## Troubleshooting

### Common Issues

**Command Not Found**
- Verify file is in correct directory
- Check YAML frontmatter syntax
- Restart Claude Code session

**GitHub API Errors**
```bash
# Check authentication
gh auth status

# Test API access
gh api user
```

**Permission Errors**
- Ensure repository access permissions
- Verify PR number exists and is accessible
- Check if repository is private/public

### Debug Mode
Add debug output by modifying the bash scripts:
```bash
set -x  # Enable debug output
echo "Debug: Variable value is $VARIABLE"
```

## Contributing

To improve PR review commands:

1. **Test Thoroughly**: Use various PR types and sizes
2. **Security First**: Ensure no false positives on security issues  
3. **Cross-Platform**: Test on both macOS and Linux
4. **Error Handling**: Add robust fallback mechanisms
5. **Documentation**: Update this README with any changes

### Adding New Commands
Follow the existing pattern:
- YAML frontmatter with metadata
- Phased analysis approach
- Structured output format
- User interaction workflow
- GitHub API integration

## Performance Considerations

- **Large PRs**: Commands may timeout on very large diffs
- **API Rate Limits**: GitHub API has rate limiting
- **Network Latency**: Commands require stable internet connection
- **Memory Usage**: Large diffs are processed in memory

## Security Considerations

- **Read-Only Operations**: Commands don't modify repository state
- **Credential Safety**: No storage of authentication tokens
- **Data Privacy**: PR content is processed locally and via Claude API
- **Permission Respect**: Commands respect repository access controls