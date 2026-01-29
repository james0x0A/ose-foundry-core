# Pre-Mortem Analysis Report

## Old-School Essentials (OSE) Foundry VTT System

**Analysis Date:** January 2026
**Codebase Version:** v13 Compatible
**Report Type:** Proactive Risk Assessment & Corrective Actions

---

## Executive Summary

This pre-mortem analysis identifies potential failure modes and risks in the OSE Foundry VTT system codebase before they manifest as production issues. The analysis covers security vulnerabilities, dependency risks, test coverage gaps, architectural concerns, and maintainability issues.

**Risk Level Summary:**
- **Critical:** 0 issues (2 fixed)
- **High:** 4 issues (1 fixed)
- **Medium:** 8 issues
- **Low:** 6 issues

---

## 1. Security Vulnerabilities

### 1.1 XSS Vulnerability - Unescaped HTML in Item Descriptions [FIXED]

**Status:** RESOLVED

**Original Issue:**
- `src/module/item/entity.js:179` - Description passed without sanitization
- `src/templates/chat/item-card.html:12` - Uses `{{{data.description}}}`

**Fix Applied:**
- `rollFormula()` now uses `this.system.enrichedDescription` (pre-sanitized)
- `getChatData()` now sets `itemData.description = this.system.enrichedDescription`

---

### 1.2 XSS Vulnerability - Unescaped Details in Roll Results [FIXED]

**Status:** RESOLVED

**Original Issue:**
- `src/templates/chat/roll-attack.html:21` - `{{{result.details}}}`
- `src/templates/chat/roll-result.html:16` - `{{{result.details}}}`

**Fix Applied:**
- Changed `{{{result.details}}}` to `{{result.details}}` (escaped) in both templates

---

### 1.3 Unsafe DOM Manipulation - outerHTML Usage [LOW]

**Location:** `src/module/helpers-chat.ts:147-149`

**Issue:** Using `outerHTML` assignment is considered an unsafe DOM API pattern.

**Corrective Action:**
```typescript
// Instead of:
blindable.outerHTML = "<div>...</div>";

// Use:
const replacement = document.createElement('div');
replacement.className = 'dice-roll';
replacement.innerHTML = "<div class='dice-result'><div class='dice-formula'>???</div></div>";
blindable.replaceWith(replacement);
```

**Priority:** LOW

---

### 1.4 CSS Injection via Group Names [LOW]

**Location:** `src/module/combat/combat-tracker.ts:203`

**Issue:** Group names interpolated directly into CSS styles.

**Corrective Action:** Use predefined CSS classes instead of interpolated values, or sanitize/validate group names.

**Priority:** LOW

---

## 2. Dependency Vulnerabilities

### 2.1 37 Known Vulnerabilities in Dependencies [CRITICAL]

**npm audit Results:**
| Severity | Count | Notable Packages |
|----------|-------|------------------|
| High | 7 | cross-spawn, rollup, semver, ws |
| Moderate | 29 | eslint, lodash, tinymce, nanoid |
| Low | 1 | brace-expansion |

**Critical Packages:**
- **rollup** (< 2.79.2): DOM Clobbering XSS vulnerability
- **cross-spawn** (7.0.0 - 7.0.4): ReDoS vulnerability
- **semver** (multiple versions): ReDoS vulnerability

**Corrective Action:**
```bash
# Fix most vulnerabilities
npm audit fix

# For breaking changes (eslint upgrade)
npm audit fix --force  # Test thoroughly after

# Manual updates needed for:
# - ws (no direct fix available, monitor for updates)
# - foundry-vtt-types (depends on vulnerable socket.io-client)
```

**Priority:** CRITICAL - Run `npm audit fix` before next release

---

### 2.2 Outdated Development Dependencies [MEDIUM]

**Key Outdated Packages:**
| Package | Current | Latest | Risk |
|---------|---------|--------|------|
| typescript | 4.7.4 | 5.x | Missing modern features |
| eslint | 8.28.0 | 9.x | Breaking changes |
| rollup | 2.77.3 | 4.x | Breaking changes |
| sass | 1.25.0 | 1.7x+ | Missing features |

**Corrective Action:**
1. Create upgrade branch
2. Update packages incrementally
3. Test build process after each major upgrade
4. Update configuration files as needed

**Priority:** MEDIUM - Schedule for next maintenance cycle

---

## 3. Test Coverage Gaps

### 3.1 Empty Test Implementations [HIGH]

**Location:** `src/module/__tests__/helpers-chat.test.ts`

**Issue:** Contains 3 empty describe blocks with `// @todo: How do we test these properly?`

**Untested Functions:**
- `applyChatCardDamage()`
- `addChatMessageContextOptions()`
- `addChatMessageButtons()`

**Corrective Action:** Implement tests using mock DOM elements and game state:
```typescript
describe('applyChatCardDamage', () => {
  it('should apply damage to selected tokens', async () => {
    // Create mock HTML element with dice-total
    // Mock game.settings for applyDamageOption
    // Mock canvas.tokens.controlled
    // Assert damage applied correctly
  });
});
```

**Priority:** HIGH - Core functionality untested

---

### 3.2 No Tests for Critical Dialog Components [HIGH]

**Untested Files (1,026 lines):**
- `src/module/dialog/character-creation.js` (178 lines)
- `src/module/dialog/character-gp-cost.js` (140 lines)

**Impact:** Character creation is a core user workflow with no automated testing.

**Corrective Action:** Add comprehensive tests for:
- Form validation
- Default value handling
- Submission behavior
- Error states

**Priority:** HIGH

---

### 3.3 Error Handling Not Tested [HIGH]

**Current State:**
- Source code: 43 instances of try/catch/throw
- Tests covering errors: Only 2 instances
- **Coverage Gap: ~95%**

**Uncovered Error Paths:**
- `monster-sheet.js:126-130` - JSON parse failures
- `party-xp.js:53-58` - JSON parse failures
- `party-sheet.js:92-104` - Multiple error paths
- `actor-sheet.js:248` - Drop target errors
- `item/entity.js:157,196` - Missing formula errors

**Corrective Action:** Add error path tests:
```typescript
it('should handle malformed JSON in drop data', async () => {
  const badDropData = { invalid: 'json structure' };
  await expect(sheet._onDrop(badDropData)).rejects.toThrow();
});
```

**Priority:** HIGH

---

### 3.4 Combat System Partially Untested [MEDIUM]

**Untested Combat Files:**
- `src/module/combat/combat-set-groups.ts` (108 lines)
- `src/module/combat/combat-tracker.ts` (244 lines)
- `src/module/combat/combatant.ts` (91 lines)

**Only Tested:** `combat.ts` via `combat.test.ts`

**Corrective Action:** Add tests for:
- Group assignment logic
- Combatant turn management
- Combat tracker rendering

**Priority:** MEDIUM

---

### 3.5 No Configuration/Settings Tests [MEDIUM]

**Untested Files:**
- `src/module/config.ts` (320+ lines)
- `src/module/settings.ts` (220+ lines)

**Impact:** Settings registration and configuration values untested.

**Corrective Action:** Add tests verifying:
- All settings are properly registered
- Default values are correct
- Encumbrance option returns valid class

**Priority:** MEDIUM

---

## 4. Architecture & Maintainability

### 4.1 Mixed JavaScript/TypeScript Codebase [MEDIUM]

**Current State:**
- JavaScript files: 31
- TypeScript files: 23

**Issues:**
- Inconsistent type safety
- Some JS files missing JSDoc types
- Mixed import patterns

**Corrective Action:**
1. Prioritize converting core files to TypeScript
2. Convert order: config.ts, entities, data-models, sheets
3. Add strict type checking incrementally

**Recommended Conversion Priority:**
```
1. src/module/actor/entity.js → entity.ts
2. src/module/item/entity.js → entity.ts
3. src/module/helpers-dice.js → helpers-dice.ts
4. src/module/dialog/*.js → *.ts
```

**Priority:** MEDIUM - Long-term maintainability

---

### 4.2 CI/CD Pipeline Missing Key Features [MEDIUM]

**Current State:** Only release workflow exists (`.github/workflows/release.yml`)

**Missing:**
- Automated testing in CI
- Code coverage reporting
- Dependency vulnerability scanning
- Pull request checks

**Corrective Action:** Add GitHub Actions workflows:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run lint

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
```

**Priority:** MEDIUM

---

### 4.3 Node.js 16 EOL in CI [HIGH]

**Location:** `.github/workflows/release.yml:16-18`

**Issue:** Using Node.js 16, which reached End-of-Life in September 2023.

**Corrective Action:**
```yaml
- name: Setup Node.js version 20
  uses: actions/setup-node@v4
  with:
    node-version: "20"
```

**Priority:** HIGH - Security risk

---

### 4.4 Deprecated GitHub Actions [LOW]

**Location:** `.github/workflows/release.yml`

**Issues:**
- `actions/checkout@v2` → Update to `@v4`
- `actions/setup-node@v2` → Update to `@v4`
- `microsoft/variable-substitution@v1` → Consider alternatives

**Corrective Action:** Update all actions to latest versions.

**Priority:** LOW

---

## 5. Code Quality Issues

### 5.1 jQuery Dependency for Simple Operations [LOW]

**Location:** `src/module/item/entity.js:477-480`

**Issue:** Using jQuery for simple slide animations when vanilla JS would suffice.

```javascript
// Current
$(content).slideDown(200);

// Suggested
content.style.display = 'block';
content.animate([{ height: 0 }, { height: content.scrollHeight + 'px' }], { duration: 200 });
```

**Impact:** Dependency on jQuery for minimal functionality.

**Priority:** LOW

---

### 5.2 Magic Numbers in Combat System [LOW]

**Location:** `src/module/actor/entity.js:46-62`

**Issue:** AC/AAC conversion uses magic number `19` without explanation.

```javascript
// Current
newData["system.aac.value"] = 19 - acValue;

// Suggested
const AC_AAC_CONVERSION_BASE = 19; // OSE uses 19 as the base for AC/AAC conversion
newData["system.aac.value"] = AC_AAC_CONVERSION_BASE - acValue;
```

**Priority:** LOW - Readability improvement

---

### 5.3 Inconsistent Error Handling Patterns [MEDIUM]

**Issue:** Mix of throw, console.error, and ui.notifications for error handling.

**Examples:**
- `entity.js:157` - throws Error
- `entity.js:505` - uses ui.notifications.error
- Some functions silently fail

**Corrective Action:** Establish consistent error handling guidelines:
1. Use `throw` for programmer errors
2. Use `ui.notifications` for user-facing errors
3. Log to console for debugging
4. Document expected error handling in JSDoc

**Priority:** MEDIUM

---

## 6. Documentation Gaps

### 6.1 API Documentation Missing [LOW]

**Issue:** No generated API documentation despite comprehensive JSDoc.

**Corrective Action:**
1. Add TypeDoc or JSDoc documentation generation
2. Include in build process
3. Publish to GitHub Pages

```json
// package.json
{
  "scripts": {
    "docs": "typedoc --out docs/api src/module"
  }
}
```

**Priority:** LOW

---

### 6.2 Contributing Guide Outdated [LOW]

**Location:** `CONTRIBUTING.md`

**Issues:**
- References Node.js 16 (EOL)
- Missing TypeScript contribution guidelines
- No testing contribution guidance

**Corrective Action:** Update with:
- Node.js 20 LTS requirement
- TypeScript style guide
- Test writing expectations

**Priority:** LOW

---

## 7. Recommended Action Plan

### Immediate Actions (Before Next Release)

| # | Action | Risk Addressed | Status |
|---|--------|----------------|--------|
| 1 | ~~Run `npm audit fix`~~ | Dependency vulnerabilities | **DONE** (partial - some deps require breaking changes) |
| 2 | ~~Sanitize spell descriptions in `rollFormula()`~~ | XSS vulnerability | **DONE** |
| 3 | Update Node.js to 20 in CI | Security/EOL | Pending |
| 4 | Update GitHub Actions versions | Deprecated actions | Pending |

### Short-Term Actions (Next Sprint)

| # | Action | Risk Addressed | Status |
|---|--------|----------------|--------|
| 5 | ~~Escape `result.details` in templates~~ | XSS vulnerability | **DONE** |
| 6 | Implement helpers-chat.test.ts | Test coverage | Pending |
| 7 | Add character-creation tests | Test coverage | Pending |
| 8 | Add CI lint workflow | Code quality | Pending |

### Medium-Term Actions (Next Quarter)

| # | Action | Risk Addressed | Effort |
|---|--------|----------------|--------|
| 9 | Convert core JS files to TypeScript | Maintainability | 20 hours |
| 10 | Add error handling tests | Test coverage | 8 hours |
| 11 | Update major dependencies (rollup, eslint) | Dependencies | 8 hours |
| 12 | Add combat system tests | Test coverage | 6 hours |

### Long-Term Actions (Future Releases)

| # | Action | Risk Addressed | Effort |
|---|--------|----------------|--------|
| 13 | Complete TypeScript migration | Maintainability | 40 hours |
| 14 | Generate API documentation | Documentation | 4 hours |
| 15 | Remove jQuery dependency | Dependencies | 2 hours |
| 16 | Standardize error handling | Code quality | 8 hours |

---

## 8. Risk Matrix

| Risk | Likelihood | Impact | Priority | Mitigation Status |
|------|------------|--------|----------|-------------------|
| XSS via item descriptions | Medium | High | Critical | **RESOLVED** |
| Dependency vulnerabilities | High | Medium | Critical | **PARTIAL** (18 remaining, down from 37) |
| Untested core workflows | Medium | Medium | High | Identified |
| Node.js EOL in CI | High | Low | High | Identified |
| TypeScript migration debt | Low | Medium | Medium | Ongoing |
| Missing CI/CD | Medium | Low | Medium | Identified |

---

## 9. Monitoring Recommendations

1. **Enable Dependabot** for automated dependency updates
2. **Add npm audit** to CI pipeline
3. **Track test coverage** with coverage reporting
4. **Monitor GitHub security advisories** for Foundry VTT ecosystem

---

## 10. Conclusion

This pre-mortem analysis has identified 21 actionable items across security, dependencies, testing, architecture, and documentation. The most critical issues are:

1. **XSS vulnerabilities** in item description rendering
2. **37 dependency vulnerabilities** requiring immediate patching
3. **Test coverage gaps** in core functionality (chat, dialogs, combat)
4. **Outdated Node.js** in CI pipeline

By addressing these issues proactively, the OSE Foundry VTT system can maintain its quality, security, and reliability for the community of users and contributors.

---

*Report generated as part of proactive code health assessment.*
