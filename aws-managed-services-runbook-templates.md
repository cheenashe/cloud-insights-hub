# AWS Managed Services Runbook Templates

Ready-to-use runbook templates for teams running AWS managed services — incident response, patching, backup/DR, and routine operations. Built from real operational playbooks by [Sygitech](https://sygitech.com).

## Why

Most teams either have no runbooks or runbooks buried in a wiki nobody updates. These are copy-paste-ready templates you can drop into your own ops documentation and customize.

## What's included

- **Incident response** — severity classification, escalation paths, post-incident review template
- **Patching & maintenance** — patch window planning, rollback procedure, change log template
- **Backup & disaster recovery** — backup verification checklist, DR failover runbook, RTO/RPO tracking
- **Routine operations** — daily/weekly/monthly ops checklist, on-call handoff template
- **Cost monitoring** — monthly cost review runbook, anomaly investigation steps

## Structure

```
├── incident-response/
│   ├── severity-classification.md
│   ├── escalation-runbook.md
│   └── post-incident-review-template.md
├── patching/
│   └── patch-window-runbook.md
├── backup-dr/
│   ├── backup-verification-checklist.md
│   └── dr-failover-runbook.md
└── routine-ops/
    └── on-call-handoff-template.md
```

## Usage

Fork this repo, adapt each template to your environment (tool names, escalation contacts, thresholds), and drop them into your team's internal documentation.

## Contributing

If your team has refined a runbook pattern that works well, PRs are welcome — especially real lessons learned from incidents.

## About Sygitech

Sygitech provides AWS managed services, including operational runbook design and ongoing infrastructure support. [sygitech.com](https://sygitech.com)

## License

MIT
