---
title: "Going Live: My Updated Website (Gatsby → Next.js)"
date: "2026-02-26"
category: "Announcement"
description: "Launching my rebuilt site migrated from Gatsby to Next.js - why I migrated and what to expect."
---

I'm excited to announce that my updated website is now live. I migrated the site from Gatsby to a Next.js stack to improve maintainability, performance, and developer experience.

Old site: https://ashish-thakur111.github.io

New site: https://ashthakur.in

Why I migrated

- Maintenance and ecosystem: Gatsby's ecosystem and plugin maintenance have slowed compared to earlier years. Many community plugins are less actively maintained, which added friction when keeping dependencies secure and up to date.
- Modern Next.js features: Next.js offers a versatile rendering model (SSG, SSR, ISR) and newer capabilities like server components and built-in routing that simplify building modern React sites.
- Performance and build times: For my use case, Next.js provides faster incremental builds and more predictable build performance on larger sites.
- Developer experience: Improved DX with first-party tooling (image optimization, analytics, edge functions) and tighter integration with deployment platforms like Vercel.

What changed

- The site codebase is now a Next.js app with pages and components optimized for SSR/SSG as appropriate.
- Content remains Markdown-based and lives under `content/` for easy editing and future migrations.
- RSS, category filtering, and slugs are preserved during the transition.

What to expect

- Faster deploys and quicker content updates.
- Better compatibility with modern hosting and edge runtimes.
- Ongoing improvements: I'll continue refining the site, improving accessibility, and adding more posts.

If you notice any issues, please reach out.

Thanks for following along - more posts coming soon.

--Ashish
