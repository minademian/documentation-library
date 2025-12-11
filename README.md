# Documentation Library

> My collected technical writing: production-tested guides, runbooks, and architecture documentation from real infrastructure work.

[![Documentation Status](https://img.shields.io/badge/docs-active-brightgreen.svg)](.)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Last Updated](https://img.shields.io/badge/updated-December%202025-blue.svg)](.)

## What This Is

This is my personal archive of technical documentation I've written throughout my career. Every guide here:

- Solves a **real problem** I encountered in production
- Contains **tested commands** that actually work
- Reflects **hard-won knowledge** from debugging, deploying, and maintaining live systems
- Is **maintained actively** as tools and best practices evolve

These aren't theoretical tutorials—they're the runbooks I wish I had when I first tackled these problems.

**My approach**: Treat documentation like production code. If it's worth writing once, it's worth versioning, testing, and keeping current.

## What's Current

| Document | Last Updated | Status |
|----------|-------------|---------|
| [Deploying to AWS EKS](./deploying-a-service-to-aws-eks/) | Dec 2025 | ✅ Active |

### Infrastructure & Platform Engineering

#### [Deploying a Service to AWS EKS](./deploying-a-service-to-aws-eks/)
The complete end-to-end playbook I developed for deploying containerized services to Amazon EKS, refined through multiple production deployments.

**What's inside:**
- Terraform infrastructure provisioning (ECR, IAM, networking)
- Docker containerization best practices
- Helm chart development and deployment
- GitHub Actions CI/CD pipeline setup
- EKS cluster management and troubleshooting
- Production-ready configuration patterns

**Who it's for:** Platform engineers, DevOps engineers, SREs  
**Level:** Intermediate to Advanced  
**Time:** 2-3 hours (first deployment), 30 min (subsequent)

---

## Using These Guides

### If You're Here to Learn

1. **Browse** [what I've written](#-what-ive-written) above
2. **Pick** a guide that matches your problem
3. **Follow along**—all prerequisites are listed upfront
4. **Jump around**—each guide has a table of contents for quick navigation

### If You Want to Contribute

I appreciate corrections, updates, and improvements:

```bash
# Fork and clone
git clone https://github.com/yourusername/documentation-library.git
cd documentation-library

# Make your improvements
# Submit a PR
```

## How I Write Documentation

Every guide I write follows the same pattern—because it works:

### Structure
- **Clear prerequisites** upfront (tools, access, assumed knowledge)
- **Time estimates** so you know what you're getting into
- **Quick navigation** for when you need to jump to a specific section
- **Step-by-step instructions** with the actual commands I used
- **Troubleshooting** for the gotchas I hit (so you don't have to)
- **Context and reasoning** not just "what" but "why"

### My Writing Style
- **Direct and imperative**: "Run this", not "you might want to consider running"
- **Code-first**: Real commands that work, not pseudocode
- **Annotated**: Comments explaining what each command does
- **Version-aware**: I note when something is version-specific
- **Security-conscious**: Production best practices, not shortcuts

### Quality Bar
- ✅ I've run every command in a real environment.
- ✅ Code blocks use proper syntax highlighting.
- ✅ Placeholders are obvious (like `<YOUR_CLUSTER_NAME>`).
- ✅ Anything based on work done for employers has been thoroughly redacted.
- ✅ Security implications are called out.
- ✅ Production considerations aren't afterthoughts.

## Technologies I Cover

My documentation spans the stack I work with day-to-day:

| Category | Technologies |
|----------|-------------|
| **Cloud** | AWS (EKS, ECR, IAM, VPC, CloudWatch) |
| **Orchestration** | Kubernetes, Helm, Docker |
| **Infrastructure** | Terraform, CloudFormation |
| **CI/CD** | GitHub Actions, GitLab CI |
| **Languages** | Python, Go, Shell scripting |
| **Observability** | Prometheus, Grafana, CloudWatch |

*This list grows as I document new systems and tools.*

## Contributing

Spotted a typo? Command didn't work? Know a better way? I'd love your input.

**What helps:**
- Bug reports when commands fail
- Updates for newer tool versions  
- Clarity improvements
- Additional troubleshooting tips

**How to contribute:**
1. Fork this repo
2. Make your changes
3. Test them (seriously—run the commands)
4. Submit a PR

I'll review and merge anything that makes the docs better.

## License

MIT License. Use it, fork it, improve it — just give credit where it's due.

---

<div align="center">

**[What I've Written](#-what-ive-written)** • **[Using These Guides](#-using-these-guides)** • **[Report Issues](../../issues)**

*Written and built with ❤️ because documentation is worthy of the same level of care as production code..*

</div>
