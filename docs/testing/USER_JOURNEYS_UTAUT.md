# PRISM-Matutum User Journeys - UTAUT Analysis

## 📋 Document Overview

This document maps out complete user journeys through the PRISM-Matutum system, analyzing each touchpoint through the UTAUT (Unified Theory of Acceptance and Use of Technology) framework. Each journey represents a real-world scenario from a user's perspective, with detailed analysis of acceptance factors.

---

## 🎯 Understanding User Journeys

A **user journey** is a complete scenario from a user's initial goal to final outcome, including:
- **Trigger**: What prompts the user to use the system
- **Steps**: All actions taken to complete the goal
- **Touchpoints**: Every interaction with the system
- **Pain Points**: Where users might struggle
- **UTAUT Analysis**: How well each step supports technology acceptance

---

## 👤 Journey 1: PA Staff - First Submission Check After Field Work

### User Profile
- **Name**: Maria Santos
- **Role**: PA Staff / Ranger
- **Context**: Has been conducting field surveys for 1 week, using KoboToolbox on tablet
- **Goal**: Check if her submissions have been reviewed
- **Tech Proficiency**: Moderate (comfortable with basic apps)

---

### Journey Map

```
┌─────────────────────────────────────────────────────────────┐
│  TRIGGER: End of field work week, back at ranger station    │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: Login to PRISM                                      │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Opens browser on ranger station computer                  │
│  • Navigates to PRISM URL (bookmarked)                      │
│  • Enters username and password                              │
│  • Clicks "Login"                                            │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis:                                             │
│  PE: ★★★★☆ Expects to see submission status quickly         │
│  EE: ★★★★★ Login is straightforward, no confusion          │
│  SI: ★★★★☆ Uses system because it's official/required      │
│  FC: ★★★★☆ Has computer access, internet available         │
├─────────────────────────────────────────────────────────────┤
│  Potential Pain Points:                                      │
│  ⚠️ Forgot password (hasn't logged in for a week)          │
│  ⚠️ Slow internet at station                                │
│  ⚠️ Account suspended if not approved                        │
├─────────────────────────────────────────────────────────────┤
│  Success Indicators:                                         │
│  ✅ Login completes in <3 seconds                           │
│  ✅ "Forgot password" link is visible                       │
│  ✅ Error messages are helpful                              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 2: View PA Staff Dashboard                             │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Automatically lands on PA Staff home dashboard            │
│  • Scans statistics at a glance                              │
│  • Looks for validation status summary                       │
├─────────────────────────────────────────────────────────────┤
│  What Maria Sees:                                            │
│  ┌──────────────────────────────────────────┐               │
│  │  📊 My Statistics                         │               │
│  │  • 12 Total Submissions                   │               │
│  │  • 8 Approved ✅                          │               │
│  │  • 2 Pending ⏳                           │               │
│  │  • 2 Rejected ❌                          │               │
│  └──────────────────────────────────────────┘               │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis:                                             │
│  PE: ★★★★★ Immediate feedback on work quality               │
│           Approval rate visible (66% approved)               │
│           Knows exactly what needs attention                 │
│  EE: ★★★★★ Information is clear at first glance            │
│           Visual indicators (✅❌⏳) aid understanding       │
│           No technical jargon                                │
│  SI: ★★★★☆ Stats show her contribution to PA work          │
│           Professional presentation builds trust             │
│  FC: ★★★★☆ Dashboard always available                      │
│           No complex setup needed                            │
├─────────────────────────────────────────────────────────────┤
│  Emotional Response:                                         │
│  😊 Positive: 8 approved - work is valued                   │
│  😟 Concerned: 2 rejected - what went wrong?                │
│  😐 Neutral: 2 pending - waiting for review                 │
├─────────────────────────────────────────────────────────────┤
│  Potential Pain Points:                                      │
│  ⚠️ High rejection rate could be discouraging               │
│  ⚠️ No explanation visible on dashboard                     │
├─────────────────────────────────────────────────────────────┤
│  Success Indicators:                                         │
│  ✅ Statistics are accurate                                 │
│  ✅ Dashboard loads in <2 seconds                           │
│  ✅ Call-to-action for rejected items is clear              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 3: Navigate to "My Submissions"                        │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Clicks "My Submissions" in navigation menu                │
│  • Page loads with list of all submissions                   │
│  • Scans list for rejected items (red badges stand out)     │
├─────────────────────────────────────────────────────────────┤
│  What Maria Sees:                                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🔍 Filter: [Status: Rejected ▼] [Search...]          │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ Date       | Location    | Species          | Status │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │ 2026-01-02 | Transect A  | Philippine Eagle | ❌ R   │   │
│  │ 2026-01-03 | Viewpoint B | Cloud Rat        | ❌ R   │   │
│  │ 2026-01-04 | Trail C     | Tarsier          | ✅ A   │   │
│  │ 2026-01-05 | Transect A  | Hornbill         | ⏳ P   │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis:                                             │
│  PE: ★★★★★ Can quickly identify what needs attention        │
│           Filters help focus on specific statuses            │
│           Saves time vs. manual tracking                     │
│  EE: ★★★★★ List is scannable and organized                 │
│           Status colors make identification instant          │
│           Familiar table format                              │
│  SI: ★★★★☆ Clean interface builds confidence in system     │
│           Professional data presentation                     │
│  FC: ★★★★★ All data is accessible, no technical barriers   │
│           Pagination handles large datasets                  │
├─────────────────────────────────────────────────────────────┤
│  Decision Point:                                             │
│  Maria decides to check the first rejection to understand   │
│  what went wrong                                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 4: View Rejected Submission Detail                     │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Clicks on rejected submission (Jan 2 - Philippine Eagle)  │
│  • Detail page loads                                         │
│  • Scrolls to validation remarks section                     │
├─────────────────────────────────────────────────────────────┤
│  What Maria Sees:                                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 📋 Submission Details                                 │   │
│  │                                                        │   │
│  │ Species: Philippine Eagle                             │   │
│  │ Date: January 2, 2026                                 │   │
│  │ Location: Transect A                                  │   │
│  │ Photos: [eagle_photo_1.jpg]                           │   │
│  │                                                        │   │
│  │ ──────────────────────────────────────────────────   │   │
│  │                                                        │   │
│  │ ✅ VALIDATION STATUS: Rejected                        │   │
│  │ Reviewed by: Dr. Juan Cruz (Validator)                │   │
│  │ Date: January 3, 2026                                 │   │
│  │                                                        │   │
│  │ 💬 Validator Remarks:                                 │   │
│  │ "The photo provided does not show clear diagnostic    │   │
│  │  features to confirm Philippine Eagle identification. │   │
│  │  The bird appears to be a juvenile, but the angle    │   │
│  │  does not show the distinctive crest or bill shape.  │   │
│  │                                                        │   │
│  │  Please refer to the Field Guide (page 42) for       │   │
│  │  photo requirements. Retake photos showing:           │   │
│  │  1. Full body view (side profile)                    │   │
│  │  2. Head with visible crest                          │   │
│  │  3. Clear view of talons if possible                 │   │
│  │                                                        │   │
│  │  If you're certain of the ID, please add             │   │
│  │  additional context in observation notes (e.g.,      │   │
│  │  behavioral observations, call heard, etc.)"          │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis:                                             │
│  PE: ★★★★★ Detailed feedback helps improve future work     │
│           Specific guidance on what to do differently        │
│           Educational value - learning from mistakes         │
│           Reference to field guide (page 42) is actionable   │
│  EE: ★★★★★ Feedback is clear and non-technical             │
│           Constructive tone, not punitive                    │
│           Numbered list makes requirements clear             │
│           No confusing jargon or acronyms                    │
│  SI: ★★★★★ Validator name shown (Dr. Juan Cruz)            │
│           Professional, respectful communication             │
│           Feedback acknowledges Maria's expertise            │
│           ("If you're certain of ID...")                     │
│           Builds collaborative relationship                  │
│  FC: ★★★★★ Provides resources (Field Guide page 42)        │
│           Guidance is immediately actionable                 │
│           System facilitates learning                        │
├─────────────────────────────────────────────────────────────┤
│  Emotional Response:                                         │
│  😌 Relief: Rejection is not about her competence           │
│  📚 Learning: Understands photo requirements better         │
│  💪 Motivated: Knows exactly how to improve next time       │
│  🤝 Respected: Validator's tone is collaborative            │
├─────────────────────────────────────────────────────────────┤
│  Key Success Factors:                                        │
│  ✅ Feedback is specific, not generic ("photo unclear")    │
│  ✅ Actionable guidance provided                            │
│  ✅ Reference materials cited                               │
│  ✅ Collaborative tone maintained                           │
│  ✅ Maria feels supported, not criticized                   │
├─────────────────────────────────────────────────────────────┤
│  Potential Pain Points (If Poorly Implemented):             │
│  ❌ Generic feedback: "Data quality insufficient"          │
│  ❌ No guidance on how to improve                           │
│  ❌ Harsh or condescending tone                             │
│  ❌ Technical validation error messages                     │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 5: Note to Self & Plan Next Steps                      │
├─────────────────────────────────────────────────────────────┤
│  Maria's Internal Dialogue:                                  │
│  "Okay, I need to take better photos. I'll check page 42    │
│   of the field guide before my next survey. For the second  │
│   rejection, let me see what went wrong there too..."       │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Clicks "Back to Submissions"                              │
│  • Opens second rejected submission                          │
│  • Reads validator feedback                                  │
│  • Mentally prepares for next field work                     │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis of Journey Completion:                       │
│  PE: ★★★★★ Journey achieved its goal completely            │
│           Maria now knows status of all submissions          │
│           Understands what needs improvement                 │
│           Feels more prepared for next survey                │
│  EE: ★★★★★ Journey was smooth with no confusion            │
│           Found information in <5 minutes                    │
│           No help documentation needed                       │
│           Navigation was intuitive throughout                │
│  SI: ★★★★★ Feels like valued team member                   │
│           Validation process feels fair and supportive       │
│           Motivated to improve data quality                  │
│           Trusts the system and validators                   │
│  FC: ★★★★★ System fully supported her needs                │
│           No technical issues encountered                    │
│           All information was accessible                     │
│           Actionable guidance provided                       │
├─────────────────────────────────────────────────────────────┤
│  Behavioral Intention (BI):                                  │
│  ★★★★★ "I will definitely use this system again"           │
│  ★★★★★ "I will check my submissions regularly"             │
│  ★★★★★ "I will recommend this to other rangers"            │
│  ★★★★★ "The feedback helps me do better work"              │
├─────────────────────────────────────────────────────────────┤
│  Journey Outcome: ✅ HIGHLY SUCCESSFUL                       │
│  • Goal achieved efficiently                                 │
│  • User feels supported and motivated                        │
│  • Learning occurred (improved skills)                       │
│  • Trust in system increased                                 │
│  • Likelihood of continued use: VERY HIGH                    │
└─────────────────────────────────────────────────────────────┘
```

---

### Journey 1: UTAUT Score Summary

| Construct | Rating | Key Drivers |
|-----------|--------|-------------|
| **Performance Expectancy** | 9.5/10 | • Fast access to submission status<br>• Detailed, actionable feedback<br>• Clear improvement path<br>• Saves time vs. manual tracking |
| **Effort Expectancy** | 9.8/10 | • Intuitive navigation<br>• No training needed<br>• Visual clarity (colors, icons)<br>• Plain language throughout |
| **Social Influence** | 9.3/10 | • Professional system builds legitimacy<br>• Respectful validator communication<br>• Collaborative tone fosters teamwork<br>• Clear roles and accountability |
| **Facilitating Conditions** | 9.5/10 | • System always available<br>• No technical barriers<br>• Resources provided (Field Guide)<br>• Error-free experience |
| **Behavioral Intention** | 9.7/10 | • Will continue using system<br>• Will check regularly<br>• Will recommend to peers<br>• Perceives high value |

**Overall Journey Success**: 9.6/10 - **Excellent**

---

## ✅ Journey 2: Validator - Morning Validation Session

### User Profile
- **Name**: Dr. Juan Cruz
- **Role**: Validator / Biodiversity Specialist
- **Context**: Starting his workday, has 1 hour dedicated to validation
- **Goal**: Review and process as many pending submissions as possible
- **Tech Proficiency**: High (uses GIS tools regularly)

---

### Journey Map

```
┌─────────────────────────────────────────────────────────────┐
│  TRIGGER: 8:00 AM - Start of workday, checking queue         │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: Login & View Validator Dashboard                    │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Opens PRISM on desktop computer                           │
│  • Logs in (credentials saved in browser)                    │
│  • Lands on Validator Dashboard                              │
├─────────────────────────────────────────────────────────────┤
│  What Dr. Cruz Sees:                                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 📊 Validation Overview                                │   │
│  │                                                        │   │
│  │ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │   │
│  │ │ 47 Pending  │ │ 12 Assigned │ │ 8 Reviewed  │     │   │
│  │ │ Submissions │ │ to Me       │ │ Today       │     │   │
│  │ └─────────────┘ └─────────────┘ └─────────────┘     │   │
│  │                                                        │   │
│  │ 📈 My Performance This Week:                          │   │
│  │ • Approval Rate: 78%                                  │   │
│  │ • Avg. Time per Validation: 3.2 minutes              │   │
│  │ • Total Validated: 52 submissions                     │   │
│  │                                                        │   │
│  │ [View Queue →]  [My Validation History →]            │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis:                                             │
│  PE: ★★★★★ Clear overview of workload                      │
│           Performance metrics show productivity              │
│           "12 assigned to me" - knows where to start         │
│           Efficiency metrics help self-improvement           │
│  EE: ★★★★★ Dashboard is professional and organized         │
│           One-click access to queue                          │
│           Metrics are auto-calculated                        │
│  SI: ★★★★★ Performance transparency fosters accountability  │
│           Metrics show contribution to PA goals              │
│           Professional role identity reinforced              │
│  FC: ★★★★★ All tools immediately accessible                │
│           No setup required                                  │
├─────────────────────────────────────────────────────────────┤
│  Emotional Response:                                         │
│  💪 Motivated: Clear workload, ready to tackle it           │
│  📊 Informed: Knows exactly what to expect                  │
│  ⏰ Time-conscious: Has 1 hour, sees 12 assigned items      │
│       (12 ÷ 3.2 min avg = ~38 min if typical - doable!)    │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 2: Access Validation Queue                             │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Clicks "View Queue" button                                │
│  • Queue page loads with filters                             │
│  • Filters to "Assigned to Me" to focus on his items         │
├─────────────────────────────────────────────────────────────┤
│  What Dr. Cruz Sees:                                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🔍 Validation Queue                                   │   │
│  │                                                        │   │
│  │ Filters: [Assigned to Me ▼] [Sort: Oldest First ▼]   │   │
│  │                                                        │   │
│  │ Showing 12 submissions                                │   │
│  │                                                        │   │
│  │ ┌─────────┬───────────┬────────────┬──────┬────────┐ │   │
│  │ │ Prio    │ PA Staff  │ Species    │ Date │ Action │ │   │
│  │ ├─────────┼───────────┼────────────┼──────┼────────┤ │   │
│  │ │ 🔴 High │ M. Santos │ Phil Eagle │ 1/2  │ [View] │ │   │
│  │ │ 🟡 Med  │ R. Garcia │ Tarsier    │ 1/3  │ [View] │ │   │
│  │ │ 🟡 Med  │ M. Santos │ Cloud Rat  │ 1/3  │ [View] │ │   │
│  │ │ 🟢 Low  │ A. Reyes  │ Hornbill   │ 1/4  │ [View] │ │   │
│  │ │ ...     │ ...       │ ...        │ ...  │ ...    │ │   │
│  │ └─────────┴───────────┴────────────┴──────┴────────┘ │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis:                                             │
│  PE: ★★★★★ Queue helps prioritize effectively              │
│           Priority indicators (🔴🟡🟢) guide decisions      │
│           Filtering saves time (no irrelevant items)         │
│           Batch view allows efficient planning               │
│  EE: ★★★★★ Queue is scannable and well-organized           │
│           Familiar table format                              │
│           Sorting options are intuitive                      │
│           One-click access to each submission                │
│  SI: ★★★☆☆ See who submitted (accountability)              │
│           Workload distribution is transparent               │
│  FC: ★★★★★ Queue always available                          │
│           No technical issues                                │
│           Handles large datasets gracefully                  │
├─────────────────────────────────────────────────────────────┤
│  Decision: Start with highest priority (Philippine Eagle)   │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 3: Review High-Priority Submission (Philippine Eagle)  │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Clicks "View" on Philippine Eagle submission              │
│  • Validation workspace loads                                │
│  • Reviews photo                                             │
│  • Checks observation details                                │
│  • Considers identification                                  │
├─────────────────────────────────────────────────────────────┤
│  What Dr. Cruz Sees:                                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 📋 Validation Workspace                               │   │
│  │                                                        │   │
│  │ Submission #2024-001-012                              │   │
│  │ Submitted by: Maria Santos (PA Staff)                 │   │
│  │ Date: January 2, 2026                                 │   │
│  │                                                        │   │
│  │ ┌─────────────────────────────────────────────────┐  │   │
│  │ │ Observation Details                              │  │   │
│  │ │                                                   │  │   │
│  │ │ Species Reported: Philippine Eagle               │  │   │
│  │ │ Location: Transect A (7.5°N, 125.1°E)           │  │   │
│  │ │ Date/Time: Jan 2, 2026, 09:30 AM                │  │   │
│  │ │ Behavior: Soaring above canopy                   │  │   │
│  │ │ Notes: Observed for ~5 minutes                   │  │   │
│  │ └─────────────────────────────────────────────────┘  │   │
│  │                                                        │   │
│  │ 📷 Photo Evidence:                                    │   │
│  │ ┌──────────────────┐                                 │   │
│  │ │  [Photo shows    │                                 │   │
│  │ │   distant bird   │  ← Dr. Cruz clicks to enlarge  │   │
│  │ │   in flight]     │                                 │   │
│  │ └──────────────────┘                                 │   │
│  │                                                        │   │
│  │ 🗺️ Map View:                                          │   │
│  │ [Interactive map showing location]                    │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  Dr. Cruz's Expert Analysis:                                 │
│  🤔 "Let me examine the photo carefully..."                 │
│  🔍 [Zooms in on photo]                                     │
│  🤔 "The bird is quite distant. I can see it's a large      │
│      raptor, but the diagnostic features aren't clear."     │
│  🤔 "No visible crest, can't confirm bill shape from        │
│      this angle. Could be juvenile, but can't be certain."  │
│  📚 [Mentally references field guide knowledge]            │
│  ⚖️ Decision: "I need to reject this and ask for better    │
│      photos, but I'll provide detailed guidance."           │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis:                                             │
│  PE: ★★★★★ Workspace provides all info needed for decision │
│           Photo viewer with zoom is essential                │
│           Location data helps verify likelihood              │
│           Behavioral notes add context                       │
│           Time estimate: 2 minutes for this submission       │
│  EE: ★★★★★ Layout is logical (data → photo → map → action)│
│           Photo zoom works smoothly                          │
│           All info visible without excessive scrolling       │
│           No confusion about what to review                  │
│  SI: ★★★★☆ Submitter info shown (Maria Santos)             │
│           Feels responsibility to provide good feedback      │
│           Professional interface matches expert role         │
│  FC: ★★★★★ System supports expert judgment                 │
│           Tools work reliably (photo viewer, map)            │
│           Can reference materials if needed                  │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 4: Reject with Detailed Feedback                       │
├─────────────────────────────────────────────────────────────┤
│  Actions:                                                    │
│  • Scrolls to validation action section                      │
│  • Selects "Reject" option                                   │
│  • Types detailed remarks (shown in Journey 1)               │
│  • Clicks "Submit Validation"                                │
├─────────────────────────────────────────────────────────────┤
│  What Dr. Cruz Sees (Action Panel):                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 🎯 Validation Action                                  │   │
│  │                                                        │   │
│  │ Decision: ○ Approve  ⦿ Reject  ○ Hold for Review    │   │
│  │                                                        │   │
│  │ 💬 Remarks: (Required for rejection)                 │   │
│  │ ┌────────────────────────────────────────────────┐   │   │
│  │ │ The photo provided does not show clear         │   │   │
│  │ │ diagnostic features to confirm Philippine      │   │   │
│  │ │ Eagle identification...                         │   │   │
│  │ │ [Dr. Cruz types constructive feedback]         │   │   │
│  │ └────────────────────────────────────────────────┘   │   │
│  │                                                        │   │
│  │ [Cancel]  [Submit Validation]                         │   │
│  └──────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis:                                             │
│  PE: ★★★★★ Rejection process is as easy as approval        │
│           Remarks field encourages quality feedback          │
│           One-click submit                                   │
│  EE: ★★★★★ Interface is clear and simple                   │
│           Radio buttons make decision unambiguous            │
│           Remarks field is appropriately sized               │
│           No confusing options                               │
│  SI: ★★★★★ Remarks will help PA Staff improve              │
│           Dr. Cruz feels he's contributing to training       │
│           Feedback builds collaborative relationship         │
│  FC: ★★★★★ System requires remarks (quality control)       │
│           Feedback is stored for future reference            │
│           Undo option available (if needed)                  │
├─────────────────────────────────────────────────────────────┤
│  Emotional Response:                                         │
│  😌 Satisfied: Provided helpful feedback                    │
│  🤝 Supportive: Helping Maria improve                       │
│  ⏱️ Efficient: Took only 2 minutes                         │
├─────────────────────────────────────────────────────────────┤
│  System Response:                                            │
│  ✅ "Validation submitted successfully"                     │
│  ↩️ Redirects back to queue (or next submission)           │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 5: Continue Validation (Batch Processing)              │
├─────────────────────────────────────────────────────────────┤
│  Next 45 Minutes:                                            │
│                                                              │
│  Submission 2 (Tarsier - 3 min):                            │
│  • Photo is clear ✅                                         │
│  • Location is appropriate ✅                                │
│  • Identification is correct ✅                              │
│  • Decision: APPROVE                                         │
│  • Remarks: "Excellent photo quality. Clear ID."            │
│                                                              │
│  Submission 3 (Cloud Rat - 2.5 min):                        │
│  • Photo shows unusual coloring 🤔                          │
│  • Could be juvenile or different subspecies                 │
│  • Decision: HOLD FOR REVIEW                                 │
│  • Remarks: "Uncertain ID. Flagging for senior review."     │
│  • Flags: @senior_validator                                 │
│                                                              │
│  Submissions 4-12 (Average 3 min each):                     │
│  • Mix of approvals (6), rejections (2), holds (1)          │
│  • Total time: ~30 minutes                                   │
│                                                              │
│  [Dr. Cruz completes all 12 assigned submissions]           │
├─────────────────────────────────────────────────────────────┤
│  Workflow Efficiency:                                        │
│  Total Time: ~50 minutes                                     │
│  Submissions Processed: 12                                   │
│  Average Time: 4.2 minutes each (within comfort zone)        │
│  • 7 Approvals                                               │
│  • 3 Rejections (with detailed feedback)                     │
│  • 2 Holds (for senior review)                               │
├─────────────────────────────────────────────────────────────┤
│  UTAUT Analysis (Overall Session):                           │
│  PE: ★★★★★ Validation session was highly productive        │
│           Queue system enabled efficient batch processing    │
│           Feedback mechanism improves PA data quality        │
│           Metrics will reflect today's productivity          │
│  EE: ★★★★★ Workflow was smooth and uninterrupted           │
│           No technical issues                                │
│           Interface supported fast decisions                 │
│           Transitions between submissions were seamless      │
│  SI: ★★★★★ Fulfilled validation role effectively           │
│           Contributed to PA biodiversity monitoring          │
│           Collaborative relationship with PA Staff           │
│           Professional identity reinforced                   │
│  FC: ★★★★★ System supported entire workflow flawlessly     │
│           All tools worked as expected                       │
│           No need for external resources                     │
│           Comfortable, efficient experience                  │
├─────────────────────────────────────────────────────────────┤
│  Behavioral Intention (End of Session):                      │
│  ★★★★★ "I will continue using this system daily"           │
│  ★★★★★ "This is much better than email-based validation"   │
│  ★★★★★ "I feel my work is making a real impact"            │
│  ★★★★☆ "Would love a few more features (batch approve?)"   │
└─────────────────────────────────────────────────────────────┘
```

---

### Journey 2: UTAUT Score Summary

| Construct | Rating | Key Drivers |
|-----------|--------|-------------|
| **Performance Expectancy** | 9.8/10 | • High productivity (12 validations in 50 min)<br>• Queue system enables efficient workflow<br>• Metrics track performance<br>• Feedback mechanism improves PA data quality |
| **Effort Expectancy** | 9.7/10 | • Intuitive workspace layout<br>• Smooth navigation<br>• No learning curve<br>• Tools work reliably |
| **Social Influence** | 9.5/10 | • Professional role identity<br>• Collaborative relationship with PA Staff<br>• Contributes to conservation goals<br>• Accountability through transparency |
| **Facilitating Conditions** | 9.8/10 | • System is reliable and fast<br>• No technical barriers<br>• All tools work as expected<br>• Supports expert judgment |
| **Behavioral Intention** | 9.6/10 | • Will use daily<br>• Prefers this over alternatives<br>• Feels work has impact<br>• Minor feature requests |

**Overall Journey Success**: 9.7/10 - **Excellent**

---

## 🔧 Journey 3: System Admin - New User Approval

### User Profile
- **Name**: Admin Sarah Lopez
- **Role**: System Administrator
- **Context**: Monday morning, checking pending user requests
- **Goal**: Approve legitimate users, reject spam/invalid requests
- **Tech Proficiency**: Very High

---

### Journey Map (Condensed)

```
TRIGGER: Weekly routine - Check pending users
   ↓
STEP 1: Login → Admin Dashboard
   • Sees "5 Pending User Approvals" alert
   • Clicks "Review Pending Users"
   • UTAUT: PE ★★★★★ (Clear workload), EE ★★★★★ (Simple access)
   ↓
STEP 2: Review Pending Users List
   • Table shows: Name, Email, Role, Organization, Date
   • Sorts by date (oldest first - fairness)
   • UTAUT: PE ★★★★★ (Efficient review), SI ★★★★★ (Fair process)
   ↓
STEP 3: Review Individual User (Carlo Mendoza, PA Staff)
   • Email: carlo.mendoza@denr.gov.ph ✅ (legitimate domain)
   • Role: PA Staff ✅
   • Organization: DENR Region 12, BMB ✅
   • Registration date: 3 days ago
   • Decision: APPROVE
   • UTAUT: PE ★★★★★ (Quick decision), EE ★★★★★ (Clear info)
   ↓
STEP 4: Approve User
   • Clicks "Approve" button
   • Confirmation: "User approved. Notification email sent."
   • User can now login
   • UTAUT: PE ★★★★★ (Instant), SI ★★★★★ (User empowered)
   ↓
STEP 5: Review Suspicious User (john.doe@example.com)
   • Email: john.doe@example.com ❌ (suspicious)
   • Role: System Admin ❌ (red flag)
   • Organization: "Test Org" ❌
   • Decision: REJECT
   • Rejection reason: "Invalid organization. Please contact IT."
   • UTAUT: FC ★★★★★ (System security maintained)
   ↓
OUTCOME: 4 approved, 1 rejected in 10 minutes
   • Efficient process
   • Security maintained
   • Users can now contribute to system
```

**UTAUT Summary**: PE 9.5/10, EE 9.8/10, SI 9.0/10, FC 9.7/10, BI 9.5/10

---

## 📊 Cross-Journey UTAUT Insights

### Common Success Factors Across All Journeys

1. **Performance Expectancy**
   - Fast page loads (<2s)
   - Efficient workflows (tasks completed quickly)
   - Immediate feedback on actions
   - Visible productivity metrics

2. **Effort Expectancy**
   - Intuitive navigation (no training needed)
   - Clear visual design (colors, icons, badges)
   - Plain language (no jargon)
   - Logical information architecture

3. **Social Influence**
   - Professional system appearance
   - Clear roles and accountability
   - Respectful, collaborative communication
   - Transparency in processes

4. **Facilitating Conditions**
   - System reliability (no crashes)
   - Help resources available
   - Error handling that guides users
   - Accessible from work locations

---

## 📈 UTAUT Journey Measurement Template

### For User Testing / Surveys

After completing a journey, ask users to rate (1-7 scale):

**Performance Expectancy:**
1. "This system helps me accomplish my work tasks quickly"
2. "Using this system improves my work efficiency"
3. "I find this system useful in my work"

**Effort Expectancy:**
1. "Learning to use this system was easy"
2. "I find this system easy to use"
3. "It is easy to find what I need in this system"

**Social Influence:**
1. "My colleagues use this system"
2. "My supervisors support use of this system"
3. "The system fits well with our organizational culture"

**Facilitating Conditions:**
1. "I have the resources needed to use this system"
2. "Technical support is available when I need it"
3. "The system is compatible with my workflow"

**Behavioral Intention:**
1. "I intend to use this system regularly"
2. "I will recommend this system to others"
3. "I plan to continue using this system"

---

## 🎯 Journey Optimization Priorities

### Based on UTAUT Analysis

**High Priority (Impact >9.0 is Critical):**
1. **Validation Feedback Quality** - Directly impacts PA Staff learning and motivation
2. **Validation Workflow Efficiency** - Enables validators to process more submissions
3. **Dashboard Load Speed** - First impression matters
4. **Clear Visual Indicators** - Status badges, colors, icons (EE)

**Medium Priority (Impact 8.0-9.0):**
1. Search and filter functionality
2. Mobile responsiveness for field use
3. Notification system
4. Batch operations

**Low Priority (Nice to Have):**
1. Dark mode
2. Customizable dashboard layouts
3. Advanced analytics

---

## ✅ Journey Testing Checklist

### For Each New Feature

- [ ] Map the complete user journey from trigger to outcome
- [ ] Identify all touchpoints and potential pain points
- [ ] Evaluate each step using UTAUT framework
- [ ] Test with real users (if possible)
- [ ] Measure quantitative metrics (time, error rate)
- [ ] Collect qualitative feedback (surveys, interviews)
- [ ] Calculate UTAUT scores
- [ ] Iterate and improve based on findings

---

*Document Version: 1.0*  
*Last Updated: January 6, 2026*  
*Part of PRISM-Matutum Testing Documentation*
