# Aaron's Security Lab

<div align="center">

![AaronSecure logo](./assets/img/favicons/logomaster.png)

**Cybersecurity · Networking · Digital Forensics**

[![Live Site](https://img.shields.io/badge/Live_Site-aaronsecure.github.io-0ea5e9?style=for-the-badge&logo=githubpages&logoColor=white)](https://aaronsecure.github.io/)
[![GitHub](https://img.shields.io/badge/GitHub-AaronSecure-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/AaronSecure)
[![Email](https://img.shields.io/badge/Email-ituraaron77@gmail.com-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:ituraaron77@gmail.com)

</div>

---

## About

This repository powers my personal security portfolio and technical blog — **Aaron's Security Lab**. It documents hands-on labs, penetration testing write-ups, networking experiments, and digital forensics research in a clean, searchable format.

I'm **Aaron Itur**, a cybersecurity practitioner focused on turning lab work into clear, reproducible documentation that shows both methodology and results.

## Focus Areas

| Area | What I Document |
|------|-----------------|
| **Cybersecurity** | Penetration testing, vulnerability analysis, privilege escalation, CTF walkthroughs |
| **Networking** | Host discovery, scanning, network topology mapping, service enumeration |
| **Digital Forensics** | Evidence handling, analysis workflows, and investigative reporting |

## Featured Work

### [PWNLAB_INIT — Full System Compromise](https://aaronsecure.github.io/posts/PWNLABINIT-enumeration/)

End-to-end penetration test of the PwnLab: init CTF target, from unauthenticated web enumeration through root access:

- Local File Inclusion (LFI) and PHP filter abuse
- Credential extraction from MySQL
- Web shell upload via MIME-type bypass
- Lateral movement across user accounts
- SUID binary exploitation for root

**Tools used:** Nmap, Burp Suite, Nikto, Netcat, MySQL CLI, CyberChef

### [Kioptrix 4 — LigGoat Web Compromise](https://aaronsecure.github.io/posts/kioptrix-4-penetration-test/)

End-to-end penetration test of Kioptrix Level 4, from host discovery through root on Ubuntu:

- SQL injection and IDOR on the LigGoat login portal
- SSH access and LigGoat restricted-shell breakout
- Empty MySQL root credentials in PHP source
- Privilege escalation via MySQL UDF `sys_exec`

**Tools used:** Netdiscover, Nmap, Gobuster, Firefox, SSH, MySQL CLI

### [Kioptrix 5 — FreeBSD Compromise](https://aaronsecure.github.io/posts/kioptrix-5-penetration-test/)

End-to-end penetration test of Kioptrix Level 5, from host discovery through root on FreeBSD 9.0:

- Hidden pChart 2.1.3 application behind a default Apache page
- Directory traversal / LFI for system file disclosure
- phptax on port 8080 and a `www` foothold
- Local privilege escalation on an end-of-life FreeBSD kernel

**Tools used:** Netdiscover, Nmap, Burp Suite, curl, Metasploit, Netcat, searchsploit

## Tech Stack

| Layer | Tools |
|-------|-------|
| **Site generator** | [Jekyll](https://jekyllrb.com/) 4.x |
| **Theme** | [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 7.x |
| **Hosting** | [GitHub Pages](https://pages.github.com/) |
| **CI/CD** | GitHub Actions (`.github/workflows/pages-deploy.yml`) |
| **Assets** | Node.js build pipeline (Rollup, PurgeCSS) |

## Project Structure

```text
├── _posts/              # Lab reports and blog posts
├── _tabs/               # Static pages (About, etc.)
├── assets/
│   ├── img/             # Avatars, favicons, lab screenshots
│   ├── css/             # Theme styles
│   └── js/              # Theme scripts
├── _config.yml          # Site configuration
└── .github/workflows/   # GitHub Pages deployment
```

## Run Locally

**Requirements:** Ruby 3.x, Bundler, Node.js (LTS)

```bash
# Install dependencies
bundle install
npm install

# Build theme assets
npm run build

# Serve the site (development)
bundle exec jekyll serve
```

Open [http://localhost:4000](http://localhost:4000) in your browser.

## Deploy

Pushing to the `main` branch triggers the GitHub Actions workflow, which builds the site and publishes it to GitHub Pages at [aaronsecure.github.io](https://aaronsecure.github.io/).

## Connect

- **Portfolio:** [aaronsecure.github.io](https://aaronsecure.github.io/)
- **GitHub:** [@AaronSecure](https://github.com/AaronSecure)
- **X (Twitter):** [@Aaronprompt](https://twitter.com/aaronprompt)
- **Email:** [ituraaron77@gmail.com](mailto:ituraaron77@gmail.com)

## License

Site content and lab write-ups are © Aaron Itur. The Jekyll theme is based on [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) (MIT License).
