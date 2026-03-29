# Full-Stack Application Verification

[![GitHub stars](https://img.shields.io/github/stars/olakunlevpn/olakunlevpn-meta-cognitive-verification?style=flat-square)](https://github.com/olakunlevpn/olakunlevpn-meta-cognitive-verification/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)

Your AI coding agent becomes a 7-person verification team. After development is complete, this skill audits the entire application against its original plan from 7 expert perspectives -- catching bugs, data mismatches, missing features, and security issues before they hit production.

This is an **agent skill** that works with Claude Code, Cursor, Cline, Gemini CLI and 40+ other AI coding agents.

## Installation

```bash
npx skills add olakunlevpn/olakunlevpn-meta-cognitive-verification
```

## The Problem

Developer says "it's done." But:
- Frontend accesses `user.firstName` while backend returns `user.first_name`
- Three pages from the plan were never built
- Payment integration is still in sandbox/test mode
- Forms accept data the backend rejects
- Admin routes have no auth middleware
- Console.logs and TODO comments are everywhere

These pass code review. They pass "it works on my machine." They crash in production.

## The Solution

This skill runs your application through 7 expert perspectives. Every claim requires evidence. Every field is traced from database to UI. Nothing ships until confidence hits 0.9.

## How to Use

The skill auto-triggers when your agent detects a verification task. You can also invoke it directly with these prompts:

**Verify a completed feature against the plan:**
```
Verify the Orders section against this plan: [paste your plan]
```

**Full app verification:**
```
Run full verification on this project. Here is the original plan: [paste plan or reference plan file]
```

**Single perspective on a specific area:**
```
Run the Tech Lead perspective on the authentication system
```

```
Run the QA Engineer perspective on the checkout flow
```

```
Run the data contract audit on all pages
```

**Pre-deployment check:**
```
We're about to deploy. Run the DevOps perspective on this project.
```

**When a developer says "it's done":**
```
The developer says the dashboard section is complete. Verify everything against the plan in docs/plan.md
```

The agent will decompose the plan, run all 7 perspectives, trace every data contract field-by-field, search for leftover junk (TODO, console.log, dummy data), and produce a structured report with a confidence score and verdict.

Always give it the plan. Without the plan, it has nothing to verify against.

## The 7 Perspectives

**QA Engineer** -- Find bugs. Break things. Verify every feature works as specified. Check every form, every error state, every edge case. Search for dummy data, debug statements, and unfinished work.

**Product Manager** -- Verify every user story was built. Trace user flows end-to-end. Check business logic, notifications, permissions, and settings persistence.

**Project Manager** -- Verify scope completeness. Create a plan-vs-implementation matrix. Flag partial implementations, placeholder pages, and missing integrations.

**Tech Lead** -- Audit code quality. Run the data contract audit. Verify security (auth, CSRF, XSS, SQL injection). Check for N+1 queries and missing indexes. Verify credentials are properly secured (without exposing values).

**Business Analyst** -- Verify business rules match requirements. Check calculations, status workflows, validation rules, audit trails, and timezone handling.

**DevOps Engineer** -- Verify production readiness. Check dependencies, migrations, environment config, logging, health checks, file storage, cache/session drivers, and error tracking.

**Stakeholder** -- Final acceptance. Walk through every section as an end user. Real data, real flows, real devices.

## The Data Contract Audit

The most critical check. For every frontend component that displays backend data:

```
Backend returns:                  Frontend accesses:
{                                 order.referenceNumber     FAIL (reference_number)
  reference_number: "ORD-001",    order.status              PASS
  status: "pending",              order.totalAmount         FAIL (total_amount)
  total_amount: 15000,            order.customer.name       FAIL (customer.full_name)
  customer: {                     item.productName          FAIL (product_name)
    full_name: "John Doe",        item.qty                  FAIL (quantity)
    email: "john@example.com"     item.price                FAIL (unit_price)
  }
}
```

7 mismatches. 7 runtime crashes waiting to happen. Caught before deployment.

## How It Works

**Phase 1: Plan Decomposition** -- Break the original plan into a verification matrix. Every feature, page, endpoint, and permission becomes a checklist item.

**Phase 2: Seven-Perspective Verification** -- Run all 7 perspectives sequentially. Each builds on findings from the previous one. Evidence required for every pass and every fail.

**Phase 3: Verification Report** -- Structured output with pass/fail counts per perspective, critical failures, data contract mismatches, confidence score, and a clear verdict.

## Example Output

```
Overall Confidence: 0.83 / 1.0
Verdict: NOT APPROVED (3 critical failures)

Critical Failures:
1. [Data Contract] 7 field name mismatches between API and frontend
2. [Security] Admin routes missing auth middleware
3. [Config] Payment integration still in sandbox/test mode

Blocking Items: Fix all 3 critical failures before release.
```

## What Gets Caught

Things this skill catches that normal code review misses:
- Field name casing mismatches (camelCase vs snake_case)
- Frontend accessing nested data that backend returns flat (or vice versa)
- Null/undefined access on optional fields
- Frontend validation looser than backend validation
- Status workflow allowing invalid transitions
- Permissions enforced in UI but not in backend
- Third-party integrations still in sandbox/test mode
- N+1 database queries hidden behind relationships
- Missing database indexes on filtered/sorted columns
- TODO comments and console.logs in production code
- Placeholder pages that were never completed
- Features in the plan that were never started

## Files

```
SKILL.md        # Main skill file (loaded by AI agents)
REFERENCE.md    # Detailed examples, grep commands, report templates
```

## Contributing

Found a verification check that should be included? Open an issue or submit a pull request.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
