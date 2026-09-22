# Stetser Infrastructure Engineering Lab

*Linux • Docker • Infrastructure Automation • Security Hardening • Networking • Observability • Databases • Backup & Recovery*

This repository documents practical infrastructure engineering work from a self-hosted Linux environment, with an emphasis on repeatable operations, security hardening, service reliability, forensic analysis, and evidence-driven troubleshooting.

The goal is not to present a polished fictional lab. It is to document real engineering work: problems encountered, decisions made, controls tested, failures analyzed, fixes validated, and lessons retained.

## Focus Areas

- Linux systems administration
- Docker and containerized services
- Infrastructure automation
- Security hardening
- Service health and dependency management
- Networking and reverse proxying
- Observability and update management
- PostgreSQL, MariaDB, and Redis operations
- Storage, backup, and recovery
- Incident analysis and root-cause investigation
- Source-code and binary forensics
- Upstream contribution and bug reporting

## Featured Case Study

### AirDC++ Scheduled Refresh Log Amplification

A large AirDC++ deployment serving a multi-terabyte media library produced extreme log amplification during scheduled recursive share refreshes.

The investigation progressed from production containment to binary comparison, source archaeology, and historical commit analysis. It identified a source-level mismatch between live filesystem monitoring and scheduled recursive refreshes:

- live monitoring uses temporal duplicate-log suppression;
- recursive refreshes aggregate blocked-share errors only at directory scope;
- on very large trees containing intentionally skiplisted files, log output can therefore scale with the number of affected directories.

The case study documents the forensic evidence, source path, historical commit behavior, operational containment, proposed fix, and upstream contribution process.

[Read the case study](case-studies/airdcpp-refresh-log-amplification/README.md)

## Engineering Approach

Work documented here generally follows this sequence:

1. Read-only reconnaissance
2. Evidence capture
3. Risk classification
4. Disposable or isolated testing where appropriate
5. Backup and rollback planning
6. Controlled production change
7. Verification
8. Cleanup
9. Documentation of results and lessons learned

## Repository Structure

```text
.
├── README.md
├── case-studies/
│   └── airdcpp-refresh-log-amplification/
│       └── README.md
├── docs/
└── assets/
```

## Status

This repository is being built incrementally from documented infrastructure work. Additional case studies, architecture notes, hardening controls, and automation examples will be added over time.
