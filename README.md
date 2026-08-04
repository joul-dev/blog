# joul.dev

Personal blog — Hugo + PaperMod, deployed on GitHub Pages.

## Setup (one time)

1. Create a **public** repo named `blog` on github.com/joul-dev, then push this folder:
   ```bash
   git init && git add -A && git commit -m "Initial blog setup"
   git branch -M main
   git remote add origin git@github.com:joul-dev/blog.git
   git push -u origin main
   ```
2. Repo → Settings → Pages → Source: **GitHub Actions**
3. Same page → Custom domain: `joul.dev` → Save (then check "Enforce HTTPS" once verified)
4. At GoDaddy, add DNS records for joul.dev:
   - 4 × `A` records @ → `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
   - `CNAME` www → `joul-dev.github.io`

DNS + certificate can take up to an hour. After that, every push to `main` deploys automatically.

## Writing

```bash
hugo new content/posts/my-post.md   # then set draft: false when ready
hugo server -D                       # local preview at localhost:1313
```
