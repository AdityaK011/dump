# AdityaK011.github.io

Thinking out loud — a personal digital garden powered by [Quartz](https://quartz.jzhao.xyz/).

## Local Development

```bash
npm install
npx quartz build --serve
```

Visit `http://localhost:8080` to preview locally.

## Adding Notes

Add Markdown files to the `content/` directory. Quartz supports Obsidian-flavored Markdown, wiki-links, tags, and more.

## Deployment

Pushes to `main` automatically trigger a GitHub Actions workflow that builds and deploys to GitHub Pages.
