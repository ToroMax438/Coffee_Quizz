# Analytics Dashboard Specification
## Basecamp Rewards - Coffee Personality Quiz

---

## Overview

**Purpose:** Track quiz performance and validate whether personality types predict actual customer behavior.

**Primary Users:**
- Marketing team (daily)
- Leadership/CFO (weekly summary)
- Store managers (type distribution for their location)

**Update Frequency:** Real-time for quiz metrics, daily for purchase behavior

---

## Data Sources Required

| Source | Data Needed | Integration |
|--------|-------------|-------------|
| Quiz Application | Completions, answers, results, timestamps | API / event tracking |
| Loyalty Database | Member profiles, personality type assignment | Database join on member_id |
| POS System | Transactions, items purchased, timestamps | Nightly batch or real-time |
| NPS Survey Tool | Survey responses, scores, timestamps | API or manual upload |

---

## Metrics Definitions

### 1. Header KPIs

#### Quiz Completions
- **Definition:** Number of members who reached a result screen
- **Calculation:** `COUNT(quiz_sessions WHERE status = 'completed')`
- **Timeframe:** Rolling 7 days, with comparison to previous period
- **Target:** Week 1: 200 | Week 4: 500 | Week 12: 1,500

#### Monthly Active Users (MAU)
- **Definition:** Unique members with at least 1 transaction OR app open in past 30 days
- **Calculation:** `COUNT(DISTINCT member_id WHERE last_activity >= NOW() - 30 days)`
- **Baseline:** 480
- **Target:** 1,200 (90-day), 2,000+ (long-term)

#### 30-Day Retention
- **Definition:** % of members who return within 30 days of signup/quiz completion
- **Calculation:** `(Members with 2+ visits in 30 days) / (Total members in cohort) * 100`
- **Baseline:** 27%
- **Target:** 45% (90-day), 50%+ (long-term)

#### Program NPS
- **Definition:** Net Promoter Score for the loyalty program specifically
- **Calculation:** `% Promoters (9-10) - % Detractors (0-6)`
- **Collection:** Post-visit survey, sample of 100+ members/week
- **Baseline:** 12
- **Target:** 30 (90-day), 40+ (long-term)

---

### 2. Quiz Funnel Metrics

| Metric | Definition | Calculation |
|--------|------------|-------------|
| Started | Member opened quiz | `COUNT(quiz_sessions)` |
| Completed | Member reached result | `COUNT(quiz_sessions WHERE status = 'completed')` |
| Completion Rate | % who finish | `Completed / Started * 100` |
| Saved to Profile | Clicked "Save to Profile" | `COUNT(events WHERE action = 'save_profile')` |
| Ordered Rec'd Drink | Purchased their recommended drink within 7 days | `COUNT(members WHERE purchased_recommended_drink = true AND days_since_quiz <= 7)` |

**Target Completion Rate:** 80%+

---

### 3. Type Distribution

| Metric | Definition |
|--------|------------|
| Type Count | Number of members assigned to each personality type |
| Type Percentage | `Type Count / Total Completions * 100` |

**Stored as:** `member_personality_type` field in loyalty database

**Expected Distribution (from barista input):**
- Creature of Habit: 25-30%
- Smooth Operator: 20-25%
- Bold Explorer: 15-20%
- Social Sipper: 10-15%
- The Purist: 10-15%
- Functional Fueler: 8-12%

**Alert:** Flag if any type exceeds 40% or falls below 5% (may indicate quiz imbalance)

---

### 4. Retention by Type

| Metric | Definition | Calculation |
|--------|------------|-------------|
| Type Retention Rate | 30-day retention for each personality type | `(Type members with 2+ visits) / (Total type members) * 100` |
| Retention vs Baseline | Difference from 27% baseline | `Type Retention - 27` |

**Segmentation:** Calculate separately for each of 6 types + overall

**Alert Threshold:** Flag any type with retention below 30%

---

### 5. Purchase Behavior by Type

| Metric | Definition | Calculation |
|--------|------------|-------------|
| Avg Visits (30d) | Average store visits per member | `SUM(visits) / COUNT(members)` by type |
| Avg Spend (30d) | Average total spend per member | `SUM(transaction_total) / COUNT(members)` by type |
| Recommended Drink Adoption | % who ordered their recommended drink at least once | `Members who ordered rec drink / Total members` by type |

**Comparison Group:** Always include non-quiz members as baseline

---

### 6. Weekly Trends

**Tracked weekly:**
- Quiz completions (total)
- Retention rate (rolling 30-day)
- MAU
- NPS (if survey volume allows)

**Visualization:** Line chart, 12-week rolling window

---

### 7. Type Validation (Behavioral Match)

**Purpose:** Verify quiz results predict actual purchasing behavior

| Type | Expected Behavior | How to Measure |
|------|-------------------|----------------|
| Creature of Habit | Orders same drink repeatedly | `% of orders that match their top item` |
| Bold Explorer | Tries new/seasonal items | `COUNT(distinct items ordered) / visits` |
| Smooth Operator | Orders sweet/creamy drinks | `% orders with milk + sweetener/flavor` |
| The Purist | Orders black coffee/minimal drinks | `% orders that are drip/americano/cortado` |
| Social Sipper | Visits during social hours, with others | `% visits on weekends OR after 2pm` |
| Functional Fueler | Morning visits, high-caffeine items | `% visits before 10am OR extra shots` |

**Match Rate:** % of members whose behavior matches expected pattern

**Target:** 70%+ overall match rate (validates quiz accuracy)

**Alert:** If match rate falls below 60%, quiz questions may need recalibration

---

## Alert Thresholds

| Metric | Alert If | Action |
|--------|----------|--------|
| Completion Rate | < 70% | Review quiz length/UX |
| Any Type Distribution | > 40% or < 5% | Review question balance |
| Type Retention | < 30% | Investigate that type's experience |
| Validation Match Rate | < 60% | Recalibrate quiz or type definitions |
| Weekly Completions | Down 25% WoW | Check for technical issues |

---

## Reporting Schedule

| Report | Audience | Frequency | Delivery |
|--------|----------|-----------|----------|
| Daily Snapshot | Marketing | Daily | Dashboard / Slack |
| Weekly Summary | Leadership | Friday | Email PDF |
| Monthly Deep Dive | All stakeholders | Monthly | Meeting + deck |

**Weekly Summary includes:**
- 4 header KPIs with WoW change
- Quiz funnel conversion
- Top/bottom performing types
- One insight or recommendation

---

## Technical Requirements

### Data Schema

```
quiz_sessions
- session_id (PK)
- member_id (FK)
- started_at (timestamp)
- completed_at (timestamp)
- status (started | completed | abandoned)
- result_type (habit | explorer | smooth | purist | social | fueler)
- q1_answer, q2_answer, ... q6_answer

member_profiles
- member_id (PK)
- personality_type
- quiz_completed_at
- recommended_drink
- signup_date

transactions
- transaction_id (PK)
- member_id (FK)
- store_id
- timestamp
- items (array)
- total_amount
```

### Access Control

| Role | Access Level |
|------|--------------|
| Marketing | Full dashboard |
| Store Manager | Their location's data only |
| Leadership | Summary view + export |
| Data Team | Full + raw data |

---

## Implementation Phases

**Phase 1 (Launch):**
- Header KPIs
- Quiz funnel
- Type distribution
- Manual weekly export

**Phase 2 (Week 2-4):**
- Retention by type
- Purchase behavior by type
- Automated daily refresh

**Phase 3 (Week 4-8):**
- Validation metrics
- Trend visualizations
- Alerts and notifications

---

## Success Criteria

Dashboard is successful if:
1. Leadership reviews weekly summary every Friday
2. Marketing uses it to make at least 1 decision per week
3. We can answer "is the quiz working?" with data, not opinions

---

*Last updated: January 2026*
