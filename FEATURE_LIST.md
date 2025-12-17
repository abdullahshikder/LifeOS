# Benji.so Feature List - Reference for Local App

## Overview
Benji.so is a comprehensive productivity and wellness application focused on habit tracking, task management, health tracking, and productivity enhancement. This document outlines all features discovered through exploration to help you build a similar local application.

**Key Discovery:** The app includes far more features than initially documented, including health tracking (Food, Weight, Hydration, Mood, Workouts, Fasting), social features (Timeline, Leaderboard), specialized tools (Routines, Countdowns, Goals, Trips, Packing items), and focus enhancement (Background Music & Ambient Sounds).

---

## 1. HABIT TRACKING

### Feature Overview

**Purpose**: Enable users to build and maintain positive habits through daily tracking, visual progress indicators, and gamified scoring systems.

**Use Cases**:
- Track daily habits like exercise, reading, meditation
- Monitor consistency over time with streaks and scores
- Organize habits into categories (work vs personal)
- Visualize progress through multiple view modes
- Identify patterns and areas for improvement

**Integration Points**:
- Links to Planner for scheduled habit times
- Can be included in Routines
- Data feeds into Dashboard widgets
- Timeline posts can be created from habit completions

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ [Habits]                                    [+] [⋮] [🔍] [Lists] │
│ ─────────────────────────────────────────────────────── │
│ [Day] [Week] [Graph]    ← Wed, 17 Dec →                  │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│  ┌────────────────────────────────────────────┐        │
│  │ Habit Name                    [✓] [Grade: A]│        │
│  │ Description text...                         │        │
│  │ Streak: 15 days | Best: 23 days            │        │
│  └────────────────────────────────────────────┘        │
│                                                          │
│  ┌────────────────────────────────────────────┐        │
│  │ Another Habit                  [ ] [Grade: B]│        │
│  └────────────────────────────────────────────┘        │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **HabitCard**: Displays habit name, description, completion checkbox, grade badge, streak info
2. **HabitForm**: Modal/form for creating/editing habits (name, description, list, category, schedule)
3. **ViewTabs**: Day/Week/Graph toggle buttons
4. **DateNavigator**: Left/right arrows with date display
5. **FilterBar**: Lists dropdown, category toggle (Both/Personal/Work), hide completed toggle
6. **HabitGrid**: Calendar grid showing completion dots for each day
7. **GraphView**: Line/bar chart showing completion trends
8. **EmptyState**: Large icon, "No habits yet", "Create your first habit?" prompt
9. **ListSidebar**: Collapsible sidebar showing habit lists with visibility toggles

#### Visual States

**Empty State**:
- Large refresh/loop icon (centered)
- Bold text: "No habits yet"
- Subtext: "Create your first habit?"
- Plus button or clickable area to create habit

**Loading State**:
- Skeleton loaders for habit cards
- Shimmer effect on grid view

**Error State**:
- Error message banner at top
- Retry button
- Error icon

**Success State**:
- Brief toast notification on completion
- Checkmark animation on habit card
- Streak counter updates with animation

#### Interactions
- **Click habit checkbox**: Toggle completion, update streak, recalculate score
- **Click habit card**: Open edit modal
- **Hover over grade badge**: Show tooltip with score breakdown
- **Click date arrows**: Navigate to previous/next day
- **Click view tabs**: Switch between Day/Week/Graph views
- **Click Lists filter**: Show dropdown with checkboxes for each list
- **Toggle hide completed**: Instantly filter out completed habits
- **Keyboard shortcuts**: 
  - `N` - New habit
  - `←`/`→` - Navigate dates
  - `T` - Toggle today
  - `C` - Toggle completed filter

#### Responsive Behavior
- **Desktop (>1024px)**: Full layout with sidebar, grid view shows 7+ columns
- **Tablet (768-1024px)**: Collapsible sidebar, grid shows 4-5 columns
- **Mobile (<768px)**: Bottom navigation, single column list, swipe for date navigation

#### Color Scheme
- **Completed habit**: Green checkmark (#10B981), light green background (#D1FAE5)
- **Incomplete habit**: Grey checkbox (#9CA3AF), white background
- **Grade badges**: 
  - A: Green (#10B981)
  - B: Blue (#3B82F6)
  - C: Yellow (#F59E0B)
  - D: Orange (#F97316)
  - F: Red (#EF4444)
- **Streak indicator**: Orange/red gradient for long streaks
- **Current day**: Red circle border (#EF4444)

#### Animations
- Checkbox toggle: Scale animation (0.8 → 1.0)
- Streak update: Counter animation with bounce
- Grade change: Color transition fade
- Grid dot appearance: Fade in with scale
- View switch: Slide transition

---

### Data Model Details

#### Complete Schema

**Habit Table**:
```typescript
interface Habit {
  id: string;                    // UUID, primary key
  name: string;                  // Required, max 100 chars
  description?: string;          // Optional, max 500 chars
  listId?: string;               // Foreign key to List, nullable
  category: 'work' | 'personal';  // Required, default 'personal'
  schedule: HabitSchedule;       // See below
  color?: string;                // Hex color, default based on category
  icon?: string;                 // Icon identifier
  targetCount?: number;           // For quantity-based habits (e.g., "drink 8 glasses")
  unit?: string;                 // Unit for quantity (e.g., "glasses", "pages")
  createdAt: Date;              // Auto-generated
  updatedAt: Date;              // Auto-updated on changes
  archived: boolean;             // Soft delete, default false
  archivedAt?: Date;            // When archived
}

interface HabitSchedule {
  type: 'daily' | 'weekly' | 'custom';
  daysOfWeek?: number[];        // 0-6 (Sun-Sat), for weekly/custom
  timesPerDay?: number;         // For multiple completions per day
  startDate?: Date;              // When habit starts
  endDate?: Date;                // Optional end date
}
```

**HabitEntry Table** (Daily completions):
```typescript
interface HabitEntry {
  id: string;                    // UUID, primary key
  habitId: string;               // Foreign key to Habit, indexed
  date: Date;                     // Date only (no time), indexed, unique with habitId
  completed: boolean;            // Default false
  count?: number;                // For quantity-based habits
  notes?: string;                // Optional notes for the day
  completedAt?: Date;           // Timestamp when marked complete
  createdAt: Date;              // Auto-generated
}
```

**HabitStreak Table**:
```typescript
interface HabitStreak {
  id: string;
  habitId: string;               // Foreign key, indexed
  startDate: Date;               // Start of current streak
  endDate?: Date;                // End date if broken, null if current
  currentStreak: number;         // Days in current streak
  longestStreak: number;         // Best streak ever
  lastUpdated: Date;            // When streak was last calculated
}
```

**HabitScore Table**:
```typescript
interface HabitScore {
  id: string;
  habitId: string;               // Foreign key, indexed
  date: Date;                     // Date of score calculation
  score: 'A' | 'B' | 'C' | 'D' | 'F';
  completionRate: number;        // 0-100, percentage
  daysCompleted: number;         // Out of 30 days
  periodStart: Date;             // Start of 30-day period
  periodEnd: Date;               // End of 30-day period
}
```

#### Relationships
- Habit → List (many-to-one, optional)
- Habit → HabitEntry (one-to-many)
- Habit → HabitStreak (one-to-one)
- Habit → HabitScore (one-to-many, latest is current)

#### Validation Rules
- **Habit name**: Required, 1-100 characters, trim whitespace
- **Description**: Optional, max 500 characters
- **Schedule daysOfWeek**: Must be array of 0-6, no duplicates
- **Date**: Must be valid date, not future dates for entries
- **Count**: If provided, must be positive number, max 1000

#### Indexes
- `HabitEntry(habitId, date)` - Composite unique index
- `HabitEntry(date)` - For date range queries
- `Habit(listId)` - For filtering by list
- `Habit(category)` - For category filtering
- `HabitScore(habitId, date DESC)` - For latest score lookup

#### Default Values
- `category`: 'personal'
- `schedule.type`: 'daily'
- `archived`: false
- `completed`: false (for entries)
- `currentStreak`: 0 (for new streaks)

---

### Business Logic & Algorithms

#### Streak Calculation Algorithm
```typescript
function calculateStreak(habitId: string, today: Date): HabitStreak {
  // Get all entries ordered by date descending
  const entries = getHabitEntries(habitId, { orderBy: 'date DESC' });
  
  let currentStreak = 0;
  let longestStreak = 0;
  let tempStreak = 0;
  let startDate = today;
  
  // Check if today is completed
  const todayEntry = entries.find(e => isSameDay(e.date, today));
  if (todayEntry?.completed) {
    currentStreak = 1;
    tempStreak = 1;
  }
  
  // Walk backwards through entries
  let checkDate = yesterday(today);
  for (const entry of entries) {
    // Fill gaps - if entry date is before checkDate, fill with incomplete
    while (isBefore(checkDate, entry.date)) {
      if (tempStreak > 0) {
        // Streak broken
        longestStreak = Math.max(longestStreak, tempStreak);
        tempStreak = 0;
      }
      checkDate = yesterday(checkDate);
    }
    
    if (entry.completed) {
      tempStreak++;
      if (currentStreak === 0 && isSameDay(entry.date, yesterday(today))) {
        // Yesterday was completed, continue streak
        currentStreak = tempStreak;
        startDate = entry.date;
      }
    } else {
      // Streak broken
      longestStreak = Math.max(longestStreak, tempStreak);
      tempStreak = 0;
      if (currentStreak > 0 && isBefore(entry.date, today)) {
        currentStreak = 0;
      }
    }
    
    checkDate = yesterday(entry.date);
  }
  
  longestStreak = Math.max(longestStreak, tempStreak);
  
  return {
    habitId,
    startDate,
    currentStreak,
    longestStreak,
    lastUpdated: today
  };
}
```

#### Scoring System (A-F Grade)
```typescript
function calculateScore(habitId: string, referenceDate: Date): HabitScore {
  const periodStart = subDays(referenceDate, 29); // 30 days including today
  const entries = getHabitEntries(habitId, {
    startDate: periodStart,
    endDate: referenceDate
  });
  
  // Filter entries based on schedule
  const scheduledDates = getScheduledDates(habitId, periodStart, referenceDate);
  const completedDates = entries.filter(e => e.completed).map(e => e.date);
  
  const completionRate = (completedDates.length / scheduledDates.length) * 100;
  
  let score: 'A' | 'B' | 'C' | 'D' | 'F';
  if (completionRate >= 90) score = 'A';
  else if (completionRate >= 75) score = 'B';
  else if (completionRate >= 60) score = 'C';
  else if (completionRate >= 40) score = 'D';
  else score = 'F';
  
  return {
    habitId,
    date: referenceDate,
    score,
    completionRate,
    daysCompleted: completedDates.length,
    periodStart,
    periodEnd: referenceDate
  };
}
```

#### Scheduled Dates Calculation
```typescript
function getScheduledDates(habit: Habit, startDate: Date, endDate: Date): Date[] {
  const dates: Date[] = [];
  let current = startDate;
  
  while (!isAfter(current, endDate)) {
    if (habit.schedule.type === 'daily') {
      dates.push(current);
    } else if (habit.schedule.type === 'weekly') {
      const dayOfWeek = current.getDay();
      if (habit.schedule.daysOfWeek?.includes(dayOfWeek)) {
        dates.push(current);
      }
    } else if (habit.schedule.type === 'custom') {
      const dayOfWeek = current.getDay();
      if (habit.schedule.daysOfWeek?.includes(dayOfWeek)) {
        dates.push(current);
      }
    }
    
    current = addDays(current, 1);
  }
  
  return dates;
}
```

#### State Management
- **Local State**: Current view (day/week/graph), selected date, filters, UI state
- **Derived State**: Filtered habits, calculated streaks, scores
- **Cache**: Recent entries, calculated streaks (invalidate on entry change)
- **Optimistic Updates**: Update UI immediately, sync to storage async

#### Edge Cases
1. **Timezone handling**: Store dates as UTC, display in user's timezone
2. **Missing entries**: Treat as incomplete, don't break streak calculation
3. **Future dates**: Prevent completion of future dates
4. **Schedule changes**: Recalculate streaks if schedule changes mid-streak
5. **Archived habits**: Don't show in main view, preserve data
6. **Deleted habits**: Soft delete, keep entries for historical data

---

### User Flows

#### Primary Flow: Complete a Habit
1. User opens Habits page (defaults to Day view, today's date)
2. Sees list of habits scheduled for today
3. Clicks checkbox on a habit card
4. Checkbox animates to checked state
5. Streak counter updates (if applicable)
6. Score badge updates color (if crossing threshold)
7. Entry saved to database
8. Toast notification: "✓ Habit completed! Streak: 15 days"

#### Alternative Flow: Create New Habit
1. User clicks "+" button or "Create your first habit?" prompt
2. Modal opens with habit form
3. User enters:
   - Name: "Morning Meditation"
   - Description: "10 minutes of mindfulness"
   - List: Selects "Health" or creates new list
   - Category: Toggles "Personal"
   - Schedule: Selects "Daily" or specific days
4. Clicks "Create" button
5. Modal closes, new habit appears in list
6. Empty state disappears if first habit

#### Alternative Flow: View Week Overview
1. User clicks "Week" tab
2. Grid view loads showing 7 days (current week)
3. Each habit shows row with 7 checkboxes (one per day)
4. User can click any checkbox to toggle that day's completion
5. Hover shows date tooltip
6. Current day highlighted with red border

#### Error Flow: Network/Storage Error
1. User attempts to complete habit
2. Action fails (storage error)
3. Checkbox reverts to unchecked
4. Error banner appears: "Failed to save. Please try again."
5. Retry button available
6. On retry, action succeeds, banner dismisses

#### Empty State Flow: First Time User
1. User navigates to Habits (no habits exist)
2. Sees empty state with large icon
3. Text: "No habits yet"
4. CTA: "Create your first habit?"
5. Clicking anywhere opens create modal
6. After creating first habit, empty state disappears

---

### API/Function Specifications

#### CRUD Operations

**createHabit(habit: CreateHabitInput): Promise<Habit>**
```typescript
interface CreateHabitInput {
  name: string;
  description?: string;
  listId?: string;
  category: 'work' | 'personal';
  schedule: HabitSchedule;
  color?: string;
  icon?: string;
  targetCount?: number;
  unit?: string;
}

// Returns: Created habit with generated ID and timestamps
// Throws: ValidationError if name missing or invalid
```

**getHabit(id: string): Promise<Habit | null>**
```typescript
// Returns: Habit or null if not found
// Includes: Related list, current streak, latest score
```

**updateHabit(id: string, updates: Partial<Habit>): Promise<Habit>**
```typescript
// Updates specified fields only
// Recalculates streaks if schedule changed
// Returns: Updated habit
// Throws: NotFoundError if habit doesn't exist
```

**deleteHabit(id: string, hardDelete?: boolean): Promise<void>**
```typescript
// If hardDelete=false (default): Sets archived=true
// If hardDelete=true: Permanently deletes habit and all entries
// Throws: NotFoundError if habit doesn't exist
```

**createHabitEntry(entry: CreateHabitEntryInput): Promise<HabitEntry>**
```typescript
interface CreateHabitEntryInput {
  habitId: string;
  date: Date;
  completed: boolean;
  count?: number;
  notes?: string;
}

// Returns: Created entry
// Throws: ValidationError if date invalid or future
// Throws: DuplicateError if entry already exists for date
// Side effect: Triggers streak recalculation
```

**getHabitEntries(filters: HabitEntryFilters): Promise<HabitEntry[]>**
```typescript
interface HabitEntryFilters {
  habitId?: string;
  startDate?: Date;
  endDate?: Date;
  completed?: boolean;
  orderBy?: 'date ASC' | 'date DESC';
  limit?: number;
}

// Returns: Array of entries matching filters
```

**toggleHabitCompletion(habitId: string, date: Date): Promise<HabitEntry>**
```typescript
// Toggles completion status for specific date
// Creates entry if doesn't exist, updates if exists
// Returns: Updated/created entry
// Side effect: Recalculates streak and score
```

#### Query Operations

**getHabitsForDate(date: Date, filters?: HabitFilters): Promise<Habit[]>**
```typescript
interface HabitFilters {
  listIds?: string[];
  category?: 'work' | 'personal' | 'both';
  hideCompleted?: boolean;
  archived?: boolean;
}

// Returns: Habits scheduled for given date, filtered
// Includes: Entry for that date (if exists), current streak, score
```

**getHabitStreak(habitId: string): Promise<HabitStreak>**
```typescript
// Returns: Current streak information
// Calculates if not cached or outdated
```

**getHabitScore(habitId: string, date?: Date): Promise<HabitScore>**
```typescript
// Returns: Score for habit (defaults to today)
// Calculates based on last 30 days
```

**getHabitGrid(habitId: string, startDate: Date, endDate: Date): Promise<HabitGridData>**
```typescript
interface HabitGridData {
  habitId: string;
  entries: { date: Date; completed: boolean }[];
  scheduledDates: Date[];
}

// Returns: Grid data for calendar view
```

#### Bulk Operations

**bulkCreateHabitEntries(entries: CreateHabitEntryInput[]): Promise<HabitEntry[]>**
```typescript
// Creates multiple entries in single transaction
// Returns: Array of created entries
// Throws: ValidationError if any entry invalid
```

**bulkToggleHabits(habitIds: string[], date: Date, completed: boolean): Promise<void>**
```typescript
// Toggles multiple habits for same date
// Useful for "complete all" functionality
```

**exportHabitData(habitId: string, format: 'json' | 'csv'): Promise<string>**
```typescript
// Exports habit and all entries
// Returns: Formatted string (JSON or CSV)
```

---

### Technical Requirements

#### Storage Structure (IndexedDB)

**Database Name**: `benji_app`
**Version**: 1

**Object Stores**:
1. `habits` - Key: `id`, Indexes: `listId`, `category`, `archived`
2. `habitEntries` - Key: `id`, Indexes: `habitId`, `date`, `(habitId, date)` unique
3. `habitStreaks` - Key: `habitId` (unique), Index: `habitId`
4. `habitScores` - Key: `id`, Indexes: `habitId`, `(habitId, date)` unique

#### Performance Considerations

**Data Volumes**:
- Habits: ~50-200 per user (reasonable)
- Entries: ~50 habits × 365 days = ~18,250 entries/year
- Optimize: Index on (habitId, date) for fast lookups
- Cache: Streak calculations cached, invalidate on entry change

**Optimization Strategies**:
1. **Lazy Loading**: Load entries only for visible date range
2. **Pagination**: Load entries in chunks (30 days at a time)
3. **Debouncing**: Debounce rapid checkbox clicks
4. **Batch Updates**: Group multiple entry updates in single transaction
5. **Background Calculation**: Calculate streaks/scores in background worker

#### Offline Support

- All operations work offline
- Use IndexedDB for local storage
- Queue failed operations for retry
- Sync indicator shows when data saved locally

#### Data Migration

**Version 1 → 2** (if adding new fields):
```typescript
function migrateHabitsV1ToV2(db: IDBDatabase) {
  const transaction = db.transaction(['habits'], 'readwrite');
  const store = transaction.objectStore('habits');
  
  store.openCursor().onsuccess = (event) => {
    const cursor = event.target.result;
    if (cursor) {
      const habit = cursor.value;
      // Add new default fields
      habit.targetCount = null;
      habit.unit = null;
      cursor.update(habit);
      cursor.continue();
    }
  };
}
```

#### Error Handling

- **Validation Errors**: Show inline form errors
- **Storage Errors**: Show toast notification, allow retry
- **Network Errors**: N/A (local app)
- **Data Corruption**: Validate on load, show recovery option

#### Testing Scenarios

1. **Streak Calculation**: Test with gaps, multiple streaks, edge dates
2. **Score Calculation**: Test with various completion rates (0%, 50%, 100%)
3. **Schedule Filtering**: Test daily, weekly, custom schedules
4. **Timezone Handling**: Test across timezones, daylight saving
5. **Concurrent Updates**: Test multiple rapid completions
6. **Large Datasets**: Test with 100+ habits, 1000+ entries

---

## 2. TODO/TASK MANAGEMENT

### Feature Overview

**Purpose**: Comprehensive task management system with organization through lists, projects, tags, and multiple viewing modes for efficient task tracking and completion.

**Use Cases**:
- Capture tasks quickly with quick add
- Organize tasks into projects and lists
- Filter and view tasks by date, category, or status
- Track progress on multi-step projects
- Manage work and personal tasks separately

**Integration Points**:
- Tasks with due dates appear in Planner calendar
- Tasks can be part of Routines
- Completed tasks can be archived or deleted
- Tasks can be linked to Habits

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Overview                                    [🔍] [⊞] [🔽] │
│ ─────────────────────────────────────────────────────── │
│ [Both] [Personal] [Work]                                 │
│ ─────────────────────────────────────────────────────── │
│ [👁 Overview] [📥 Inbox] [☀ Today] [📅 Upcoming] [⋯]   │
│ ─────────────────────────────────────────────────────── │
│ Quick add todo...                                        │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ Projects                                                 │
│ ┌────────────────────────────────────────────┐         │
│ │ Project Name                    [3/5 tasks]│         │
│ │  ☐ Task 1                                    │         │
│ │  ☑ Task 2                                    │         │
│ │  ☐ Task 3                                    │         │
│ └────────────────────────────────────────────┘         │
│                                                          │
│ Lists                                                    │
│ ┌────────────────────────────────────────────┐         │
│ │ List Name                       [2 tasks]  │         │
│ │  ☐ Task A                                    │         │
│ │  ☐ Task B                                    │         │
│ └────────────────────────────────────────────┘         │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **QuickAddInput**: Persistent input at top for fast task creation
2. **TaskCard**: Displays task with checkbox, title, due date, tags, priority indicator
3. **TaskForm**: Modal/form for detailed task creation/editing
4. **ViewTabs**: Overview/Inbox/Today/Upcoming tabs
5. **CategoryFilter**: Both/Personal/Work toggle buttons
6. **SearchBar**: Search input with magnifying glass icon
7. **FilterModal**: Advanced filter options (date range, tags, lists, projects)
8. **ProjectSection**: Collapsible section showing project with nested tasks
9. **ListSection**: Collapsible section showing list with tasks
10. **EmptyState**: Different states for each view mode
11. **LayoutToggle**: Grid/list view switcher

#### Visual States

**Empty States** (view-specific):
- **Overview**: "No tasks yet" with task icon, "Add your first task" prompt
- **Inbox**: "Inbox is empty" with inbox icon, "All tasks organized!"
- **Today**: "Nothing due today" with sun icon, "Enjoy your free day!"
- **Upcoming**: "No upcoming tasks" with calendar icon

**Loading State**:
- Skeleton loaders for task cards
- Shimmer effect during data fetch

**Error State**:
- Error banner: "Failed to load tasks"
- Retry button

**Success State**:
- Toast on task completion: "Task completed!"
- Toast on task creation: "Task added"
- Animation on checkbox toggle

#### Interactions
- **Quick Add**: Type and press Enter to create task (uses defaults)
- **Click task card**: Opens edit modal
- **Click checkbox**: Toggles completion (strikethrough animation)
- **Click view tab**: Switches view, filters tasks accordingly
- **Click category filter**: Filters by Work/Personal/Both
- **Search**: Real-time filtering as user types
- **Drag task**: Reorder within list/project (optional)
- **Keyboard shortcuts**:
  - `Q` - Focus quick add
  - `N` - New task modal
  - `Escape` - Close modal/cancel
  - `Ctrl+F` - Focus search

#### Responsive Behavior
- **Desktop**: Full layout with sidebar, grid view shows 2-3 columns
- **Tablet**: Collapsible sections, single column list
- **Mobile**: Bottom sheet for task form, swipe to navigate views

#### Color Scheme
- **Completed task**: Grey text with strikethrough (#9CA3AF)
- **Due today**: Orange indicator (#F97316)
- **Overdue**: Red indicator (#EF4444)
- **Priority High**: Red dot (#EF4444)
- **Priority Medium**: Yellow dot (#F59E0B)
- **Priority Low**: Blue dot (#3B82F6)
- **Work category**: Blue accent (#3B82F6)
- **Personal category**: Green accent (#10B981)

#### Animations
- Task completion: Strikethrough animation, fade out
- Task creation: Slide in from top
- View switch: Fade transition
- Project/list collapse: Smooth height transition

---

### Data Model Details

#### Complete Schema

**Todo Table**:
```typescript
interface Todo {
  id: string;                    // UUID, primary key
  title: string;                 // Required, max 200 chars
  description?: string;          // Optional, max 2000 chars
  completed: boolean;           // Default false
  completedAt?: Date;           // When completed
  dueDate?: Date;               // Optional due date
  priority: 'low' | 'medium' | 'high' | null; // Default null
  category: 'work' | 'personal'; // Required, default 'personal'
  listId?: string;              // Foreign key to List, nullable
  projectId?: string;            // Foreign key to Project, nullable
  tagIds: string[];             // Array of Tag IDs
  order: number;                // For sorting within list/project
  createdAt: Date;              // Auto-generated
  updatedAt: Date;              // Auto-updated
  archived: boolean;            // Soft delete, default false
  archivedAt?: Date;            // When archived
}
```

**List Table**:
```typescript
interface List {
  id: string;
  name: string;                  // Required, max 100 chars
  type: 'todo' | 'habit' | 'planner'; // Default 'todo'
  color?: string;                // Hex color
  icon?: string;                 // Icon identifier
  order: number;                 // For sorting
  createdAt: Date;
}
```

**Project Table**:
```typescript
interface Project {
  id: string;
  name: string;                  // Required, max 100 chars
  description?: string;         // Optional
  color?: string;               // Hex color
  icon?: string;                // Icon identifier
  order: number;                // For sorting
  createdAt: Date;
}
```

**Tag Table**:
```typescript
interface Tag {
  id: string;
  name: string;                  // Required, max 50 chars, unique
  color?: string;               // Hex color
  createdAt: Date;
}
```

#### Relationships
- Todo → List (many-to-one, optional)
- Todo → Project (many-to-one, optional)
- Todo → Tags (many-to-many via tagIds array)
- List → Todos (one-to-many)
- Project → Todos (one-to-many)

#### Validation Rules
- **Title**: Required, 1-200 characters, trim whitespace
- **Description**: Optional, max 2000 characters
- **Due date**: Must be valid date, can be future or past
- **Priority**: Must be one of: low, medium, high, null
- **Category**: Must be 'work' or 'personal'
- **Tag IDs**: Must reference existing tags

#### Indexes
- `Todo(listId)` - For filtering by list
- `Todo(projectId)` - For filtering by project
- `Todo(category)` - For category filtering
- `Todo(dueDate)` - For date-based queries
- `Todo(completed)` - For filtering completed/incomplete
- `Todo(createdAt)` - For sorting by creation date
- `List(type)` - For filtering list types

#### Default Values
- `completed`: false
- `priority`: null
- `category`: 'personal'
- `tagIds`: []
- `order`: 0
- `archived`: false

---

### Business Logic & Algorithms

#### Quick Add Parsing
```typescript
function parseQuickAddInput(input: string): Partial<Todo> {
  const todo: Partial<Todo> = { title: input.trim() };
  
  // Parse due date: "task name @tomorrow" or "task name @2024-12-25"
  const dateMatch = input.match(/@(tomorrow|today|\d{4}-\d{2}-\d{2})/i);
  if (dateMatch) {
    if (dateMatch[1] === 'tomorrow') {
      todo.dueDate = addDays(new Date(), 1);
    } else if (dateMatch[1] === 'today') {
      todo.dueDate = new Date();
    } else {
      todo.dueDate = parseISO(dateMatch[1]);
    }
    todo.title = todo.title.replace(dateMatch[0], '').trim();
  }
  
  // Parse priority: "task name !high" or "task name !!"
  const priorityMatch = input.match(/!{1,3}/);
  if (priorityMatch) {
    const level = priorityMatch[0].length;
    todo.priority = level === 1 ? 'low' : level === 2 ? 'medium' : 'high';
    todo.title = todo.title.replace(priorityMatch[0], '').trim();
  }
  
  // Parse tags: "task name #work #urgent"
  const tagMatches = input.match(/#(\w+)/g);
  if (tagMatches) {
    // Extract tag names, will be matched to existing tags or created
    todo.tagNames = tagMatches.map(t => t.substring(1));
    todo.title = todo.title.replace(/#\w+/g, '').trim();
  }
  
  return todo;
}
```

#### Task Filtering Logic
```typescript
function filterTasks(todos: Todo[], filters: TodoFilters): Todo[] {
  let filtered = [...todos];
  
  // Filter by view
  if (filters.view === 'inbox') {
    filtered = filtered.filter(t => !t.listId && !t.projectId);
  } else if (filters.view === 'today') {
    const today = startOfDay(new Date());
    filtered = filtered.filter(t => 
      t.dueDate && isSameDay(t.dueDate, today)
    );
  } else if (filters.view === 'upcoming') {
    const today = startOfDay(new Date());
    filtered = filtered.filter(t => 
      t.dueDate && isAfter(t.dueDate, today)
    );
  }
  
  // Filter by category
  if (filters.category !== 'both') {
    filtered = filtered.filter(t => t.category === filters.category);
  }
  
  // Filter by completion
  if (filters.hideCompleted) {
    filtered = filtered.filter(t => !t.completed);
  }
  
  // Filter by list
  if (filters.listIds && filters.listIds.length > 0) {
    filtered = filtered.filter(t => 
      t.listId && filters.listIds.includes(t.listId)
    );
  }
  
  // Filter by project
  if (filters.projectIds && filters.projectIds.length > 0) {
    filtered = filtered.filter(t => 
      t.projectId && filters.projectIds.includes(t.projectId)
    );
  }
  
  // Filter by tags
  if (filters.tagIds && filters.tagIds.length > 0) {
    filtered = filtered.filter(t => 
      filters.tagIds.some(tagId => t.tagIds.includes(tagId))
    );
  }
  
  // Search
  if (filters.searchQuery) {
    const query = filters.searchQuery.toLowerCase();
    filtered = filtered.filter(t => 
      t.title.toLowerCase().includes(query) ||
      t.description?.toLowerCase().includes(query)
    );
  }
  
  return filtered;
}
```

#### Task Ordering
```typescript
function sortTasks(todos: Todo[], sortBy: SortOption): Todo[] {
  const sorted = [...todos];
  
  switch (sortBy) {
    case 'dueDate':
      return sorted.sort((a, b) => {
        if (!a.dueDate && !b.dueDate) return 0;
        if (!a.dueDate) return 1;
        if (!b.dueDate) return -1;
        return a.dueDate.getTime() - b.dueDate.getTime();
      });
    
    case 'priority':
      const priorityOrder = { high: 3, medium: 2, low: 1, null: 0 };
      return sorted.sort((a, b) => 
        priorityOrder[b.priority || 'null'] - priorityOrder[a.priority || 'null']
      );
    
    case 'createdAt':
      return sorted.sort((a, b) => 
        b.createdAt.getTime() - a.createdAt.getTime()
      );
    
    case 'order':
      return sorted.sort((a, b) => a.order - b.order);
    
    default:
      return sorted;
  }
}
```

#### Project Progress Calculation
```typescript
function calculateProjectProgress(projectId: string): ProjectProgress {
  const todos = getTodosByProject(projectId);
  const total = todos.length;
  const completed = todos.filter(t => t.completed).length;
  const percentage = total > 0 ? (completed / total) * 100 : 0;
  
  return {
    projectId,
    total,
    completed,
    percentage,
    remaining: total - completed
  };
}
```

---

### User Flows

#### Primary Flow: Quick Add Task
1. User focuses on "Quick add todo..." input
2. Types: "Review design mockups @tomorrow !!"
3. Presses Enter
4. Task created with:
   - Title: "Review design mockups"
   - Due date: Tomorrow
   - Priority: Medium (!!)
5. Task appears in Today view (if due tomorrow)
6. Input clears, ready for next task

#### Alternative Flow: Create Detailed Task
1. User clicks "+" button or "New Task"
2. Modal opens with full form
3. User fills:
   - Title: "Complete project proposal"
   - Description: "Include budget and timeline"
   - Due date: Selects from date picker
   - List: Selects "Work"
   - Project: Selects "Q4 Planning"
   - Tags: Adds "urgent", "client-facing"
   - Priority: Selects "High"
4. Clicks "Create"
5. Modal closes, task appears in Overview
6. Task also appears in selected List and Project

#### Alternative Flow: Complete Task
1. User sees task in Today view
2. Clicks checkbox
3. Task animates with strikethrough
4. Task moves to bottom of list (or hidden if filter active)
5. Project progress updates if task was in project
6. Toast: "Task completed!"

#### Alternative Flow: Filter by Category
1. User clicks "Work" category filter
2. Only work tasks displayed
3. Personal tasks hidden
4. Filter persists across view switches
5. Clear filter by clicking "Both"

#### Error Flow: Validation Error
1. User tries to create task with empty title
2. Form shows error: "Title is required"
3. Title field highlighted in red
4. Submit button disabled
5. User enters title, error clears

---

### API/Function Specifications

#### CRUD Operations

**createTodo(todo: CreateTodoInput): Promise<Todo>**
```typescript
interface CreateTodoInput {
  title: string;
  description?: string;
  dueDate?: Date;
  priority?: 'low' | 'medium' | 'high';
  category: 'work' | 'personal';
  listId?: string;
  projectId?: string;
  tagIds?: string[];
  order?: number;
}

// Returns: Created todo with generated ID
// Throws: ValidationError if title missing
// Side effect: Updates project progress if projectId provided
```

**quickAddTodo(input: string, defaults?: Partial<Todo>): Promise<Todo>**
```typescript
// Parses quick add input, creates task with parsed + default values
// Returns: Created todo
// Throws: ValidationError if parsing fails
```

**getTodo(id: string): Promise<Todo | null>**
```typescript
// Returns: Todo or null if not found
// Includes: Related list, project, tags
```

**updateTodo(id: string, updates: Partial<Todo>): Promise<Todo>**
```typescript
// Updates specified fields
// Returns: Updated todo
// Side effect: Recalculates project progress if projectId changed
// Throws: NotFoundError if todo doesn't exist
```

**deleteTodo(id: string, hardDelete?: boolean): Promise<void>**
```typescript
// If hardDelete=false: Sets archived=true
// If hardDelete=true: Permanently deletes
// Side effect: Updates project progress
```

**toggleTodoCompletion(id: string): Promise<Todo>**
```typescript
// Toggles completed status
// Sets completedAt if completing, clears if uncompleting
// Returns: Updated todo
```

#### Query Operations

**getTodos(filters: TodoFilters): Promise<Todo[]>**
```typescript
interface TodoFilters {
  view?: 'overview' | 'inbox' | 'today' | 'upcoming';
  category?: 'both' | 'work' | 'personal';
  listIds?: string[];
  projectIds?: string[];
  tagIds?: string[];
  completed?: boolean;
  hideCompleted?: boolean;
  searchQuery?: string;
  sortBy?: 'dueDate' | 'priority' | 'createdAt' | 'order';
  limit?: number;
}

// Returns: Filtered and sorted todos
```

**getTodosByList(listId: string): Promise<Todo[]>**
```typescript
// Returns: All todos in list, sorted by order
```

**getTodosByProject(projectId: string): Promise<Todo[]>**
```typescript
// Returns: All todos in project, sorted by order
// Includes: Completion status for progress calculation
```

**searchTodos(query: string, filters?: TodoFilters): Promise<Todo[]>**
```typescript
// Full-text search across title and description
// Returns: Matching todos ranked by relevance
```

#### Bulk Operations

**bulkUpdateTodos(updates: { id: string; updates: Partial<Todo> }[]): Promise<Todo[]>**
```typescript
// Updates multiple todos in single transaction
// Returns: Array of updated todos
```

**bulkDeleteTodos(ids: string[]): Promise<void>**
```typescript
// Archives or deletes multiple todos
// Side effect: Updates affected project progress
```

**reorderTodos(todoIds: string[], newOrder: number[]): Promise<void>**
```typescript
// Updates order field for multiple todos
// Used for drag-and-drop reordering
```

---

### Technical Requirements

#### Storage Structure (IndexedDB)

**Object Stores**:
1. `todos` - Key: `id`, Indexes: `listId`, `projectId`, `category`, `dueDate`, `completed`, `createdAt`
2. `lists` - Key: `id`, Indexes: `type`
3. `projects` - Key: `id`
4. `tags` - Key: `id`, Index: `name` (unique)

#### Performance Considerations

**Data Volumes**:
- Todos: ~500-2000 per user (reasonable)
- Lists: ~10-50
- Projects: ~5-20
- Tags: ~20-100

**Optimization Strategies**:
1. **Lazy Loading**: Load todos for current view only
2. **Virtual Scrolling**: For large lists (100+ todos)
3. **Debouncing**: Debounce search input (300ms)
4. **Caching**: Cache filtered results, invalidate on changes
5. **Indexed Queries**: Use indexes for fast filtering

#### Offline Support

- All operations work offline
- Use IndexedDB for local storage
- Optimistic UI updates
- Sync indicator

#### Data Migration

**Version 1 → 2** (adding new fields):
```typescript
function migrateTodosV1ToV2(db: IDBDatabase) {
  const transaction = db.transaction(['todos'], 'readwrite');
  const store = transaction.objectStore('todos');
  
  store.openCursor().onsuccess = (event) => {
    const cursor = event.target.result;
    if (cursor) {
      const todo = cursor.value;
      if (!todo.tagIds) todo.tagIds = [];
      if (!todo.order) todo.order = 0;
      cursor.update(todo);
      cursor.continue();
    }
  };
}
```

#### Error Handling

- **Validation Errors**: Inline form errors
- **Storage Errors**: Toast notification with retry
- **Duplicate Tags**: Auto-merge or show warning

#### Testing Scenarios

1. **Quick Add Parsing**: Test all parsing scenarios (@dates, !priority, #tags)
2. **Filtering**: Test all filter combinations
3. **Project Progress**: Test with various completion states
4. **Bulk Operations**: Test with 100+ todos
5. **Search**: Test with special characters, long queries
6. **Date Handling**: Test timezone, DST transitions

---

## 3. PLANNER/SCHEDULING

### Feature Overview

**Purpose**: Visual calendar-based scheduling system for events, appointments, and time blocks with integration of habits and todos.

**Use Cases**:
- Schedule meetings and appointments
- Block time for focused work
- View todos with due dates on calendar
- See habit schedule alongside events
- Plan week/month ahead

**Integration Points**:
- Todos with due dates appear automatically
- Habits scheduled for specific days shown
- Events can be linked to Lists for organization
- Can export calendar data

---

### UI/UX Specifications

#### Layout Structure (Week View)
```
┌─────────────────────────────────────────────────────────┐
│ ← Today →    Dec 15 - Dec 21 [This week]    [Day] [Week] ⚙ │
│ ─────────────────────────────────────────────────────── │
│        Mon 15  Tue 16  Wed 17  Thu 18  Fri 19  Sat 20  Sun 21│
│ All day│      │      │      │      │      │      │      │
│ ───────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│ 11 AM  │      │      │      │      │      │      │      │
│ 12 PM  │      │      │──────│      │      │      │      │
│        │      │      │Event │      │      │      │      │
│ 1 PM   │      │      │──────│      │      │      │      │
│ 2 PM   │      │      │      │      │      │      │      │
│ 3 PM   │      │      │      │      │      │      │      │
│        │      │      │      │      │      │      │      │
└─────────────────────────────────────────────────────────┘
         │
    [My Lists ▼]
    ┌─────────────┐
    │ No lists    │
    │ found.      │
    │             │
    │ Create a    │
    │ list        │
    └─────────────┘
```

#### Components Required
1. **WeekCalendar**: Main week grid with 7 columns (days) and time slots
2. **DayCalendar**: Single day view with hourly slots
3. **EventCard**: Draggable card showing event title, time, color
4. **EventForm**: Modal/form for creating/editing events
5. **TimeIndicator**: Red horizontal line showing current time
6. **DateNavigator**: Left/right arrows, "Today" button, date range display
7. **ViewToggle**: Day/Week toggle buttons
8. **ListSidebar**: Right sidebar with "My Lists" section
9. **TimeSlot**: Clickable time slot for creating events
10. **AllDaySection**: Section above time slots for all-day events

#### Visual States

**Empty State**:
- Empty calendar grid with time slots
- Click any time slot to create event
- Hover shows "+ Add event" tooltip

**Loading State**:
- Skeleton loaders for calendar grid
- Placeholder event cards

**Event States**:
- **Scheduled**: Colored card with title and time
- **All-day**: Bar at top of day column
- **Multi-day**: Spans across multiple columns
- **Hover**: Slight elevation, shows drag handle
- **Dragging**: Semi-transparent, follows cursor

#### Interactions
- **Click time slot**: Opens event form with pre-filled time
- **Click event**: Opens edit modal
- **Drag event**: Move to different time/day
- **Resize event**: Drag top/bottom edge to change duration
- **Click date arrows**: Navigate to previous/next week
- **Click "Today"**: Jump to current week
- **Click view toggle**: Switch between Day/Week views
- **Keyboard shortcuts**:
  - `N` - New event
  - `←`/`→` - Navigate weeks
  - `T` - Jump to today
  - `D`/`W` - Switch Day/Week view

#### Responsive Behavior
- **Desktop**: Full week view, sidebar visible
- **Tablet**: Week view, collapsible sidebar
- **Mobile**: Day view default, swipe for navigation

#### Color Scheme
- **Current day**: Red circle border (#EF4444)
- **Current time**: Red horizontal line (#EF4444)
- **Event colors**: Based on list color or category
- **Time slots**: Light grey background (#F3F4F6)
- **Hour markers**: Dark grey (#6B7280)
- **Weekend**: Slightly different background (#FAFAFA)

#### Animations
- Event creation: Slide in from clicked time slot
- Event drag: Smooth movement, snap to time slots
- View switch: Fade transition
- Time indicator: Smooth movement every minute

---

### Data Model Details

#### Complete Schema

**Event Table**:
```typescript
interface Event {
  id: string;                    // UUID, primary key
  title: string;                 // Required, max 200 chars
  description?: string;          // Optional, max 2000 chars
  date: Date;                    // Date of event (date only)
  startTime?: Date;              // Start time (datetime), null for all-day
  endTime?: Date;                // End time (datetime), null for all-day
  allDay: boolean;               // Default false
  listId?: string;               // Foreign key to List
  color?: string;                // Hex color, defaults to list color
  location?: string;             // Optional location
  recurrence?: RecurrenceRule;   // Optional recurrence
  createdAt: Date;               // Auto-generated
  updatedAt: Date;               // Auto-updated
}

interface RecurrenceRule {
  frequency: 'daily' | 'weekly' | 'monthly' | 'yearly';
  interval: number;              // Every N days/weeks/months/years
  endDate?: Date;                // Optional end date
  count?: number;                // Optional number of occurrences
  daysOfWeek?: number[];         // For weekly: 0-6 (Sun-Sat)
  dayOfMonth?: number;          // For monthly: 1-31
}
```

#### Relationships
- Event → List (many-to-one, optional)
- List → Events (one-to-many)

#### Validation Rules
- **Title**: Required, 1-200 characters
- **Date**: Required, valid date
- **Start time**: Required if not all-day, must be before end time
- **End time**: Required if not all-day, must be after start time
- **Recurrence**: If provided, must have valid frequency and interval

#### Indexes
- `Event(date)` - For date range queries
- `Event(startTime)` - For time-based sorting
- `Event(listId)` - For filtering by list

#### Default Values
- `allDay`: false
- `color`: Based on list color or default (#3B82F6)

---

### Business Logic & Algorithms

#### Get Events for Date Range
```typescript
function getEventsForRange(startDate: Date, endDate: Date): Event[] {
  const events = getAllEvents();
  
  return events.filter(event => {
    const eventDate = startOfDay(event.date);
    const rangeStart = startOfDay(startDate);
    const rangeEnd = startOfDay(endDate);
    
    // Check if event falls within range
    if (isWithinRange(eventDate, rangeStart, rangeEnd)) {
      return true;
    }
    
    // Check recurring events
    if (event.recurrence) {
      return checkRecurrence(event, startDate, endDate);
    }
    
    return false;
  });
}
```

#### Check Recurrence
```typescript
function checkRecurrence(event: Event, startDate: Date, endDate: Date): boolean {
  const rule = event.recurrence;
  if (!rule) return false;
  
  let current = event.date;
  const occurrences: Date[] = [];
  
  while (!isAfter(current, endDate)) {
    if (isAfter(current, startDate) || isSameDay(current, startDate)) {
      occurrences.push(current);
    }
    
    // Calculate next occurrence
    if (rule.frequency === 'daily') {
      current = addDays(current, rule.interval);
    } else if (rule.frequency === 'weekly') {
      current = addWeeks(current, rule.interval);
    } else if (rule.frequency === 'monthly') {
      current = addMonths(current, rule.interval);
    } else if (rule.frequency === 'yearly') {
      current = addYears(current, rule.interval);
    }
    
    // Check end conditions
    if (rule.endDate && isAfter(current, rule.endDate)) break;
    if (rule.count && occurrences.length >= rule.count) break;
  }
  
  return occurrences.length > 0;
}
```

#### Calculate Event Position in Grid
```typescript
function calculateEventPosition(event: Event, dayStart: number): EventPosition {
  if (event.allDay) {
    return {
      type: 'all-day',
      dayIndex: getDayIndex(event.date),
      row: 0
    };
  }
  
  const startHour = event.startTime.getHours();
  const startMinute = event.startTime.getMinutes();
  const endHour = event.endTime.getHours();
  const endMinute = event.endTime.getMinutes();
  
  const startSlot = (startHour - dayStart) * 2 + (startMinute >= 30 ? 1 : 0);
  const duration = ((endHour - startHour) * 60 + (endMinute - startMinute)) / 30;
  
  return {
    type: 'timed',
    dayIndex: getDayIndex(event.date),
    startSlot,
    duration,
    height: duration * 30 // pixels
  };
}
```

---

### User Flows

#### Primary Flow: Create Event
1. User clicks time slot (e.g., 2 PM on Wednesday)
2. Event form opens with:
   - Date: Pre-filled (Wednesday)
   - Start time: Pre-filled (2:00 PM)
   - End time: Pre-filled (3:00 PM)
3. User enters title: "Team Meeting"
4. Optionally selects list, adds description
5. Clicks "Create"
6. Event appears on calendar at correct time
7. Form closes

#### Alternative Flow: Drag Event
1. User clicks and holds event card
2. Event becomes semi-transparent
3. User drags to new time slot
4. Event snaps to nearest 30-minute interval
5. User releases mouse
6. Event updates with new time
7. Changes saved automatically

#### Alternative Flow: View Todos on Calendar
1. User opens Planner
2. Todos with due dates automatically appear
3. Todos shown as all-day events at top of day
4. Color indicates category (work/personal)
5. Clicking todo opens todo edit modal

---

### API/Function Specifications

#### CRUD Operations

**createEvent(event: CreateEventInput): Promise<Event>**
```typescript
interface CreateEventInput {
  title: string;
  description?: string;
  date: Date;
  startTime?: Date;
  endTime?: Date;
  allDay: boolean;
  listId?: string;
  color?: string;
  location?: string;
  recurrence?: RecurrenceRule;
}

// Returns: Created event
// Throws: ValidationError if times invalid
```

**getEvent(id: string): Promise<Event | null>**
```typescript
// Returns: Event or null
```

**updateEvent(id: string, updates: Partial<Event>): Promise<Event>**
```typescript
// Updates event
// Handles recurrence updates
// Returns: Updated event
```

**deleteEvent(id: string, deleteRecurring?: boolean): Promise<void>**
```typescript
// If deleteRecurring=true: Deletes all occurrences
// If deleteRecurring=false: Deletes single occurrence only
```

#### Query Operations

**getEventsForDateRange(startDate: Date, endDate: Date): Promise<Event[]>**
```typescript
// Returns: All events in range, including recurring
// Includes: Todos with due dates, habits scheduled for days
```

**getEventsForDay(date: Date): Promise<Event[]>**
```typescript
// Returns: Events for specific day
// Includes: All-day events, timed events, todos, habits
```

**getTodosForCalendar(startDate: Date, endDate: Date): Promise<CalendarTodo[]>**
```typescript
interface CalendarTodo {
  id: string;
  title: string;
  dueDate: Date;
  category: 'work' | 'personal';
  completed: boolean;
}

// Returns: Todos formatted for calendar display
```

**getHabitsForCalendar(startDate: Date, endDate: Date): Promise<CalendarHabit[]>**
```typescript
interface CalendarHabit {
  id: string;
  name: string;
  scheduledDates: Date[];
  category: 'work' | 'personal';
}

// Returns: Habits with scheduled dates in range
```

---

### Technical Requirements

#### Storage Structure (IndexedDB)

**Object Stores**:
1. `events` - Key: `id`, Indexes: `date`, `startTime`, `listId`

#### Performance Considerations

**Data Volumes**:
- Events: ~100-500 per user
- Recurring events: Generate instances on-the-fly

**Optimization Strategies**:
1. **Lazy Loading**: Load events for visible date range only
2. **Virtual Scrolling**: For month view with many events
3. **Debouncing**: Debounce drag operations
4. **Caching**: Cache date range queries

#### Integration with Todos and Habits

- Todos with due dates appear automatically
- Habits scheduled for days appear as indicators
- All shown in unified calendar view
- Clicking opens respective edit modals

#### Timezone Handling

- Store all times in UTC
- Display in user's local timezone
- Handle DST transitions correctly
- Show timezone indicator in settings

---

## 4. POMODORO TIMER

### Feature Overview

**Purpose**: Focus timer using Pomodoro Technique (25-minute work sessions with breaks) to improve productivity and track focused work time.

**Use Cases**:
- Time-boxed work sessions
- Track daily productivity goals
- Monitor sitting vs standing time during breaks
- Analyze work patterns over time

**Integration Points**:
- Can be linked to specific tasks/projects
- Data feeds into Dashboard widgets
- Statistics available for insights

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Pomodoros                                    ← Today →  │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │                                            │          │
│  │            [Large Timer Display]           │          │
│  │                 25:00                       │          │
│  │                                            │          │
│  │         [Start] [Pause] [Reset]            │          │
│  │                                            │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │ Pomodoros                    🍅🍅🍅🍅🍅🍅🍅🍅│          │
│  │ 0/8                                         │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │ Sitting vs Standing Time                  │          │
│  │ No Pomodoro data today                    │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **TimerDisplay**: Large circular timer showing minutes:seconds
2. **TimerControls**: Start, Pause, Reset buttons
3. **SessionTypeIndicator**: Shows "Focus" or "Break" mode
4. **ProgressWidget**: Circular progress showing completed/target pomodoros
5. **TomatoIcons**: Row of tomato icons (filled for completed)
6. **SittingStandingWidget**: Shows sitting vs standing time during breaks
7. **StatisticsView**: Historical data, charts, trends
8. **SettingsModal**: Configure durations, goals, notifications

#### Visual States

**Idle State**:
- Timer shows default duration (25:00)
- Start button enabled
- No progress indicators active

**Running State**:
- Timer counts down
- Pause button enabled
- Progress ring animates
- Background color changes (subtle)

**Paused State**:
- Timer frozen at current time
- Resume button shown
- Reset button enabled

**Completed State**:
- Timer shows 00:00
- Success animation
- Notification shown
- Auto-transition to break timer

#### Interactions
- **Click Start**: Begins countdown, starts session tracking
- **Click Pause**: Pauses timer, preserves state
- **Click Reset**: Resets to default duration, clears current session
- **Timer completes**: Auto-starts break timer, shows notification
- **Click tomato icon**: View session details
- **Keyboard shortcuts**:
  - `Space` - Start/Pause
  - `R` - Reset
  - `Esc` - Cancel current session

#### Responsive Behavior
- **Desktop**: Large timer display, side widgets
- **Mobile**: Full-screen timer, widgets below

#### Color Scheme
- **Focus timer**: Red/orange (#EF4444)
- **Break timer**: Green (#10B981)
- **Completed pomodoro**: Red tomato icon
- **Incomplete pomodoro**: Grey tomato icon
- **Progress ring**: Gradient based on completion

#### Animations
- Timer countdown: Smooth number transitions
- Progress ring: Animated fill
- Completion: Celebration animation (confetti optional)
- Tomato fill: Scale animation when completed

---

### Data Model Details

#### Complete Schema

**PomodoroSession Table**:
```typescript
interface PomodoroSession {
  id: string;                    // UUID, primary key
  date: Date;                     // Date of session (date only)
  startTime: Date;                // When session started
  endTime?: Date;                 // When session ended (null if ongoing)
  duration: number;               // Duration in minutes (25, 5, 15)
  type: 'focus' | 'short-break' | 'long-break'; // Session type
  completed: boolean;             // Whether completed or cancelled
  taskId?: string;                // Optional link to Todo
  projectId?: string;             // Optional link to Project
  sittingTime?: number;           // Minutes sitting during break
  standingTime?: number;          // Minutes standing during break
  notes?: string;                 // Optional notes
  createdAt: Date;                // Auto-generated
}
```

**PomodoroGoal Table**:
```typescript
interface PomodoroGoal {
  id: string;
  userId: string;                 // For future multi-user support
  date: Date;                     // Date of goal (date only)
  target: number;                 // Target pomodoros (default 8)
  completed: number;              // Completed pomodoros
  createdAt: Date;
  updatedAt: Date;
}
```

#### Relationships
- PomodoroSession → Todo (many-to-one, optional)
- PomodoroSession → Project (many-to-one, optional)

#### Validation Rules
- **Duration**: Must be positive number, typically 5, 15, or 25
- **Type**: Must be one of: focus, short-break, long-break
- **Date**: Must be valid date, not future

#### Indexes
- `PomodoroSession(date)` - For date range queries
- `PomodoroSession(type)` - For filtering by type
- `PomodoroGoal(date)` - For daily goal lookup

#### Default Values
- `duration`: 25 (for focus), 5 (for short break), 15 (for long break)
- `completed`: false
- `target`: 8 (daily goal)

---

### Business Logic & Algorithms

#### Timer State Machine
```typescript
type TimerState = 'idle' | 'running' | 'paused' | 'completed';

function handleTimerAction(state: TimerState, action: TimerAction): TimerState {
  switch (action) {
    case 'start':
      return state === 'idle' || state === 'paused' ? 'running' : state;
    case 'pause':
      return state === 'running' ? 'paused' : state;
    case 'reset':
      return 'idle';
    case 'complete':
      return 'completed';
    default:
      return state;
  }
}
```

#### Break Timer Logic
```typescript
function getNextBreakType(completedFocusSessions: number): 'short-break' | 'long-break' {
  // Every 4 focus sessions, take a long break
  return completedFocusSessions % 4 === 0 ? 'long-break' : 'short-break';
}
```

#### Daily Progress Calculation
```typescript
function calculateDailyProgress(date: Date): PomodoroProgress {
  const goal = getPomodoroGoal(date) || { target: 8, completed: 0 };
  const sessions = getPomodoroSessions(date, { type: 'focus', completed: true });
  
  return {
    date,
    target: goal.target,
    completed: sessions.length,
    percentage: (sessions.length / goal.target) * 100,
    remaining: Math.max(0, goal.target - sessions.length)
  };
}
```

#### Sitting vs Standing Time Tracking
```typescript
function trackActivity(sessionId: string, activity: 'sitting' | 'standing', duration: number): void {
  const session = getPomodoroSession(sessionId);
  if (session.type === 'focus') return; // Only track during breaks
  
  if (activity === 'sitting') {
    session.sittingTime = (session.sittingTime || 0) + duration;
  } else {
    session.standingTime = (session.standingTime || 0) + duration;
  }
  
  updatePomodoroSession(sessionId, session);
}
```

---

### User Flows

#### Primary Flow: Complete Pomodoro Session
1. User clicks "Start" on 25:00 timer
2. Timer begins countdown
3. Progress ring animates
4. After 25 minutes, timer reaches 00:00
5. Notification appears: "Focus session complete! Time for a break"
6. Break timer auto-starts (5:00)
7. Session saved to database
8. Tomato icon fills (if under daily goal)

#### Alternative Flow: Pause and Resume
1. User starts timer
2. Clicks "Pause" after 10 minutes
3. Timer freezes at 15:00
4. User clicks "Resume"
5. Timer continues from 15:00
6. Total session time tracked correctly

#### Alternative Flow: Set Daily Goal
1. User clicks settings icon
2. Opens settings modal
3. Changes "Daily Pomodoro Goal" from 8 to 10
4. Saves settings
5. Progress widget updates to show "0/10"
6. Tomato icons row expands to 10 icons

---

### API/Function Specifications

#### CRUD Operations

**startPomodoroSession(type: 'focus' | 'short-break' | 'long-break', duration?: number): Promise<PomodoroSession>**
```typescript
// Creates new session, starts timer
// Returns: Created session with startTime
// Side effect: Updates daily goal progress
```

**completePomodoroSession(sessionId: string): Promise<PomodoroSession>**
```typescript
// Marks session as completed, sets endTime
// Returns: Updated session
// Side effect: Updates daily goal, starts break timer if focus session
```

**cancelPomodoroSession(sessionId: string): Promise<void>**
```typescript
// Cancels ongoing session
// Deletes session or marks as cancelled
```

**getPomodoroSessions(filters: SessionFilters): Promise<PomodoroSession[]>**
```typescript
interface SessionFilters {
  startDate?: Date;
  endDate?: Date;
  type?: 'focus' | 'short-break' | 'long-break';
  completed?: boolean;
}

// Returns: Filtered sessions
```

**setPomodoroGoal(date: Date, target: number): Promise<PomodoroGoal>**
```typescript
// Sets or updates daily goal
// Returns: Goal object
```

**getPomodoroStatistics(startDate: Date, endDate: Date): Promise<PomodoroStatistics>**
```typescript
interface PomodoroStatistics {
  totalSessions: number;
  totalFocusTime: number;        // Minutes
  averageSessionsPerDay: number;
  longestStreak: number;          // Days
  sittingTime: number;           // Minutes
  standingTime: number;          // Minutes
  completionRate: number;        // Percentage
}

// Returns: Calculated statistics
```

---

### Technical Requirements

#### Storage Structure (IndexedDB)

**Object Stores**:
1. `pomodoroSessions` - Key: `id`, Indexes: `date`, `type`, `completed`
2. `pomodoroGoals` - Key: `id`, Index: `date` (unique)

#### Performance Considerations

**Data Volumes**:
- Sessions: ~8-16 per day × 365 = ~3000-6000 per year
- Goals: ~365 per year (one per day)

**Optimization Strategies**:
1. **Timer Updates**: Use requestAnimationFrame for smooth updates
2. **Background Timer**: Use Web Workers for accurate timing
3. **Notification API**: Browser notifications when tab not active
4. **Local Storage**: Cache current session state

#### Timer Implementation

**Accurate Timing**:
- Use `performance.now()` for high-resolution timing
- Account for tab visibility (pause when hidden)
- Handle system sleep/wake events
- Compensate for timer drift

**Background Execution**:
- Use Service Worker for background timers
- Web Notifications API for alerts
- Wake Lock API to prevent sleep during session

#### Notifications

- **Browser Notification**: When timer completes
- **Sound**: Optional audio alert
- **Visual**: In-app notification banner
- **Settings**: User can enable/disable each type

#### Offline Support

- All timer functionality works offline
- Sessions saved locally
- Sync when online (if multi-device support added)

---

## 5. DASHBOARD & HOME

### Feature Overview

**Purpose**: Central hub for organizing and viewing widgets from different features, customizable dashboard system for personalized productivity overview.

**Use Cases**:
- Create multiple dashboards for different contexts (work, personal, health)
- Add widgets from habits, todos, pomodoros, health tracking
- Quick overview of daily progress
- Customize layout and organization

**Integration Points**:
- Aggregates data from all features
- Widgets pull from respective feature data
- Can navigate to features from widgets

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Select Dashboard                              [⚙ Settings]│
│ ─────────────────────────────────────────────────────── │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │                                            │          │
│  │    [Grid Icon - Large]                    │          │
│  │                                            │          │
│  │    Create or Select a Dashboard            │          │
│  │    Click here to get started               │          │
│  │                                            │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Dashboard with Widgets**:
```
┌─────────────────────────────────────────────────────────┐
│ My Dashboard                    [Edit] [⚙]              │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │ Today's      │  │ Pomodoros   │                    │
│  │ Habits       │  │ 3/8 🍅🍅🍅   │                    │
│  │ ✓ 5/7        │  └──────────────┘                    │
│  └──────────────┘                                       │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │ Today's      │  │ Weight      │                    │
│  │ Todos        │  │ Trend 📈    │                    │
│  │ 3 tasks      │  └──────────────┘                    │
│  └──────────────┘                                       │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **DashboardSelector**: Dropdown/modal for selecting dashboard
2. **DashboardGrid**: Drag-and-drop grid layout for widgets
3. **Widget**: Base widget component with resize, drag, settings
4. **WidgetLibrary**: Modal showing available widgets to add
5. **EmptyState**: "Create or Select a Dashboard" prompt
6. **DashboardSettings**: Modal for dashboard name, layout options
7. **WidgetSettings**: Per-widget configuration modal

#### Visual States

**Empty State**:
- Large grid icon (centered)
- "Create or Select a Dashboard" text
- "Click here to get started" subtext
- Clickable area opens dashboard selector

**Dashboard States**:
- **View Mode**: Widgets displayed, no drag handles
- **Edit Mode**: Widgets draggable, resize handles visible, add widget button

#### Interactions
- **Click empty area**: Opens dashboard selector
- **Drag widget**: Reorder widgets in grid
- **Resize widget**: Change widget size (small, medium, large)
- **Click widget**: Navigate to feature (if clickable)
- **Click widget settings**: Open widget configuration
- **Click edit**: Enter edit mode
- **Click add widget**: Open widget library

#### Widget Types Available
1. **Habits Widget**: Today's habits, completion count
2. **Todos Widget**: Today's tasks, completion count
3. **Pomodoro Widget**: Daily progress, current session
4. **Weight Widget**: Current weight, trend chart
5. **Hydration Widget**: Daily intake, progress bar
6. **Mood Widget**: Today's mood, weekly chart
7. **Calendar Widget**: Upcoming events, today's schedule

---

### Data Model Details

#### Complete Schema

**Dashboard Table**:
```typescript
interface Dashboard {
  id: string;                    // UUID, primary key
  name: string;                   // Required, max 100 chars
  isDefault: boolean;            // Default dashboard
  layout: DashboardLayout;       // Grid layout configuration
  widgets: DashboardWidget[];    // Array of widget configurations
  createdAt: Date;               // Auto-generated
  updatedAt: Date;               // Auto-updated
}

interface DashboardLayout {
  columns: number;               // Grid columns (default 3)
  gap: number;                   // Gap between widgets (pixels)
  breakpoints: Breakpoint[];     // Responsive breakpoints
}

interface DashboardWidget {
  id: string;                     // Widget instance ID
  type: WidgetType;              // Widget type
  position: GridPosition;        // Grid position
  size: WidgetSize;              // Small, medium, large
  config: WidgetConfig;          // Widget-specific config
  data?: any;                    // Cached widget data
}

interface GridPosition {
  x: number;                     // Grid column
  y: number;                     // Grid row
  w: number;                     // Width in grid units
  h: number;                     // Height in grid units
}

type WidgetSize = 'small' | 'medium' | 'large';
type WidgetType = 'habits' | 'todos' | 'pomodoros' | 'weight' | 'hydration' | 'mood' | 'calendar';
```

#### Validation Rules
- **Name**: Required, 1-100 characters
- **Widget positions**: Must not overlap, must fit in grid
- **Widget type**: Must be valid widget type

#### Indexes
- `Dashboard(isDefault)` - For finding default dashboard
- `Dashboard(name)` - For searching dashboards

---

### Business Logic & Algorithms

#### Grid Layout Algorithm
```typescript
function calculateGridLayout(widgets: DashboardWidget[], columns: number): DashboardWidget[] {
  // Sort widgets by position (top to bottom, left to right)
  const sorted = [...widgets].sort((a, b) => {
    if (a.position.y !== b.position.y) return a.position.y - b.position.y;
    return a.position.x - b.position.x;
  });
  
  // Check for overlaps and adjust
  const placed: DashboardWidget[] = [];
  for (const widget of sorted) {
    let position = widget.position;
    
    // Find first available position
    while (hasOverlap(position, placed)) {
      position = findNextPosition(position, columns);
    }
    
    widget.position = position;
    placed.push(widget);
  }
  
  return placed;
}
```

#### Widget Data Fetching
```typescript
async function fetchWidgetData(widget: DashboardWidget): Promise<any> {
  switch (widget.type) {
    case 'habits':
      return getTodayHabitsSummary();
    case 'todos':
      return getTodayTodosSummary();
    case 'pomodoros':
      return getPomodoroProgress();
    case 'weight':
      return getWeightTrend(7); // Last 7 days
    // ... other widget types
  }
}
```

---

### User Flows

#### Primary Flow: Create Dashboard
1. User clicks empty state area
2. Modal opens: "Create New Dashboard"
3. User enters name: "Work Dashboard"
4. Clicks "Create"
5. Dashboard created, enters edit mode
6. Widget library opens automatically
7. User adds widgets (Habits, Todos, Pomodoros)
8. Widgets appear on dashboard
9. User clicks "Done" to exit edit mode

#### Alternative Flow: Add Widget
1. User clicks "Add Widget" button
2. Widget library modal opens
3. Shows available widgets with previews
4. User clicks "Habits" widget
5. Widget added to dashboard
6. Widget settings modal opens
7. User configures widget (show completed, list filter)
8. Settings saved, widget displays data

---

### API/Function Specifications

**createDashboard(name: string, isDefault?: boolean): Promise<Dashboard>**
```typescript
// Creates new dashboard with default layout
// Returns: Created dashboard
```

**getDashboard(id: string): Promise<Dashboard | null>**
```typescript
// Returns: Dashboard with widgets and data
```

**updateDashboard(id: string, updates: Partial<Dashboard>): Promise<Dashboard>**
```typescript
// Updates dashboard configuration
// Returns: Updated dashboard
```

**addWidgetToDashboard(dashboardId: string, widget: Omit<DashboardWidget, 'id'>): Promise<DashboardWidget>**
```typescript
// Adds widget to dashboard
// Returns: Created widget with ID
```

**updateWidgetPosition(dashboardId: string, widgetId: string, position: GridPosition): Promise<void>**
```typescript
// Updates widget position (drag and drop)
```

**refreshWidgetData(widgetId: string): Promise<any>**
```typescript
// Fetches fresh data for widget
// Returns: Widget data
```

---

### Technical Requirements

#### Storage Structure (IndexedDB)

**Object Stores**:
1. `dashboards` - Key: `id`, Indexes: `isDefault`, `name`
2. `dashboardWidgets` - Key: `id`, Index: `dashboardId`

#### Performance Considerations

- **Widget Data Caching**: Cache widget data, refresh on demand
- **Lazy Loading**: Load widget data only when dashboard visible
- **Debouncing**: Debounce drag operations
- **Virtual Scrolling**: For dashboards with many widgets

#### Responsive Grid

- **Desktop**: 3-4 columns
- **Tablet**: 2 columns
- **Mobile**: 1 column
- Widgets automatically resize based on breakpoints

---

## 6. TIMELINE (SOCIAL FEED)

### Feature Overview

**Purpose**: Social feed for sharing achievements (weight, workouts) and viewing community activity with leaderboard gamification. Note: For local app, this can be simplified to personal timeline only or made optional.

**Use Cases**:
- Share weight progress
- Share workout achievements
- View personal activity history
- Compete on leaderboard (if multi-user)
- Get motivation from community

**Integration Points**:
- Auto-creates posts from weight logs
- Auto-creates posts from workout logs
- Links to Weight and Workout features

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Timeline                    [Leaderboard]                │
│ ─────────────────────────────────────────────────────── │
│ Post something...                                        │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ ┌──────────────────────────────────────────┐          │
│ │ [Avatar] @username  Dec 15                │          │
│ │ ⬆ 91.8kg                                  │          │
│ │ gained 0.3kg                              │          │
│ │ ❤ 0  💬 0                                 │          │
│ └──────────────────────────────────────────┘          │
│                                                          │
│ ┌──────────────────────────────────────────┐          │
│ │ [Avatar] @username  Dec 13                │          │
│ │ 🏋️ Chest, Legs, Core, Arms, Shoulders    │          │
│ │ ⏱ 35 mins                                  │          │
│ │ • 4 x Dumbbell Bench Press                │          │
│ │ • 3 x Standing Calf Raise                 │          │
│ │ ❤ 2  💬 1                                 │          │
│ └──────────────────────────────────────────┘          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Leaderboard Sidebar**:
```
┌──────────────────────────────────────────┐
│ 🔥 Leaderboard                    [?]    │
│ ─────────────────────────────────────── │
│ [Today] [Week] [Month] [All time]       │
│ ─────────────────────────────────────── │
│ 🥇 1 [Avatar] User Name          🔥 17  │
│ 🥈 2 [Avatar] User Name          🔥 10  │
│ 🥉 3 [Avatar] User Name          🔥 2   │
│    4 [Avatar] User Name          🔥 2   │
│    5 [Avatar] Your Name          🔥 1   │
└──────────────────────────────────────────┘
```

#### Components Required
1. **PostInput**: "Post something..." input field
2. **PostCard**: Displays post with avatar, content, interactions
3. **PostTypeIndicator**: Icon showing post type (weight, workout, general)
4. **LikeButton**: Heart icon with count, toggle on click
5. **CommentButton**: Comment icon with count, opens comments
6. **LeaderboardWidget**: Right sidebar with rankings
7. **TimeframeTabs**: Today/Week/Month/All time tabs
8. **MedalIcons**: Gold/silver/bronze medals for top 3

#### Data Models

**Post Table**:
```typescript
interface Post {
  id: string;
  userId: string;
  type: 'weight' | 'workout' | 'general';
  content: PostContent;
  date: Date;
  likeCount: number;
  commentCount: number;
  createdAt: Date;
}

interface WeightPostContent {
  weight: number;
  unit: 'kg' | 'lbs';
  change?: number;              // Weight change
  changeType?: 'gain' | 'loss';
}

interface WorkoutPostContent {
  muscleGroups: string[];
  duration: number;             // Minutes
  exercises: ExerciseSummary[];
}

interface PostContent extends WeightPostContent, WorkoutPostContent {
  text?: string;                // For general posts
}
```

**Like Table**:
```typescript
interface Like {
  id: string;
  postId: string;
  userId: string;
  createdAt: Date;
}
```

**Comment Table**:
```typescript
interface Comment {
  id: string;
  postId: string;
  userId: string;
  content: string;
  createdAt: Date;
}
```

**LeaderboardEntry Table**:
```typescript
interface LeaderboardEntry {
  userId: string;
  score: number;                // Flame/points
  rank: number;
  timeframe: 'today' | 'week' | 'month' | 'all-time';
  date: Date;                   // For caching
}
```

#### Business Logic

**Score Calculation**:
```typescript
function calculateScore(userId: string, timeframe: Timeframe): number {
  const startDate = getTimeframeStart(timeframe);
  const posts = getPostsByUser(userId, startDate);
  
  let score = 0;
  for (const post of posts) {
    if (post.type === 'weight') score += 1;
    if (post.type === 'workout') score += 2; // Workouts worth more
    score += post.likeCount * 0.5;           // Likes add points
  }
  
  return Math.floor(score);
}
```

---

### API/Function Specifications

**createPost(post: CreatePostInput): Promise<Post>**
```typescript
interface CreatePostInput {
  type: 'weight' | 'workout' | 'general';
  content: PostContent;
}

// Auto-creates from weight/workout logs
// Returns: Created post
```

**getTimeline(filters?: TimelineFilters): Promise<Post[]>**
```typescript
interface TimelineFilters {
  userId?: string;
  type?: PostType;
  startDate?: Date;
  limit?: number;
}

// Returns: Posts in reverse chronological order
```

**toggleLike(postId: string): Promise<Like | null>**
```typescript
// Toggles like, creates or deletes
// Returns: Like if created, null if deleted
// Side effect: Updates post likeCount
```

**getLeaderboard(timeframe: Timeframe): Promise<LeaderboardEntry[]>**
```typescript
// Returns: Top users ranked by score
// Cached and refreshed periodically
```

---

## 7. FOOD TRACKING

### Feature Overview

**Purpose**: Track daily food intake with macro-nutrient (protein, carbs, fat) and calorie tracking against daily goals.

**Use Cases**:
- Log meals and snacks
- Track macro-nutrient intake
- Monitor calorie consumption
- Set and achieve macro goals
- View daily/weekly nutrition trends

**Integration Points**:
- Can create posts to Timeline
- Data feeds into Dashboard widgets
- Can be part of Routines

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ ← Today →                                    ⚙ [List]   │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ No items found                                           │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │                                            │          │
│  │              [Carrot Icon]                 │          │
│  │                                            │          │
│  │          No food logs                      │          │
│  │    Add a food log to get started           │          │
│  │                                            │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
│                    [Macro Goals ▼]                      │
│  ┌──────────────────────────────────────────┐          │
│  │ Macro goals                               │          │
│  │                                            │          │
│  │ Create a macro goal                        │          │
│  │ No items found                            │          │
│  └──────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **FoodLogForm**: Modal/form for logging food (name, quantity, macros)
2. **FoodLogCard**: Displays food item with macros breakdown
3. **MacroSummary**: Daily totals for protein, carbs, fat, calories
4. **MacroProgressBar**: Visual progress bars for each macro
5. **MacroGoalSidebar**: Right sidebar for setting goals
6. **FoodDatabase**: Searchable database of common foods (optional)
7. **EmptyState**: Carrot icon with prompt

#### Data Models

**FoodLog Table**:
```typescript
interface FoodLog {
  id: string;
  date: Date;                    // Date only
  name: string;                   // Food name
  quantity: number;              // Amount
  unit: string;                   // "g", "oz", "serving", etc.
  protein: number;               // Grams
  carbs: number;                  // Grams
  fat: number;                    // Grams
  calories: number;              // Calories
  meal?: 'breakfast' | 'lunch' | 'dinner' | 'snack';
  notes?: string;
  createdAt: Date;
}
```

**MacroGoal Table**:
```typescript
interface MacroGoal {
  id: string;
  name: string;                   // Goal name (e.g., "Cutting", "Bulking")
  protein: number;               // Target grams
  carbs: number;                  // Target grams
  fat: number;                    // Target grams
  calories: number;              // Target calories
  startDate: Date;
  endDate?: Date;                // Optional end date
  isActive: boolean;             // Currently active goal
  createdAt: Date;
}
```

#### Business Logic

**Daily Macro Calculation**:
```typescript
function calculateDailyMacros(date: Date): MacroSummary {
  const logs = getFoodLogs(date);
  
  return {
    protein: logs.reduce((sum, log) => sum + log.protein, 0),
    carbs: logs.reduce((sum, log) => sum + log.carbs, 0),
    fat: logs.reduce((sum, log) => sum + log.fat, 0),
    calories: logs.reduce((sum, log) => sum + log.calories, 0)
  };
}
```

**Progress Calculation**:
```typescript
function calculateMacroProgress(date: Date, goal: MacroGoal): MacroProgress {
  const macros = calculateDailyMacros(date);
  
  return {
    protein: {
      current: macros.protein,
      target: goal.protein,
      percentage: (macros.protein / goal.protein) * 100
    },
    carbs: {
      current: macros.carbs,
      target: goal.carbs,
      percentage: (macros.carbs / goal.carbs) * 100
    },
    fat: {
      current: macros.fat,
      target: goal.fat,
      percentage: (macros.fat / goal.fat) * 100
    },
    calories: {
      current: macros.calories,
      target: goal.calories,
      percentage: (macros.calories / goal.calories) * 100
    }
  };
}
```

---

### API/Function Specifications

**createFoodLog(log: CreateFoodLogInput): Promise<FoodLog>**
```typescript
interface CreateFoodLogInput {
  date: Date;
  name: string;
  quantity: number;
  unit: string;
  protein: number;
  carbs: number;
  fat: number;
  calories: number;
  meal?: string;
  notes?: string;
}

// Returns: Created food log
```

**getFoodLogs(date: Date): Promise<FoodLog[]>**
```typescript
// Returns: All food logs for date
```

**createMacroGoal(goal: CreateMacroGoalInput): Promise<MacroGoal>**
```typescript
interface CreateMacroGoalInput {
  name: string;
  protein: number;
  carbs: number;
  fat: number;
  calories: number;
  startDate: Date;
  endDate?: Date;
}

// Returns: Created goal
// Side effect: Deactivates other active goals
```

**getMacroProgress(date: Date): Promise<MacroProgress>**
```typescript
// Returns: Current macros vs active goal
```

---

## 8. WEIGHT TRACKING

### Feature Overview

**Purpose**: Track body weight over time with trend visualization and optional weight clubs for community motivation.

**Use Cases**:
- Daily/weekly weight logging
- Track weight trends over time
- Set weight goals
- View progress charts
- Join weight clubs for motivation (optional for local app)

**Integration Points**:
- Can create Timeline posts
- Data feeds into Dashboard widgets
- Can be part of Routines

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Weight                                    [Weight Clubs]│
│ ─────────────────────────────────────────────────────── │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │                                            │          │
│  │            [Scale Icon]                   │          │
│  │                                            │          │
│  │      No weight logs yet                    │          │
│  │ Start tracking your weight journey by     │          │
│  │ adding your first log                      │          │
│  │                                            │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
│                    [Weight Clubs ▼]                     │
│  ┌──────────────────────────────────────────┐          │
│  │ Weight Clubs                        [?]   │          │
│  │                                            │          │
│  │ 50kg: 53.5                                 │          │
│  │ 60kg: 67.7 [Avatar]                       │          │
│  │ 90kg: 91.8 [Avatar]                       │          │
│  │ 100kg: 108, 103 [Avatar]                  │          │
│  │ 110kg: 112.6 [Avatar]                     │          │
│  └──────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **WeightLogForm**: Modal/form for logging weight
2. **WeightChart**: Line chart showing weight trend over time
3. **WeightSummary**: Current weight, change from last, change from start
4. **WeightClubsWidget**: Sidebar showing weight categories
5. **EmptyState**: Scale icon with prompt

#### Data Models

**WeightLog Table**:
```typescript
interface WeightLog {
  id: string;
  date: Date;                    // Date only
  weight: number;                // Weight value
  unit: 'kg' | 'lbs';            // Unit of measurement
  bodyFat?: number;              // Optional body fat %
  muscleMass?: number;           // Optional muscle mass
  notes?: string;
  createdAt: Date;
}
```

**WeightClub Table** (Optional for local app):
```typescript
interface WeightClub {
  id: string;
  name: string;                  // e.g., "50kg", "60kg"
  minWeight: number;             // Minimum weight for club
  maxWeight: number;             // Maximum weight for club
  unit: 'kg' | 'lbs';
  memberCount: number;            // Number of members
}
```

#### Business Logic

**Weight Trend Calculation**:
```typescript
function calculateWeightTrend(logs: WeightLog[]): WeightTrend {
  if (logs.length === 0) return { trend: 'stable', change: 0 };
  
  const sorted = [...logs].sort((a, b) => a.date.getTime() - b.date.getTime());
  const first = sorted[0];
  const last = sorted[sorted.length - 1];
  const change = last.weight - first.weight;
  
  let trend: 'increasing' | 'decreasing' | 'stable';
  if (change > 0.5) trend = 'increasing';
  else if (change < -0.5) trend = 'decreasing';
  else trend = 'stable';
  
  return { trend, change, firstWeight: first.weight, lastWeight: last.weight };
}
```

**Weight Change Calculation**:
```typescript
function calculateWeightChange(current: WeightLog, previous?: WeightLog): WeightChange {
  if (!previous) return { change: 0, percentage: 0, isNew: true };
  
  const change = current.weight - previous.weight;
  const percentage = (change / previous.weight) * 100;
  
  return {
    change,
    percentage,
    isNew: false,
    direction: change > 0 ? 'gain' : change < 0 ? 'loss' : 'stable'
  };
}
```

---

### API/Function Specifications

**createWeightLog(log: CreateWeightLogInput): Promise<WeightLog>**
```typescript
interface CreateWeightLogInput {
  date: Date;
  weight: number;
  unit: 'kg' | 'lbs';
  bodyFat?: number;
  muscleMass?: number;
  notes?: string;
}

// Returns: Created weight log
// Side effect: Can auto-create Timeline post
```

**getWeightLogs(startDate: Date, endDate: Date): Promise<WeightLog[]>**
```typescript
// Returns: Weight logs in date range
```

**getWeightTrend(startDate: Date, endDate: Date): Promise<WeightTrend>**
```typescript
// Returns: Trend analysis
```

**getCurrentWeight(): Promise<WeightLog | null>**
```typescript
// Returns: Most recent weight log
```

---

## 9. HYDRATION TRACKING

### Feature Overview

**Purpose**: Track daily water/fluid intake with customizable goals, templates for quick logging, and calendar visualization.

**Use Cases**:
- Log water intake throughout the day
- Set daily hydration goals
- Use templates for common amounts (glass, bottle, etc.)
- View weekly/monthly intake patterns
- Track hydration consistency

**Integration Points**:
- Data feeds into Dashboard widgets
- Can be part of Routines

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ ← Today →                    [ml] [oz]        ⚙         │
│ ─────────────────────────────────────────────────────── │
│ 0 ml / 2000 ml                                          │
│ ─────────────────────────────────────────────────────── │
│ Hydration log                                           │
│ No items found                                           │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │                                            │          │
│  │          [Water Drop Icon]                │          │          │
│  │                                            │          │
│  │        No hydration logs                    │          │
│  │  Add a hydration log to get started         │          │
│  │                                            │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
│                    [Calendar ▼]                        │
│  ┌──────────────────────────────────────────┐          │
│  │ December 2025                             │          │
│  │ Mo Tu We Th Fr Sa Su                      │          │
│  │  1  2  3  4  5  6  7                      │          │
│  │  8  9 10 11 12 13 14                      │          │
│  │ 15 16[17]18 19 20 21                      │          │
│  │ 22 23 24 25 26 27 28                      │          │
│  │ 29 30 31                                  │          │
│  │                                            │          │
│  │ Templates +                                │          │
│  │ No items found                             │          │
│  └──────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **HydrationLogForm**: Modal/form for logging intake
2. **DailySummary**: Progress bar showing current/goal
3. **UnitToggle**: ml/oz toggle buttons
4. **CalendarWidget**: Right sidebar calendar with activity dots
5. **TemplateList**: List of reusable templates
6. **TemplateForm**: Create/edit template modal
7. **EmptyState**: Water drop icon with prompt

#### Data Models

**HydrationLog Table**:
```typescript
interface HydrationLog {
  id: string;
  date: Date;                    // Date only
  amount: number;                // Amount consumed
  unit: 'ml' | 'oz';             // Unit
  templateId?: string;            // Optional template used
  time?: Date;                    // Optional time of consumption
  notes?: string;
  createdAt: Date;
}
```

**HydrationTemplate Table**:
```typescript
interface HydrationTemplate {
  id: string;
  name: string;                   // e.g., "Glass", "Bottle", "Large Bottle"
  amount: number;                // Amount
  unit: 'ml' | 'oz';             // Unit
  icon?: string;                  // Optional icon
  order: number;                  // For sorting
  createdAt: Date;
}
```

**HydrationGoal Table**:
```typescript
interface HydrationGoal {
  id: string;
  dailyGoal: number;             // Target amount
  unit: 'ml' | 'oz';             // Unit (default: ml, 2000)
  createdAt: Date;
  updatedAt: Date;
}
```

#### Business Logic

**Daily Intake Calculation**:
```typescript
function calculateDailyIntake(date: Date, unit: 'ml' | 'oz'): number {
  const logs = getHydrationLogs(date);
  
  let total = logs.reduce((sum, log) => {
    // Convert to target unit
    const amount = log.unit === unit ? log.amount : convertUnit(log.amount, log.unit, unit);
    return sum + amount;
  }, 0);
  
  return total;
}
```

**Unit Conversion**:
```typescript
function convertUnit(amount: number, from: 'ml' | 'oz', to: 'ml' | 'oz'): number {
  if (from === to) return amount;
  // 1 oz = 29.5735 ml
  if (from === 'oz' && to === 'ml') return amount * 29.5735;
  if (from === 'ml' && to === 'oz') return amount / 29.5735;
  return amount;
}
```

**Progress Calculation**:
```typescript
function calculateHydrationProgress(date: Date): HydrationProgress {
  const goal = getHydrationGoal();
  const intake = calculateDailyIntake(date, goal.unit);
  const percentage = (intake / goal.dailyGoal) * 100;
  
  return {
    current: intake,
    goal: goal.dailyGoal,
    unit: goal.unit,
    percentage: Math.min(100, percentage),
    remaining: Math.max(0, goal.dailyGoal - intake)
  };
}
```

---

### API/Function Specifications

**createHydrationLog(log: CreateHydrationLogInput): Promise<HydrationLog>**
```typescript
interface CreateHydrationLogInput {
  date: Date;
  amount: number;
  unit: 'ml' | 'oz';
  templateId?: string;
  time?: Date;
  notes?: string;
}

// Returns: Created log
```

**quickAddFromTemplate(templateId: string, date?: Date): Promise<HydrationLog>**
```typescript
// Creates log from template
// Uses current date if not provided
```

**createHydrationTemplate(template: CreateTemplateInput): Promise<HydrationTemplate>**
```typescript
interface CreateTemplateInput {
  name: string;
  amount: number;
  unit: 'ml' | 'oz';
  icon?: string;
}

// Returns: Created template
```

**getHydrationProgress(date: Date): Promise<HydrationProgress>**
```typescript
// Returns: Current intake vs goal
```

**setHydrationGoal(goal: number, unit: 'ml' | 'oz'): Promise<HydrationGoal>**
```typescript
// Sets or updates daily goal
// Default: 2000 ml
```

---

## 10. MOOD TRACKING

### Feature Overview

**Purpose**: Track daily mood/emotional state with visualizations to identify patterns and trends over time.

**Use Cases**:
- Log mood multiple times per day
- Track emotional patterns
- View mood trends over weeks/months
- Identify triggers and patterns
- Monitor mental health trends

**Integration Points**:
- Data feeds into Dashboard widgets
- Can be part of Routines

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ ← Today →                                    [+]         │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ No items found                                           │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │                                            │          │
│  │          [Large Circle]                   │          │
│  │        (Mood Chart Placeholder)           │          │
│  │                                            │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Mood Entry Form**:
```
┌──────────────────────────────────────────┐
│ Log Mood                                  │
│ ──────────────────────────────────────── │
│                                            │
│ How are you feeling?                      │
│                                            │
│ [😊] [😐] [😢] [😴] [😡] [😰] [😍]        │
│                                            │
│ Notes (optional)                           │
│ [Text area]                                │
│                                            │
│ [Cancel] [Save]                            │
└──────────────────────────────────────────┘
```

#### Components Required
1. **MoodSelector**: Emoji/scale selector for mood
2. **MoodChart**: Line/area chart showing mood over time
3. **MoodEntryCard**: Displays mood entry with emoji, notes, time
4. **MoodForm**: Modal/form for logging mood
5. **EmptyState**: Large circle placeholder

#### Data Models

**MoodLog Table**:
```typescript
interface MoodLog {
  id: string;
  date: Date;                    // Date only
  time?: Date;                   // Optional time
  mood: MoodValue;               // 1-10 scale or emoji code
  notes?: string;                // Optional notes
  tags?: string[];                // Optional tags (e.g., "work", "family")
  createdAt: Date;
}

type MoodValue = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10;
// Or use emoji codes: 'happy', 'neutral', 'sad', 'tired', 'angry', 'anxious', 'excited'
```

#### Business Logic

**Mood Average Calculation**:
```typescript
function calculateDailyMoodAverage(date: Date): number | null {
  const logs = getMoodLogs(date);
  if (logs.length === 0) return null;
  
  const sum = logs.reduce((acc, log) => acc + log.mood, 0);
  return sum / logs.length;
}
```

**Mood Trend Analysis**:
```typescript
function analyzeMoodTrend(startDate: Date, endDate: Date): MoodTrend {
  const dailyAverages: { date: Date; average: number }[] = [];
  let current = startDate;
  
  while (!isAfter(current, endDate)) {
    const avg = calculateDailyMoodAverage(current);
    if (avg !== null) {
      dailyAverages.push({ date: current, average: avg });
    }
    current = addDays(current, 1);
  }
  
  // Calculate trend
  if (dailyAverages.length < 2) {
    return { trend: 'insufficient-data', data: dailyAverages };
  }
  
  const first = dailyAverages[0].average;
  const last = dailyAverages[dailyAverages.length - 1].average;
  const change = last - first;
  
  let trend: 'improving' | 'declining' | 'stable';
  if (change > 1) trend = 'improving';
  else if (change < -1) trend = 'declining';
  else trend = 'stable';
  
  return { trend, change, data: dailyAverages };
}
```

---

### API/Function Specifications

**createMoodLog(log: CreateMoodLogInput): Promise<MoodLog>**
```typescript
interface CreateMoodLogInput {
  date: Date;
  time?: Date;
  mood: MoodValue;
  notes?: string;
  tags?: string[];
}

// Returns: Created mood log
```

**getMoodLogs(date: Date): Promise<MoodLog[]>**
```typescript
// Returns: All mood logs for date
```

**getMoodTrend(startDate: Date, endDate: Date): Promise<MoodTrend>**
```typescript
// Returns: Trend analysis with daily averages
```

**getMoodStatistics(startDate: Date, endDate: Date): Promise<MoodStatistics>**
```typescript
interface MoodStatistics {
  averageMood: number;
  mostCommonMood: MoodValue;
  entriesLogged: number;
  daysWithEntries: number;
  trend: 'improving' | 'declining' | 'stable';
}

// Returns: Statistical summary
```

---

## 11. ROUTINES

### Feature Overview

**Purpose**: Create and execute multi-step routines combining habits, todos, and other actions into repeatable sequences.

**Use Cases**:
- Morning routine (meditation, exercise, breakfast)
- Evening routine (journal, plan tomorrow, wind down)
- Work routine (check emails, standup, deep work)
- Exercise routine (warmup, workout, cooldown)

**Integration Points**:
- Combines Habits, Todos, and other features
- Can be scheduled or triggered manually
- Items can be checked off as routine executes

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Routines                              [🔽] [+]          │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │                                            │          │
│  │        [Refresh/Loop Icon]                │          │
│  │                                            │          │
│  │        No routines yet                     │          │
│  │  Create your first routine to get started │          │
│  │                                            │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Routine Card**:
```
┌──────────────────────────────────────────┐
│ Morning Routine              [▶ Start]   │
│ ─────────────────────────────────────── │
│ ☐ Meditate (10 min)                      │
│ ☐ Exercise                               │
│ ☐ Healthy Breakfast                      │
│ ☐ Review Today's Goals                   │
│                                            │
│ Progress: 0/4                             │
└──────────────────────────────────────────┘
```

#### Components Required
1. **RoutineForm**: Modal/form for creating/editing routines
2. **RoutineCard**: Displays routine with items and progress
3. **RoutineItemEditor**: Add/edit items in routine
4. **RoutineExecutor**: Active routine view with checkboxes
5. **EmptyState**: Refresh icon with prompt

#### Data Models

**Routine Table**:
```typescript
interface Routine {
  id: string;
  name: string;                   // Required, max 100 chars
  description?: string;           // Optional
  items: RoutineItem[];           // Array of routine items
  schedule?: RoutineSchedule;     // Optional scheduling
  color?: string;                 // Optional color
  icon?: string;                  // Optional icon
  createdAt: Date;
  updatedAt: Date;
}

interface RoutineItem {
  id: string;
  type: 'habit' | 'todo' | 'custom'; // Item type
  itemId?: string;                // ID of habit/todo if linked
  name: string;                   // Display name
  description?: string;           // Optional description
  order: number;                  // Order in routine
  estimatedDuration?: number;     // Minutes
  completed: boolean;            // Current completion status
}

interface RoutineSchedule {
  type: 'manual' | 'daily' | 'weekly' | 'custom';
  time?: string;                 // Time of day (HH:mm)
  daysOfWeek?: number[];         // For weekly/custom
  trigger?: 'wake' | 'sleep' | 'custom'; // Auto-trigger
}
```

#### Business Logic

**Routine Execution**:
```typescript
function startRoutine(routineId: string): RoutineExecution {
  const routine = getRoutine(routineId);
  
  // Create execution instance
  const execution: RoutineExecution = {
    id: generateId(),
    routineId,
    startedAt: new Date(),
    items: routine.items.map(item => ({
      ...item,
      completed: false
    })),
    completed: false
  };
  
  saveRoutineExecution(execution);
  return execution;
}
```

**Complete Routine Item**:
```typescript
function completeRoutineItem(executionId: string, itemId: string): void {
  const execution = getRoutineExecution(executionId);
  const item = execution.items.find(i => i.id === itemId);
  
  if (!item) return;
  
  item.completed = true;
  
  // If linked to habit/todo, mark that as complete too
  if (item.type === 'habit' && item.itemId) {
    toggleHabitCompletion(item.itemId, new Date());
  } else if (item.type === 'todo' && item.itemId) {
    toggleTodoCompletion(item.itemId);
  }
  
  // Check if routine complete
  const allComplete = execution.items.every(i => i.completed);
  if (allComplete) {
    execution.completed = true;
    execution.completedAt = new Date();
  }
  
  updateRoutineExecution(execution);
}
```

---

### API/Function Specifications

**createRoutine(routine: CreateRoutineInput): Promise<Routine>**
```typescript
interface CreateRoutineInput {
  name: string;
  description?: string;
  items: Omit<RoutineItem, 'id' | 'completed'>[];
  schedule?: RoutineSchedule;
  color?: string;
  icon?: string;
}

// Returns: Created routine
```

**getRoutine(id: string): Promise<Routine | null>**
```typescript
// Returns: Routine with items
```

**startRoutine(routineId: string): Promise<RoutineExecution>**
```typescript
// Starts routine execution
// Returns: Execution instance
```

**completeRoutineItem(executionId: string, itemId: string): Promise<void>**
```typescript
// Marks item as complete
// Side effect: Updates linked habit/todo if applicable
```

**getRoutineProgress(routineId: string): Promise<RoutineProgress>**
```typescript
interface RoutineProgress {
  routineId: string;
  totalExecutions: number;
  completedExecutions: number;
  averageCompletionRate: number; // Percentage
  lastExecuted?: Date;
}

// Returns: Progress statistics
```

---

## 12. COUNTDOWNS

### Feature Overview

**Purpose**: Create countdown timers for upcoming events, deadlines, or milestones with visual time remaining display.

**Use Cases**:
- Countdown to vacation
- Deadline tracking
- Event anticipation (birthday, anniversary)
- Goal deadline countdown
- Project milestones

**Integration Points**:
- Can link to Goals
- Can link to Trips
- Can create reminders/notifications

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Countdowns                                    [+]        │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ No items found                                           │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Countdown Card**:
```
┌──────────────────────────────────────────┐
│ Vacation to Paris                         │
│                                            │
│ 45 days                                    │
│ 1,080 hours                                │
│ 64,800 minutes                             │
│                                            │
│ Target: Dec 25, 2024                      │
│                                            │
│ [Edit] [Delete]                            │
└──────────────────────────────────────────┘
```

#### Components Required
1. **CountdownForm**: Modal/form for creating countdowns
2. **CountdownCard**: Displays countdown with time remaining
3. **CountdownTimer**: Live updating timer display
4. **EmptyState**: "No items found" message

#### Data Models

**Countdown Table**:
```typescript
interface Countdown {
  id: string;
  name: string;                   // Required, max 100 chars
  targetDate: Date;               // Target date/time
  description?: string;           // Optional description
  color?: string;                 // Optional color
  icon?: string;                  // Optional icon
  completed: boolean;             // True if target date passed
  createdAt: Date;
  updatedAt: Date;
}
```

#### Business Logic

**Time Remaining Calculation**:
```typescript
function calculateTimeRemaining(countdown: Countdown): TimeRemaining {
  const now = new Date();
  const target = countdown.targetDate;
  const diff = target.getTime() - now.getTime();
  
  if (diff <= 0) {
    return {
      expired: true,
      days: 0,
      hours: 0,
      minutes: 0,
      seconds: 0
    };
  }
  
  const days = Math.floor(diff / (1000 * 60 * 60 * 24));
  const hours = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
  const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));
  const seconds = Math.floor((diff % (1000 * 60)) / 1000);
  
  return {
    expired: false,
    days,
    hours,
    minutes,
    seconds,
    totalSeconds: Math.floor(diff / 1000)
  };
}
```

**Auto-Update Timer**:
```typescript
function startCountdownTimer(countdownId: string, callback: (remaining: TimeRemaining) => void): () => void {
  const interval = setInterval(() => {
    const countdown = getCountdown(countdownId);
    if (!countdown) {
      clearInterval(interval);
      return;
    }
    
    const remaining = calculateTimeRemaining(countdown);
    callback(remaining);
    
    if (remaining.expired) {
      clearInterval(interval);
      markCountdownCompleted(countdownId);
    }
  }, 1000); // Update every second
  
  return () => clearInterval(interval); // Return cleanup function
}
```

---

### API/Function Specifications

**createCountdown(countdown: CreateCountdownInput): Promise<Countdown>**
```typescript
interface CreateCountdownInput {
  name: string;
  targetDate: Date;
  description?: string;
  color?: string;
  icon?: string;
}

// Returns: Created countdown
```

**getCountdown(id: string): Promise<Countdown | null>**
```typescript
// Returns: Countdown
```

**updateCountdown(id: string, updates: Partial<Countdown>): Promise<Countdown>**
```typescript
// Updates countdown
// Returns: Updated countdown
```

**deleteCountdown(id: string): Promise<void>**
```typescript
// Deletes countdown
```

**getActiveCountdowns(): Promise<Countdown[]>**
```typescript
// Returns: Countdowns that haven't expired
// Sorted by target date ascending
```

**getExpiredCountdowns(): Promise<Countdown[]>**
```typescript
// Returns: Countdowns that have passed target date
```

---

## 13. FASTING TRACKING

### Feature Overview

**Purpose**: Track intermittent fasting sessions with duration monitoring, progress visualization, and historical calendar view.

**Use Cases**:
- Track intermittent fasting (16:8, 18:6, OMAD, etc.)
- Monitor fasting duration
- View fasting history and patterns
- Set fasting goals
- Optional: See community members fasting (for multi-user)

**Integration Points**:
- Can create Timeline posts
- Data feeds into Dashboard widgets

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Fasting                                    ⚙            │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │                                            │          │
│  │            [Clock Icon]                  │          │
│  │                                            │          │
│  │          Start a fast                      │          │
│  │    Track your fasting progress             │          │
│  │                                            │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
│  [Start a fast] Button (top right)                      │
│                                                          │
│  Currently Fasting                              [?]     │
│  [Avatar] [Avatar] [Avatar]                             │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │ December 2025                             │          │
│  │ [Calendar with fasting days marked]        │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Active Fast View**:
```
┌──────────────────────────────────────────┐
│ Current Fast                               │
│                                            │
│ 14:32:15                                   │
│                                            │
│ Started: 8:00 PM                           │
│ Target: 16 hours (12:00 PM)                 │
│                                            │
│ Progress: ████████░░░░ 87%                 │
│                                            │
│ [End Fast]                                 │
└──────────────────────────────────────────┘
```

#### Components Required
1. **FastingTimer**: Large timer display showing elapsed time
2. **StartFastButton**: Button to start new fast
3. **EndFastButton**: Button to end current fast
4. **FastingCalendar**: Monthly calendar with fasting days marked
5. **FastingHistory**: List of past fasting sessions
6. **EmptyState**: Clock icon with prompt

#### Data Models

**Fast Table**:
```typescript
interface Fast {
  id: string;
  startTime: Date;               // When fast started
  endTime?: Date;                // When fast ended (null if ongoing)
  targetDuration?: number;       // Target duration in hours (e.g., 16)
  type: '16:8' | '18:6' | '20:4' | 'OMAD' | 'custom'; // Fasting type
  status: 'active' | 'completed' | 'cancelled';
  actualDuration?: number;       // Actual duration in hours (if completed)
  notes?: string;               // Optional notes
  createdAt: Date;
  updatedAt: Date;
}
```

#### Business Logic

**Fasting Duration Calculation**:
```typescript
function calculateFastingDuration(fast: Fast): FastingDuration {
  const now = new Date();
  const start = fast.startTime;
  const end = fast.endTime || now;
  
  const diffMs = end.getTime() - start.getTime();
  const hours = diffMs / (1000 * 60 * 60);
  const minutes = (hours % 1) * 60;
  const seconds = (minutes % 1) * 60;
  
  return {
    totalHours: hours,
    hours: Math.floor(hours),
    minutes: Math.floor(minutes),
    seconds: Math.floor(seconds),
    isActive: !fast.endTime
  };
}
```

**Progress Calculation**:
```typescript
function calculateFastingProgress(fast: Fast): FastingProgress {
  if (!fast.targetDuration) {
    return { percentage: 0, remainingHours: null };
  }
  
  const duration = calculateFastingDuration(fast);
  const percentage = Math.min(100, (duration.totalHours / fast.targetDuration) * 100);
  const remainingHours = Math.max(0, fast.targetDuration - duration.totalHours);
  
  return {
    percentage,
    remainingHours,
    targetReached: duration.totalHours >= fast.targetDuration
  };
}
```

---

### API/Function Specifications

**startFast(fast: StartFastInput): Promise<Fast>**
```typescript
interface StartFastInput {
  targetDuration?: number;       // Hours
  type: FastType;
  notes?: string;
}

// Returns: Created fast with startTime
// Side effect: Ends any existing active fast
```

**endFast(fastId: string, notes?: string): Promise<Fast>**
```typescript
// Ends active fast, sets endTime
// Calculates actual duration
// Returns: Updated fast
```

**getActiveFast(): Promise<Fast | null>**
```typescript
// Returns: Currently active fast or null
```

**getFastingHistory(startDate: Date, endDate: Date): Promise<Fast[]>**
```typescript
// Returns: Fasting sessions in date range
// Includes completed and cancelled
```

**getFastingStatistics(startDate: Date, endDate: Date): Promise<FastingStatistics>**
```typescript
interface FastingStatistics {
  totalSessions: number;
  totalHours: number;
  averageDuration: number;
  longestFast: number;           // Hours
  completionRate: number;        // Percentage
}

// Returns: Statistical summary
```

---

## 14. WORKOUTS

### Feature Overview

**Purpose**: Log and track workout sessions with exercise database, sets/reps tracking, and muscle group organization.

**Use Cases**:
- Log completed workouts
- Track exercises, sets, reps, weight
- View workout history
- Plan future workouts
- Track muscle group training frequency

**Integration Points**:
- Can create Timeline posts
- Data feeds into Dashboard widgets
- Can be part of Routines

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Workouts                    [Exercises] [Start Workout] │
│ ─────────────────────────────────────────────────────── │
│ Log completed workout                                    │
│ No items found                                           │
│                                                          │
│                    [Calendar ▼]                        │
│  ┌──────────────────────────────────────────┐          │
│  │ December 2025                             │          │
│  │ [Calendar with workout days marked]        │          │
│  └──────────────────────────────────────────┘          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Workout Form**:
```
┌──────────────────────────────────────────┐
│ Log Workout                                │
│ ──────────────────────────────────────── │
│ Date: [Dec 17, 2024]                      │
│                                            │
│ Exercises:                                 │
│ ┌──────────────────────────────────────┐  │
│ │ Dumbbell Bench Press                 │  │
│ │ Sets: 4  Reps: 10  Weight: 25kg     │  │
│ │ [Add Set] [Remove]                   │  │
│ └──────────────────────────────────────┘  │
│                                            │
│ [+ Add Exercise]                           │
│                                            │
│ Duration: [35] minutes                     │
│ Muscle Groups: Chest, Arms, Shoulders     │
│                                            │
│ [Cancel] [Save]                            │
└──────────────────────────────────────────┘
```

#### Components Required
1. **WorkoutForm**: Modal/form for logging workouts
2. **ExerciseSelector**: Searchable exercise database
3. **ExerciseSetEditor**: Add/edit sets with reps and weight
4. **WorkoutCard**: Displays workout summary
5. **WorkoutCalendar**: Right sidebar calendar
6. **EmptyState**: "No items found" message

#### Data Models

**Workout Table**:
```typescript
interface Workout {
  id: string;
  date: Date;                    // Date only
  startTime?: Date;              // Optional start time
  endTime?: Date;                // Optional end time
  duration: number;              // Duration in minutes
  exercises: WorkoutExercise[];  // Array of exercises
  muscleGroups: string[];         // e.g., ["chest", "arms", "shoulders"]
  notes?: string;                // Optional notes
  createdAt: Date;
  updatedAt: Date;
}

interface WorkoutExercise {
  id: string;
  exerciseId: string;            // Reference to Exercise
  name: string;                  // Exercise name (denormalized)
  sets: ExerciseSet[];           // Array of sets
  notes?: string;               // Exercise-specific notes
  order: number;                 // Order in workout
}

interface ExerciseSet {
  reps: number;                  // Number of repetitions
  weight?: number;               // Weight (optional for bodyweight)
  unit?: 'kg' | 'lbs';          // Weight unit
  duration?: number;             // Duration in seconds (for time-based)
  restTime?: number;             // Rest time in seconds
  completed: boolean;           // Whether set was completed
}
```

**Exercise Table** (Database):
```typescript
interface Exercise {
  id: string;
  name: string;                  // Required, unique
  muscleGroups: string[];        // Primary muscle groups
  secondaryMuscles?: string[];   // Secondary muscles
  equipment?: string[];          // Required equipment
  instructions?: string;         // How to perform
  category: 'strength' | 'cardio' | 'flexibility' | 'other';
  createdAt: Date;
}
```

#### Business Logic

**Workout Duration Calculation**:
```typescript
function calculateWorkoutDuration(workout: Workout): number {
  if (workout.startTime && workout.endTime) {
    const diff = workout.endTime.getTime() - workout.startTime.getTime();
    return Math.floor(diff / (1000 * 60)); // Minutes
  }
  return workout.duration; // Use manual duration
}
```

**Volume Calculation**:
```typescript
function calculateWorkoutVolume(workout: Workout): WorkoutVolume {
  let totalVolume = 0;
  let totalSets = 0;
  let totalReps = 0;
  
  for (const exercise of workout.exercises) {
    for (const set of exercise.sets) {
      if (set.completed && set.weight) {
        const volume = set.reps * set.weight;
        totalVolume += volume;
        totalSets++;
        totalReps += set.reps;
      }
    }
  }
  
  return {
    totalVolume,
    totalSets,
    totalReps,
    averageVolumePerSet: totalSets > 0 ? totalVolume / totalSets : 0
  };
}
```

---

### API/Function Specifications

**createWorkout(workout: CreateWorkoutInput): Promise<Workout>**
```typescript
interface CreateWorkoutInput {
  date: Date;
  startTime?: Date;
  endTime?: Date;
  duration: number;
  exercises: Omit<WorkoutExercise, 'id'>[];
  muscleGroups: string[];
  notes?: string;
}

// Returns: Created workout
// Side effect: Can auto-create Timeline post
```

**getWorkouts(startDate: Date, endDate: Date): Promise<Workout[]>**
```typescript
// Returns: Workouts in date range
```

**getExercise(id: string): Promise<Exercise | null>**
```typescript
// Returns: Exercise from database
```

**searchExercises(query: string, filters?: ExerciseFilters): Promise<Exercise[]>**
```typescript
interface ExerciseFilters {
  muscleGroup?: string;
  equipment?: string;
  category?: string;
}

// Returns: Matching exercises
```

**getWorkoutStatistics(startDate: Date, endDate: Date): Promise<WorkoutStatistics>**
```typescript
interface WorkoutStatistics {
  totalWorkouts: number;
  totalDuration: number;         // Minutes
  totalVolume: number;           // kg or lbs
  averageWorkoutDuration: number;
  muscleGroupFrequency: Record<string, number>; // How many times each muscle group trained
}

// Returns: Statistical summary
```

---

## 15. TRIPS

### Feature Overview

**Purpose**: Plan and organize trips with dates, destinations, and related packing lists.

**Use Cases**:
- Plan upcoming vacations
- Track trip dates and destinations
- Link to packing lists
- View trip history
- Countdown to trips

**Integration Points**:
- Links to Packing Items feature
- Can create Countdowns
- Can be part of Planner calendar

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Trips                        [Cards] [Table]    [+]     │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ No items found                                           │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Trip Card**:
```
┌──────────────────────────────────────────┐
│ 🏖️ Paris Vacation                         │
│                                            │
│ Dec 20 - Dec 27, 2024                     │
│ Paris, France                              │
│                                            │
│ 3 days until departure                     │
│                                            │
│ [View] [Edit] [Delete]                     │
└──────────────────────────────────────────┘
```

#### Components Required
1. **TripForm**: Modal/form for creating/editing trips
2. **TripCard**: Card display with dates, destination, countdown
3. **TripTable**: Table view with sortable columns
4. **ViewToggle**: Cards/Table toggle buttons
5. **EmptyState**: "No items found" message

#### Data Models

**Trip Table**:
```typescript
interface Trip {
  id: string;
  name: string;                   // Required, max 100 chars
  destination: string;             // Required, max 200 chars
  startDate: Date;                // Trip start date
  endDate: Date;                  // Trip end date
  notes?: string;                 // Optional notes
  packingListId?: string;         // Link to packing list
  countdownId?: string;           // Link to countdown
  color?: string;                 // Optional color
  icon?: string;                  // Optional icon
  createdAt: Date;
  updatedAt: Date;
}
```

#### Business Logic

**Trip Duration Calculation**:
```typescript
function calculateTripDuration(trip: Trip): TripDuration {
  const start = startOfDay(trip.startDate);
  const end = startOfDay(trip.endDate);
  const diff = differenceInDays(end, start) + 1; // Inclusive
  
  return {
    days: diff,
    nights: Math.max(0, diff - 1),
    isUpcoming: isAfter(start, new Date()),
    isActive: isWithinInterval(new Date(), { start, end }),
    isPast: isBefore(end, new Date())
  };
}
```

**Days Until Trip**:
```typescript
function calculateDaysUntil(trip: Trip): number | null {
  const now = startOfDay(new Date());
  const start = startOfDay(trip.startDate);
  
  if (isBefore(start, now)) return null; // Trip already started
  
  return differenceInDays(start, now);
}
```

---

### API/Function Specifications

**createTrip(trip: CreateTripInput): Promise<Trip>**
```typescript
interface CreateTripInput {
  name: string;
  destination: string;
  startDate: Date;
  endDate: Date;
  notes?: string;
  color?: string;
  icon?: string;
}

// Returns: Created trip
// Side effect: Can auto-create countdown
```

**getTrip(id: string): Promise<Trip | null>**
```typescript
// Returns: Trip with related data
```

**getTrips(filters?: TripFilters): Promise<Trip[]>**
```typescript
interface TripFilters {
  status?: 'upcoming' | 'active' | 'past';
  startDate?: Date;
  endDate?: Date;
}

// Returns: Filtered trips
```

**updateTrip(id: string, updates: Partial<Trip>): Promise<Trip>**
```typescript
// Updates trip
// Returns: Updated trip
```

**deleteTrip(id: string): Promise<void>**
```typescript
// Deletes trip
// Optionally deletes linked packing list and countdown
```

---

## 16. PACKING ITEMS

### Feature Overview

**Purpose**: Create and manage packing lists for trips with Kanban workflow for organizing items by status (to pack, packing, packed).

**Use Cases**:
- Create packing list for trip
- Organize items by category
- Track packing progress
- Use Kanban board for visual organization
- Check off items as packed

**Integration Points**:
- Links to Trips
- Can be part of Routines

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Packing items              [Cards] [Kanban]      [+]    │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ No items found                                           │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Kanban View**:
```
┌─────────────────────────────────────────────────────────┐
│ Packing items                              [Kanban] [+] │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ [To Pack]        [Packing]        [Packed]             │
│ ──────────       ──────────       ──────────            │
│ ☐ Passport       ☐ Toothbrush     ☑ Clothes            │
│ ☐ Tickets        ☐ Charger        ☑ Shoes              │
│ ☐ Wallet                              ☑ Phone            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **PackingItemForm**: Modal/form for adding items
2. **PackingItemCard**: Card display with checkbox
3. **KanbanBoard**: Three-column Kanban layout
4. **KanbanColumn**: Draggable column (To Pack, Packing, Packed)
5. **ViewToggle**: Cards/Kanban toggle
6. **EmptyState**: "No items found" message

#### Data Models

**PackingItem Table**:
```typescript
interface PackingItem {
  id: string;
  tripId?: string;               // Link to trip (optional)
  name: string;                   // Required, max 200 chars
  category?: string;              // e.g., "clothing", "electronics", "documents"
  status: 'to-pack' | 'packing' | 'packed'; // Kanban status
  quantity: number;               // Default 1
  packed: boolean;                // Whether item is packed
  notes?: string;                 // Optional notes
  order: number;                  // For sorting
  createdAt: Date;
  updatedAt: Date;
}
```

#### Business Logic

**Packing Progress Calculation**:
```typescript
function calculatePackingProgress(items: PackingItem[]): PackingProgress {
  const total = items.length;
  const packed = items.filter(i => i.status === 'packed' || i.packed).length;
  const packing = items.filter(i => i.status === 'packing').length;
  const toPack = items.filter(i => i.status === 'to-pack').length;
  
  return {
    total,
    packed,
    packing,
    toPack,
    percentage: total > 0 ? (packed / total) * 100 : 0
  };
}
```

**Category Grouping**:
```typescript
function groupItemsByCategory(items: PackingItem[]): Record<string, PackingItem[]> {
  return items.reduce((groups, item) => {
    const category = item.category || 'uncategorized';
    if (!groups[category]) {
      groups[category] = [];
    }
    groups[category].push(item);
    return groups;
  }, {} as Record<string, PackingItem[]>);
}
```

---

### API/Function Specifications

**createPackingItem(item: CreatePackingItemInput): Promise<PackingItem>**
```typescript
interface CreatePackingItemInput {
  tripId?: string;
  name: string;
  category?: string;
  status?: 'to-pack' | 'packing' | 'packed';
  quantity?: number;
  notes?: string;
}

// Returns: Created item
```

**getPackingItems(tripId?: string): Promise<PackingItem[]>**
```typescript
// Returns: Items, optionally filtered by trip
```

**updatePackingItemStatus(id: string, status: PackingItemStatus): Promise<PackingItem>**
```typescript
// Updates item status (drag and drop in Kanban)
// Returns: Updated item
```

**togglePackingItem(id: string): Promise<PackingItem>**
```typescript
// Toggles packed status
// Returns: Updated item
```

**getPackingProgress(tripId?: string): Promise<PackingProgress>**
```typescript
// Returns: Progress statistics
```

---

## 17. GOALS

### Feature Overview

**Purpose**: Set and track personal goals with progress monitoring, milestones, and completion tracking.

**Use Cases**:
- Set short-term and long-term goals
- Track progress toward goals
- Set milestones/checkpoints
- View goal completion history
- Link goals to habits/todos for tracking

**Integration Points**:
- Can link to Habits for automatic tracking
- Can link to Todos for task-based goals
- Can create Countdowns for deadline goals
- Data feeds into Dashboard widgets

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ My goals                                      [+]       │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ No items found                                           │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Goal Card**:
```
┌──────────────────────────────────────────┐
│ 🎯 Learn Spanish                          │
│                                            │
│ Progress: ████████░░ 80%                  │
│                                            │
│ Target: Complete 50 lessons               │
│ Current: 40/50 lessons                    │
│                                            │
│ Deadline: Mar 1, 2024                     │
│ 12 days remaining                          │
│                                            │
│ [View Details] [Edit]                     │
└──────────────────────────────────────────┘
```

#### Components Required
1. **GoalForm**: Modal/form for creating/editing goals
2. **GoalCard**: Displays goal with progress bar
3. **ProgressBar**: Visual progress indicator
4. **MilestoneList**: List of goal milestones
5. **EmptyState**: "No items found" message

#### Data Models

**Goal Table**:
```typescript
interface Goal {
  id: string;
  name: string;                   // Required, max 200 chars
  description?: string;           // Optional
  category: 'health' | 'fitness' | 'learning' | 'career' | 'personal' | 'other';
  targetValue: number;            // Target number/value
  currentValue: number;            // Current progress
  unit?: string;                  // Unit of measurement (e.g., "lessons", "kg", "hours")
  targetDate?: Date;              // Optional deadline
  status: 'active' | 'completed' | 'paused' | 'cancelled';
  progress: number;               // Percentage (0-100), calculated
  milestones?: Milestone[];      // Optional milestones
  linkedHabitId?: string;         // Link to habit for auto-tracking
  linkedTodoIds?: string[];       // Link to todos for task-based goals
  createdAt: Date;
  updatedAt: Date;
  completedAt?: Date;            // When goal was completed
}

interface Milestone {
  id: string;
  name: string;                   // Milestone name
  targetValue: number;            // Value at which milestone is reached
  achieved: boolean;              // Whether milestone achieved
  achievedAt?: Date;              // When milestone achieved
}
```

#### Business Logic

**Progress Calculation**:
```typescript
function calculateGoalProgress(goal: Goal): number {
  if (goal.targetValue === 0) return 0;
  
  const progress = (goal.currentValue / goal.targetValue) * 100;
  return Math.min(100, Math.max(0, progress));
}
```

**Check Milestones**:
```typescript
function checkMilestones(goal: Goal): Goal {
  if (!goal.milestones) return goal;
  
  const updatedMilestones = goal.milestones.map(milestone => {
    if (!milestone.achieved && goal.currentValue >= milestone.targetValue) {
      return {
        ...milestone,
        achieved: true,
        achievedAt: new Date()
      };
    }
    return milestone;
  });
  
  goal.milestones = updatedMilestones;
  
  // Check if goal completed
  if (goal.currentValue >= goal.targetValue && goal.status === 'active') {
    goal.status = 'completed';
    goal.completedAt = new Date();
    goal.progress = 100;
  } else {
    goal.progress = calculateGoalProgress(goal);
  }
  
  return goal;
}
```

**Auto-Update from Habit**:
```typescript
function updateGoalFromHabit(goalId: string, habitId: string): void {
  const goal = getGoal(goalId);
  if (!goal || goal.linkedHabitId !== habitId) return;
  
  // Get habit completion count or streak
  const habit = getHabit(habitId);
  const entries = getHabitEntries(habitId);
  const completedCount = entries.filter(e => e.completed).length;
  
  // Update goal current value
  goal.currentValue = completedCount;
  goal = checkMilestones(goal);
  
  updateGoal(goalId, goal);
}
```

---

### API/Function Specifications

**createGoal(goal: CreateGoalInput): Promise<Goal>**
```typescript
interface CreateGoalInput {
  name: string;
  description?: string;
  category: GoalCategory;
  targetValue: number;
  currentValue?: number;          // Default 0
  unit?: string;
  targetDate?: Date;
  milestones?: Omit<Milestone, 'id' | 'achieved' | 'achievedAt'>[];
  linkedHabitId?: string;
  linkedTodoIds?: string[];
}

// Returns: Created goal with calculated progress
```

**getGoal(id: string): Promise<Goal | null>**
```typescript
// Returns: Goal with milestones
```

**updateGoalProgress(id: string, currentValue: number): Promise<Goal>**
```typescript
// Updates current value, recalculates progress
// Checks milestones
// Returns: Updated goal
```

**completeGoal(id: string): Promise<Goal>**
```typescript
// Marks goal as completed
// Sets completedAt
// Returns: Updated goal
```

**getGoals(filters?: GoalFilters): Promise<Goal[]>**
```typescript
interface GoalFilters {
  status?: GoalStatus;
  category?: GoalCategory;
  targetDate?: Date;            // Goals with deadline before/on date
}

// Returns: Filtered goals
```

**getGoalStatistics(): Promise<GoalStatistics>**
```typescript
interface GoalStatistics {
  totalGoals: number;
  activeGoals: number;
  completedGoals: number;
  averageProgress: number;       // Average progress across all active goals
  goalsByCategory: Record<string, number>;
}

// Returns: Statistical summary
```

---

## 18. BACKGROUND MUSIC & AMBIENT SOUNDS

### Feature Overview

**Purpose**: Provide background music (Lofi) and ambient sounds for focus sessions, Pomodoro timers, and general productivity work.

**Use Cases**:
- Play Lofi music for focused work sessions
- Mix multiple ambient sounds (rain, fire, waves, etc.)
- Control master volume and individual sound volumes
- Create custom sound mixes for different work scenarios
- Enhance focus during Pomodoro sessions

**Integration Points**:
- Integrates with Pomodoro timer (auto-play during focus sessions)
- Can be part of Routines
- Can be linked to specific work contexts
- Settings persist across sessions

---

### UI/UX Specifications

#### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│ Sounds                                    [X]            │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ ┌──────────────────────────────────────────┐          │
│ │ [▶] Master Volume              100% [━━━━━━━]│          │
│ └──────────────────────────────────────────┘          │
│                                                          │
│ ┌──────────────────────────────────────────┐          │
│ │ [▶] Lofi  [🔀] [🔊] [━━━━━━━]        [X]   │          │
│ │ [Gradient: Pink to Purple]                  │          │
│ └──────────────────────────────────────────┘          │
│                                                          │
│ Ambient Sounds                          [🔀]            │
│ ─────────────────────────────────────────────────────── │
│                                                          │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                   │
│ │ ☁️🌧️ │ │ 🔥  │ │ 🌊  │ │ 🐦  │                   │
│ │ Rain │ │ Fire │ │Waves │ │Birds │                   │
│ │ [🔊] │ │ [🔊] │ │ [🔊] │ │ [🔊] │                   │
│ │[━━] │ │[━━] │ │[━━] │ │[━━] │                   │
│ └──────┘ └──────┘ └──────┘ └──────┘                   │
│                                                          │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                   │
│ │ 🦗  │ │ ⚡  │ │ 💨  │ │ 🔊  │                   │
│ │Crick │ │Thund│ │ Wind │ │White │                   │
│ │ ets │ │ er  │ │      │ │Noise │                   │
│ │ [🔊] │ │ [🔊] │ │ [🔊] │ │ [🔊] │                   │
│ │[━━] │ │[━━] │ │[━━] │ │[━━] │                   │
│ └──────┘ └──────┘ └──────┘ └──────┘                   │
│                                                          │
│ ┌──────┐                                                 │
│ │ ☕  │                                                 │
│ │Coffe│                                                 │
│ │ e   │                                                 │
│ │ [🔊] │                                                 │
│ │[━━] │                                                 │
│ └──────┘                                                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Components Required
1. **MasterVolumeControl**: Top-level volume slider affecting all sounds
2. **LofiPlayer**: Music player with play/pause, shuffle, volume controls
3. **AmbientSoundGrid**: Grid of ambient sound tiles
4. **AmbientSoundTile**: Individual sound tile with icon, name, volume control
5. **SoundMixPreset**: Save/load custom sound combinations
6. **PlaybackControls**: Play/pause, shuffle, loop controls

#### Visual States

**Idle State**:
- All sounds paused
- Volume sliders at 0% or saved position
- Play buttons show play icon (▶)

**Playing State**:
- Active sounds show pause icon (⏸)
- Volume sliders show current level
- Visual feedback (subtle animation) on active sounds

**Loading State**:
- Loading spinner on sound tiles
- Disabled controls during load

#### Interactions
- **Click play button**: Toggle play/pause for sound
- **Drag volume slider**: Adjust volume (0-100%)
- **Click shuffle**: Randomize ambient sound selection
- **Master volume**: Controls overall output, affects all sounds proportionally
- **Keyboard shortcuts**:
  - `Space` - Play/pause master
  - `↑`/`↓` - Adjust master volume
  - `M` - Mute/unmute master

#### Responsive Behavior
- **Desktop**: Full grid layout, all sounds visible
- **Tablet**: 2-column grid
- **Mobile**: Single column, scrollable list

#### Color Scheme
- **Lofi player**: Pink-to-purple gradient (#FFB6C1 to #9370DB)
- **Ambient sound tiles**: Pastel colors
  - Rain: Light blue (#ADD8E6)
  - Fire: Orange (#FFA500)
  - Waves: Light blue (#87CEEB)
  - Birds: Light green (#90EE90)
  - Crickets: Light green (#98FB98)
  - Thunder: Light purple (#DDA0DD)
  - Wind: Light blue (#B0E0E6)
  - White Noise: Light gray (#D3D3D3)
  - Coffee: Light yellow (#FFFACD)

#### Animations
- Play button: Smooth icon transition (▶ ↔ ⏸)
- Volume slider: Smooth value updates
- Sound tile: Subtle pulse when playing
- Master volume: Visual feedback on change

---

### Data Model Details

#### Complete Schema

**SoundSettings Table**:
```typescript
interface SoundSettings {
  id: string;                    // Single record, use 'default'
  masterVolume: number;          // 0-100, default 100
  lofiEnabled: boolean;          // Default false
  lofiVolume: number;            // 0-100, default 50
  lofiShuffle: boolean;          // Default false
  ambientSounds: AmbientSoundConfig[]; // Array of ambient sound configs
  currentMix?: string;           // Name of current preset mix
  autoPlayWithPomodoro: boolean; // Auto-start with Pomodoro sessions
  createdAt: Date;
  updatedAt: Date;
}

interface AmbientSoundConfig {
  id: string;                    // Sound identifier
  name: string;                   // Display name
  enabled: boolean;              // Whether sound is active
  volume: number;                // 0-100, individual volume
  loop: boolean;                 // Whether to loop sound
  category: 'nature' | 'urban' | 'abstract'; // Sound category
}
```

**SoundMixPreset Table**:
```typescript
interface SoundMixPreset {
  id: string;
  name: string;                  // Preset name (e.g., "Focus", "Relax", "Rainy Day")
  description?: string;           // Optional description
  lofiEnabled: boolean;
  lofiVolume: number;
  ambientSounds: string[];       // Array of enabled sound IDs
  ambientVolumes: Record<string, number>; // Volume for each sound
  masterVolume: number;
  createdAt: Date;
  updatedAt: Date;
}
```

**Available Ambient Sounds** (Predefined):
```typescript
const AMBIENT_SOUNDS: AmbientSound[] = [
  { id: 'rain', name: 'Rain', icon: '☁️🌧️', category: 'nature', color: '#ADD8E6' },
  { id: 'fire', name: 'Fire', icon: '🔥', category: 'nature', color: '#FFA500' },
  { id: 'waves', name: 'Waves', icon: '🌊', category: 'nature', color: '#87CEEB' },
  { id: 'birds', name: 'Birds', icon: '🐦', category: 'nature', color: '#90EE90' },
  { id: 'crickets', name: 'Crickets', icon: '🦗', category: 'nature', color: '#98FB98' },
  { id: 'thunder', name: 'Thunder', icon: '⚡', category: 'nature', color: '#DDA0DD' },
  { id: 'wind', name: 'Wind', icon: '💨', category: 'nature', color: '#B0E0E6' },
  { id: 'white-noise', name: 'White Noise', icon: '🔊', category: 'abstract', color: '#D3D3D3' },
  { id: 'coffee', name: 'Coffee', icon: '☕', category: 'abstract', color: '#FFFACD' }
];
```

#### Relationships
- SoundSettings → SoundMixPreset (many-to-one, optional)
- SoundMixPreset → AmbientSoundConfig (one-to-many via arrays)

#### Validation Rules
- **Master volume**: 0-100, integer
- **Individual volumes**: 0-100, integer
- **Sound IDs**: Must be from predefined list
- **Preset name**: Required, max 50 characters

#### Indexes
- `SoundMixPreset(name)` - For searching presets

#### Default Values
- `masterVolume`: 100
- `lofiEnabled`: false
- `lofiVolume`: 50
- `ambientSounds[].enabled`: false
- `ambientSounds[].volume`: 0
- `autoPlayWithPomodoro`: false

---

### Business Logic & Algorithms

#### Volume Calculation
```typescript
function calculateEffectiveVolume(soundVolume: number, masterVolume: number): number {
  // Effective volume = (soundVolume / 100) * (masterVolume / 100) * 100
  return Math.round((soundVolume * masterVolume) / 100);
}
```

#### Sound Mixing
```typescript
function mixSounds(sounds: AmbientSoundConfig[], masterVolume: number): AudioMix {
  const activeSounds = sounds.filter(s => s.enabled && s.volume > 0);
  
  // Calculate total volume (for normalization if needed)
  const totalVolume = activeSounds.reduce((sum, s) => sum + calculateEffectiveVolume(s.volume, masterVolume), 0);
  
  return {
    sounds: activeSounds.map(s => ({
      id: s.id,
      volume: calculateEffectiveVolume(s.volume, masterVolume),
      loop: s.loop
    })),
    totalVolume,
    normalized: totalVolume > 100 // Flag if normalization needed
  };
}
```

#### Auto-Play with Pomodoro
```typescript
function handlePomodoroStart(sessionId: string): void {
  const settings = getSoundSettings();
  
  if (settings.autoPlayWithPomodoro) {
    // Start last used mix or default mix
    const preset = settings.currentMix 
      ? getSoundMixPreset(settings.currentMix)
      : getDefaultPreset();
    
    if (preset) {
      applySoundMix(preset);
      playAllSounds();
    }
  }
}

function handlePomodoroEnd(sessionId: string): void {
  const settings = getSoundSettings();
  
  if (settings.autoPlayWithPomodoro) {
    // Optionally fade out or stop sounds
    fadeOutSounds(2000); // 2 second fade
  }
}
```

#### Shuffle Ambient Sounds
```typescript
function shuffleAmbientSounds(count: number = 3): AmbientSoundConfig[] {
  const allSounds = AMBIENT_SOUNDS;
  const shuffled = [...allSounds].sort(() => Math.random() - 0.5);
  
  // Select random sounds and set to moderate volume
  return shuffled.slice(0, count).map(sound => ({
    id: sound.id,
    name: sound.name,
    enabled: true,
    volume: 50, // Moderate volume
    loop: true,
    category: sound.category
  }));
}
```

---

### User Flows

#### Primary Flow: Start Focus Session with Sounds
1. User opens Sounds panel (from Pomodoro or directly)
2. Sees current sound settings
3. Clicks play on Lofi player
4. Lofi music starts playing
5. User clicks on "Rain" ambient sound tile
6. Rain sound starts playing, mixes with Lofi
7. User adjusts Rain volume slider to 60%
8. User adjusts master volume to 80%
9. All sounds adjust proportionally
10. User starts Pomodoro session
11. Sounds continue playing (if auto-play enabled)

#### Alternative Flow: Create Custom Mix
1. User configures multiple ambient sounds
2. Sets individual volumes
3. Adjusts master volume
4. Clicks "Save Mix" button
5. Enters name: "Rainy Focus"
6. Mix saved as preset
7. Preset appears in dropdown for quick access

#### Alternative Flow: Shuffle Sounds
1. User clicks shuffle button
2. System randomly selects 3-4 ambient sounds
3. Sets them to moderate volume (50%)
4. Enables selected sounds
5. Disables others
6. Sounds start playing automatically

---

### API/Function Specifications

#### CRUD Operations

**getSoundSettings(): Promise<SoundSettings>**
```typescript
// Returns: Current sound settings (creates default if not exists)
```

**updateSoundSettings(updates: Partial<SoundSettings>): Promise<SoundSettings>**
```typescript
// Updates sound settings
// Returns: Updated settings
```

**playSound(soundId: string): Promise<void>**
```typescript
// Starts playing specific sound
// Creates audio context if needed
```

**stopSound(soundId: string): Promise<void>**
```typescript
// Stops playing specific sound
```

**setSoundVolume(soundId: string, volume: number): Promise<void>**
```typescript
// Sets volume for specific sound (0-100)
// Updates settings
```

**setMasterVolume(volume: number): Promise<void>**
```typescript
// Sets master volume (0-100)
// Updates all active sounds immediately
```

**toggleLofi(): Promise<boolean>**
```typescript
// Toggles Lofi music play/pause
// Returns: New playing state
```

**createSoundMixPreset(preset: CreateSoundMixPresetInput): Promise<SoundMixPreset>**
```typescript
interface CreateSoundMixPresetInput {
  name: string;
  description?: string;
  lofiEnabled: boolean;
  lofiVolume: number;
  ambientSounds: string[];
  ambientVolumes: Record<string, number>;
  masterVolume: number;
}

// Returns: Created preset
```

**applySoundMix(presetId: string): Promise<void>**
```typescript
// Applies preset to current settings
// Starts/stops sounds as needed
```

**getSoundMixPresets(): Promise<SoundMixPreset[]>**
```typescript
// Returns: All saved presets
```

**deleteSoundMixPreset(presetId: string): Promise<void>**
```typescript
// Deletes preset
```

#### Query Operations

**getActiveSounds(): Promise<ActiveSound[]>**
```typescript
interface ActiveSound {
  id: string;
  name: string;
  volume: number;
  effectiveVolume: number; // After master volume applied
  playing: boolean;
}

// Returns: Currently active sounds with calculated volumes
```

**shuffleAmbientSounds(count?: number): Promise<AmbientSoundConfig[]>**
```typescript
// Randomly selects and enables ambient sounds
// Returns: Selected sound configs
```

---

### Technical Requirements

#### Storage Structure (IndexedDB)

**Object Stores**:
1. `soundSettings` - Key: `id` (single record: 'default')
2. `soundMixPresets` - Key: `id`, Index: `name`

#### Audio Implementation

**Web Audio API**:
- Use `AudioContext` for audio playback
- Support multiple simultaneous audio sources
- Volume control via `GainNode`
- Loop support for ambient sounds

**Audio Files**:
- Store audio files locally or use CDN
- Support MP3, OGG, WAV formats
- Preload sounds for instant playback
- Cache audio buffers for performance

**Performance Considerations**:
- **Lazy Loading**: Load audio files on first use
- **Preloading**: Preload commonly used sounds
- **Audio Context**: Reuse single AudioContext instance
- **Volume Normalization**: Prevent clipping when multiple sounds play
- **Memory Management**: Unload unused audio buffers

#### Integration with Pomodoro

**Auto-Play Configuration**:
- User can enable "Auto-play with Pomodoro" in settings
- When Pomodoro starts, last used mix or default mix plays
- When Pomodoro ends, sounds can fade out or continue
- Settings persist across sessions

#### Offline Support

- All audio files stored locally
- No internet required for playback
- Settings saved to IndexedDB
- Works completely offline

#### Browser Compatibility

- **Chrome/Edge**: Full Web Audio API support
- **Firefox**: Full support
- **Safari**: Full support (iOS 14+)
- **Fallback**: Use HTML5 Audio if Web Audio API unavailable

#### Error Handling

- **Audio Load Errors**: Show error message, disable sound
- **Permission Errors**: Request audio permission, show instructions
- **Format Errors**: Try alternative format, show fallback message
- **Performance Issues**: Reduce simultaneous sounds, show warning

#### Testing Scenarios

1. **Volume Mixing**: Test multiple sounds at different volumes
2. **Master Volume**: Test master volume affects all sounds
3. **Auto-Play**: Test Pomodoro integration
4. **Presets**: Test save/load/apply presets
5. **Shuffle**: Test random sound selection
6. **Performance**: Test with 5+ simultaneous sounds
7. **Offline**: Test without internet connection
8. **Browser Compatibility**: Test across browsers

---

## 19. USER INTERFACE & EXPERIENCE

### Design Elements
- **Clean, Modern UI**: Minimalist and intuitive interface
- **Dark/Light Mode**: Theme switching capability
- **Responsive Design**: Works on desktop, tablet, and mobile
- **Visual Feedback**: Clear indicators for completed/incomplete items
- **Smooth Animations**: Polished user interactions

### Navigation
- **Main Dashboard**: Overview of all features
- **Quick Access**: Easy switching between habits, todos, planner, pomodoro
- **Search Functionality**: Search across habits and todos
- **Keyboard Shortcuts**: Power user shortcuts for efficiency

---

## 19. DATA MANAGEMENT

### Data Storage
- **Local Storage**: Data stored locally (for your local app)
- **Data Export**: Export data (CSV, JSON, etc.)
- **Data Import**: Import existing data
- **Backup & Restore**: Backup and restore functionality

### Data Persistence
- **Auto-save**: Automatic saving of changes
- **Offline Support**: Works without internet connection
- **Sync** (if multi-device): Sync across devices (optional for local app)

---

## 20. ADDITIONAL UI ELEMENTS OBSERVED

### Navigation Patterns
- **Persistent Sidebar**: Left sidebar always visible with all features
- **Favorites**: Ability to pin features to favorites section
- **More Menu**: "More" option with ellipsis for additional features
- **Active State**: Highlighted background for currently selected feature

### Common UI Components
- **Empty States**: Consistent empty state design with icons and prompts
- **Add Buttons**: Plus icons for adding new items
- **Filter Icons**: Funnel/filter icons for filtering content
- **Settings Icons**: Gear icons for feature-specific settings
- **Date Navigation**: Consistent left/right arrows for date navigation
- **View Toggles**: Tabs or buttons for switching between view modes
- **Search**: Magnifying glass icons for search functionality

### Visual Design
- **Color Scheme**: Light theme with white backgrounds, grey sidebars
- **Icons**: Simple, monochromatic line icons for each feature
- **Typography**: Clean, readable fonts with clear hierarchy
- **Spacing**: Generous whitespace for clean appearance
- **Cards**: Card-based layouts for organized content display

---

## 21. ADDITIONAL FEATURES (Potential)

### Notifications & Reminders
- **Habit Reminders**: Notifications to complete habits
- **Todo Reminders**: Reminders for upcoming due dates
- **Daily Review**: End-of-day summary/reminder

### Gamification
- **Achievements**: Badges for milestones
- **Levels**: Progress levels based on consistency
- **Rewards**: Unlock features or themes

### Reports & Insights
- **Weekly Reports**: Summary of week's progress
- **Monthly Reports**: Monthly analytics and insights
- **Trend Analysis**: Long-term trend visualization

---

## TECHNICAL REQUIREMENTS FOR LOCAL APP

### Technology Stack Suggestions
- **Frontend Framework**: React, Vue, or vanilla JavaScript
- **Styling**: CSS/Tailwind/Styled Components
- **State Management**: Redux, Zustand, or Context API
- **Local Storage**: IndexedDB, LocalStorage, or SQLite
- **Build Tool**: Vite, Webpack, or Create React App
- **PWA Support**: Make it installable as a Progressive Web App

### Core Data Models

#### Main Feature Models (18 features)
1. **Habit**: id, name, description, listId, category, schedule, createdAt
2. **HabitEntry**: id, habitId, date, completed, notes
3. **Todo**: id, title, description, listId, projectId, tagIds, dueDate, completed, priority, category
4. **Event** (Planner): id, title, description, date, startTime, endTime, listId
5. **PomodoroSession**: id, date, duration, type (work/break), sittingTime, standingTime
6. **FoodLog**: id, date, items, macros (protein, carbs, fat, calories), macroGoalId
7. **WeightLog**: id, date, weight, unit
8. **HydrationLog**: id, date, amount, unit, templateId
9. **MoodLog**: id, date, mood, notes, moodValue
10. **Routine**: id, name, description, items, schedule, createdAt
11. **Countdown**: id, name, targetDate, createdAt, completed
12. **Fast**: id, startTime, endTime, duration, type, status
13. **Workout**: id, date, exercises, duration, muscleGroups, exerciseIds
14. **Trip**: id, name, startDate, endDate, destination, notes
15. **PackingItem**: id, tripId, name, category, status, packed
16. **Goal**: id, name, description, targetDate, progress, category, status
17. **Dashboard**: id, name, widgets, layout, isDefault
18. **SoundSettings**: id, masterVolume, lofiEnabled, lofiVolume, ambientSounds, autoPlayWithPomodoro
19. **SoundMixPreset**: id, name, description, lofiEnabled, lofiVolume, ambientSounds, ambientVolumes, masterVolume

#### Supporting/Reference Models
18. **List**: id, name, type (habit/todo/planner), color, icon, createdAt
19. **Project**: id, name, description, color, createdAt
20. **Tag**: id, name, color, createdAt
21. **MacroGoal**: id, name, protein, carbs, fat, calories, startDate, endDate
22. **WeightClub**: id, name, weightRange (min, max), unit
23. **HydrationTemplate**: id, name, amount, unit, icon
24. **Exercise**: id, name, muscleGroups, equipment, instructions
25. **WorkoutExercise**: id, workoutId, exerciseId, sets, reps, weight, duration, notes
26. **RoutineItem**: id, routineId, type (habit/todo/exercise), itemId, order, completed
27. **DashboardWidget**: id, dashboardId, type, position, config, data

#### User & Social Models
28. **User**: id, name, avatar, favorites, preferences, proStatus
29. **Post** (Timeline): id, userId, content, type (weight/workout/general), date, likes, commentCount
30. **Like**: id, postId, userId, createdAt
31. **Comment**: id, postId, userId, content, createdAt
32. **LeaderboardEntry**: userId, score, rank, timeframe, date

#### Analytics & Tracking Models
33. **HabitStreak**: id, habitId, startDate, endDate, currentStreak, longestStreak
34. **HabitScore**: id, habitId, date, score (A-F), period (30 days)
35. **PomodoroGoal**: id, userId, date, target, completed

---

## IMPLEMENTATION PRIORITY

### Phase 1 (MVP - Core Productivity)
1. Basic habit tracking (create, complete, view)
2. Simple day view for habits
3. Basic todo management with quick add
4. Planner with week view
5. Pomodoro timer with daily goals
6. Local data storage (IndexedDB or LocalStorage)
7. Basic navigation sidebar

### Phase 2 (Organization & Views)
1. Grid view for habits
2. Streak tracking and scoring system
3. Habit lists/organization
4. Todo views (Overview, Inbox, Today, Upcoming)
5. Projects, Lists, and Tags for todos
6. Filters and advanced views
7. Dashboard selection/creation

### Phase 3 (Health Tracking)
1. Food logging with macro goals
2. Weight tracking
3. Hydration tracking with templates
4. Mood tracking
5. Workout logging
6. Calendar integration for all health features

### Phase 4 (Advanced Features)
1. Routines
2. Countdowns
3. Goals
4. Fasting tracking
5. Trips and Packing items
6. Background Music & Ambient Sounds
7. Analytics and reports
8. Export/Import

### Phase 5 (Social & Gamification - Optional)
1. Timeline/Social feed
2. Leaderboard
3. User profiles
4. Likes and comments
5. Community features

### Phase 6 (Polish)
1. Advanced analytics
2. Themes (dark/light mode)
3. Notifications
4. Keyboard shortcuts
5. PWA optimization

---

## NOTES
- Focus on local-first architecture (no cloud dependency)
- Ensure data privacy (all data stays on device)
- Consider PWA for mobile-like experience
- Start with MVP and iterate based on usage

