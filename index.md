---
layout: home
---

# Welcome to My Cybersecurity Blog

This is where I share my CTF writeups, security research, and penetration testing notes.

## Latest Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <span class="post-date"> - {{ post.date | date: "%B %d, %Y" }}</span>
    </li>
  {% endfor %}
</ul>

## About Me

I'm Rishabh Arora, a cybersecurity professional passionate about offensive security, malware analysis, and vulnerability research. Currently working toward OSCP certification.

- **GitHub:** [R1shabh-Arora](https://github.com/R1shabh-Arora)
- **LinkedIn:** [R1shabh-Arora](https://linkedin.com/in/R1shabh-Arora)
- **TryHackMe:** Top 9%
