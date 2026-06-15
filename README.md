# Personal Engineering Blog

A minimalist Hugo site.

## Layout

- `layouts/index.html` renders the homepage with nested blog/project lists.
- `layouts/_default/single.html` renders posts and project pages.
- `static/css/styles.css` contains the dark monospace styling.

## Content

- `content/posts/` contains Markdown blog posts.
- `content/projects/` contains Markdown project pages.
- `content/about.md` is an about page.

## Build

Install Hugo and run:

```bash
hugo --minify
```

The generated static site will be in `public/`.

## Deploy

Any static host works:

- GitHub Pages
- Cloudflare Pages
- Netlify
- Vercel
