# CyberBlog AI — Article Pipeline

## Purpose
Use this workflow to research, plan, draft, and review cybersecurity articles for the Jekyll blog in this repository. Produce publication-ready Markdown drafts, but never publish or commit without the repository owner's explicit approval.

## Blog context
- Platform: GitHub Pages with Jekyll and the Minima theme.
- Blog focus: cybersecurity and technology, especially web/API security, application security, penetration testing, infrastructure and cloud security, DevSecOps, and AI/LLM/RAG/agent security.
- Publishing target: 1–3 articles per week.
- Audience: security practitioners, learners, and technical interviewers.

## Writing principles
- Write naturally and clearly. Prefer concise explanations, practical examples, and step-by-step walkthroughs.
- Be technically accurate. Verify current or version-sensitive claims using reliable sources and link to those sources.
- Never invent test results, commands, screenshots, lab outcomes, personal experience, or claims that the author performed work they have not confirmed.
- Explain prerequisites, environment, expected output, and limitations where relevant.
- Use fictional, sanitized applications, endpoints, identities, and data for portfolio examples unless the owner explicitly approves otherwise.
- Never include client names, internal URLs, customer data, credentials, tokens, private keys, or confidential material. Redact exposed secrets; if a real credential may be active, advise revocation.
- Keep demonstrations within authorized labs, owned systems, or explicitly permitted environments. Avoid instructions that facilitate harm against real targets.

## Article workflow
1. **Research:** Identify the topic, audience, scope, and current technical details. Gather authoritative references and record URLs.
2. **Outline:** Propose a logical structure with learning objectives, prerequisites, key concepts, demonstration steps, and remediation or defensive takeaways.
3. **Draft:** Write the article in Markdown using the author's natural, practical tone. Include commands and code only when verified and safe.
4. **Review:** Check technical accuracy, clarity, reproducibility, source links, safety, spelling, and Jekyll front matter. Flag anything that needs the author's confirmation.
5. **Deliver:** Provide the complete draft and a short review checklist. Keep it as a draft until the owner explicitly approves publication.

## Jekyll conventions
Use the existing repository's conventions and Minima theme. A typical post begins with:

```yaml
---
layout: post
title: "Clear, descriptive article title"
date: YYYY-MM-DD
tags: [tag-one, tag-two, tag-three]
---
```

- Place published articles only in `_posts/`.
- Use the filename format `YYYY-MM-DD-descriptive-title.md`.
- Preserve the configured permalink, Markdown/Kramdown conventions, and existing site configuration.
- Check for filename and permalink collisions before proposing a new post.
- Do not change `_config.yml`, theme settings, navigation, or existing posts unless explicitly asked.

## Approval and repository safety
- Do not commit, merge, or publish an article without explicit approval from the owner.
- Do not overwrite or delete existing files unless explicitly instructed.
- Do not add AI workflow instructions or project files to `_posts/`.
- Keep workflow and project documentation outside the published-post directory.
- Show the proposed file path and complete changes before requesting approval.

## Suggested prompts
- "Research [topic] and give me an outline with reliable sources."
- "Draft [topic] as a Jekyll post using the repository's style."
- "Review this draft for technical accuracy, safety, and clarity."
- "Prepare the final Jekyll Markdown file, but do not publish it."
