# Sentinel Projects

A collection of KQL queries, workbooks, and operational tools for **Microsoft Sentinel** and **Log Analytics** — built by a practitioner, for practitioners.

**Author: Venicia Solomons, Solution Engineer - Cloud and AI Security, Microsoft**

---

## What This Repo Is About

This repository is a growing library of practical, production-tested resources for teams running Microsoft Sentinel as their SIEM. The focus is on the work that sits between the documentation and the day-to-day reality of operating a security platform at scale: cost governance, ingestion analysis, benefit tracking, detection tuning, and operational visibility.

Everything here is designed to be **copied straight into the Log Analytics query editor** or adapted into workbooks and scheduled rules with minimal effort.

---

## Projects

| Project | Description |
|---------|-------------|
| [Defender for Servers P2 — Cost & Savings Analysis](Defender For Servers/) | Compound KQL queries that calculate daily billable ingestion after the Defender for Servers P2 benefit, track actual savings, and surface per-table breakdowns for FinOps reporting. |

*More projects coming soon.*

---

## Who This Is For

- **SOC analysts** who need to understand what data is flowing into their workspace and why
- **Security engineers** building and tuning detections in Sentinel
- **FinOps / cloud cost teams** responsible for Log Analytics and Sentinel spend
- **Architects** planning Sentinel deployments, commitment tiers, or multi-workspace designs

---

## How to Use

1. Browse the project folders above
2. Each project has its own full documentation files
3. `.kql` files can be pasted directly into the **Log Analytics Logs blade** in the Azure portal
4. Adjust the configuration variables at the top of each query to match your environment

---

## Contributing

Contributions, suggestions, and feedback are welcome. If you have a query or tool that has saved you time in your Sentinel environment, feel free to open a pull request.

