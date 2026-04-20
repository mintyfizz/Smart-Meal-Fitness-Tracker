# Smart Meal & Fitness Tracker

![Build](https://github.com/mintyfizz/Smart-Meal-Fitness-Tracker/actions/workflows/build.yml/badge.svg)

A Windows desktop application built with **WPF (.NET 8)** and **Supabase** that helps users track their meals, physical activities, weight, and daily calorie goals — with an AI-powered meal recommendation engine powered by Google Gemini.

---

## Table of Contents

1. [Features](#features)
2. [Technology Stack](#technology-stack)
3. [Architecture Overview](#architecture-overview)
4. [Project Structure](#project-structure)
5. [Database Schema](#database-schema)
6. [Setup & Installation](#setup--installation)
7. [Configuration](#configuration)
8. [Running the App](#running-the-app)
9. [Views & Navigation](#views--navigation)
10. [Service Layer](#service-layer)
11. [AI Meal Recommendations](#ai-meal-recommendations)
12. [Admin Panel](#admin-panel)
13. [CI/CD Pipeline](#cicd-pipeline)
14. [Roadmap](#roadmap)

---

## Features

### User Features
- **Register & Login** — Secure email/password authentication via Supabase Auth with email confirmation support
- **Dashboard** — Real-time summary of daily calories consumed, calories burned, meal count, activity count, calorie balance, and BMI
- **Meal Logging** — Search the USDA FoodData Central database (44 seeded public foods + live USDA API search), select grams and meal type (Breakfast/Lunch/Dinner/Snack), and log to the database
- **Meal History** — Full paginated list of all logged meals with food name, grams, meal type, date, and one-click delete
- **Activity Tracking** — Log physical activities with name, calories burned, and duration in minutes
- **Weight History** — Log weigh-ins over time with an interactive line chart showing 7-day, 30-day, and all-time views, plus a target weight dashed line
- **Daily Calorie Goal** — Set a daily calorie target and optional target weight; goal is displayed on the dashboard with a progress bar
- **Profile Management** — Edit full name, age, height, weight, gender, dietary preferences (8 checkboxes), and allergies (8 checkboxes)
- **AI Meal Recommendations** — Generate a personalised one-day meal plan (Breakfast/Lunch/Dinner/Snacks) using Google Gemini 2.5 Flash, based on calorie goal, dietary preferences, and allergies

### Admin Features
- **Admin Dashboard** — View all registered users in a data grid with registration date, email, role, and ban status
- **Ban / Unban Users** — Toggle ban status per user; banned users are blocked at login with a descriptive error message
- **User Statistics** — Total user count, admin count, banned count, and new users this week

---

## Technology Stack

| Layer | Technology |
|---|---|
| UI Framework | WPF (Windows Presentation Foundation) |
| Language | C# 12 / .NET 8.0 |
| Backend / Auth | Supabase (PostgreSQL + GoTrue Auth) |
| ORM | Supabase Postgrest C# SDK v1.1.1 |
| AI | Google Gemini 2.5 Flash REST API |
| Food Database | USDA FoodData Central API |
| CI/CD | GitHub Actions (windows-latest) |
| Secrets | GitHub Actions Secrets |

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│                        WPF App (SmartMeal)                 │
│                                                            │
│   MainWindow (shell + service container + navigation)      │
│        │                                                   │
│        ├── LoginView / RegisterView                        │
│        ├── DashboardView                                   │
│        ├── AddMealView / MealsView                         │
│        ├── AddActivityView                                 │
│        ├── SetGoalView                                     │
│        ├── WeightHistoryView                               │
│        ├── ProfileView                                     │
│        ├── RecommendationsView                             │
│        └── AdminDashboardView                              │
│                                                            │
│   Services (injected from MainWindow):                     │
│   AuthService · MealService · FoodService · GoalService    │
│   ActService · WeightLogService · AdminService             │
│   FoodSearchService · GeminiService                        │
└────────────────────┬───────────────────────────────────────┘
                     │ HTTPS / JWT
         ┌───────────▼────────────┐     ┌─────────────────────┐
         │  Supabase Backend      │     │  External APIs       │
         │  ─ PostgreSQL (RLS)    │     │  ─ Gemini 2.5 Flash  │
         │  ─ GoTrue Auth         │     │  ─ USDA FoodData     │
         └────────────────────────┘     └─────────────────────┘
```

**Navigation pattern:** There is no router. `MainWindow.Navigate(UserControl)` swaps the content area. Every view casts `Application.Current.MainWindow` to `MainWindow` to access services.

**Config loading priority (highest wins):**
```
OS environment variables  →  .env file  →  supabase.config.local.json  →  supabase.config.json
```

---

## Project Structure

```
SmartMealSolution/
│
├── SmartMeal/                        ← WPF UI project
│   ├── MainWindow.xaml / .cs         ← App shell, service init, navigation
│   ├── Views/
│   │   ├── LoginView.xaml/.cs
│   │   ├── RegisterView.xaml/.cs
│   │   ├── DashboardView.xaml/.cs
│   │   ├── AddMealView.xaml/.cs
│   │   ├── MealsView.xaml/.cs
│   │   ├── AddActivityView.xaml/.cs
│   │   ├── SetGoalView.xaml/.cs
│   │   ├── WeightHistoryView.xaml/.cs
│   │   ├── ProfileView.xaml/.cs
│   │   ├── RecommendationsView.xaml/.cs
│   │   └── AdminDashboardView.xaml/.cs
│   ├── Helpers/
│   │   └── SessionHelper.cs          ← Shared user-ID guard used across views
│   └── supabase.config.example.json  ← Template — copy and fill in your keys
│
├── SmartMeal.core/                   ← Business logic (no UI dependency)
│   ├── Models/
│   │   ├── User.cs                   → public.users
│   │   ├── FoodItem.cs               → public.food_items
│   │   ├── Meal.cs (MealLog)         → public.meal_logs
│   │   ├── MealType.cs               → public.meal_types
│   │   ├── Goal.cs                   → public.goals
│   │   ├── Activity.cs               → public.activities
│   │   └── WeightLog.cs              → public.weight_logs
│   └── Services/
│       ├── AuthService.cs
│       ├── MealService.cs
│       ├── FoodService.cs
│       ├── FoodSearchService.cs      ← USDA API search + local cache
│       ├── GoalService.cs
│       ├── ActService.cs
│       ├── WeightLogService.cs
│       ├── AdminService.cs
│       └── GeminiService.cs          ← AI meal plan generation
│
├── SmartMeal.Data/                   ← Data access bootstrap
│   └── Context/
│       └── SupabaseClientProvider.cs ← Creates and holds the shared Supabase.Client
│
├── SmartUnit.Tests/                  ← xUnit test project (scaffolded)
│
├── database/                         ← SQL migration scripts
│   ├── add_user_preferences.sql      ← Adds food_preferences + allergies columns
│   └── rls_food_items_policies.sql   ← RLS policies for food_items table
│
├── .env.example                      ← Environment variable template
├── .gitignore
└── .github/
    └── workflows/
        └── build.yml                 ← CI: build on every push
```

---

## Database Schema

All tables live in the `public` schema in Supabase (PostgreSQL).

### `users`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | Matches Supabase Auth UID |
| `full_name` | text | Required |
| `email` | text UNIQUE | Required |
| `role` | text | `'user'` or `'admin'` |
| `age` | int | Optional |
| `height_cm` | numeric(5,2) | Optional |
| `weight_kg` | numeric(5,2) | Snapshot at registration |
| `gender` | text | `male` / `female` / `other` |
| `is_banned` | boolean | Default false; set by admins |
| `food_preferences` | text | Comma-separated tags e.g. `Vegetarian,Keto` |
| `allergies` | text | Comma-separated tags e.g. `Nuts,Gluten` |
| `created_at` | timestamptz | Auto |

### `food_items`
| Column | Type | Notes |
|---|---|---|
| `food_id` | bigint PK | Auto |
| `name` | text | |
| `food_category_id` | smallint FK | Nullable for USDA-sourced items |
| `calories_per_100g` | numeric(8,2) | |
| `protein_per_100g` | numeric(8,2) | |
| `carbs_per_100g` | numeric(8,2) | |
| `fats_per_100g` | numeric(8,2) | |
| `is_public` | boolean | True = seeded / visible to all |
| `is_active` | boolean | False = soft deleted |
| `created_by_user_id` | uuid FK | Null for public items |

### `meal_logs`
| Column | Type | Notes |
|---|---|---|
| `meal_log_id` | bigint PK | Auto |
| `user_id` | uuid FK | → users ON DELETE CASCADE |
| `food_id` | bigint FK | → food_items ON DELETE RESTRICT |
| `grams` | numeric(8,2) | > 0 |
| `meal_type_id` | smallint FK | → meal_types |
| `log_date` | date | `yyyy-MM-dd` |
| `created_at` | timestamptz | Auto |

### `goals`
| Column | Type | Notes |
|---|---|---|
| `goal_id` | bigint PK | Auto |
| `user_id` | uuid UNIQUE FK | One goal per user |
| `calorie_goal` | numeric | Daily target kcal |
| `target_weight_kg` | numeric | Optional |
| `protein_goal` | numeric | Optional (not yet in UI) |
| `carbs_goal` | numeric | Optional (not yet in UI) |
| `fat_goal` | numeric | Optional (not yet in UI) |

### `activities`
| Column | Type | Notes |
|---|---|---|
| `activity_id` | bigint PK | Auto |
| `user_id` | uuid FK | → users ON DELETE CASCADE |
| `name` | text | e.g. "Running" |
| `calories_burned` | int | |
| `duration_minutes` | int | |
| `logged_at` | timestamptz | |
| `notes` | text | Optional |

### `weight_logs`
| Column | Type | Notes |
|---|---|---|
| `weight_log_id` | bigint PK | Auto |
| `user_id` | uuid FK | → users ON DELETE CASCADE |
| `weight_kg` | numeric | > 0 |
| `logged_at` | timestamptz | |
| `notes` | text | Optional |

---

## Setup & Installation

### Prerequisites
- Windows 10 / 11
- [Visual Studio 2022+](https://visualstudio.microsoft.com/) with the **.NET desktop development** workload
- .NET 8.0 SDK
- A free [Supabase](https://supabase.com) account
- A free [Google AI Studio](https://aistudio.google.com) API key (for meal recommendations)
- A free [USDA FoodData Central](https://fdc.nal.usda.gov/api-guide.html) API key (for food search)

### 1. Clone the repository
```bash
git clone https://github.com/mintyfizz/Smart-Meal-Fitness-Tracker.git
cd Smart-Meal-Fitness-Tracker
```

### 2. Set up Supabase

1. Create a new Supabase project at [supabase.com](https://supabase.com)
2. In the **SQL Editor**, run the following scripts in order:

```sql
-- Create the core schema (users, food_items, meal_logs, goals, activities, weight_logs, etc.)
-- Use the Supabase Table Editor or run the schema SQL from your project setup
```

3. Run the migration scripts in the `database/` folder:
```sql
-- database/add_user_preferences.sql
-- database/rls_food_items_policies.sql
```

4. Copy your **Project URL** and **anon public key** from **Settings → API**

### 3. Configure credentials

Copy the example env file:
```bash
cp .env.example .env
```

Fill in your values in `.env`:
```env
SMARTMEAL_SUPABASE_URL=https://your-project.supabase.co
SMARTMEAL_SUPABASE_ANON_KEY=your-anon-key
SMARTMEAL_SUPABASE_EMAIL_REDIRECT_URL=
SMARTMEAL_USDA_API_KEY=your-usda-key
SMARTMEAL_GEMINI_API_KEY=your-gemini-key
```

Or use `supabase.config.json` (copy from `supabase.config.example.json`):
```json
{
  "SupabaseUrl": "https://your-project.supabase.co",
  "SupabaseAnonKey": "your-anon-key",
  "SupabaseEmailRedirectUrl": "",
  "UsdaApiKey": "your-usda-key",
  "GeminiApiKey": "your-gemini-key"
}
```

---

## Configuration

The app loads configuration in this priority order (first non-empty value wins):

| Source | Location | Committed to git? |
|---|---|---|
| OS environment variables | System / CI | N/A |
| `.env` file | Project root | No (gitignored) |
| `supabase.config.local.json` | `SmartMeal/` | No (gitignored) |
| `supabase.config.json` | `SmartMeal/` | No (gitignored) |

**Environment variable names:**
| Variable | Purpose |
|---|---|
| `SMARTMEAL_SUPABASE_URL` | Supabase project URL |
| `SMARTMEAL_SUPABASE_ANON_KEY` | Supabase anon public key |
| `SMARTMEAL_SUPABASE_EMAIL_REDIRECT_URL` | OAuth email redirect (optional) |
| `SMARTMEAL_USDA_API_KEY` | USDA FoodData Central API key |
| `SMARTMEAL_GEMINI_API_KEY` | Google Gemini API key |

---

## Running the App

1. Open `SmartMealSolution.sln` in Visual Studio
2. Set **SmartMeal** as the startup project
3. Press **F5** to build and run

The app starts on the **Register** screen. New users register and are routed to the **Dashboard**. Existing users can switch to **Login**.

---

## Views & Navigation

| View | Route | Description |
|---|---|---|
| `RegisterView` | App start | Create a new account with optional profile fields |
| `LoginView` | From Register | Sign in with email + password |
| `DashboardView` | After login | Daily stats: calories, activities, balance, BMI, goal |
| `AddMealView` | Dashboard → Add Meal | Search USDA foods, select grams + meal type, save |
| `MealsView` | Sidebar → Meals | Full meal log history with delete per row |
| `AddActivityView` | Sidebar → Activities | Log activity name, calories burned, duration |
| `SetGoalView` | Dashboard → Set Goal | Set daily calorie goal and optional target weight |
| `WeightHistoryView` | Sidebar → History | Log weigh-ins, view line chart with filter buttons |
| `ProfileView` | Sidebar → Profile | Edit personal details, dietary preferences, allergies |
| `RecommendationsView` | Sidebar → Recommendations | Generate AI meal plan via Gemini |
| `AdminDashboardView` | Admins only | User table with ban/unban controls |

**Sidebar navigation** is consistent across all views. The current view is highlighted blue. Every view has a **Log Out** button that signs out and returns to Login.

---

## Service Layer

| Service | Responsibility |
|---|---|
| `AuthService` | Register, login, logout, profile update, session management (`CurrentUser`) |
| `MealService` | Fetch today's logs, fetch all logs, add meal log, delete meal log |
| `FoodService` | Fetch all public food items, upsert USDA-sourced foods as private items |
| `FoodSearchService` | Search USDA FoodData Central API, return results as `FoodItem` objects |
| `GoalService` | Get user goal, upsert calorie goal + target weight |
| `ActService` | Fetch today's activities, add activity |
| `WeightLogService` | Fetch all weight logs for user, add new weigh-in |
| `AdminService` | Fetch all users, ban user, unban user |
| `GeminiService` | Build prompt from user profile, call Gemini API, parse structured JSON response |

All services receive the shared `Supabase.Client` from `MainWindow` at startup and use the Supabase Postgrest ORM for database operations.

---

## AI Meal Recommendations

The **Recommendations** view uses **Google Gemini 2.5 Flash** to generate a personalised one-day meal plan.

**How it works:**
1. The app reads the user's calorie goal, age, gender, weight, height, dietary preferences, and allergies from their profile
2. A prompt is built and sent to the Gemini REST API with:
   - `responseMimeType: "application/json"` — forces valid JSON output
   - `responseSchema` — constrains the response to the exact `{breakfast, lunch, dinner, snacks}` shape
   - `maxOutputTokens: 8192` — prevents truncation
3. The response is deserialized into a `MealPlan` object and displayed in a tab control with meal cards showing name, description, and calorie badge

**Example output structure:**
```json
{
  "breakfast": [{ "name": "Oats with Berries", "calories": 320, "description": "High-fibre oat bowl with mixed berries" }],
  "lunch":     [{ "name": "Grilled Chicken Salad", "calories": 450, "description": "Lean protein with leafy greens" }],
  "dinner":    [{ "name": "Salmon with Vegetables", "calories": 520, "description": "Omega-3 rich with roasted vegetables" }],
  "snacks":    [{ "name": "Greek Yoghurt", "calories": 150, "description": "High-protein low-sugar snack" }]
}
```

---

## Admin Panel

Users with `role = 'admin'` are automatically routed to the **Admin Dashboard** after login instead of the regular dashboard.

**Admin capabilities:**
- View all registered users with their details
- Ban a user — sets `is_banned = true` in the database; the user is immediately blocked from logging in
- Unban a user — sets `is_banned = false`, restoring access
- View aggregate stats (total users, admins, banned, new this week)

To create an admin, manually update the `role` column in the `public.users` table in Supabase:
```sql
UPDATE public.users SET role = 'admin' WHERE email = 'admin@example.com';
```

---

## CI/CD Pipeline

GitHub Actions runs a build on every push to `main` or `feature/supabase-ui-sync`.

**Workflow:** `.github/workflows/build.yml`

```
Trigger: push to main / feature/supabase-ui-sync
         pull request to main
         ↓
Runner: windows-latest
         ↓
Steps:
  1. Checkout code
  2. Setup .NET 8
  3. dotnet restore
  4. dotnet build --configuration Release
```

All credentials are stored as **GitHub Actions Secrets** and injected as environment variables at build time — no keys are ever stored in the repository.

---

## Roadmap

| Feature | Status |
|---|---|
| Activity history view (list + delete) | Planned |
| Edit meal log entries | Planned |
| Macro goal tracking (protein/carbs/fat UI) | Planned |
| Add recommended meal directly to meal log | Planned |
| Calorie trend chart over 30/90 days | Planned |
| Activity stats & trends chart | Planned |
| Custom food creation by users | Planned |
| Unit tests | Scaffolded (empty) |

---

## License

This project is for educational purposes.
