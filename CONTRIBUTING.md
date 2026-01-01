# Contributing to Open Source System Design

Thank you for your interest in contributing! This document provides guidelines for contributing to this project.

## Ways to Contribute

### Content Contributions
- **Fix errors**: Correct technical inaccuracies or typos
- **Improve explanations**: Make concepts clearer
- **Add examples**: Provide additional code examples or diagrams
- **Add new topics**: Propose and write new content

### Code Contributions
- **Diagrams**: Create or improve architecture diagrams
- **Interactive examples**: Add runnable code samples
- **Tools**: Build helpful utilities

## Contribution Guidelines

### Content Style

1. **Be concise**: System design interviews are time-boxed. Keep explanations focused.

2. **Use diagrams**: A picture is worth a thousand words. Include ASCII diagrams or images.

3. **Show trade-offs**: Don't just present one solution. Discuss alternatives and when to use them.

4. **Include examples**: Real-world examples help concepts stick.

5. **Cite sources**: Link to papers, blog posts, or documentation when relevant.

### File Structure

Each topic should follow this structure:

```
topic-name/
├── README.md          # Main content
├── examples/          # Code examples (if applicable)
├── diagrams/          # Images and diagrams
├── quiz.md           # Self-assessment questions
└── resources.md      # Further reading
```

### System Design Format

Each system design should include:

```
system-name/
├── README.md          # Complete design walkthrough
├── problem.md         # Requirements and constraints
├── solution.md        # Step-by-step solution
├── deep-dives/        # Component deep-dives
├── diagrams/          # Architecture diagrams
└── alternatives.md    # Trade-offs and alternatives
```

### Writing Style

- Use **active voice**
- Write in **second person** ("You should consider...")
- Use **present tense**
- Keep paragraphs short (3-4 sentences max)
- Use bullet points for lists
- Include code blocks with syntax highlighting

### Diagrams

- Use clear, simple diagrams
- Include labels for all components
- Show data flow direction with arrows
- Use consistent styling across diagrams
- ASCII diagrams are acceptable for simple flows

Example ASCII diagram:
```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  Client  │────▶│ Load Balancer│────▶│  Server  │
└──────────┘     └──────────────┘     └──────────┘
```

## Submission Process

### For Small Changes

1. Fork the repository
2. Create a branch: `git checkout -b fix/typo-in-caching`
3. Make your changes
4. Commit with a clear message: `git commit -m "Fix typo in caching chapter"`
5. Push to your fork: `git push origin fix/typo-in-caching`
6. Open a Pull Request

### For New Content

1. **Open an issue first** to discuss your proposal
2. Wait for approval before starting work
3. Follow the submission process above
4. Include in your PR:
   - Summary of changes
   - Why this content is valuable
   - Any references or sources

## Review Process

1. Maintainers will review your PR within 1 week
2. Feedback may be provided for revisions
3. Once approved, your PR will be merged
4. You'll be added to the contributors list

## Code of Conduct

- Be respectful and inclusive
- Focus on constructive feedback
- Help others learn
- No self-promotion or spam

## Questions?

Open an issue with the `question` label if you need clarification.

Thank you for helping make system design education accessible to everyone!
