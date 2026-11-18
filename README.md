# Lead AI Governance Agent

Automated model package validation against SR 11-7, the EU AI Act, and the NIST AI RMF, built with CrewAI and Gemini 2.5 Flash.

## Overview

Before a machine learning model reaches production in a regulated setting, someone has to check it. In financial services, that review draws on the Federal Reserve's SR 11-7 guidance on model risk management, the EU AI Act's requirements for high-risk systems, and the NIST AI Risk Management Framework. Doing this by hand is slow, inconsistent between reviewers, and hard to repeat as models and regulations change.

The Lead AI Governance Agent automates the first-pass validation of a model package against these three frameworks. It takes a serialized classifier and its documentation, extracts what it needs, has a crew of specialized agents evaluate it, and produces a structured, machine-readable audit report.

## The Problem

A model package is more than a trained file. It includes the model object, its feature schema, hyperparameters, training configuration, and documentation. Regulators expect validators to check that each of these is sound and consistent with the others.

Three problems make this hard to do manually:

1. **Metadata is buried.** Feature names, types, and hyperparameters are locked inside serialized objects, and key context lives in PDFs.
2. **Frameworks overlap but don't align.** A single issue, such as reliance on one dominant feature, can be a model risk concern under SR 11-7, a transparency issue under the EU AI Act, and a "Measure" function gap under NIST AI RMF.
3. **Free-text reviews don't scale.** Narrative audit notes can't be aggregated, compared across models, or fed into governance dashboards.

## Architecture

The system has three layers: ingestion, a multi-agent evaluation crew, and schema-enforced reporting.

### 1. Metadata Ingestion Pipeline

Built in Python with PyPDF2 and pickle, this stage turns an opaque model package into structured facts the agents can reason over.

- **Serialized classifiers:** Loads pickled models and introspects them to extract the feature schema (names, count, ordering), hyperparameters (tree depth, learning rate, number of estimators, regularization), and model specifications such as algorithm family and objective.
- **Documentation PDFs:** PyPDF2 extracts text from accompanying documents so agents can compare what the documentation claims against what the model actually contains.

Extracting this programmatically means the agents work from ground truth rather than a reviewer's summary, and it exposes mismatches between documentation and artifact.

> **Security note:** Unpickling can execute arbitrary code. Only load trusted artifacts, in a controlled environment.

### 2. The Multi-Agent Auditing Crew

CrewAI orchestrates a team of agents, each with its own role, goal, and framing, mirroring a real model risk function:

| Agent | Responsibility |
|---|---|
| **Performance** | Assesses predictive quality and technical soundness, including whether hyperparameters and configuration are reasonable and whether evidence supports claimed performance. |
| **Risk** | Evaluates model risk in the SR 11-7 sense: conceptual soundness, input sensitivity, concentration of reliance, and potential for failure or misuse. |
| **Counsel** | Maps model characteristics to regulatory obligations, checking documentation, transparency, and governance requirements under the EU AI Act and NIST AI RMF. |
| **Gatekeeper** | Synthesizes the other agents' conclusions, resolves conflicts, and produces the overall assessment and recommendation. |

Each specialist analyzes the extracted metadata from its own perspective, and the Gatekeeper consolidates the results. Narrower mandates and clearer standards give broader coverage than a single "audit this" prompt.

### 3. Schema-Enforced Output with Pydantic

Instead of free-form prose, agent outputs are mapped onto a Pydantic model, `AgentReport`, which captures:

- the auditor's **score**
- a list of **findings**
- a list of **recommendations**

Every report is validated at creation. A non-conforming response fails fast instead of passing malformed content downstream.

Benefits:

- **Automated JSON export:** Reports serialize cleanly for storage, versioning, and consumption by other systems.
- **Consistency:** Every model is evaluated against the same structure, enabling comparison and trend analysis.
- **Auditability:** Structured findings are easier to trace, review, and challenge than narrative text.
- **Integration readiness:** Output can feed governance registries, ticketing systems, or dashboards without manual transcription.

## Case Study: XGBoost Credit Model

The system was tested on an XGBoost credit model, a high-stakes use case since credit decisioning is regulated and often classed as high-risk under the EU AI Act. The crew surfaced two significant issues.

**1. Over-reliance on the credit score.**
The model leaned heavily on the credit score feature. This is a concentration risk: a model dominated by one input can be fragile if that input shifts or becomes unavailable. It can also mask the influence of other variables and complicates explainability and fairness assessments. It is a conceptual-soundness and transparency concern under SR 11-7 and the EU AI Act, and a gap in the NIST AI RMF Measure function.

**2. Raw data parsing anomalies.**
The pipeline flagged irregularities in how raw data was parsed. Problems at this layer can quietly corrupt features before the model ever sees them, undermining data quality and any conclusions drawn from model performance. Catching them requires cross-checking the extracted schema against documentation and data handling.

Both findings arrived in the standardized `AgentReport` format with scores and concrete recommendations, ready for export.

## Impact and Takeaways

- **Role specialization improves coverage.** Distinct mandates surfaced different classes of issues than a single-pass review would.
- **Structure makes AI output usable in governance.** A Pydantic schema converts agent reasoning into auditable, exportable records.
- **Ingestion matters as much as reasoning.** Grounding agents in programmatically extracted metadata reduces reliance on assumptions and makes documentation-versus-artifact discrepancies visible.
- **Automation augments human reviewers.** It accelerates first-pass validation and standardizes findings, leaving human validators to focus on judgment and remediation.

## Tech Stack

Python, CrewAI, Gemini 2.5 Flash, Pydantic, PyPDF2, pickle, XGBoost
