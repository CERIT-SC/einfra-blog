---
title: "Contribution Guidelines"
date: "2025-08-08"
thumbnail: "/img/capybara-clustery-grafika-m.jpg"
author: "e-INFRA CZ"
---

# e-INFRA CZ Blog Contribution Guidelines  
**Version 1.0 — Effective from 1 August 2025**

## Purpose of the Blog
The e-INFRA CZ blog serves as a **technical showcase** of expertise, innovations, and solutions developed within **CERIT-SC, CESNET, and IT4Innovations**.


## Key Technical Areas for Contributions

### 1. Unique Infrastructure and Hardware
- In-depth analyses of newly deployed systems (e.g., HPC cluster parameters, storage solutions)  
- Case studies of non-standard problem-solving (e.g., exascale cooling, power optimisation, unique system architectures)

### 2. In-House Software and Tools
- Presentations of open-source tools developed by our teams (with GitHub links)

### 3. Operational Know-how
- Workflow optimisation for extreme workloads (HPC, AI/ML, big data)  
- Lessons learned from operating critical infrastructure (monitoring, security incidents, disaster recovery)

### 4. Scientific Use Cases
- Collaborations with research teams on high-impact projects (e.g., medicine, climate modelling)  
- Quantifying the impact of our infrastructure on specific research

## Requirements for Technical Depth

| Requirement        | Expectation |
|--------------------|-------------|
| **Accuracy**       | All claims must be supported by data/metrics |
| **Reproducibility**| Include verified configurations and code snippets |
| **Originality**    | Focus on unique or technically interesting solutions |
| **Visualisation**  | Mandatory technical diagrams (architecture, performance graphs, tables, etc.) |

## Prohibited Topics
- General technology descriptions without direct relation to our implementation  
- Marketing language (“the best solution on the market”)  
- Theoretical discussions without practical validation in our environment  
- Commercial advertising, political content, unverified speculation  
- Any content violating copyright or ethical standards  

## Who Can Contribute
- **Internal Authors:** Staff from CERIT-SC, CESNET, IT4Innovations (subject to editorial approval)  
- **Guest Authors:** External experts (subject to editorial approval)  


## Author Best Practices
- Posts must relate to **e-infrastructures, HPC, cloud computing, data storage, scientific computing, or related technologies**.

### Recommended Article Structure
1. **Problem** → **Our Solution** → **Technical Details** → **Measured Results** → **Lessons Learned**
2. **Referencing:**
   - Include references to internal documentation/projects (unless confidential)  
   - Link to relevant open-source repositories
3. **Transparency:**
   - Clearly state limitations (“This works for X, but not for Y due to Z”)  
   - Describe failed experiments as valuable lessons

## Contribution Process

### 1. GitHub Registration
- Create a GitHub account (if you do not have one)  
- Request contributor access from the editors (contact: `k8s@cerit-sc.cz`)

### 2. Creating a Post
- Fork the repository: [https://github.com/CERIT-SC/einfra-blog](https://github.com/CERIT-SC/einfra-blog)  
- Add posts to `/content/blog/TITLE` in **Markdown** format  
- Place images in `/assets/img/TITLE` (`.jpg`, `.png`)  
- Use standard Markdown syntax
- Submit a **Pull Request (PR)** to the main branch when the post is ready  
- In the PR description, include:
  - Content summary  
  - Target category (e.g., HPC, Cloud, Storage)  
  - Suggested publication date

### 3. Alternative Submission
- You may also send the post via email to `k8s@cerit-sc.cz` preferably in Markdown format, including all images as attachments.

## Post Formatting

- Mandatory Frontmatter Example
```yaml
---
date: '2025-02-22T12:26:32Z'
title: 'Machine Learning in Mass Spectrometry'
thumbnail: '/img/spectus/alchymista_v_knihovne1.jpeg'
author: "Ivana Krenkova"
description: "SpecTUS: a new tool in da house"
tags: ["Jiří Novák", "CERIT-SC", "SpecTUS", "Machine Learning", "Research"]
colormode: true
---
```

- Language: English

- Recommended Length: 800–2500 words

## Editorial Process

- **Review:** All PRs are reviewed by editors for grammar, technical accuracy, and compliance with these guidelines.

- **Publication:** Approved posts will be published by editors. Editors may adjust the title, metadata, or categories for better discoverability. Editors may request additional information or images.

- **Post-publication Edits:** For corrections, update the Markdown file and submit a new PR.

## Copyright and Licensing
All posts are published under `CC BY-SA 4.0`.
- The author guarantees they are the original creator of the text and images (or have the rights to use them).
- External sources must be explicitly attributed (e.g., data sources, stock images).



## Why These Standards Exist
The blog aims to demonstrate technical competence to:
- The scientific community (credibility for grant applications)
- European partners
- Students and potential recruits

Every post should help build e-INFRA CZ’s reputation as a leader in e-infrastructures.

## Contact
Technical support, topics, and content queries: k8s@cerit-sc.cz



