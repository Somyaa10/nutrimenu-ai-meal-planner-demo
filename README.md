# NutriMenu: AI Household Meal Planner

A product case study for an app that decides each day's menu so every household member's protein and fibre targets are met, without the user having to check or approve it each day. This is Part 1 of 3: the menu decision engine.

## Links
- Live PRD and Specs: https://ubiquitous-hummingbird-5b5550.netlify.app
- Clickable prototype (sample data): https://meal-planner-prototype.lovable.app

## What's in this repo
- `01_PRD_Menu_Decision_Engine.md`: problem, users, requirements, success metrics
- `02_Specs_Menu_Decision_Engine.md`: data model, screens, AI and system logic
- `04_UI_Specs.md`: visual direction and screen notes

## Status
Specs and UI design are complete. The prototype is a clickable demo with hardcoded sample data. The working app is being rebuilt in phases.

## Key product decisions
- Protein and fibre are hard daily targets
- Zero daily verification: menus are generated ahead of time, and editing is only for exceptions
- Nutrition values are calculated from a database, not guessed by the AI
