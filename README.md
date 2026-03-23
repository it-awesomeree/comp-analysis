# Comp Analysis

Documentation of the Competitive Analysis project — bots, web app (frontend & backend), database, and infrastructure.

## Structure

| Folder | Scope | Description |
|--------|-------|-------------|
| [`comp 1/`](comp%201/) | Shopee MY | Original comp analysis page — legacy `Shopee_Comp` table |
| [`comp 2/`](comp%202/) | Shopee MY VVIP & Shopee SG | VVIP + SG pages — normalized tables (`Shopee_My_Products` + `Shopee_Comp_Data`), region-based isolation |
| [`plans/`](plans/) | Cross-cutting | Investigation and fix plans (e.g., AW-386 performance) |

## Quick Links

### Comp 1 — Shopee MY
- [Frontend Doc](comp%201/comp-analysis-frontend.md) — UI components, data flow, interactions
- [Backend Doc](comp%201/comp-analysis-backend.md) — API routes, repository, DB schema, caching

### Comp 2 — VVIP & SG
- [Frontend Doc](comp%202/comp-analysis-2-frontend.md) — UI components, 3-level hierarchy, icons, data flow
- [Backend Doc](comp%202/comp-analysis-backend.md) — API routes, repository, DB schema, query patterns, caching
