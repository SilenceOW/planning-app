# Command Center - Technical Architecture & Design

## Tech Stack

### Frontend
- **Framework:** Next.js 14+ (App Router)
- **Styling:** Tailwind CSS
- **Components:** shadcn/ui
- **State Management:** React Context + Hooks (or Zustand if needed)
- **Calendar:** react-big-calendar or custom implementation
- **Date Handling:** date-fns
- **Forms:** react-hook-form + zod
- **HTTP Client:** fetch with custom wrapper

### Backend
- **Framework:** FastAPI
- **Database:** PostgreSQL
- **ORM:** SQLAlchemy 2.0 with asyncpg
- **Auth:** Session-based authentication
- **Session Store:** Redis (or PostgreSQL-based sessions)
- **Migrations:** Alembic
- **Validation:** Pydantic v2

### Infrastructure
- **Frontend Hosting:** Vercel
- **Backend Hosting:** Railway / Render / DigitalOcean
- **Database:** Railway PostgreSQL / Supabase
- **Redis:** Upstash (free tier) or Railway

### Development
- **API Documentation:** FastAPI built-in (Swagger/OpenAPI)
- **Type Safety:** TypeScript (frontend) + Pydantic (backend)
- **Linting:** ESLint (frontend) + Ruff (backend)
- **Formatting:** Prettier (frontend) + Black (backend)

---

## Database Schema (SQLAlchemy)

### Models Structure

```python
# models/base.py
from datetime import datetime
from sqlalchemy import Column, Integer, DateTime
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class TimestampMixin:
    """Mixin for created_at and updated_at timestamps"""
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow, nullable=False)


# models/user.py
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import relationship
from .base import Base, TimestampMixin

class User(Base, TimestampMixin):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    full_name = Column(String, nullable=True)
    google_calendar_token = Column(String, nullable=True)  # For Google Calendar integration
    google_calendar_refresh_token = Column(String, nullable=True)
    
    # Relationships
    projects = relationship("Project", back_populates="user", cascade="all, delete-orphan")
    tasks = relationship("Task", back_populates="user", cascade="all, delete-orphan")
    calendar_events = relationship("CalendarEvent", back_populates="user", cascade="all, delete-orphan")
    time_entries = relationship("TimeEntry", back_populates="user", cascade="all, delete-orphan")
    cycles = relationship("Cycle", back_populates="user", cascade="all, delete-orphan")


# models/project.py
from sqlalchemy import Column, Integer, String, Float, ForeignKey, Enum as SQLEnum, Text
from sqlalchemy.orm import relationship
from .base import Base, TimestampMixin
import enum

class ProjectStatus(enum.Enum):
    ON_TRACK = "on_track"
    NEEDS_ATTENTION = "needs_attention"
    BLOCKED = "blocked"
    COMPLETED = "completed"
    ARCHIVED = "archived"

class Project(Base, TimestampMixin):
    __tablename__ = "projects"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    
    # Basic info
    name = Column(String(200), nullable=False)
    description = Column(Text, nullable=True)
    color = Column(String(7), nullable=True)  # Hex color for visual representation
    icon = Column(String(50), nullable=True)  # Emoji or icon name
    
    # Status
    status = Column(SQLEnum(ProjectStatus), default=ProjectStatus.ON_TRACK, nullable=False)
    
    # Time tracking
    hours_per_week = Column(Float, nullable=True)  # Expected hours per week
    
    # Next actions
    next_action = Column(Text, nullable=True)
    next_action_after = Column(Text, nullable=True)  # The action after next
    last_worked_on = Column(DateTime, nullable=True)
    
    # Schedule
    schedule_notes = Column(Text, nullable=True)  # e.g., "Mondays 9 AM - 6 PM"
    
    # Order for display
    display_order = Column(Integer, default=0)
    
    # Relationships
    user = relationship("User", back_populates="projects")
    tasks = relationship("Task", back_populates="project", cascade="all, delete-orphan")
    time_entries = relationship("TimeEntry", back_populates="project", cascade="all, delete-orphan")


# models/task.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey, Enum as SQLEnum, Text
from sqlalchemy.orm import relationship
from .base import Base, TimestampMixin
import enum

class TaskPriority(enum.Enum):
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"

class Task(Base, TimestampMixin):
    __tablename__ = "tasks"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    project_id = Column(Integer, ForeignKey("projects.id", ondelete="CASCADE"), nullable=True)
    
    # Task details
    title = Column(String(500), nullable=False)
    description = Column(Text, nullable=True)
    completed = Column(Boolean, default=False, nullable=False)
    completed_at = Column(DateTime, nullable=True)
    
    # Scheduling
    due_date = Column(DateTime, nullable=True)
    start_time = Column(DateTime, nullable=True)
    end_time = Column(DateTime, nullable=True)
    
    # Metadata
    priority = Column(SQLEnum(TaskPriority), default=TaskPriority.MEDIUM, nullable=False)
    estimated_minutes = Column(Integer, nullable=True)
    actual_minutes = Column(Integer, nullable=True)
    
    # Order within project
    display_order = Column(Integer, default=0)
    
    # Relationships
    user = relationship("User", back_populates="tasks")
    project = relationship("Project", back_populates="tasks")


# models/calendar_event.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey, Text
from sqlalchemy.orm import relationship
from .base import Base, TimestampMixin

class CalendarEvent(Base, TimestampMixin):
    __tablename__ = "calendar_events"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    
    # Event details
    title = Column(String(500), nullable=False)
    description = Column(Text, nullable=True)
    location = Column(String(500), nullable=True)
    
    # Timing
    start_time = Column(DateTime, nullable=False)
    end_time = Column(DateTime, nullable=False)
    all_day = Column(Boolean, default=False, nullable=False)
    
    # Google Calendar integration
    google_event_id = Column(String(500), nullable=True, unique=True)
    is_from_google = Column(Boolean, default=False, nullable=False)
    
    # Color coding
    color = Column(String(7), nullable=True)
    
    # Relationships
    user = relationship("User", back_populates="calendar_events")


# models/time_entry.py
from sqlalchemy import Column, Integer, DateTime, ForeignKey, Text
from sqlalchemy.orm import relationship
from .base import Base, TimestampMixin

class TimeEntry(Base, TimestampMixin):
    __tablename__ = "time_entries"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    project_id = Column(Integer, ForeignKey("projects.id", ondelete="CASCADE"), nullable=False)
    
    # Time tracking
    start_time = Column(DateTime, nullable=False)
    end_time = Column(DateTime, nullable=True)  # Null if still running
    duration_minutes = Column(Integer, nullable=True)  # Calculated on end
    
    # Notes
    notes = Column(Text, nullable=True)
    
    # Relationships
    user = relationship("User", back_populates="time_entries")
    project = relationship("Project", back_populates="time_entries")


# models/cycle.py
from sqlalchemy import Column, Integer, String, Date, ForeignKey, Text, Enum as SQLEnum
from sqlalchemy.orm import relationship
from .base import Base, TimestampMixin
import enum

class CyclePeriod(enum.Enum):
    DAY = "day"
    WEEK = "week"
    CUSTOM = "custom"  # For 6-week cycles

class Cycle(Base, TimestampMixin):
    __tablename__ = "cycles"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    
    # Cycle details
    name = Column(String(200), nullable=False)  # e.g., "Cycle 1: Ship, Ship, Ship"
    period = Column(SQLEnum(CyclePeriod), nullable=False)
    start_date = Column(Date, nullable=False)
    end_date = Column(Date, nullable=False)
    
    # Goals
    goals = Column(Text, nullable=True)  # JSON or text with cycle goals
    
    # Priority projects for this cycle
    priority_project_ids = Column(Text, nullable=True)  # JSON list of project IDs
    
    # Relationships
    user = relationship("User", back_populates="cycles")
```

### Additional Indexes

```python
# Add these to respective models or via migrations

# Indexes for common queries
Index('idx_tasks_user_project', Task.user_id, Task.project_id)
Index('idx_tasks_due_date', Task.due_date)
Index('idx_tasks_completed', Task.completed)
Index('idx_calendar_events_user_time', CalendarEvent.user_id, CalendarEvent.start_time)
Index('idx_time_entries_user_project', TimeEntry.user_id, TimeEntry.project_id)
Index('idx_cycles_user_dates', Cycle.user_id, Cycle.start_date, Cycle.end_date)
```

---

## API Structure (FastAPI)

### Endpoints Overview

```python
# main.py structure
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="Command Center API")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],  # Update for production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routers
app.include_router(auth.router, prefix="/api/auth", tags=["auth"])
app.include_router(projects.router, prefix="/api/projects", tags=["projects"])
app.include_router(tasks.router, prefix="/api/tasks", tags=["tasks"])
app.include_router(calendar.router, prefix="/api/calendar", tags=["calendar"])
app.include_router(time.router, prefix="/api/time", tags=["time-tracking"])
app.include_router(cycles.router, prefix="/api/cycles", tags=["cycles"])
app.include_router(dashboard.router, prefix="/api/dashboard", tags=["dashboard"])
```

### Key API Endpoints

```python
# Auth
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout
GET    /api/auth/me

# Projects
GET    /api/projects              # List all user's projects
POST   /api/projects              # Create project
GET    /api/projects/{id}         # Get project details
PATCH  /api/projects/{id}         # Update project
DELETE /api/projects/{id}         # Delete project
PATCH  /api/projects/{id}/status  # Update status
POST   /api/projects/reorder      # Reorder projects

# Tasks
GET    /api/tasks                 # List tasks (with filters)
POST   /api/tasks                 # Create task
GET    /api/tasks/{id}            # Get task
PATCH  /api/tasks/{id}            # Update task
DELETE /api/tasks/{id}            # Delete task
POST   /api/tasks/{id}/complete   # Mark complete
GET    /api/tasks/today           # Get today's tasks
GET    /api/tasks/week            # Get week's tasks

# Calendar
GET    /api/calendar/events       # List events (date range)
POST   /api/calendar/events       # Create event
PATCH  /api/calendar/events/{id}  # Update event
DELETE /api/calendar/events/{id}  # Delete event
GET    /api/calendar/google/auth  # Start Google OAuth
GET    /api/calendar/google/callback  # OAuth callback
POST   /api/calendar/google/sync  # Sync from Google

# Time Tracking
POST   /api/time/start            # Start tracking time
POST   /api/time/stop             # Stop tracking
GET    /api/time/entries          # Get time entries
GET    /api/time/stats            # Time stats (by project, date range)

# Cycles
GET    /api/cycles                # List cycles
POST   /api/cycles                # Create cycle
GET    /api/cycles/current        # Get current cycle
PATCH  /api/cycles/{id}           # Update cycle

# Dashboard
GET    /api/dashboard/overview    # Main dashboard data
GET    /api/dashboard/today       # Today view data
GET    /api/dashboard/week        # Week view data
```

### Pydantic Schemas Example

```python
# schemas/project.py
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional
from enum import Enum

class ProjectStatus(str, Enum):
    ON_TRACK = "on_track"
    NEEDS_ATTENTION = "needs_attention"
    BLOCKED = "blocked"
    COMPLETED = "completed"
    ARCHIVED = "archived"

class ProjectBase(BaseModel):
    name: str = Field(..., max_length=200)
    description: Optional[str] = None
    color: Optional[str] = Field(None, pattern="^#[0-9A-Fa-f]{6}$")
    icon: Optional[str] = None
    hours_per_week: Optional[float] = Field(None, ge=0)
    next_action: Optional[str] = None
    next_action_after: Optional[str] = None
    schedule_notes: Optional[str] = None

class ProjectCreate(ProjectBase):
    pass

class ProjectUpdate(BaseModel):
    name: Optional[str] = Field(None, max_length=200)
    description: Optional[str] = None
    color: Optional[str] = Field(None, pattern="^#[0-9A-Fa-f]{6}$")
    icon: Optional[str] = None
    status: Optional[ProjectStatus] = None
    hours_per_week: Optional[float] = Field(None, ge=0)
    next_action: Optional[str] = None
    next_action_after: Optional[str] = None
    last_worked_on: Optional[datetime] = None
    schedule_notes: Optional[str] = None

class ProjectResponse(ProjectBase):
    id: int
    status: ProjectStatus
    last_worked_on: Optional[datetime]
    display_order: int
    created_at: datetime
    updated_at: datetime
    
    # Computed fields
    task_count: int
    completed_task_count: int
    hours_this_week: float
    
    class Config:
        from_attributes = True
```

---

## UI/UX Design

### Color Palette

```css
/* Tailwind config extension */
{
  colors: {
    primary: {
      50: '#f0f9ff',
      100: '#e0f2fe',
      500: '#0ea5e9',
      600: '#0284c7',
      700: '#0369a1',
    },
    success: {
      500: '#10b981',
      600: '#059669',
    },
    warning: {
      500: '#f59e0b',
      600: '#d97706',
    },
    danger: {
      500: '#ef4444',
      600: '#dc2626',
    },
    neutral: {
      50: '#fafafa',
      100: '#f5f5f5',
      200: '#e5e5e5',
      300: '#d4d4d4',
      700: '#404040',
      800: '#262626',
      900: '#171717',
    }
  }
}
```

### Component Design System

#### 1. Project Card

```tsx
// Component structure
<ProjectCard>
  <Header>
    <Icon + Name>
    <StatusBadge>
  </Header>
  
  <Stats>
    <TimeThisWeek />
    <TaskProgress />
  </Stats>
  
  <NextAction>
    Primary action (bold)
    Secondary action (muted)
  </NextAction>
  
  <Footer>
    <LastWorked />
    <QuickActions />
  </Footer>
</ProjectCard>
```

**Visual Design:**
```
┌────────────────────────────────────────┐
│ 💼 Alles Work           ✅ On Track    │
│────────────────────────────────────────│
│ ⏱️  12.5h / 15h this week              │
│ ✓ 3 / 7 tasks                          │
│────────────────────────────────────────│
│ Next: Review Q4 metrics                │
│ Then: Prepare founder report           │
│────────────────────────────────────────│
│ 🕐 2h ago    [▶️ Start] [📝] [⚙️]      │
└────────────────────────────────────────┘
```

**Tailwind Classes:**
- Card: `bg-white dark:bg-neutral-800 rounded-xl shadow-sm border border-neutral-200 p-6 hover:shadow-md transition-shadow`
- Status badge: `inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium`
  - On track: `bg-success-100 text-success-800`
  - Needs attention: `bg-warning-100 text-warning-800`
  - Blocked: `bg-danger-100 text-danger-800`

---

#### 2. Main Dashboard Layout

```
┌─────────────────────────────────────────────────────────────┐
│ Command Center                    [Today] [Week] [Profile]  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ ⚡ Quick Capture                                             │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ [Type task and press Enter...] 🎯                     │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 📊 Today's Focus                              [View All →]  │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ ☐ Review user metrics (Alles) · 30min · Due 2:00 PM  │  │
│ │ ☐ Prep Session 3 (AI Intensive) · 1h · Due today     │  │
│ │ ☐ Email sponsors (MIPT Club) · 15min · High priority │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 🎯 Active Projects (6)                       [+ Add Project] │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│ │ 💼 Alles     │ │ 🎓 AI Int.   │ │ 📱 TG Chan.  │        │
│ │ 12.5h / 15h  │ │ 2h / 3h      │ │ 0.5h / 1h    │        │
│ │ ✅ On Track  │ │ ⚠️  Needs     │ │ ✅ On Track  │        │
│ │              │ │  Attention   │ │              │        │
│ │ 3/7 tasks    │ │ 5/8 tasks    │ │ 1/2 tasks    │        │
│ └──────────────┘ └──────────────┘ └──────────────┘        │
│                                                               │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│ │ 🎙️ Podcasts  │ │ 🧠 Mastermind│ │ 🏢 MIPT Club │        │
│ │ ...          │ │ ...          │ │ ...          │        │
│ └──────────────┘ └──────────────┘ └──────────────┘        │
│                                                               │
│ 📈 This Week Overview                                        │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ Total time: 32h / 48h available                       │  │
│ │ ████████████░░░░░░░░░  67%                            │  │
│ │                                                         │  │
│ │ Most focused: Alles (12.5h)                            │  │
│ │ Needs attention: AI Intensive (behind 1h)              │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Layout Structure:**
- Max width: `max-w-7xl mx-auto`
- Padding: `px-4 sm:px-6 lg:px-8 py-8`
- Grid: `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6`

---

#### 3. Today View

```
┌─────────────────────────────────────────────────────────────┐
│ ← Dashboard          Today - Thursday, Nov 7                │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ 🌅 Morning (8 AM - 12 PM)                                   │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ 8:00  [FREE TIME] ──────────────────────────────      │  │
│ │ 10:00 ☐ Deep Work: AI Intensive prep (2h)             │  │
│ │ 12:00                                                   │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 🌤️  Afternoon (12 PM - 6 PM)                                │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ 12:00 [LUNCH BREAK] ──────────────────────────        │  │
│ │ 1:00  [FREE TIME] ──────────────────────────────      │  │
│ │ 6:00                                                    │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 🌙 Evening (6 PM - 11 PM)                                   │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ 6:00  💪 Exercise ──────────────────                   │  │
│ │ 7:00  [DINNER] ──────────────────────────────          │  │
│ │ 8:00  ☐ Building time (3h)                             │  │
│ │ 11:00                                                   │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 📋 Unscheduled Tasks                                         │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ ☐ Email sponsors (15min) - MIPT Club                  │  │
│ │ ☐ Edit podcast (1h) - Podcasts                         │  │
│ │ ☐ Write TG post (30min) - TG Channel                  │  │
│ │                                                         │  │
│ │ [Drag to schedule →]                                   │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 💡 Smart Suggestions                                         │
│ • You have 12h free time today (your power day!)            │
│ • AI Intensive is behind schedule - prioritize today        │
│ • Consider batching all email tasks (saves 15min)           │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Features:**
- Drag-and-drop tasks into time slots
- Visual blocks for scheduled vs free time
- Color-coded by project
- Smart suggestions based on patterns

---

#### 4. Weekly View

```
┌─────────────────────────────────────────────────────────────┐
│ Week of Nov 4-10, 2025                  [← Prev] [Next →]  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│      Mon    Tue    Wed    Thu    Fri    Sat    Sun          │
│ 8  ┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐      │
│    │Alles││     ││MIPT ││     ││MIPT ││     ││     │      │
│ 9  │Work ││     ││     ││     ││     ││     ││     │      │
│    │     ││     ││     ││FREE ││     ││FREE ││FREE │      │
│ 10 │     ││     ││Calc ││     ││Disc ││     ││     │      │
│    │     ││     ││     ││     ││     ││     ││     │      │
│ 11 │     ││     ││     ││     ││     ││     ││     │      │
│    │     ││     ││     ││     ││     ││     ││     │      │
│ 12 │     ││Eng  ││Prob ││     ││     ││AI   ││     │      │
│    │     ││     ││     ││     ││     ││Int  ││     │      │
│ 1  │     ││     ││     ││     ││     ││     ││     │      │
│    │     ││     ││     ││     ││     ││     ││     │      │
│ 2  │     ││     ││     ││     ││     ││     ││Mstr │      │
│    │     ││     ││     ││     ││     ││     ││mind │      │
│ 3  │     ││     ││     ││     ││Diff ││     ││     │      │
│    │     ││     ││     ││     ││Eq   ││     ││     │      │
│ 4  │     ││     ││     ││     ││     ││     ││     │      │
│    │     ││     ││     ││     ││     ││     ││     │      │
│ 5  │     ││     ││     ││     ││     ││     ││     │      │
│ 6  └─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘      │
│                                                               │
│ 📊 Week Summary                                              │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ Total scheduled: 36h / 48h available                   │  │
│ │ Free time remaining: 12h                                │  │
│ │                                                         │  │
│ │ By Project:                                             │  │
│ │ • Alles: 15h (target: 15h) ✅                          │  │
│ │ • AI Intensive: 2h (target: 3h) ⚠️ 1h behind           │  │
│ │ • MIPT Club: 2h (target: 3h) ⚠️                        │  │
│ │ • Podcasts: 1h (target: 2h) ⚠️                         │  │
│ │ • TG Channel: 1h (target: 1h) ✅                       │  │
│ │ • Mastermind: 2h (target: 2h) ✅                       │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 🎯 Focus Projects This Week                                 │
│ [Alles] [AI Intensive] [+ Add focus]                        │
│ (Blur other projects in calendar view)                      │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Interaction:**
- Click cell to add task/event
- Drag to resize time blocks
- Hover shows full event details
- Click focus project to blur others

---

#### 5. Cycle View (Unique Feature!)

```
┌─────────────────────────────────────────────────────────────┐
│ Cycle 1: Ship, Ship, Ship                  Nov 7 - Dec 20   │
│ Week 1 of 6                                    [Edit Cycle]  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ 🎯 Cycle Goals                                               │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ • Ship 3 projects                                      │  │
│ │ • Get 50+ real users                                   │  │
│ │ • Build in public: 100+ followers                      │  │
│ │ • Identify 3-5 potential co-founders                   │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 🔥 Priority Projects for This Cycle                         │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│ │ Building     │ │ Networking   │ │ Content      │        │
│ │ 40h/week     │ │ 2.5h/week    │ │ 2.5h/week    │        │
│ │ ███████████  │ │ █████        │ │ ████         │        │
│ └──────────────┘ └──────────────┘ └──────────────┘        │
│                                                               │
│ 📊 Progress Tracking                                         │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ Projects shipped: 0 / 3                                │  │
│ │ Users acquired: 0 / 50                                  │  │
│ │ Social followers: 45 / 100                              │  │
│ │ Co-founders identified: 1 / 5                           │  │
│ │                                                         │  │
│ │ Time remaining: 5 weeks, 6 days                         │  │
│ │ On track: ⚠️  Behind on users & projects                │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 📅 Week-by-Week Plan                                         │
│ ┌─────┬─────────────────────────────────────┬─────────┐   │
│ │ W1  │ Resume AI + Setup                   │ Current │   │
│ │ W2  │ Resume AI launch + Project #2       │         │   │
│ │ W3  │ Project #2 + Main project research  │         │   │
│ │ W4  │ Main project foundation             │         │   │
│ │ W5  │ Main project alpha                  │         │   │
│ │ W6  │ Review + Holiday prep               │         │   │
│ └─────┴─────────────────────────────────────┴─────────┘   │
│                                                               │
│ 💭 Cycle Reflections                                         │
│ [Add notes about what's working, what's not...]              │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Unique Features:**
- Visual progress bars for cycle goals
- Week-by-week breakdown
- Reflection space for learnings
- Comparison to planned vs actual

---

#### 6. Project Detail Page

```
┌─────────────────────────────────────────────────────────────┐
│ ← Back to Dashboard                                          │
│                                                               │
│ 💼 Alles Work                              [Edit] [Archive]  │
│ ✅ On Track · 15h/week · Monday 9 AM - 6 PM                 │
│ Last worked: 2 hours ago                                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ 🎯 Next Actions                                              │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ 1. Review Q4 metrics and prepare dashboard             │  │
│ │    [Start Timer] [Mark Complete] [Edit]                │  │
│ │                                                         │  │
│ │ 2. Weekly sync with team about new features            │  │
│ │    (After completing above)                             │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 📋 Tasks                      [All] [Active] [Completed]     │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ This Week                                               │  │
│ │ ☐ Review user metrics (30min) · Due Today 2:00 PM     │  │
│ │ ☐ Weekly sync (60min) · Monday                         │  │
│ │ ☐ Product roadmap update (2h) · Thursday               │  │
│ │ ☑ Bug priority list (15min) · Completed ✓             │  │
│ │                                                         │  │
│ │ Next Week                                               │  │
│ │ ☐ User research interviews (3h)                        │  │
│ │ ☐ Mobile app testing                                   │  │
│ │                                                         │  │
│ │ [+ Add Task]                                           │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 📊 Time Tracking                                             │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ This Week: 12.5h / 15h (83%)                           │  │
│ │ ████████████████████░░░░                                │  │
│ │                                                         │  │
│ │ Recent Sessions:                                        │  │
│ │ • Today: 2.5h (User metrics review)                    │  │
│ │ • Yesterday: 4h (Feature planning)                     │  │
│ │ • Tuesday: 3h (Team meetings)                          │  │
│ │ • Monday: 3h (Bug fixes)                               │  │
│ │                                                         │  │
│ │ [View Full History]                                    │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
│ 📝 Notes & Context                                           │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ Focus areas this month:                                 │  │
│ │ - Retention metrics improvement                         │  │
│ │ - Mobile app v2 launch                                  │  │
│ │                                                         │  │
│ │ Key contacts:                                           │  │
│ │ - Founder wants weekly updates                          │  │
│ │ - Dev team sync Mondays 10 AM                          │  │
│ │                                                         │  │
│ │ [Edit Notes]                                           │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

#### 7. Quick Capture Component

**Always visible at top of every page:**

```
┌─────────────────────────────────────────────────────────────┐
│ ⚡ Quick Capture                                             │
│ ┌───────────────────────────────────────────────────────┐  │
│ │ Type anything: task, idea, note... [Enter to save] 🎯 │  │
│ └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**On Enter, shows quick options:**
```
┌───────────────────────────────────────────┐
│ "Review user metrics"                      │
│                                            │
│ Assign to project:                         │
│ • 💼 Alles Work                            │
│ • 🎓 AI Intensive                          │
│ • [No project - Inbox]                     │
│                                            │
│ Due date: [Today] [Tomorrow] [This week]  │
│ Time: [30min] [1h] [2h] [Custom]          │
│                                            │
│      [Save] [Cancel]                       │
└───────────────────────────────────────────┘
```

**Keyboard Shortcuts:**
- `Cmd/Ctrl + K`: Open quick capture
- `Cmd/Ctrl + Enter`: Save
- `/p Project Name`: Auto-assign to project
- `/d tomorrow`: Set due date
- `/t 30m`: Set time estimate

---

### Mobile Responsive Design

**Breakpoints:**
```css
sm: 640px   /* Mobile landscape */
md: 768px   /* Tablet */
lg: 1024px  /* Desktop */
xl: 1280px  /* Large desktop */
```

**Mobile Layout:**
- Bottom navigation bar (Dashboard, Today, Add, Week, Profile)
- Swipe gestures (left/right for prev/next day)
- Collapsible project cards (tap to expand)
- FAB (Floating Action Button) for Quick Capture
- Pull-to-refresh on dashboard

---

### Dark Mode Support

**Implementation:**
```tsx
// Use Tailwind dark mode classes
<div className="bg-white dark:bg-neutral-900">
  <h1 className="text-neutral-900 dark:text-neutral-100">
    Command Center
  </h1>
</div>
```

**Toggle in header:**
```tsx
<Button
  onClick={toggleDarkMode}
  variant="ghost"
  size="icon"
>
  {isDark ? <Sun /> : <Moon />}
</Button>
```

---

### Loading States & Animations

**Skeleton Loaders:**
```tsx
<ProjectCardSkeleton>
  <div className="animate-pulse">
    <div className="h-4 bg-neutral-200 rounded w-3/4 mb-4" />
    <div className="h-3 bg-neutral-200 rounded w-1/2" />
  </div>
</ProjectCardSkeleton>
```

**Transitions:**
- Page transitions: `transition-opacity duration-300`
- Hover effects: `hover:scale-105 transition-transform`
- Smooth scrolling: `scroll-smooth`

---

### Accessibility

**Key Considerations:**
- All interactive elements keyboard accessible
- ARIA labels on icons and buttons
- Focus visible states
- Color contrast ratios (WCAG AA minimum)
- Screen reader friendly

```tsx
<button
  aria-label="Mark task as complete"
  className="focus:ring-2 focus:ring-primary-500 focus:outline-none"
>
  <Check className="w-4 h-4" />
</button>
```

---

## Component Library (shadcn/ui)

### Components to Install

```bash
npx shadcn-ui@latest add button
npx shadcn-ui@latest add card
npx shadcn-ui@latest add input
npx shadcn-ui@latest add label
npx shadcn-ui@latest add select
npx shadcn-ui@latest add dialog
npx shadcn-ui@latest add dropdown-menu
npx shadcn-ui@latest add badge
npx shadcn-ui@latest add calendar
npx shadcn-ui@latest add checkbox
npx shadcn-ui@latest add form
npx shadcn-ui@latest add popover
npx shadcn-ui@latest add progress
npx shadcn-ui@latest add separator
npx shadcn-ui@latest add tabs
npx shadcn-ui@latest add textarea
npx shadcn-ui@latest add toast
npx shadcn-ui@latest add tooltip
```

---

## Additional Visual Details

### Status Indicators

**On Track:**
```tsx
<Badge className="bg-success-100 text-success-800">
  <CheckCircle className="w-3 h-3 mr-1" />
  On Track
</Badge>
```

**Needs Attention:**
```tsx
<Badge className="bg-warning-100 text-warning-800">
  <AlertCircle className="w-3 h-3 mr-1" />
  Needs Attention
</Badge>
```

**Blocked:**
```tsx
<Badge className="bg-danger-100 text-danger-800">
  <XCircle className="w-3 h-3 mr-1" />
  Blocked
</Badge>
```

### Time Progress Bars

```tsx
<div className="w-full bg-neutral-200 rounded-full h-2">
  <div 
    className="bg-primary-600 h-2 rounded-full transition-all duration-300"
    style={{ width: `${(12.5 / 15) * 100}%` }}
  />
</div>
<p className="text-sm text-neutral-600 mt-1">
  12.5h / 15h (83%)
</p>
```

---

## Final Notes

This design provides:
- ✅ Clean, professional UI
- ✅ Full responsiveness (mobile-first)
- ✅ Dark mode support
- ✅ Accessibility compliance
- ✅ Smooth interactions
- ✅ Scalable component system
- ✅ Type-safe backend with FastAPI
- ✅ Proper database relationships
- ✅ RESTful API structure

The system is designed to grow with you - start with MVP features and add AI recommendations, advanced analytics, and integrations later.
