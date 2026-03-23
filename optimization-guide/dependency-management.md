# Dependency Management

> Every `yarn add` is a long-term maintenance commitment. Be intentional.

[Back to Index](./README.md) | Previous: [Keeping Updated](./keeping-rn-updated.md) | Next: [Performance](./performance-fundamentals.md)

---

## Why It Matters

Each library you add:
- Increases **bundle size** (affects startup — see [Performance](./performance-fundamentals.md))
- Adds **transitive dependencies** you don't control
- Creates **potential failure points** (see [ANR Analysis](./anr-analysis.md))
- Introduces **upgrade friction** (see [Keeping Updated](./keeping-rn-updated.md))

### The Hidden Cost

```
What you see:       cool-library        25KB
What you get:       + sub-dependencies  250KB
                    + peer-dependencies 150KB
                    ─────────────────────────
                    Total impact:       425KB
```

## Auditing Unused Dependencies {#auditing-unused-dependencies}

### Using Depcheck

```bash
# Install
npm install -g depcheck

# Run with RN-appropriate ignores
depcheck --ignores="@types/*,react-native-*,expo-*"
```

Depcheck identifies:
- **Unused dependencies** — candidates for removal
- **Missing dependencies** — used but not declared
- **Misplaced dependencies** — should be devDependencies

### Safe Removal Process

**Do not** blindly remove what depcheck flags. Some packages are transitive dependencies.

```bash
# Step 1: Check WHY a package exists
yarn why <package-name>

# Step 2: Check peer dependency requirements
# Look at the output — is another package depending on it?

# Step 3: Remove
yarn remove <package-name>

# Step 4: Verify
yarn ci          # type-check + lint + tests
yarn android     # build Android
yarn ios         # build iOS
```

## Lighter Alternatives {#lighter-alternatives}

### High-Impact Swaps

| Heavy Library | Lighter Alternative | Size Reduction | Notes |
|--------------|-------------------|---------------|-------|
| **Moment.js** (232KB) | **Day.js** (2KB) | ~99% | Near-identical API, plugin-based |
| **Lottie** (~1.5MB) | **Rive** (~850KB) | ~45% | State machine support, better perf |
| **Lodash** (full, 72KB) | **Lodash-es** (tree-shakeable) or native JS | varies | Most lodash methods have native equivalents |

### When Evaluating Alternatives

Ask:
1. Is the API similar enough for a low-risk migration?
2. Is the alternative actively maintained?
3. Does it support the [RN version](./keeping-rn-updated.md) you're targeting?
4. What's the transitive dependency cost?

## Before Adding a Library {#before-adding-a-library}

Run through this checklist:

- [ ] **Can it be done in <100 lines of custom code?** If yes, write it yourself.
- [ ] **Is it actively maintained?** Check last commit date, open issues, bus factor.
- [ ] **What's the total bundle impact?** Check with `yarn why` after install.
- [ ] **Are there lighter alternatives?** See [table above](#lighter-alternatives).
- [ ] **Does it conflict with current RN version?** Check compatibility.
- [ ] **Do we only need 5% of its API?** If so, extract just that functionality.

### Benefits of Custom Code

- Complete control — no surprise breaking changes
- No transitive dependencies
- Easier debugging — you wrote it, you understand it
- Smaller bundle size
- Customizable to your exact needs

### When Libraries ARE the Right Choice

- Complex domains (crypto, image processing, navigation)
- Platform-specific native modules
- Standards compliance (date formatting across locales)
- Active ecosystem with strong community support

## Measuring Impact

After removing or replacing libraries, track:

```bash
# Bundle size (use react-native-bundle-visualizer or similar)
npx react-native-bundle-visualizer

# Startup time — measure on real low-end devices
# ANR rates — check Crashlytics/Play Console over 48-72 hours
```

Cross-reference with [ANR Analysis](./anr-analysis.md) to confirm improvements.

---

**Next**: [Performance Fundamentals](./performance-fundamentals.md) — optimize animations, rendering, and data fetching
