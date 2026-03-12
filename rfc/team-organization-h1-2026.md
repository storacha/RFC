## Development Team Structure & Cadence Proposal

## Authors

- [Hannah Howard](https://github.com/hannahhoward), [Storacha Network](https://storacha.network/)

### Goals

As we enter H1 2026, Storacha has a significant workload and needs to move quickly. This proposal optimizes our team processes to deliver more efficiently.

This proposal is organized around two guiding principles:
- **Tighter organization**: More organized backlog and project board, improved visibility on velocity and metrics, and greater accountability to deliverables
- **Focused work time**: More opportunity for deep work with fewer unnecessary meetings

### Overall Team Structure

**Leadership & Planning Core**
- CTO + 2 Tech Leads form the planning core
- Tech Leads will typically lead project sub-teams

**Dynamic Sub-Teams By Initiative**
- 2-3 active sub-teams at any given time, each focused on a major initiative
- Each sub-team includes a Tech Lead and Product representative
- Sub-teams are responsible for detailed estimation and execution
- Sub-teams hold 1-2 synchronous meetings per week
- Most people are on exactly one sub-team; in rare cases, individuals may be assigned to multiple teams
- Current active initiatives (for reference): "Forge 1.1", "AWS Cost Measurement Test for Forge Network"

**Workstream Organization**
- Three consistent H1 2026 workstreams: Network Launch, Cost Optimization, and Forge Product Development

### Tracking

**GitHub Integration**
- All work tracked in GitHub via issues and the main GH Project

**Initiative Tracking**
- Each initiative must have **a single common ancestor issue** with type "Initiative"

**Kanban View**
- Swim lanes organized by "Initiative" (see Automation section for implementation details)
- "Backlog" status ordered by priority to inform Sprint Backlog selection

**Timeline View (GANTT Chart)**
- Sliceable by workstream
- Managed by leadership and planning team

### Sprint & Planning Rhythm

**Weekly Sprints** (replacing bi-weekly)

**Monday**
- Automated Discord message: assigned issues + status update request
- All developers update out-of-date issues
- 10-minute team standup (synchronous, strict format)

**Tuesday**
- Async standup via Discord thread
- Tech Leads + CTO + Product sync: velocity review, backlog grooming, and high-level sprint planning (synchronous, 1 hour)

**Wednesday**
- Async standup via Discord thread
- Team-wide sprint overview meeting: priorities and context for the week (synchronous, 30 minutes)

**Thursday**
- Async standup via Discord thread
- Optional open discussion: informal chat and reserved space for focused technical discussions
- Good day for sub-team estimation sessions

**Friday**
- 10-minute team standup (synchronous, strict format)

**Continuous**
- Daily Discord automation flags in-progress work without status updates for more than 3 days
- Sub-team meetings as scheduled (1-2 per week)

### Standup Format

**Synchronous Standups (Monday/Friday):**
- Strict 10-minute timebox
- Each person shares:
  - One sentence: what you worked on
  - One sentence: what you're working on
  - One sentence: blockers (if any)
- No discussion during standup
- Enforced parking lot: any discussion after 10 minutes is optional and clearly marked

**Asynchronous Standups (Tuesday/Wednesday/Thursday):**
- Posted in Discord thread

### Monthly Cadence

**Retrospectives** (monthly)
- Session 1: Tech Leads + CTO + Product review project progress, metrics, and velocity
- Session 2: All-team retrospective, beginning with an overview of accomplishments

### Automation

We're developing automation tools to support project organization. See: https://github.com/storacha/project-agent

**Active Automations:**
- **Weekly issue status digest**: Private message with the status of issues you own
- **Stale issues summary**: Daily report sent to #core-dev
- **Auto icebox**: Issues with no updates for more than 6 months automatically move to icebox
- **Auto pull-request finder**: Linking a GitHub issue in a pull request automatically updates the project status
- **Initiative setter**: All sub-issues and descendants in an initiative automatically populate into the project with the "Initiative" field set to the initiative issue title (this works around GitHub's lack of "slice by root ancestor" filtering)
- **Standup thread poster**: Automated Discord thread creation for async standups

**Disabled/In Development:**
- **Duplicate detector**: Not yet accurate enough to enable

**Future Automations to Consider:**
- Automatic timeline adjustment based on team velocity and blocking relationships
- Additional suggestions welcome

### Ongoing Practices

**Demos & Celebrations**
- Continue current format

---

**Implementation Timeline**
- Transition to new structure: Week of February 9th, 2025

