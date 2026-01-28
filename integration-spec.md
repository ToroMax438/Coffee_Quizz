# Coffee Personality Quiz - Integration Spec
## Technical Handoff Document

---

## Overview

**What:** Integrate the Coffee Personality Quiz into the Basecamp loyalty app.

**Why:** Give members a coffee identity to increase engagement and retention.

**Prototype:** https://toromax438.github.io/Coffee_Quizz/coffee-quiz.html

**Target:** MVP live in 3 weeks, full rollout in 4 weeks.

---

## Architecture

### MVP Approach: Webview + Native Storage

```
┌─────────────────────────────────────────────────────────────┐
│                      MOBILE APP                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    WEBVIEW                           │   │
│  │         (loads quiz HTML/CSS/JS)                     │   │
│  │                                                      │   │
│  │   Quiz completes → JS Bridge → Native callback       │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              NATIVE LAYER                            │   │
│  │   - Capture result from JS bridge                    │   │
│  │   - Call API to save result                          │   │
│  │   - Update local profile cache                       │   │
│  │   - Fire analytics event                             │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      BACKEND                                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  Quiz API    │───▶│  Member DB   │◀───│  POS Lookup  │  │
│  │  /quiz/save  │    │  (profile)   │    │  (optional)  │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│          │                                                  │
│          ▼                                                  │
│  ┌──────────────┐                                          │
│  │  Analytics   │                                          │
│  │  (events)    │                                          │
│  └──────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

### Why Webview for MVP

| Factor | Webview | Native |
|--------|---------|--------|
| Dev time | 2-3 weeks | 6-8 weeks |
| Iteration speed | Update without app release | Requires app update |
| UX quality | Good enough | Best |
| Offline support | Limited | Full |

**Decision:** Ship webview for MVP. Migrate to native post-validation if engagement is strong.

---

## Data Flow

### Quiz Completion Flow

```
1. User opens quiz (webview loads)
2. User completes 6 questions
3. JS calculates result type
4. JS calls native bridge with result payload
5. Native app calls POST /api/quiz/save
6. Backend saves to member profile
7. Backend fires analytics event
8. Native app updates local cache
9. Native app shows confirmation + result
```

### API Endpoint

**POST /api/quiz/save**

Request:
```json
{
  "member_id": "abc123",
  "personality_type": "bold_explorer",
  "answers": {
    "q1": "explorer",
    "q2": "purist",
    "q3": "smooth",
    "q4": "explorer",
    "q5": "explorer",
    "q6": "explorer"
  },
  "completed_at": "2026-01-28T10:30:00Z",
  "quiz_version": "1.0"
}
```

Response:
```json
{
  "success": true,
  "personality": {
    "type": "bold_explorer",
    "name": "Bold Explorer",
    "tagline": "Surprise me is my love language",
    "description": "You ask \"what's new?\" before the barista says hello. Predictable? Never. You trust us to surprise you.",
    "recommended_drink": "Ethiopian Single Origin Pour Over",
    "icon": "compass"
  }
}
```

Error Response:
```json
{
  "success": false,
  "error": "ALREADY_COMPLETED",
  "message": "Quiz can only be retaken after 90 days",
  "next_retake_date": "2026-04-28"
}
```

---

## Database Schema

### Member Profile (existing table, add fields)

```sql
ALTER TABLE members ADD COLUMN coffee_personality VARCHAR(20);
ALTER TABLE members ADD COLUMN personality_updated_at TIMESTAMP;
ALTER TABLE members ADD COLUMN quiz_version VARCHAR(10);
```

### Quiz Results (new table, for history)

```sql
CREATE TABLE quiz_results (
  id UUID PRIMARY KEY,
  member_id UUID REFERENCES members(id),
  personality_type VARCHAR(20) NOT NULL,
  answers JSONB NOT NULL,
  quiz_version VARCHAR(10) NOT NULL,
  completed_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_quiz_results_member ON quiz_results(member_id);
CREATE INDEX idx_quiz_results_type ON quiz_results(personality_type);
```

### Personality Types (reference table)

```sql
CREATE TABLE personality_types (
  type_key VARCHAR(20) PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  tagline VARCHAR(100) NOT NULL,
  description TEXT NOT NULL,
  recommended_drink VARCHAR(100) NOT NULL,
  icon VARCHAR(20) NOT NULL,
  color_hex VARCHAR(7) NOT NULL
);

-- Seed data
INSERT INTO personality_types VALUES
  ('habit', 'The Regular', 'I know what I love', 'You found your drink. The baristas start making it when you walk in. That''s not boring—that''s confidence.', 'Classic House Drip', 'home', '#D4A574'),
  ('explorer', 'Bold Explorer', 'Surprise me is my love language', 'You ask "what''s new?" before the barista says hello. Predictable? Never. You trust us to surprise you.', 'Ethiopian Single Origin Pour Over', 'compass', '#2D5A3D'),
  ('smooth', 'Smooth Operator', 'Sweet, creamy, unapologetic', 'Creamy, sweet, unapologetic. You want coffee that feels like a treat. You know what you like, and you''re not sorry about it.', 'Oat Milk Vanilla Latte', 'palette', '#C4956A'),
  ('purist', 'The Purist', 'Just the coffee, please', 'You want to taste the coffee, not the syrup. Black or a cortado, keep it simple. For you, less is more.', 'Cortado', 'coffee', '#4A3728'),
  ('social', 'Social Sipper', 'I''m here for the vibes', 'You''re here for the vibe, not the caffeine. The corner table, the familiar faces, the conversation. Coffee''s good—company''s better.', 'Shareable Cold Brew Pitcher', 'hand-wave', '#E8846B'),
  ('fueler', 'Functional Fueler', 'Coffee is my productivity hack', 'Coffee is a tool and you respect it. Large, dark, let''s go. You''ve got things to do.', 'Red Eye', 'lightning', '#5B7B8C');
```

---

## Analytics Events

### Event: quiz_started

```json
{
  "event": "quiz_started",
  "member_id": "abc123",
  "timestamp": "2026-01-28T10:28:00Z",
  "properties": {
    "entry_point": "home_banner",
    "quiz_version": "1.0"
  }
}
```

### Event: quiz_question_answered

```json
{
  "event": "quiz_question_answered",
  "member_id": "abc123",
  "timestamp": "2026-01-28T10:28:30Z",
  "properties": {
    "question_number": 3,
    "answer_type": "smooth",
    "time_on_question_ms": 4200
  }
}
```

### Event: quiz_abandoned

```json
{
  "event": "quiz_abandoned",
  "member_id": "abc123",
  "timestamp": "2026-01-28T10:29:00Z",
  "properties": {
    "last_question_seen": 3,
    "time_in_quiz_ms": 45000
  }
}
```

### Event: quiz_completed

```json
{
  "event": "quiz_completed",
  "member_id": "abc123",
  "timestamp": "2026-01-28T10:30:00Z",
  "properties": {
    "personality_type": "bold_explorer",
    "time_to_complete_ms": 72000,
    "quiz_version": "1.0"
  }
}
```

### Event: quiz_result_shared

```json
{
  "event": "quiz_result_shared",
  "member_id": "abc123",
  "timestamp": "2026-01-28T10:31:00Z",
  "properties": {
    "personality_type": "bold_explorer",
    "share_method": "native_share"
  }
}
```

---

## JavaScript Bridge

### Webview → Native Communication

The quiz webview needs to communicate results to the native app.

**iOS (WKWebView):**
```javascript
// In quiz HTML, when result is calculated:
window.webkit.messageHandlers.quizComplete.postMessage({
  personalityType: 'bold_explorer',
  answers: ['explorer', 'purist', 'smooth', 'explorer', 'explorer', 'explorer']
});
```

**Android (WebView):**
```javascript
// In quiz HTML, when result is calculated:
Android.onQuizComplete(JSON.stringify({
  personalityType: 'bold_explorer',
  answers: ['explorer', 'purist', 'smooth', 'explorer', 'explorer', 'explorer']
}));
```

### Native Handlers

**iOS (Swift):**
```swift
func userContentController(_ userContentController: WKUserContentController,
                          didReceive message: WKScriptMessage) {
    if message.name == "quizComplete" {
        guard let body = message.body as? [String: Any],
              let type = body["personalityType"] as? String else { return }
        saveQuizResult(type: type, answers: body["answers"])
    }
}
```

**Android (Kotlin):**
```kotlin
@JavascriptInterface
fun onQuizComplete(resultJson: String) {
    val result = JSONObject(resultJson)
    val type = result.getString("personalityType")
    saveQuizResult(type, result.getJSONArray("answers"))
}
```

---

## Business Rules

### Retake Policy

| Rule | Value |
|------|-------|
| Can retake? | Yes |
| Cooldown period | 90 days |
| Keep history? | Yes (all results stored) |
| Which result shown? | Most recent |

### Versioning

| Scenario | Behavior |
|----------|----------|
| User took v1.0, now on v1.1 | Show v1.0 result, prompt to retake |
| Scoring logic changes | Bump quiz version, don't retroactively change results |
| New personality type added | Existing users keep old type until retake |

---

## Edge Cases

### Offline Handling

```
1. User completes quiz while offline
2. App stores result locally (SQLite/UserDefaults)
3. App queues API call
4. When online, sync result to backend
5. If sync fails 3x, show error + retry option
```

### Interrupted Quiz

```
1. User answers 3 questions, leaves app
2. On return within 24 hours: resume from Q4
3. On return after 24 hours: start fresh
4. Store progress: { question: 3, answers: [...], started_at: timestamp }
```

### Already Completed (Within 90 Days)

```
1. User opens quiz
2. Check last completion date
3. If < 90 days: show current result + countdown to retake
4. If >= 90 days: allow quiz
```

---

## MVP Scope

### Week 1: Core Integration

| Task | Owner | Notes |
|------|-------|-------|
| Set up webview container | Mobile | iOS + Android |
| Implement JS bridge | Mobile | Capture result |
| Create /api/quiz/save endpoint | Backend | Save to DB |
| Add personality fields to member table | Backend | Schema migration |
| Fire quiz_completed event | Backend | Basic analytics |

### Week 2: Profile + Polish

| Task | Owner | Notes |
|------|-------|-------|
| Show personality badge in profile | Mobile | Display saved type |
| Add retake logic (90-day check) | Backend | Enforce cooldown |
| Handle offline completion | Mobile | Queue + sync |
| Deep link: basecamp://quiz | Mobile | For marketing |
| QA pass | QA | All flows |

### Week 3: Soft Launch

| Task | Owner | Notes |
|------|-------|-------|
| Feature flag for 10% rollout | Backend | Gradual release |
| Monitor completion rates | Data | Dashboard check |
| Bug fixes | All | Fast response |
| Expand to 50%, then 100% | Backend | If metrics good |

---

## NOT in MVP (Post-Launch)

| Feature | Why Deferred |
|---------|--------------|
| POS barista visibility | Requires POS integration work |
| Personalized drink recommendations | Hardcoded per type for now |
| Social sharing with custom image | Native share is enough |
| A/B testing questions | Sample size too small |
| Push notifications for incomplete | Add after baseline data |

---

## Testing Checklist

### Functional

- [ ] Quiz loads in webview (iOS)
- [ ] Quiz loads in webview (Android)
- [ ] All 6 questions display correctly
- [ ] Result calculates correctly for each type
- [ ] Result saves to backend
- [ ] Result displays in profile
- [ ] Retake blocked within 90 days
- [ ] Retake allowed after 90 days
- [ ] Deep link opens quiz directly

### Edge Cases

- [ ] Offline completion queues correctly
- [ ] Interrupted quiz resumes
- [ ] Slow network doesn't cause timeout
- [ ] Already completed shows current result
- [ ] Invalid member_id returns error

### Analytics

- [ ] quiz_started fires on open
- [ ] quiz_question_answered fires each question
- [ ] quiz_completed fires on result
- [ ] quiz_abandoned fires on exit without completion

---

## Contacts

| Role | Name | For |
|------|------|-----|
| Product | [Name] | Requirements, prioritization |
| Mobile Lead | [Name] | iOS/Android questions |
| Backend Lead | [Name] | API, database |
| QA | [Name] | Testing, bugs |
| Analytics | [Name] | Event tracking |

---

## Resources

- **Prototype:** https://toromax438.github.io/Coffee_Quizz/coffee-quiz.html
- **Design spec:** See barista-training-guide.html for type details
- **Analytics spec:** See dashboard-spec.md for metrics definitions

---

*Last updated: January 2026*
