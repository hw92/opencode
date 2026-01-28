

## For Your Financial Agents

Apply these patterns:

| OpenCode          | Your System                            |
| ----------------- | -------------------------------------- |
| Agent configs     | Analyst, Researcher, Trader roles      |
| Tool registry     | Market data, SEC filings, calculations |
| Permission system | Prevent unauthorized trades            |
| Execution loop    | Keep analyzing until complete          |
| Event bus         | Real-time updates to dashboard         |
| Storage           | Persist analysis results               |
| Snapshot          | Audit trail for decisions              |







## Lessons for Investment Agents

### 1. Define Multiple Specialized Agents

```typescript
const agents = [
  {
    name: "fundamental-analyst",
    instructions: "Analyze balance sheets, P/E ratios, DCF models",
    tools: ["SecEdgar", "FinancialCalculator"],
    model: "claude-opus-4-5"
  },
  {
    name: "technical-analyst",
    instructions: "Chart patterns, indicators, support/resistance",
    tools: ["MarketData", "ChartAnalysis"],
    model: "claude-sonnet-4-5"
  },
  {
    name: "risk-assessor",
    instructions: "Portfolio risk, correlation, VaR calculations",
    tools: ["PortfolioAnalyzer", "RiskMetrics"],
    model: "claude-sonnet-4-5"
  },
  {
    name: "news-monitor",
    instructions: "Real-time news impact, sentiment analysis",
    tools: ["NewsAPI", "SentimentAnalysis"],
    model: "claude-haiku-4-5"  // Fast for real-time
  }
]
```

### 2. Use Zod/Pydantic for Config Validation

```typescript
const InvestmentAgentSchema = z.object({
  name: z.string(),
  assetClass: z.enum(["equity", "bonds", "crypto", "commodities"]),
  riskTolerance: z.number().min(0).max(1),
  maxPositionSize: z.number().default(0.10),
  rebalanceFrequency: z.enum(["daily", "weekly", "monthly"]),
  allowedExchanges: z.array(z.string()),
  constraints: z.object({
    noShortSelling: z.boolean().default(true),
    maxLeverage: z.number().default(1.0),
    requiredApproval: z.boolean().default(true)
  })
})
```

### 3. Layer Safety Constraints

```typescript
// Base constraints for ALL investment agents
const baseConstraints = {
  permissions: [
    "deny:trade-execution",  // Never auto-trade
    "deny:portfolio-deletion",
    "allow:read-market-data",
    "allow:generate-reports",
    "ask:large-transactions"  // Requires approval
  ]
}

// Agent-specific additions
const equityAgent = {
  ...baseConstraints,
  permissions: [
    ...baseConstraints.permissions,
    "allow:sec-filings",
    "allow:earnings-calls"
  ]
}
```

### 4. Multi-Phase Workflow

```
Phase 1: Analysis (read-only)
    ├─ fundamental-analyst → P/E, DCF, balance sheet
    ├─ technical-analyst → Chart patterns, momentum
    └─ news-monitor → Recent sentiment

Phase 2: Synthesis
    └─ portfolio-manager → Combines all analyses

Phase 3: User Review
    └─ Present findings, recommendations

Phase 4: Execution (if approved)
    └─ trade-executor → Execute orders
```

### 5. Agent Hierarchy

```
Primary Agent:
  portfolio-manager (has full context, makes decisions)
      │
      ├─ Subagents (specialists):
      │   ├─ fundamental-analyst
      │   ├─ technical-analyst
      │   ├─ risk-assessor
      │   └─ news-monitor
      │
      └─ Utility Agents (hidden):
          ├─ report-generator
          ├─ alert-monitor
          └─ compliance-checker
```

## 

/Users/hai/learn_docs/26-01-22-16-agents-vs-llms-concept.md



