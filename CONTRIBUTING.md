# Contributing to HLD-Content

Thank you for your interest in contributing to the HLD-Content repository! This document provides guidelines and best practices for contributing.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Content Guidelines](#content-guidelines)
- [Style Guide](#style-guide)
- [Submission Process](#submission-process)

## Code of Conduct

This project adheres to a code of professionalism and respect. By participating, you are expected to uphold this standard. Please:

- Be respectful and inclusive
- Accept constructive criticism gracefully
- Focus on what is best for the community
- Show empathy towards other contributors

## How Can I Contribute?

### Reporting Issues

- Check if the issue already exists
- Use a clear and descriptive title
- Provide specific examples and context
- Include relevant screenshots if applicable

### Suggesting Enhancements

- Clearly describe the enhancement and its benefits
- Explain why this would be useful to most users
- Provide examples of how it would work

### Content Contributions

You can contribute by:

1. **Adding new topics**: Introduce new system design concepts
2. **Improving existing content**: Enhance clarity, add examples, or fix errors
3. **Adding diagrams**: Visual representations of concepts
4. **Adding resources**: Links to articles, videos, or books
5. **Fixing typos**: Grammar and spelling corrections

## Content Guidelines

### Structure

- Follow the existing folder structure
- Each topic should have its own folder within the appropriate section
- Include a README.md in each folder with:
  - Overview of the topic
  - Learning objectives
  - Prerequisites
  - Key concepts
  - Examples
  - Additional resources

### Quality Standards

1. **Accuracy**: Ensure all technical information is correct
2. **Clarity**: Write in clear, concise language
3. **Completeness**: Cover topics thoroughly but avoid unnecessary verbosity
4. **Examples**: Include practical examples where applicable
5. **References**: Cite sources for complex concepts

### Content Organization

```markdown
# Topic Name

## Overview
Brief introduction to the topic

## Key Concepts
- Concept 1
- Concept 2

## Detailed Explanation
In-depth coverage with examples

## Use Cases
Real-world applications

## Best Practices
Recommendations and guidelines

## Common Pitfalls
Mistakes to avoid

## Additional Resources
- Links to articles
- Video tutorials
- Books
```

## Style Guide

### Markdown Formatting

- Use `#` for main headings, `##` for subheadings
- Use **bold** for important terms
- Use `code blocks` for code, commands, and technical terms
- Use bullet points for lists
- Use numbered lists for sequential steps

### Code Examples

```markdown
```language
// Your code here with comments
// Explain what the code does
```
```

### Diagrams

- Use Mermaid for simple diagrams when possible
- For complex diagrams, use Draw.io or similar tools
- Save diagrams as PNG with transparent background
- Include both source file and rendered image

### Naming Conventions

- **Files**: Use lowercase with hyphens (e.g., `load-balancing.md`)
- **Folders**: Use lowercase with hyphens (e.g., `01-introduction`)
- **Images**: Descriptive names (e.g., `cap-theorem-diagram.png`)

## Submission Process

### Step 1: Fork the Repository

```bash
# Fork via GitHub UI, then clone your fork
git clone https://github.com/YOUR_USERNAME/HLD-Content.git
cd HLD-Content
```

### Step 2: Create a Branch

```bash
# Create a descriptive branch name
git checkout -b feature/add-kafka-notes
# or
git checkout -b fix/typo-in-caching
```

### Step 3: Make Your Changes

- Add or modify content following the guidelines
- Test all links and code examples
- Ensure markdown renders correctly

### Step 4: Commit Your Changes

```bash
git add .
git commit -m "Add detailed notes on Apache Kafka"
```

**Commit Message Guidelines:**
- Use present tense ("Add feature" not "Added feature")
- Use imperative mood ("Move cursor to..." not "Moves cursor to...")
- Be specific and descriptive
- Reference issues if applicable (#123)

### Step 5: Push and Create Pull Request

```bash
git push origin feature/add-kafka-notes
```

Then create a Pull Request via GitHub:

1. Go to your fork on GitHub
2. Click "Pull Request"
3. Provide a clear title and description
4. Reference any related issues
5. Wait for review

### Pull Request Guidelines

- **Title**: Clear and descriptive (e.g., "Add comprehensive notes on message queues")
- **Description**: Explain what changes you made and why
- **Scope**: Keep PRs focused on a single topic/fix
- **Size**: Smaller PRs are easier to review

## Review Process

1. **Automated checks**: Markdown linting and link validation
2. **Content review**: Accuracy and quality assessment
3. **Feedback**: Suggestions for improvements
4. **Approval**: Once all checks pass and reviewers approve
5. **Merge**: Your contribution will be merged

## Recognition

All contributors will be:
- Listed in the repository contributors
- Acknowledged in release notes for significant contributions
- Given credit in the README for major additions

## Questions?

If you have questions about contributing:
- Check existing issues and discussions
- Open a new issue with the "question" label
- Reach out via GitHub discussions

## Additional Notes

### What to Avoid

- Plagiarism: Always cite sources
- Opinions as facts: Distinguish between best practices and personal preferences
- Outdated information: Keep content current
- Over-complication: Keep explanations accessible

### Priority Areas

We especially welcome contributions in:
- Real-world case studies
- Interview problem solutions
- Diagrams and visualizations
- Video content recommendations
- Practice exercises

---

Thank you for contributing to HLD-Content! Your efforts help make system design knowledge accessible to everyone.
