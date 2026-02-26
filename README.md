# Blogs

## Overview

This repository contains technical blog posts about AI, technology, and related topics.

## Structure

- Blog posts should be located in the `content/blog/` directory.
- Supported post formats:
	1. Folder-based: `content/blog/post-name/index.md` (recommended for posts with images)
	2. File-based: `content/blog/post-name.md` (simple posts)

## Frontmatter

Each Markdown file must include YAML frontmatter containing the following fields:

- `title`: Post title (string)
- `date`: Publication date (YYYY-MM-DD)
- `category`: Post category (used for filtering)
- `description`: Brief post description

## Example

Example YAML frontmatter (use at the top of your post file):

```yaml
---
title: "Mastering Kubernetes with Fabric8"
date: "2026-02-25"
category: "Kubernetes"
description: "A deep dive into using the Fabric8 client for Java applications."
---
```

## Features

- Automatic slug generation from folder/filename for URLs
- Dynamic category and year filtering on the `/blog` page
- Automatic RSS feed generation at `/rss.xml`
- Standard Markdown support with metadata extraction

My blogs related to tech, AI and other things.