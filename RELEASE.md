# Release Strategy

We follow [Semantic Versioning](https://semver.org/).

## Release Process

1. **Prepare the Release**:
   - Update the `version` in `package.json`.
   - Update `CHANGELOG.md` (if applicable) with new features and fixes.
2. **Commit and Tag**:
   ```bash
   git add package.json
   git commit -m "chore: release v1.0.0"
   git tag -a v1.0.0 -m "Release version 1.0.0"
   ```
3. **Push**:
   ```bash
   git push origin main --tags
   ```
4. **GitHub Release**:
   - Go to the [Releases page](https://github.com/sharmasundip/playwright-qa-skills/releases).
   - Draft a new release.
   - Select the tag and title it "v1.0.0".
   - Use the "Generate release notes" button.
   - Publish the release.

## Versioning Policy
- **Major (x.0.0)**: Breaking changes to skill interfaces or core pipeline.
- **Minor (0.x.0)**: New skills or significant enhancements to existing ones.
- **Patch (0.0.x)**: Bug fixes and performance improvements.
