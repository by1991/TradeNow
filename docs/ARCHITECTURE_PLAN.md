# TradeNow Frontend Architecture Plan

## Goal

Build a single monorepo for a trading application frontend with:

- Web app using the latest stable React + TypeScript.
- Mobile app using the latest stable React Native + TypeScript.
- Shared business logic across web and mobile.
- Shared platform API surface across web and mobile.
- Separate platform UI for web and mobile.

This plan assumes a pnpm workspace monorepo managed with Turborepo. The exact framework versions should be resolved to the latest stable releases at project initialization time.

## Recommended Stack

- Monorepo: pnpm workspaces + Turborepo
- Web: React + TypeScript + Vite
- Mobile: React Native + TypeScript using the React Native Community CLI
- Routing:
  - Web: React Router
  - Mobile: React Navigation
- UI system: Tamagui with shared tokens/theme config and separate web/mobile component packages
- Server state: TanStack Query
- Local state: Zustand or feature-local state machines
- Forms: React Hook Form + Zod
- Shared validation: Zod
- Testing:
  - Unit: Vitest for shared packages and web-compatible packages
  - Mobile unit/UI: Jest + React Native Testing Library
  - Web UI: Testing Library
- Linting/formatting: ESLint + Prettier

## Architecture Principles

1. Business rules must live outside platform apps.
2. UI components must not be shared between web and mobile.
3. Platform API contracts should be shared, while platform-specific implementations can differ behind the same interface.
4. Shared packages should avoid direct browser or native imports unless using platform-specific entry files.
5. Apps should compose shared logic and platform services, then render their own UI.

## Proposed Folder Structure

```text
TradeNow/
  apps/
    web/
      src/
        app/
        routes/
        screens/
        components/
        providers/
        platform/
        main.tsx
      public/
      index.html
      package.json
      tsconfig.json
      vite.config.ts

    mobile/
      src/
        navigation/
        screens/
        components/
        providers/
        platform/
      assets/
      android/
      ios/
      babel.config.js
      metro.config.js
      package.json
      tsconfig.json

  packages/
    domain/
      src/
        entities/
        value-objects/
        services/
        rules/
        events/
        index.ts
      package.json
      tsconfig.json

    application/
      src/
        use-cases/
        stores/
        hooks/
        mappers/
        facades/
        index.ts
      package.json
      tsconfig.json

    platform/
      src/
        api/
          storage/
            index.ts
            types.ts
            createStorage.ts
            storage.web.ts
            storage.native.ts
          auth/
            index.ts
            types.ts
            auth.web.ts
            auth.native.ts
          network/
            index.ts
            httpClient.ts
          notifications/
            index.ts
            notifications.web.ts
            notifications.native.ts
          device/
            index.ts
            device.web.ts
            device.native.ts
        index.ts
      package.json
      tsconfig.json

    ui-web/
      src/
        components/
        layouts/
        theme/
        index.ts
      package.json
      tsconfig.json

    ui-mobile/
      src/
        components/
        layouts/
        theme/
        index.ts
      package.json
      tsconfig.json

    assets/
      src/
        icons/
        illustrations/
        tokens/
        index.ts
      package.json

    config/
      eslint/
      typescript/
      jest/
      package.json

  docs/
    decisions/

  package.json
  pnpm-workspace.yaml
  turbo.json
  tsconfig.base.json
  README.md
```

## Package Responsibilities

### apps/web

- Owns browser bootstrapping, routing, web-specific providers, and web screens.
- Uses shared domain, application, and platform packages.
- Uses only `ui-web` for rendering.

### apps/mobile

- Owns native bootstrapping, navigation shell, device entrypoints, and mobile screens.
- Uses shared domain, application, and platform packages.
- Uses only `ui-mobile` for rendering.

### packages/domain

- Pure business logic.
- Trading entities, order rules, portfolio calculations, market models, and validation rules.
- No React, no browser APIs, no React Native APIs.

Examples:

- Order validation
- Margin and fee calculations
- Portfolio summary calculations
- Instrument formatting rules

### packages/application

- Orchestrates use cases around domain logic.
- Hosts app-facing hooks, query orchestration, shared state stores, and feature facades.
- May depend on `domain` and `platform`, but must not depend on `ui-web` or `ui-mobile`.

Examples:

- `usePlaceOrder`
- `usePortfolioOverview`
- `watchlistStore`
- `loginFacade`

### packages/platform

- Shared platform API layer.
- Defines stable interfaces for storage, auth, networking, device info, notifications, and secure persistence.
- Uses platform-specific files such as `*.web.ts` and `*.native.ts` when implementations differ.
- Exposes one shared import surface to both apps.

This is the key package for the requirement that platform APIs are shared between web and mobile.

Example idea:

```ts
export interface SecureStorageApi {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
  remove(key: string): Promise<void>;
}
```

Then implement it separately in:

- `secureStorage.web.ts` using browser storage or IndexedDB
- `secureStorage.native.ts` using `react-native-keychain`, encrypted storage, or another native-secure storage solution

The application layer imports only the shared contract and factory, not the platform-specific implementation directly.

### packages/ui-web

- Web-only presentation components.
- Tables, desktop charts, order forms, responsive layouts, and browser-oriented interactions.
- Built with Tamagui for web, with web-specific composition and interaction patterns.

### packages/ui-mobile

- Mobile-only presentation components.
- Native navigation shells, touch-first order tickets, mobile chart containers, and handset layouts.
- Built with Tamagui for native, with mobile-specific composition and interaction patterns.

Both UI packages can share design tokens and theme primitives while keeping screen-level UI and platform interaction patterns separate.

### packages/assets

- Shared non-behavior assets and design tokens.
- Use only assets that both apps can consume safely.

### packages/config

- Centralized config presets for TypeScript, ESLint, and test tooling.

## Dependency Rules

Use this dependency direction strictly:

```text
apps/web      -> application, domain, platform, ui-web, assets
apps/mobile   -> application, domain, platform, ui-mobile, assets

ui-web        -> application, domain, assets
ui-mobile     -> application, domain, assets

application   -> domain, platform
platform      -> domain (optional, minimal)
domain        -> no internal package dependencies
```

Rules:

- `domain` must remain framework-agnostic.
- `application` must never import from `ui-web` or `ui-mobile`.
- `ui-web` and `ui-mobile` must never import each other.
- `platform` should expose common contracts and resolution helpers.
- App entrypoints are responsible for wiring providers and app lifecycle concerns.

## How Sharing Should Work

### Shared Business Logic

Put these in `packages/domain` and `packages/application`:

- Authentication flows
- Order placement rules
- Portfolio aggregation
- Watchlist management
- Market data transformation
- Shared hooks and state orchestration

### Shared Platform APIs

Put these in `packages/platform`:

- HTTP client
- Token storage abstraction
- Session persistence abstraction
- Feature flag API
- Analytics API
- Notification API
- Device capability API

Implementation strategy:

- Shared types and factories in common files
- Browser implementation in `*.web.ts`
- Native implementation in `*.native.ts`
- Resolver exports chosen by bundler/platform rules

### Separate Platform UI

Keep UI isolated by platform:

- Web screens and components live in `apps/web` and `packages/ui-web`
- Mobile screens and components live in `apps/mobile` and `packages/ui-mobile`

Do not attempt to share:

- Screen components
- Layout systems
- Navigation containers
- DOM-specific or React Native-specific widgets

## Example Feature Slice

For an `orders` feature:

```text
packages/domain/src/services/orders/
packages/application/src/use-cases/orders/
packages/platform/src/api/network/
packages/ui-web/src/components/orders/
packages/ui-mobile/src/components/orders/
apps/web/src/screens/orders/
apps/mobile/src/screens/orders/
```

Flow:

1. UI gathers user input.
2. Application use case validates and transforms data.
3. Domain rules enforce trading behavior.
4. Platform API sends the request and persists session data.
5. Platform-specific UI renders success, error, and loading states independently.

## Project Setup Sequence

1. Initialize pnpm workspace and Turborepo.
2. Create `apps/web` with React + TypeScript using Vite.
3. Create `apps/mobile` with React Native Community CLI + TypeScript.
4. Add shared packages: `domain`, `application`, `platform`, `assets`, `config`.
5. Add separated UI packages: `ui-web`, `ui-mobile`.
6. Configure TypeScript path aliases and package exports.
7. Configure platform file resolution for web and native.
8. Add lint, test, and build pipelines in Turborepo.
9. Implement one vertical slice end-to-end, such as auth or watchlist.
10. Scale feature-by-feature after validating the boundaries.

## Tamagui Setup (Specific)

This section defines where Tamagui configuration lives, how providers are wired, and how package boundaries are enforced.

### Recommended Tamagui File Locations

```text
packages/
  ui-core/
    src/
      tamagui/
        tokens.ts
        themes.ts
        shorthands.ts
        animations.ts
        media.ts
        config.ts
      index.ts
    package.json
    tsconfig.json

  ui-web/
    src/
      providers/
        AppProviders.tsx
      components/
      screens/
      index.ts

  ui-mobile/
    src/
      providers/
        AppProviders.tsx
      components/
      screens/
      index.ts
```

Notes:

- `ui-core` is shared design-system infrastructure only: tokens, themes, and tamagui config.
- `ui-web` and `ui-mobile` own platform-specific composed UI.
- Do not place web or native screen components in `ui-core`.

### Provider Wiring

#### Web Provider Wiring

Location:

- `apps/web/src/providers/AppProviders.tsx`

Example:

```tsx
import { PropsWithChildren } from 'react';
import { TamaguiProvider } from 'tamagui';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import tamaguiConfig from '@tradenow/ui-core/tamagui/config';

const queryClient = new QueryClient();

export function AppProviders({ children }: PropsWithChildren) {
  return (
    <TamaguiProvider config={tamaguiConfig} defaultTheme="light">
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    </TamaguiProvider>
  );
}
```

Web app entrypoint (`apps/web/src/main.tsx`) should wrap the root router/app with `AppProviders`.

#### Mobile Provider Wiring

Location:

- `apps/mobile/src/providers/AppProviders.tsx`

Example:

```tsx
import { PropsWithChildren } from 'react';
import { TamaguiProvider } from 'tamagui';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import tamaguiConfig from '@tradenow/ui-core/tamagui/config';

const queryClient = new QueryClient();

export function AppProviders({ children }: PropsWithChildren) {
  return (
    <TamaguiProvider config={tamaguiConfig} defaultTheme="light">
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    </TamaguiProvider>
  );
}
```

Mobile app entrypoint should wrap `NavigationContainer` (or root navigator) with `AppProviders`.

### Package Boundaries with Tamagui

Use these strict boundaries:

- `packages/ui-core`
  - Allowed: tamagui config, tokens, themes, typed design primitives.
  - Forbidden: route-level screens, navigation containers, web DOM logic, native-only modules.
- `packages/ui-web`
  - Allowed: web-only composed components and screen UI using `ui-core` primitives.
  - Forbidden: React Native-specific modules.
- `packages/ui-mobile`
  - Allowed: native-only composed components and screen UI using `ui-core` primitives.
  - Forbidden: browser DOM APIs.

Dependency direction:

```text
ui-web      -> ui-core, application, domain, assets
ui-mobile   -> ui-core, application, domain, assets
ui-core     -> assets
```

This preserves shared tokens/themes while maintaining separate platform UI implementations.

### Tamagui Ownership Rules

- All token and theme changes start in `packages/ui-core/src/tamagui/`.
- Platform-specific spacing/layout exceptions stay in `ui-web` or `ui-mobile`.
- Screens are always platform packages or app folders, never `ui-core`.
- Avoid importing app code into any UI package.

## Initial Aliases Recommendation

```text
@tradenow/domain
@tradenow/application
@tradenow/platform
@tradenow/ui-web
@tradenow/ui-mobile
@tradenow/assets
@tradenow/config
```

## Suggested First Milestone

Build these first:

1. Authentication flow
2. Session storage abstraction
3. Market watchlist
4. Portfolio overview
5. Order ticket submission flow

These features will prove the separation between:

- shared domain rules
- shared app orchestration
- shared platform APIs
- separate web and mobile UI

## Recommended Next Step

Scaffold the monorepo first, then implement a single feature slice such as authentication using the exact package boundaries above. If the auth slice remains clean, the rest of the trading application will scale more predictably.