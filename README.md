# Rishabh's Cyber Blog

[![Website](https://img.shields.io/website?url=https%3A%2F%2Fblogs.rishabh.uk)](https://blogs.rishabh.uk)
[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-deployed-brightgreen)](https://blogs.rishabh.uk)
[![Jekyll](https://img.shields.io/badge/Jekyll-4.3-blue)](https://jekyllrb.com/)

Personal cybersecurity blog featuring CTF writeups, penetration testing notes, and security research.

**Live Site:** [https://blogs.rishabh.uk](https://blogs.rishabh.uk)

---

## About

This blog documents my journey in cybersecurity — from Hack The Box machines to vulnerability research. It's built with Jekyll and hosted on GitHub Pages with a custom domain.

## Tech Stack

- **Static Site Generator:** [Jekyll](https://jekyllrb.com/)
- **Theme:** [Minima](https://github.com/jekyll/minima) (dark skin)
- **Hosting:** GitHub Pages
- **DNS:** Cloudflare (CNAME + SSL)
- **Domain:** blogs.rishabh.uk

## Project Structure

```
blog/
├── _config.yml              # Jekyll configuration
├── _posts/                  # Blog posts (Markdown)
│   └── YYYY-MM-DD-title.md
├── about.md                 # About page
├── index.md                 # Homepage
├── CNAME                    # Custom domain config
└── Gemfile                  # Ruby dependencies
```

## Adding a New Post

1. Create a new Markdown file in `_posts/` with the naming convention:
   ```
   YYYY-MM-DD-your-post-title.md
   ```

2. Add front matter at the top:
   ```markdown
   ---
   title: "Your Post Title"
   date: YYYY-MM-DD HH:MM:SS +0000
   description: "Brief description of the post"
   tags: [tag1, tag2, tag3]
   categories: [ctf, writeup, research]
   ---
   ```

3. Write your content in Markdown below the front matter

4. Commit and push:
   ```bash
   git add _posts/YYYY-MM-DD-your-post-title.md
   git commit -m "Add: [Post Title]"
   git push origin main
   ```

5. GitHub Pages will automatically rebuild and deploy

## Writing Tips

- Use `##` for main headings (H1 is reserved for post title)
- Include code blocks with language tags: ` ```bash `, ` ```python `
- Add screenshots with: `![alt text](/assets/images/filename.png)`
- Link to tools/resources for reader convenience
- Tag appropriately for discoverability

## Local Development

To preview changes locally before pushing:

```bash
# Install dependencies
bundle install

# Serve locally
bundle exec jekyll serve

# Open browser to http://localhost:4000
```

## Posts

| Date | Title | Tags |
|------|-------|------|
| 2026-03-03 | [HackTheBox: WingData Writeup](https://blogs.rishabh.uk/2026/03/03/wingdata-htb-writeup/) | HTB, Linux, CVE-2025-4517, Privilege Escalation |

## Connect

- **Main Site:** [rishabh.uk](https://rishabh.uk)
- **Portfolio:** [portfolio.rishabh.uk](https://portfolio.rishabh.uk)
- **GitHub:** [@R1shabh-Arora](https://github.com/R1shabh-Arora)
- **LinkedIn:** [R1shabh-Arora](https://linkedin.com/in/R1shabh-Arora)
- **TryHackMe:** Top 9%

---

*Built with ❤️ and caffeine. Opinions are my own.*
