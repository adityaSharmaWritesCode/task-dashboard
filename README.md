# TaskTree Project Specification

## Project Overview

**TaskTree** is a progressive web application for hierarchical task management with advanced analytics. It allows users to create nested projects/tasks, track completion status, and gain insights through burndown charts, deadline heatmaps, and completion rate analytics.

**Key Philosophy:** Recursive, tree-based task structure with localStorage persistence and real-time analytics.

---

## Core Architecture

### Tech Stack
- **Frontend:** Vanilla JavaScript (ES6+)
- **Styling:** CSS3 (custom properties for theming)
- **Storage:** Browser localStorage (key: `tasktree_v3`)
- **Rendering:** HTML5 Canvas for analytics charts
- **Fonts:** Playfair Display (headings), Inter (UI), DM Mono (legacy, being phased out)

### File Structure
Single-file HTML application (`taskdash.html`) containing:
- HTML structure (navigation, panels, modals)
- CSS styling (themes, layouts, components)
- JavaScript logic (data model, rendering, analytics)

---

## Data Model

### Task Object Structure
```javascript
{
  id: "string",              // Unique identifier (timestamp-based)
  name: "string",            // Task/project name
  deadline: "YYYY-MM-DD",    // Optional deadline date
  weightage: 1-100,          // Relative priority/effort weight
  done: boolean,             // Completion flag
  createdAt: "YYYY-MM-DD",   // Date task was created
  completedAt: "YYYY-MM-DD", // Date task was marked done (null if not done)
  children: [Task]           // Nested subtasks (recursive array)
}
```

### Storage
- **Key:** `tasktree_v3` in browser localStorage
- **Format:** JSON stringified array of root tasks
- **Persistence:** Automatic on any task modification
- **Migration:** Legacy tasks auto-populated with new fields on load

---

## UI Layout

### Three-Column Design (Desktop)

```
┌─────────────────────────────────────────────────────────────┐
│ Dashboard  ☰  🌙  + Add                                     │ ← Top Bar
├──────────────┬────────────────────────────────┬─────────────┤
│ NAVIGATE     │ Chart / Overview / Details     │ PROJECTS    │
│ • Home       │ • Weight Distribution Pie      │ • Task List │
│ • Breadcrumb │ • Overview Stats (4 cards)     │ • Show/Hide │
│              │ • Due This Week / Overdue      │   Done      │
│ ANALYTICS    │                                │             │
│ • Completion │                                │             │
│ • Burndown   │                                │             │
│ • On-Time %  │                                │             │
│ • Heatmap    │                                │             │
│              │                                │             │
│ TOOLS        │                                │             │
│ • Export     │                                │             │
│ • Import     │                                │             │
└──────────────┴────────────────────────────────┴─────────────┘
```

#### Left Sidebar (260px)
- **Navigate Section:** Home link + breadcrumb path of current location
- **Analytics Section:** Clickable links to Burndown, Completion Rate, On-Time %, Heatmap
- **Tools Section:** Export/Import JSON task data
- **Mobile Behavior:** Slides in from left as overlay when hamburger clicked

#### Middle Panel (Flex 1fr)
- **Default View:** Chart + stats + details (toggles away for analytics)
  - Weight Distribution donut chart (top-left, 240px)
  - Overview stats grid (4 boxes: Total, Done %, Overdue, Due Soon)
  - Details card (Due This Week or Overdue list)
- **Analytics Views:** Full-screen visualization when analytics clicked
  - Burndown Chart (line chart, cumulative created vs completed)
  - Completion Rate (donut, % complete + counts)
  - On-Time % (donut, % on-deadline + counts)
  - Deadline Heatmap (3-month calendar with task density)

#### Right Panel (380px)
- **Panel Header:** "Current Level" label + context title + "Hide Done" toggle
- **Task List:** Hierarchical list of tasks at current nesting level
  - Checkbox to mark done
  - Task name (clickable to drill in)
  - Deadline badge (colored: overdue red, today red, soon amber, future blue)
  - Weight + percentage of current level's total
  - Subtask count
  - Edit / Delete buttons
- **Empty State:** When no tasks exist at level

#### Top Bar (54px, full width)
- **Left:** Hamburger (mobile only) + "Dashboard" brand
- **Right:** Theme toggle (🌙/☀️) + "+ Add" button

---

## State Management

### Global State Variables
```javascript
let tasks = [];              // Root task array (persisted)
let viewStack = [];          // Navigation path [id1, id2, id3, ...]
let editingId = null;        // ID of task being edited
let detailsMode = 'soon';    // 'soon' or 'overdue' for details card
let showDone = true;         // Show/hide completed tasks
let currentTheme = 'dark';   // 'dark' or 'light'
let segments = [];           // Pie chart segment data
```

### State Flow
1. **Load Data:** `loadData()` retrieves from localStorage, auto-migrates missing fields
2. **Navigation:** Push/pop viewStack IDs, call `render()`
3. **Modification:** Update task, call `saveData()` + `render()`
4. **Rendering:** Compute UI from current state
5. **Persistence:** Every change auto-saves to localStorage

---

## Navigation System

### View Stack
- Empty `[]` = Home (root projects level)
- `['proj1']` = Inside project proj1
- `['proj1', 'task2']` = Inside task2 (child of proj1)
- `['proj1', 'task2', 'subtask3']` = Nested indefinitely

### Navigation Functions
- `navigateTo(id)` — Push ID onto stack, render
- `navigateToIndex(i)` — Slice stack to specific depth, render
- `goHome()` — Reset to root, close analytics view

### Breadcrumb Navigation
- Sidebar shows current path as clickable items
- Top bar shows: `🏠 Home › Project › Task › Current`
- Clicking any breadcrumb jumps to that level

---

## Feature Breakdown

### 1. Task Management

#### Create Task
- Modal form with: Name, Deadline, Weightage (1-100 slider)
- New tasks get: unique ID, createdAt = today, done = false
- Added to current list (`getCurrentList()`)
- Auto-saves to localStorage

#### Edit Task
- Click ✎ button on task row
- Modal pre-fills current values
- Updates task in place (no structural changes)

#### Delete Task
- Confirm dialog prevents accidental deletion
- Removes task and all descendants
- If current view is inside deleted task, pops navigation stack

#### Mark Complete
- Click checkbox on task row
- Sets `done = true`, `completedAt = today`
- Task fades to 50% opacity with strikethrough
- Can be toggled back to incomplete

#### Show/Hide Done
- "Hide Done" / "Show Done" button in projects panel
- Filters task list (excludes done tasks)
- Chart and stats respect filter
- Does NOT delete or change completed tasks

---

### 2. Weight Distribution (Pie Chart)

#### Data Computation
- Excludes completed tasks (shows only active workload)
- Gets current level's tasks
- Total = sum of all weightages
- Calculates percentage for each task

#### Visual Rendering
- Donut chart (inner radius 27%, outer 42% of canvas)
- 240px x 240px canvas
- Gap between segments (0.014 rad)
- Center label: task count + "projects" / "tasks"

#### Legend
- Color dot + task name + weight (number and %)
- Clickable items navigate into that task

#### Tooltip
- Hover segment → shows task name, weight, % of total
- "Click to open →" hint
- Positioned to not overflow chart area

#### Click-to-Navigate
- Click any segment → drill into that task
- Click legend item → drill into that task

---

### 3. Overview Statistics

#### Four Stat Cards
1. **Total** — All tasks across entire hierarchy (recursive count)
2. **Done %** — Green percentage + progress bar, current level completion
3. **Overdue** — Count of past-deadline tasks (red if > 0)
4. **Due Soon** — Count of tasks due within 7 days (amber if > 0)

#### Interactivity
- Click "Done %" card → toggle Hide/Show Done
- Click "Overdue" or "Due Soon" → populate details card with matching tasks

#### Progress Bar
- Green bar fills to completion percentage
- Width animates on state change

---

### 4. Details Card (Due This Week / Overdue)

#### Data Source
- All tasks across hierarchy (recursive)
- Filter by: `daysLeft(deadline)` in range [0, 7] OR overdue

#### Display
- Title: "Due This Week" or "Overdue Tasks"
- Subtitle: Task count badge
- List of matching tasks:
  - Task name
  - Deadline badge ("📅 In 2 days" / "⚠ 3 days overdue")
  - Weight value

#### Interactions
- None (read-only display)
- Hidden when no matching tasks

---

### 5. Theming System

#### CSS Custom Properties
- 21 CSS variables per theme (colors, shadows, radii)
- Dark theme: `#0f1117` bg, `#58a6ff` accent
- Light theme: `#f6f8fa` bg, `#0969da` accent
- Smooth 0.25s transition on theme change

#### Toggle
- Moon (🌙) / Sun (☀️) icon button in top bar
- Saved to localStorage (`tasktree_theme` key)
- Applies immediately, persists across sessions

#### Chart Rendering
- Charts re-render on theme change to use new colors
- CSS var lookups via `getCSSVar('--var-name')`

---

### 6. Modal System

#### Task Modal (Create/Edit)
- Single modal, repurposed for both
- Fields: Name (text), Deadline (date picker), Weightage (1-100 slider)
- Range display shows current weight value
- Cancel / Save buttons
- ESC key closes modal
- Backdrop click closes modal

#### Validation
- Name required (non-empty trim)
- Deadline optional (empty string = null)
- Weightage defaults to 50, allows 1-100

#### Behavior
- Create: Sets up new task with all fields, createdAt = today
- Edit: Updates existing task (in-place)

---

### 7. Analytics Views

#### Burndown Chart
**Purpose:** Show if you're creating tasks faster than completing them

**Data:**
- Group all tasks by week created
- Compute cumulative created and completed per week
- Plot as line chart: X = weeks, Y = cumulative count

**Rendering:**
- 800px wide, 400px tall canvas
- Blue line (created), Green line (completed)
- Grid background, axes, legend
- High-DPI scaling for sharpness

**Stats Box:**
- Total created (blue) + total completed (green)
- Helpful for: spotting backlog buildup, velocity trends

#### Completion Rate Chart
**Purpose:** Show overall progress toward finishing all tasks

**Data:**
- Count tasks with `done = true` vs. total
- Percentage = (done / total) * 100

**Rendering:**
- 240px donut chart
- Green arc (done) + gray arc (remaining)
- Center text: percentage + "complete"
- Stat boxes: count completed + count remaining

**Helpful for:** motivation, project progress snapshot

#### On-Time Completion %
**Purpose:** Show if you're realistic with deadline estimates

**Data:**
- Only tasks with both deadline AND completedAt
- Compare: completedAt <= deadline = on-time, else late
- Percentage = (onTime / total) * 100

**Rendering:**
- 240px donut chart
- Green arc (on-time) + red arc (late)
- Center text: percentage + "on time"
- Stat boxes: count on-time + count late
- Success rate callout box

**Helpful for:** identifying overcommitment, improving estimates

#### Deadline Heatmap
**Purpose:** Visualize task clustering and spread deadlines evenly

**Data:**
- Group tasks by `deadline` date
- Count per day
- Show 3 months: current + next 2

**Rendering:**
- Calendar grid per month
- Day headers (Sun-Sat)
- Color intensity by count:
  - Empty (0 tasks) = white
  - Low (1-2) = light blue
  - Medium (3-5) = medium blue
  - High (6+) = dark blue
- Hover tooltips show exact count
- Legend explains color scale

**Helpful for:** spreading deadlines, identifying bottleneck dates

#### Closing Analytics
- Click "← Back" button in top-left
- Returns to default view
- If in nested view, pops to home

---

## Helper Functions

### Tree Traversal
- `findTask(id, list?)` — Recursive search for task by ID anywhere in tree
- `deleteTask(id, list?)` — Recursive deletion from tree
- `countDescendants(task)` — Count all nested children
- `getTasksFromList(list?)` — Flatten tree into single array

### Data Queries
- `getCurrentList()` — Tasks visible at current view level
- `getCurrentTask()` — Current context task (null if home)
- `getTasksFromList()` — All tasks recursively from a point

### Date Utilities
- `isOverdue(task)` — Boolean, is deadline in past?
- `daysLeft(deadline)` — Integer, days until deadline (negative if overdue)
- `formatDate(date)` — Format as "16 May 24"

### Aggregation (Recursive)
- `countAll()` — Total tasks in entire tree
- `countOverdue()` — Total overdue tasks anywhere
- `countSoon()` — Total due within 7 days anywhere

### UI Utilities
- `getColor(index)` — Returns chart color from palette by index
- `getCSSVar(name)` — Reads CSS custom property value
- `uid()` — Generate unique task ID (timestamp + random)

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `ESC` | Close modal OR pop navigation (if in nested view) OR close sidebar (mobile) |
| `N` | Open add task modal |
| `Cmd/Ctrl + Enter` | Save task in modal |

---

## Storage & Persistence

### localStorage Keys
- `tasktree_v3` — Main task tree (JSON array)
- `tasktree_theme` — Current theme ('dark' or 'light')

### Auto-Migration
On load, any task missing new fields gets:
- `createdAt` = today's date (YYYY-MM-DD)
- `done` = false
- `completedAt` = null

This ensures backward compatibility with old saved data.

### Save Behavior
- Called after: create, edit, delete, toggle done, import
- Entire task tree serialized atomically
- No partial saves (all-or-nothing)

---

## Responsive Design

### Desktop (>1200px)
- All three columns visible side-by-side
- Hamburger hidden
- Full width layout

### Tablet (768px - 1200px)
- Middle and right panels stack
- Sidebar visible (not overlay)
- Reduced padding/spacing

### Mobile (<768px)
- Single column layout
- Hamburger menu visible (☰)
- Sidebar slides in as fixed overlay from left
- Semi-transparent backdrop on sidebar open
- Right panel becomes bottom section

---

## File Export/Import

### Export
- Click "Export Data" in Tools sidebar section
- Generates JSON file: `tasks-YYYY-MM-DD.json`
- Contains full task tree with all metadata
- Downloaded to user's device

### Import
- Click "Import Data" in Tools sidebar section
- Opens file picker (JSON only)
- Parses file, validates structure
- **Warning:** Replaces entire current task tree
- Resets navigation to home
- Shows success/error alert

### Format
```json
[
  {
    "id": "...",
    "name": "Project",
    "deadline": "2025-05-20",
    "weightage": 50,
    "done": false,
    "createdAt": "2025-05-01",
    "completedAt": null,
    "children": [...]
  }
]
```

---

## Color Palette

### Dark Theme
| Element | Color | Hex |
|---------|-------|-----|
| Background | Surface | #0f1117 |
| Cards | Surface2 | #102128 |
| Accent Primary | Blue | #58a6ff |
| Accent Secondary | Lighter Blue | #388bfd |
| Success | Green | #3fb950 |
| Danger | Red | #f78166 |
| Warning | Amber | #ffa657 |

### Light Theme
| Element | Color | Hex |
|---------|-------|-----|
| Background | Off-white | #f6f8fa |
| Cards | White | #ffffff |
| Accent Primary | Blue | #0969da |
| Accent Secondary | Darker Blue | #0550ae |
| Success | Green | #1a7f37 |
| Danger | Red | #cf222e |
| Warning | Amber | #9a6700 |

---

## Browser Compatibility

- **Tested:** Chrome, Safari, Firefox, Edge (latest)
- **Requirements:** ES6+, CSS Grid, Canvas, localStorage
- **Not Supported:** IE11 and below

---

## Performance Considerations

### Rendering Optimization
- Chart re-renders only on state change (not on every keystroke)
- `setTimeout(..., 10)` defers heavy canvas work off main thread
- Recursive tree traversal acceptable for typical task counts (<1000)

### Memory
- Entire task tree in memory (JSON stringified on save)
- Chart segment data cached in `segments[]` array
- No caching layer needed for typical use

### Storage
- localStorage limit typically 5-10MB per domain
- Practical limit: tens of thousands of tasks

---

## Future Enhancement Ideas

### Quick Wins
1. Task search/filter by keyword
2. Bulk select + mark done
3. "Due Today/Tomorrow/This Week" deadline buttons
4. Recurring tasks (daily/weekly/monthly)

### Analytics
1. Velocity trend (tasks completed per week)
2. Average completion time (days from creation to done)
3. Per-project health scores
4. Forecast warnings ("47 weight due next week")

### Workflow
1. Floating + button for quick add
2. Drag-and-drop task reordering
3. Task templates/subtask presets
4. Notes field on tasks

### Mobile
1. Swipe right to mark done
2. Swipe left to delete
3. Home screen widget
4. Offline sync support

### Data
1. CSV/spreadsheet export
2. Weekly digest emails
3. Time tracking (estimated vs actual)
4. Goal/milestone tracking

---

## Known Limitations

1. **No Authentication:** Data lives only in browser localStorage, not synced across devices
2. **No Undo:** Deleted tasks cannot be recovered (except via JSON import)
3. **No Sorting:** Tasks display in creation order (no custom drag-to-reorder yet)
4. **No Collaboration:** Single-user only (no sharing or team features)
5. **Mobile Charts:** Donuts sized fixed (240px); may be small on small screens
6. **Storage Limit:** Browser localStorage ~5-10MB practical limit

---

## Development Notes

### Code Organization
- Single file (taskdash.html) for ease of hosting
- ~1600 lines total (HTML + CSS + JS)
- No build process, transpiler, or package manager needed
- Pure vanilla JS (no frameworks or libraries)

### Testing Approach
- Manual testing on desktop + mobile browsers
- Add tasks, navigate, toggle states
- Export/import cycle
- Theme switching
- All analytics render correctly

### Debugging Tips
- Open DevTools console to inspect `tasks` array directly
- Use `localStorage.getItem('tasktree_v3')` to view raw data
- Use `localStorage.clear()` to reset all data
- Check network tab for localStorage size

---

## Summary

**TaskTree** is a lightweight, self-contained task management tool emphasizing recursive hierarchy, analytics-driven insights, and offline-first storage. It demonstrates advanced canvas rendering, state management without frameworks, and responsive design principles in a single HTML file.

The core innovation is treating tasks as trees (projects contain subtasks), combined with temporal data (createdAt, completedAt) to enable burndown and on-time metrics. Analytics surface whether the user is drowning in backlog, how realistic estimates are, and where deadlines cluster.

**Total LOC:** ~1600  
**Dependencies:** None (pure HTML/CSS/JS)  
**Storage:** Browser localStorage  
**Hosting:** Static file server (GitHub Pages, Netlify, etc.)
