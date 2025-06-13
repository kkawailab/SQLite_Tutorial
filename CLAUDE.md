# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a comprehensive SQLite tutorial written in Japanese (日本語) consisting of 9 chapters covering SQLite from basics to advanced topics. The repository contains only documentation files (Markdown) with embedded code examples - no actual executable code files.

## Project Structure

The tutorial is organized into the following chapters:
- `README.md` - Main overview and table of contents
- `chapter01-introduction.md` - SQLite fundamentals and concepts
- `chapter02-setup.md` - Installation and environment setup
- `chapter03-basic-sql.md` - Basic SQL operations (CRUD)
- `chapter04-datatypes-constraints.md` - Data types and constraints
- `chapter05-advanced-queries.md` - JOINs, subqueries, views
- `chapter06-indexes-performance.md` - Performance optimization
- `chapter07-transactions.md` - Transaction management and ACID
- `chapter08-programming-integration.md` - Integration with Python, JavaScript, Java, C#
- `chapter09-practice.md` - Practical projects (ToDo app, inventory system)

## Language and Localization

**Important**: All content in this repository is written in Japanese. When making modifications or additions:
- Maintain consistency with Japanese technical writing conventions
- Use appropriate Japanese terminology for database concepts
- Keep code comments in Japanese unless specifically for international examples

## Common Tasks

### Adding New Content
When adding new sections or examples:
1. Follow the existing Markdown formatting style
2. Include practical code examples within code blocks
3. Add new chapters to the table of contents in README.md
4. Maintain the progressive learning structure (basic → advanced)

### Code Examples
- Code examples are embedded within Markdown files using triple backticks
- Support multiple languages: SQL, Python, JavaScript, Java, C#
- Include both basic examples and complete application implementations

### Cross-Chapter References
- Use relative links for chapter navigation (e.g., `[← 第1章へ戻る](./chapter01-introduction.md)`)
- Maintain navigation links at the bottom of each chapter
- Keep the table of contents in README.md updated

## Content Guidelines

### Tutorial Philosophy
- Focus on practical, hands-on learning
- Progress from simple concepts to complex implementations
- Include real-world application examples (ToDo app, inventory system)
- Provide troubleshooting sections and best practices

### Code Example Standards
- SQL examples should work with SQLite 3.x
- Programming language examples should use standard libraries where possible
- Include error handling and transaction management in practical examples
- Show both basic usage and optimized implementations

## Repository Maintenance

### Documentation Updates
- Keep examples current with SQLite best practices
- Update deprecated syntax or methods
- Add new SQLite features as they become available
- Maintain consistency across all chapters

### Future Enhancements
Consider adding:
- English translations (as separate files or parallel structure)
- Interactive exercises or quizzes
- Video tutorial links
- Sample databases for practice
- GitHub Pages deployment for web viewing