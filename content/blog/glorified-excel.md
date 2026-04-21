---
title: "Glorified Excel: The Dashboard Feature Request You Should Push Back On"
date: 2023-09-01
tags: ["software-engineering", "biotech", "dashboards", "product"]
summary: "In biotech, the most common dashboard feature request is a filterable, sortable table. The fastest solution is usually a CSV download and the spreadsheet software people already know."
---

The most common feature request I get from scientists and lab teams is some variation of: "Can we add a table to the dashboard where I can filter by X, sort by Y, and search for Z?"

Every time, I have the same internal reaction: you're asking me to build a worse version of Excel.

## The Pattern

A pipeline produces data. Someone asks for a dashboard. The first iteration has some charts. Then the requests start: add a column, filter by date, sort by this metric, export the filtered view, conditional formatting, pivot by batch.

Each request is reasonable in isolation. Taken together, they describe a spreadsheet. And there is nothing I can build — no React table component, no ag-Grid config — that will beat what Excel or Google Sheets already do. These are tools with decades of development behind them, used daily by the people requesting the feature.

## What I Do Instead

When someone asks for a filterable table, I ask what they're actually trying to do once they've found the rows they care about:

- **"I want to check specific samples"** — a search bar is faster to build and use than a table.
- **"I want to explore the data"** — a "Download CSV" button and five minutes in their spreadsheet of choice gets them further than anything I could build in a sprint.
- **"I want to spot anomalies"** — they need charts with thresholds or alerting, not a sortable column.

The CSV download is the one I reach for most. It's trivial to implement, it respects that scientists already have workflows in Excel or R or Python, and it avoids the maintenance burden of a custom table UI that will never be as good as the tools it's imitating.

A filterable, sortable, paginated table with search sounds simple. In practice it means pagination, server-side filtering, state management, shareable URLs, accessibility, performance testing, and an endless stream of requests to tweak columns and filters. That's not a feature — that's a product. And it's a product that already exists.

## When a Table Is the Right Call

Tables make sense when the data is live and changes too frequently for a static export, when clicking a row navigates somewhere, when it combines data from multiple sources, or when access control matters. But when the request is "I want to look at my data and filter it," the honest answer is usually: here's a download button.
