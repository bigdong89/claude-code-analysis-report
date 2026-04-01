# Contributing to Claude Code Analysis Report

Thank you for your interest in contributing to the Claude Code Analysis Report! This document provides guidelines for contributing.

## ⚠️ Important Copyright Notice

**Before contributing, please understand:**

- **Claude Code** source code is copyrighted by **Anthropic, PBC**
- This repository contains **analysis reports only**, NOT the original Claude Code source
- This project is for **educational purposes only** and is NOT officially affiliated with Anthropic
- Any contributions must respect intellectual property rights and applicable licenses

By contributing, you agree that your contributions:
1. Are your original work or properly attributed
2. Do not infringe on Anthropic's copyrights or any third-party rights
3. Align with the educational mission of this project
4. Include appropriate attribution where due

If you have concerns about copyright or licensing, please open an issue with the `copyright-inquiry` label BEFORE contributing.

---

## 🎯 Contribution Areas

We welcome contributions in the following areas:

### 1. Bug Fixes & Corrections
- Typographical errors
- Factual corrections about Claude Code's architecture
- Broken links or references
- Chart/visualization improvements

### 2. New Analysis Content
- Additional code examples
- New architectural insights
- Performance analysis updates
- Documentation of new Claude Code features

### 3. Documentation Improvements
- Clarifying technical explanations
- Adding more diagrams/visualizations
- Improving code examples
- Translating to other languages

### 4. Tool Enhancements
- Better data visualization
- Improved HTML/CSS styling
- Accessibility improvements
- Performance optimizations

## 📝 How to Contribute

### Reporting Issues

1. Check existing issues first
2. Use clear, descriptive titles
3. Provide context and evidence
4. Include screenshots if applicable

### Submitting Pull Requests

1. **Fork the repository** and create your branch
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Make your changes** following the style guidelines
   - Keep HTML reports self-contained
   - Maintain consistent styling
   - Test in multiple browsers

3. **Commit your changes**
   ```bash
   git add .
   git commit -m "feat: add your feature description"
   ```

4. **Push and create PR**
   ```bash
   git push origin feature/your-feature-name
   ```

## 🎨 Style Guidelines

### HTML Reports
- Keep files self-contained (embedded CSS/JS)
- Use consistent color scheme (#667eea, #764ba2 gradients)
- Maintain responsive design
- Test charts render correctly

### Markdown Files
- Use proper markdown syntax
- Include code examples with syntax highlighting
- Add clear section headers
- Maintain Chinese/English parallel structure

### Code Examples
```typescript
// Use proper TypeScript syntax
// Include comments for clarity
// Follow Claude Code conventions
```

## 🧪 Testing

### HTML Reports
- Test in Chrome, Firefox, Safari, Edge
- Verify charts render correctly
- Check responsive behavior
- Validate HTML/CSS

### Links & References
- Verify all external links work
- Check anchor links within documents
- Ensure language switcher functions

## 📋 Pull Request Checklist

Before submitting a PR, please ensure:

- [ ] Code follows the project's style guidelines
- [ ] Self-review has been performed
- [ ] Comments document changes clearly
- [ ] Documentation is updated if needed
- [ ] No new warnings/errors introduced
- [ ] Links and references work correctly
- [ ] HTML reports tested in multiple browsers

## 🏷️ Commit Message Format

We follow conventional commits:

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

### Examples
```
feat(html): add dark mode toggle to reports
docs(readme): update installation instructions
fix(charts): correct data flow chart labels
```

## 🌐 Language Translations

We welcome translations of the analysis reports:

1. Create a new file: `ANALYSIS_REPORT_<LANG>.html`
2. Maintain the same structure and styling
3. Add language switch link in the header
4. Update README.md with the new language

## 📧 Contact

For questions or discussions:
- Open a GitHub issue
- Start a discussion in the Discussions tab

## 📜 Code of Conduct

Be respectful, constructive, and professional in all interactions.

---

**Happy Contributing!** 🚀
