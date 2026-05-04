---
title: "Maintenance Log - Ninth Week: React Frontend Kickoff (Vite, Routing, Layout)"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Frontend", "React", "Vite", "ReactRouter"]
series: ["Maintenance Log"]
date: 2026-05-04
draft: false
---

# Devlog Week 9: React Frontend Kickoff (Vite, Routing, Layout)

This week starts the frontend phase of the Maintenance Log project. The focus was not new domain functionality yet, but establishing a clean React baseline: routing, layout composition, reusable UI building blocks, and a consistent styling foundation.

## What Changed This Week

- Set up a React app using Vite and removed starter template code
- Added declarative routing with React Router and a nested layout structure
- Established a domain-based folder structure (`components/` + `pages/`)
- Defined a small design system via CSS custom properties (tokens)
- Implemented shared UI components (`InputField`, `Button`)
- Added a layout shell (`NavBar` + drawer menu) using state + conditional rendering
- Added initial pages and placeholders for upcoming features

---

## Project Setup

- Created the project with Vite
- Installed React Router
- Wrapped the app in `BrowserRouter` in `main.jsx`

## Folder Structure

The goal was to keep features grouped by domain and co-locate styles.

- `src/pages/` for route-level components
- `src/components/` grouped by domain:
  - `auth/`, `layout/`, `assets/`, `shared/`
- Component-specific CSS files live next to the component that uses them

## Routing (App.jsx)

Routing is centralized in `App.jsx`.

- Public route:
  - `/login`
- Authenticated area as a nested layout route:
  - `/` renders `Layout`
  - Child routes render inside `<Outlet />`
- Dynamic segments:
  - `:id` for asset/user details

The routing structure is intentionally minimal at this stage, but it establishes the pattern for adding the real pages later.

## Layout Shell (Layout + NavBar + DrawerMenu)

The app shell is implemented as a reusable layout route.

- `Layout` owns drawer open/close state
  - `toggleMenu` and `closeMenu` are passed down as props
- `NavBar` triggers `onToggle` from the burger menu
- `DrawerMenu` renders conditionally (`isMenuOpen && ...`)
  - `NavLink` items close the drawer on click
  - Logout button is a placeholder

## Design System (index.css + App.css)

Styling is token-based to keep UI consistent and easy to adjust.

- CSS custom properties for:
  - Colors, typography, spacing, radius, shadows, semantic status colors
- Mobile-first layout
- "Phone frame" shell on larger screens
- Media query switches to full-screen on real mobile devices

## Shared Components

- `InputField`
  - Controlled input via props
  - Optional label via ternary rendering
  - Props: `type`, `placeholder`, `required`, `value`, `onChange`
- `Button`
  - Reusable button component
  - Accepts `onClick` handler and `className`

## Auth Components

- `LoginForm`
  - Owns `email` and `password` state via `useState`
  - Uses `<form onSubmit={...}>` and `e.preventDefault()`
  - Currently logs credentials as a placeholder (API integration comes later)

## Pages

- `Login`: branding wrapper + `LoginForm`
- `AssetList`: holds mock assets in state and renders an `AssetCard` per asset
- `AssetDetail`, `EmployeeList`, `UserProfile`: placeholders for upcoming implementation

## Asset Components

- `AssetCard`
  - Receives an `asset` prop
  - Shows key fields (name, description, active status, last log date)
  - Uses ternary rendering for boolean status
  - Uses `useNavigate` to route to `/assets/:id/logs` on click

---

## Key React Concepts Practiced

- Props + prop drilling
- Controlled inputs with `useState`
- Conditional rendering (`&&` and ternary)
- Component composition and reuse
- Lifting state to the lowest common ancestor
- Route patterns: nested routes + `<Outlet />`
- `NavLink` vs `Link` (active styling)
- `useNavigate` for programmatic navigation

## Component Hierarchy

```
App
├── Login
│   └── LoginForm
│       ├── InputField (email)
│       ├── InputField (password)
│       └── Button (login)
└── Layout
    ├── NavBar
    ├── DrawerMenu
    │   ├── NavLink (Home)
    │   ├── NavLink (Assets)
    │   ├── NavLink (Your Profile)
    │   ├── NavLink (User List)
    │   ├── NavLink (Manage Assets)
    │   ├── NavLink (Manage Users)
    │   └── Button (Log Out)
    └── <Outlet>
        ├── AssetList
        │   └── AssetCard (× n)
        ├── AssetDetail
        ├── EmployeeList
        └── UserProfile
```

---

## Frontend Screenshots

{{< figure
  src="img/frontend/loginv1.png"
  alt="Login page (v1)"
  caption="Login page (v1)"
  >}}

{{< figure
  src="img/frontend/drawerMenuv1.png"
  alt="Drawer menu (v1)"
  caption="Drawer menu (v1)"
  >}}

{{< figure
  src="img/frontend/assetListv1.png"
  alt="Asset list (v1)"
  caption="Asset list (v1)"
  >}}
