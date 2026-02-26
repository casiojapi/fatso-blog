# fat blog

Blog for Fat Solutions. Hugo-based, markdown posts.

## Run locally

```
hugo server
```

Open `http://localhost:1313`.

## Write a post

1. Create a new post:

```
hugo new posts/my-slop.md
```

2. Edit the generated file in `content/posts/`. The and fill the post data:

```yaml
---
title: "Your Post Title"
date: 2026-02-26
description: "previou text on the home page."
tags: ["zk", "privacy", "audits", "whatever you feel makes sense, try to keep it less than 3"]
---

Your markdown content here.
```

- `title` -- shows as the post heading
- `date` -- determines ordering (latest first)
- `description` -- preview text on the home page card
- `tags` -- used for filtering/grouping on the tags page. Use lowercase, kebab-case
- `subtitle` (optional) -- shows below the title on the post page

3. Add images by placing them in `static/img/` and referencing them:

```markdown
![alt text](/img/my-image.png)
```

4. Preview locally with `hugo server` before pushing.

## Publish

**If you're part of fatso:**

1. Push your post to the `review` branch
2. Open a PR from `review` to `master`
3. A preview deployment is built automatically -- the preview link gets posted as a comment on the PR
4. Once reviewed, merge to `master` -- production deploys automatically via GitHub Pages

**If you're not:** fork, write your post, open a PR to `review`. PRs are welcome.

## Deployments

- **Production**: auto-deploys on push to `master` via GitHub Actions + GitHub Pages
- **Preview**: auto-deploys on push to `review` -- preview URL gets commented on any open PR to `master`

Repo settings needed (one-time): go to Settings > Pages > Source > set to "GitHub Actions".

## Tags

Tags are defined per-post in the frontmatter. No need to register them anywhere -- Hugo picks them up automatically and generates the tag pages.

Some existing tags: `zk`, `security`, `exploits`, `tornado-cash`, `zcash`, `aztec`, `polygon`, `starknet`.

Use whatever makes sense. Keep them lowercase.

## Structure

```
content/posts/    -- markdown posts go here
static/img/       -- images
themes/fat/       -- blog theme (matches fatsolutions.xyz aesthetic)
hugo.toml         -- site config
```
