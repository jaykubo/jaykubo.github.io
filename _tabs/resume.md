---
icon: fas fa-file-lines
order: 5
layout: page
title: Resume
---

<style>
  .resume-contact {
    color: var(--text-muted-color);
    font-size: 0.9rem;
    margin-bottom: 1.25rem;
  }
  .resume-dl {
    display: inline-block;
    padding: 0.4rem 0.9rem;
    margin-bottom: 1.5rem;
    border: 1px solid var(--main-border-color);
    border-radius: 0.4rem;
    font-size: 0.85rem;
    text-decoration: none;
    color: var(--link-color);
    background: var(--card-bg);
  }
  .resume-dl:hover {
    background: var(--card-hover-bg);
    text-decoration: none;
  }
  .rows p {
    margin: 0;
    padding: 0.2rem 0;
  }
  .rows .who,
  .rows .when {
    color: var(--text-muted-color);
    font-size: 0.85rem;
  }
  .rows .when {
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
  }
</style>

<p class="resume-contact">
Tokyo, Japan · <a href="https://linkedin.com/in/jaykubo">linkedin.com/in/jaykubo</a> · <a href="https://github.com/jkubo">github.com/jkubo</a> · Japan and U.S. dual citizen
</p>

<a class="resume-dl" href="/assets/Jay_Kubo_Resume.pdf">Download PDF</a>

Applied AI architect in Tokyo. A decade advising Japanese enterprises from inside Google Cloud, CrowdStrike, and Rakuten—now building agent orchestration and LLM evaluation on a 20-node cluster across four sites. Native Japanese and English, technical register.

## EXPERIENCE

### Founder, kub0 LLC

**Tokyo, Japan** | Jul 2020 - Present

#### Agent Orchestration

- Claude Code orchestrates the fleet: it distills work to Grok Build and Gemini workers under kernel sandboxes, and a write needs a YubiKey grant. Done is a re-run predicate, not the worker's self-report.
- Claude Code TUI status line for remaining context (advisory); same on Grok sessions. Gaius as a Claude Code plugin (OSS, PyPI, MCP). ~30 skills, including a scoped-diff security audit that stays inside the context window.
- Project Gaius: 10,000+ facts from Claude and Gemini sessions. 98.6% retrieval hit@10 on LongMemEval-S, offline, no GPU.

#### Evaluation

- LLM-as-judge (base, RAG, fine-tuned) across two model lineages. Fine-tune scored 2.79 vs RAG 3.92, so I shipped RAG. Judge agreement measured 47% against a 70% bar, so I shipped on the margin, not the grade.
- Found alerts that passed review and could never fire (always-zero compare; missing credential). Adversarial harness: 85 detected, 5 bypassed, all test debt.

#### Platform

- Packaged Kubernetes security product for GCP Marketplace (Helm, GKE). Approved in Anthropic's Cyber Verification Program.
- Daily kube-bench (k3s-cis-1.9) into a CIS posture API: living hardening state, not a one-time audit.
- Cross-compiled a custom ARM64 kernel with BPF_LSM, absent from the stock kernel, to run eBPF enforcement on ARM nodes.
- 20-node Kubernetes cluster, Austin, Los Angeles, Torrance, Tokyo. Ansible, WireGuard, ~1 TB GPU memory, 47 TB DRBD-replicated block (NVMe + SATA). OpenTelemetry/Grafana, two-proxy OAuth2. Site outage shifts traffic automatically.
- vLLM serving Gemma, Qwen, and an embedding model on AMD ROCm and NVIDIA CUDA. LoRA fine-tuning on a DGX Spark.

### Technical Advisor, Japan Deluxe Tours, Inc.

**Remote** | May 2015 - Present

- Claude in production for their staff: a Google Chat bot that answers operations questions and ships approved changes, gated by Google Groups roles (admin vs staff) and logged. The same roles gate ChatOps merges and experimental features.
- Recommended and stood up their first AWS cloud in 2015. Cut RDS cost ~50% while still on AWS. Migrated the customer site to Cloudflare Workers and on-prem Kubernetes.
- MySQL 8.0 on NVMe with GTID replication, two streaming replicas at 0 s lag.
- Stripe payments (deposits and self-service balance, no stored card). Sign in with Apple and Google on the customer site.
- Legacy staff tooling behind Cloudflare Access. Staff Macs on Tailscale; FleetDM for inventory and CVE.

### Associate Analyst, CrowdStrike (Falcon Complete Japan)

**Austin, TX** | Jan 2025 - May 2025

- Japanese-language translator on high-priority Falcon Complete Japan escalations—analysts and senior analysts pulled me in, and what I wrote is what went out. Also worked with a GovCloud senior analyst on Japan commercial-cloud operations.
- Claude workflow for Japanese customer responses (SLA docs, JIRA, Confluence). Daily threat-intel feeds and ~30 SOC automations the team kept.

### Application Support Analyst, EPAM Systems (Google Cloud)

**Austin, TX** | Nov 2019 - Mar 2021

- Frontline support for Japanese enterprise customers on Google Cloud, customer-facing under Google: BigQuery, Dataflow, Cloud Composer, Cloud Spanner, Cloud SQL, App Engine, Dialogflow. Gaming, e-commerce, rail, trading houses, telecom, retail, banking.
- Bridged U.S. and Japan on high-priority escalations. Chrome extension cut Platinum P0/P1 SLA misses to near zero.
- Carried Won't Fix rulings back to enterprise customers; moved stalled private cases to the public tracker.
- Filed P2s in Google Issue Tracker from those accounts, including: Cloud SQL export blocked for viewer IAM, Dataproc fluentd leak on the master until OOM, billing accounts missing in Console, Cloud SDK output drift between Python 2 and 3.

### Security Engineer & Data Analyst, Rakuten

**San Mateo, CA** | Sep 2015 - Jul 2018

- Created an internal vulnerability-intelligence platform to collect, search, alert, and visualize CVEs in real time. Sold across subsidiaries (Ebates, Slice, Buy.com, Linkshare). Whitebox and blackbox pentests for the same brands.
- Kafka pipeline: Palo Alto PAN-OS syslog into Elasticsearch. Authored the company MySQL and Oracle security standards from CIS benchmarks, across thousands of instances. On-call DBA for 100+ Japan-region services (~5,000 VMs).
- Azure pipelines and DOMO dashboards. A/B and multivariate tests in Oracle Maxymiser across six markets. 4-month OJT with the Rakuten HQ data-science team in Tokyo (CNN product-image matching, OpenCV, Chainer).
- Led Rakuten's first PII/SPII sweep for GDPR compliance, embedded with the DBA team as a dual post across the company's full database landscape. Coordinated with thousands of internal teams, much of it in Japanese.

## EDUCATION

<div class="rows">
<p>B.A., Mathematics and Computer Science (Courant Institute) · <span class="who">New York University</span> · <span class="when">May 2015</span></p>
</div>

Coursework: Artificial Intelligence · Algorithmic Problem Solving · Database Systems (Courant, graduate) · Mathematics of Finance · Meaning of Leadership (Wagner) · Digital Marketing and Social Media Analytics (Stern)

## CERTIFICATIONS

<div class="rows">
<p>Certified Falcon Administrator (CCFA) · <span class="who">CrowdStrike</span> · <span class="when">Apr 2025 - Apr 2028</span></p>
<p>Remote Pilot Certification (Part 107) · <span class="who">FAA</span> · <span class="when">Issued Oct 2025</span></p>
<p>Advanced Open Water Diver · <span class="who">PADI</span> · <span class="when">Issued Aug 2025</span></p>
<p>Private Pilot License · <span class="who">FAA</span> · <span class="when">Issued Dec 2024</span></p>
<p>Continuous Monitoring Certification (GMON) · <span class="who">GIAC</span> · <span class="when">Expired May 2026</span></p>
</div>

## TRAINING

<div class="rows">
<p>Assessing and Exploiting Control Systems and IIoT · <span class="who">Black Hat</span> · <span class="when">Apr 2024</span></p>
<p>Abusing and Protecting Kubernetes, Linux and Containers · <span class="who">Black Hat</span> · <span class="when">Mar 2024</span></p>
<p>Reverse Engineering Firmware with Ghidra · <span class="who">Black Hat</span> · <span class="when">Dec 2023</span></p>
<p>Machine Learning (inaugural) · <span class="who">Black Hat</span> · <span class="when">Aug 2023</span></p>
<p>Reversing Signal with Software-Defined Radio · <span class="who">Black Hat</span> · <span class="when">Aug 2023</span></p>
<p>Advanced Infrastructure Hacking · <span class="who">Black Hat</span> · <span class="when">Mar 2023</span></p>
<p>ICS/SCADA Security Essentials · <span class="who">SANS</span> · <span class="when">Dec 2022</span></p>
<p>A Crash Course in Practical Fast Forensics with a Red Teaming Perspective (Japanese) · <span class="who">Black Hat</span> · <span class="when">Nov 2022</span></p>
</div>

## OPEN SOURCE & WRITING

PyPI [`gaius-memory`](https://pypi.org/project/gaius-memory/) · Ubuntu kernel bug LP#2150798 · cluster engineering writing [across this site](/archives/)

## SKILLS

**Spoken languages:** Japanese (native; business and technical) · English (native; business and technical)

**AI:** Claude API · Claude Code · Grok Build · Gemini · RAG · LLM-as-judge eval · vLLM · LoRA · MCP · Python · Go

**Cloud:** GCP (BigQuery, Dataflow, Cloud Composer, Cloud Spanner, Cloud SQL, GKE, Marketplace, Dialogflow) · AWS · Azure · Kubernetes · Ansible · OpenTelemetry/Grafana

**Security:** Linux · eBPF · YubiKey · endpoint · threat intelligence · penetration testing
