# Implementation Plan: Plant Watering Tracker

## Overview

Incremental implementation of a React Native (Expo) iOS app in TypeScript. Each task builds on the previous, starting with the project scaffold and data layer, then services, then screens, and finally deployment configuration. Property-based tests (fast-check) are placed close to the logic they validate.

## Tasks

- [ ] 1. Scaffold Expo project with TypeScript and navigation
  - Run `npx create-expo-app plant-watering-tracker --template expo-template-blank-typescript`
  - Install dependencies: `react-navigation`, `@react-navigation/native-stack`, `@react-navigation/bottom-tabs`, `expo-sqlite`, `expo-camera`, `expo-image-picker`, `expo-image-manipulator`, `expo-file-system`, `expo-notifications`, `lottie-react-native`, `fast-check`, `uuid`
  - Create `src/` directory structure: `components/`, `screens/`, `services/`, `db/`, `hooks/`, `utils/`, `types/`
  - Define all shared TypeScript interfaces in `src/types/index.ts`: `Plant`, `NewPlant`, `SpeciesSuggestion`, `AppSettings`
  - Set up bottom-tab navigator with placeholder screens: Dashboard, Settings
  - Set up stack navigator for Add Plant and Plant Detail flows
  - Configure `app.json` with bundle identifier, permissions (`NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSUserNotificationsUsageDescription`)
  - _Requirements: 13.1_

- [ ] 2. Implement SQLite schema and PlantRepository
  - [ ] 2.1 Create database initialisation and schema migrations in `src/db/database.ts`
    - Open SQLite database with `expo-sqlite`
    - Create `plants` table with all columns matching the `Plant` interface
    - Add index on `nextWaterDate` column
    - Implement schema version tracking for future migrations
    - _Requirements: 10.1, 10.2, 10.3_

  - [ ] 2.2 Implement PlantRepository in `src/db/PlantRepository.ts`
    - Implement `getAllPlants()`, `getPlantById()`, `insertPlant()`, `updatePlant()`, `deletePlant()`, `getPlantsNeedingWaterToday()`
    - `insertPlant` must generate UUID v4 `id`, set `addedAt` to current ISO timestamp, compute `nextWaterDate = addedAt + waterIntervalDays`
    - `updatePlant` must recompute `nextWaterDate` when `waterIntervalDays` changes: `lastWateredAt ?? addedAt + new waterIntervalDays`
    - Wrap all write operations in try/catch; on failure throw a typed `StorageError`
    - _Requirements: 1.8, 3.2, 3.3, 4.1, 4.2, 9.1, 10.1, 10.4_

  - [ ]* 2.3 Write property test for nextWaterDate invariant
    - **Property 1: nextWaterDate consistency** — for any plant `p`, `p.nextWaterDate === addDays(p.lastWateredAt ?? p.addedAt, p.waterIntervalDays)`
    - Use fast-check arbitraries for `waterIntervalDays` (integer 1–365) and ISO date strings
    - **Validates: Requirements 1.8, 3.3**

  - [ ]* 2.4 Write unit tests for PlantRepository
    - Test insert, update, delete, and query operations against an in-memory SQLite instance
    - Test `nextWaterDate` recomputation on `waterIntervalDays` change
    - Test `StorageError` is thrown on write failure
    - _Requirements: 10.1, 10.3, 10.4_

- [ ] 3. Implement utility functions
  - [ ] 3.1 Implement `computePlantAge(birthDate: string | null): string` in `src/utils/plantAge.ts`
    - Return `"Unknown"` when `birthDate` is null
    - Return human-readable string e.g. `"2 years, 3 months"` for valid ISO 8601 dates
    - Wrap in try/catch to ensure it never throws
    - _Requirements: 8.1, 8.2, 8.3_

  - [ ]* 3.2 Write property test for computePlantAge
    - **Property 4: computePlantAge never throws** — for any arbitrary string or null input, `computePlantAge` must not throw
    - Use fast-check `fc.option(fc.string())` as input arbitrary
    - **Validates: Requirements 8.3**

  - [ ] 3.3 Implement `buildNotificationMessage(plants: Plant[]): string` in `src/utils/notifications.ts`
    - Return `"Your {nickname} needs water today"` when `plants.length === 1`
    - Return `"{n} plants need water today"` when `plants.length > 1`
    - _Requirements: 6.5, 6.6_

  - [ ]* 3.4 Write property test for buildNotificationMessage
    - **Property 3: buildNotificationMessage never returns empty string** — for any non-empty array of plants, the result must be a non-empty string
    - Use fast-check `fc.array(plantArbitrary, { minLength: 1 })` as input
    - **Validates: Requirements 6.5, 6.6**

  - [ ] 3.5 Implement `addDays(isoDate: string, days: number): string` and `isToday(isoDate: string): boolean` helpers in `src/utils/dates.ts`
    - Handle leap years and month-end edge cases
    - _Requirements: 1.8, 3.3, 4.1_

- [ ] 4. Checkpoint — Ensure all tests pass
  - Run `npx jest --runInBand` (or equivalent) and confirm all unit and property tests pass. Ask the user if any questions arise.

- [ ] 5. Implement SpeciesRecognitionService
  - [ ] 5.1 Implement image compression in `src/services/imageUtils.ts`
    - Use `expo-image-manipulator` to compress to ≤ 1 MB JPEG without modifying the original file
    - Implement `compressImage(uri: string, maxSizeKB: number): Promise<string>`
    - _Requirements: 1.2, 11.4_

  - [ ]* 5.2 Write property test for compressImage
    - **Property (image size): compressImage output ≤ maxSizeKB** — for any valid image URI and maxSizeKB > 0, the returned file size must be ≤ maxSizeKB
    - Note: test with a set of real fixture images; use fast-check for `maxSizeKB` values
    - **Validates: Requirements 1.2, 11.4**

  - [ ] 5.3 Implement `SpeciesRecognitionService` in `src/services/SpeciesRecognitionService.ts`
    - Call `POST /v3/identification` with `details=watering,common_names` and base64-encoded compressed image
    - Map `watering.max` → `waterIntervalDays`: 1→14, 2→7, 3→3
    - Return `SpeciesSuggestion[]` sorted by confidence descending
    - On API error, timeout, or missing `watering` field: return `[]` without throwing
    - Implement recognition result cache keyed by SHA-256 hash of the image file
    - Read Plant.id API key from `Constants.expoConfig.extra.plantIdApiKey`
    - _Requirements: 1.2, 1.3, 1.4, 1.5, 1.6, 2.1, 2.2, 11.3_

  - [ ]* 5.4 Write unit tests for SpeciesRecognitionService
    - Test successful response mapping for all three `watering.max` values (1, 2, 3)
    - Test that API error returns empty array without throwing
    - Test that missing `watering` field returns empty array
    - Test cache hit avoids second API call
    - _Requirements: 1.3, 1.4, 1.5, 1.6, 2.1, 2.2, 11.3_

- [ ] 6. Implement NotificationScheduler
  - [ ] 6.1 Implement `NotificationScheduler` in `src/services/NotificationScheduler.ts`
    - Implement `requestPermissions(): Promise<boolean>` using `expo-notifications`
    - Implement `rescheduleAll(plants: Plant[]): Promise<void>`: cancel all existing notifications, group plants by `nextWaterDate`, schedule one notification per day for dates within the next 7 days at 9:00 AM local time using `buildNotificationMessage`
    - Implement `cancelForPlant(plantId: string): Promise<void>`
    - Read notification time from `AppSettings` (default `"09:00"`)
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 12.2, 12.3_

  - [ ]* 6.2 Write property test for rescheduleAll notification uniqueness
    - **Property 2: at most one notification per calendar day** — for any list of plants with arbitrary `nextWaterDate` values within the next 7 days, `rescheduleAll` must schedule at most one notification per calendar day
    - Use fast-check `fc.array(plantArbitrary)` with dates constrained to next 7 days
    - **Validates: Requirements 6.3**

  - [ ]* 6.3 Write unit tests for NotificationScheduler
    - Test that `rescheduleAll` cancels all previous notifications before scheduling new ones
    - Test single-plant message format
    - Test multi-plant message format
    - Test that dates beyond 7 days are not scheduled
    - _Requirements: 6.2, 6.3, 6.5, 6.6, 6.7_

- [ ] 7. Checkpoint — Ensure all tests pass
  - Run all tests and confirm services layer is fully covered. Ask the user if any questions arise.

- [ ] 8. Implement DashboardScreen
  - [ ] 8.1 Create `PlantCard` component in `src/components/PlantCard.tsx`
    - Display nickname, last watered relative duration (e.g. "7 days ago"), plant age via `computePlantAge`, next water date
    - Show "Water" and "Spray" action buttons
    - Highlight card visually when plant is overdue or due today
    - Handle invalid `photoUri` by showing a placeholder image and a toast "Photo unavailable"
    - _Requirements: 4.3, 5.2, 11.2_

  - [ ] 8.2 Create stats summary row component in `src/components/StatsRow.tsx`
    - Display total plant count, due-today count, overdue count
    - _Requirements: 5.1_

  - [ ] 8.3 Implement `DashboardScreen` in `src/screens/DashboardScreen.tsx`
    - Load plants from `PlantRepository` on mount and on focus
    - Sort cards: overdue → due today → upcoming by `nextWaterDate` ascending
    - Render `StatsRow` and list of `PlantCard` components
    - Show Lottie celebration animation when no plants are overdue or due today
    - Provide "Add Plant" button navigating to AddPlantScreen
    - Wire "Water" and "Spray" button taps to `markWatered` / `markSprayed` actions that call `PlantRepository.updatePlant` and `NotificationScheduler.rescheduleAll`
    - Display `StorageError` toast "Couldn't save plant — storage may be full" on write failure
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 5.1, 5.2, 5.3, 5.4, 5.5, 9.3, 10.4_

  - [ ]* 8.4 Write unit tests for DashboardScreen sort order
    - Test that overdue plants appear before due-today plants, which appear before upcoming plants
    - Test that Lottie animation renders only when no plants are due or overdue
    - _Requirements: 5.3, 5.4_

- [ ] 9. Implement AddPlantScreen
  - [ ] 9.1 Create camera/gallery picker step in `src/screens/AddPlantScreen.tsx`
    - Use `expo-image-picker` for gallery and `expo-camera` for camera capture
    - On photo selected, call `SpeciesRecognitionService.identifyFromUri` and show loading indicator
    - _Requirements: 1.1, 1.2_

  - [ ] 9.2 Implement recognition result and edit form step
    - Display top suggestion (common name, scientific name, confidence %)
    - Show editable fields: nickname, species name, `waterIntervalDays`, `sprayIntervalDays`, `birthDate`
    - When suggestion list is empty, show inline message "Couldn't identify — enter species manually" and pre-fill `waterIntervalDays` with 7
    - Validate `waterIntervalDays >= 1`; show validation error if not
    - Fall back to `speciesCommonName` as display name when nickname is cleared
    - _Requirements: 1.3, 1.7, 2.3, 2.4, 3.5, 3.6_

  - [ ] 9.3 Implement save action
    - On "Save Plant" tap: compress image, call `PlantRepository.insertPlant`, call `NotificationScheduler.rescheduleAll`, navigate back to Dashboard
    - Store image as local file via `expo-file-system` and record URI in plant record
    - Display `StorageError` toast on failure
    - _Requirements: 1.8, 1.9, 1.10, 10.4, 11.1_

- [ ] 10. Implement PlantDetailScreen
  - [ ] 10.1 Create `PlantDetailScreen` in `src/screens/PlantDetailScreen.tsx`
    - Load plant by `id` from route params
    - Display all fields: nickname, species names, age, `waterIntervalDays`, `sprayIntervalDays`, `lastWateredAt`, `lastSprayedAt`, `nextWaterDate`, photo
    - Handle invalid `photoUri` with placeholder image and toast
    - _Requirements: 3.1, 8.1, 8.2, 11.2_

  - [ ] 10.2 Implement edit and save
    - Editable fields: nickname, species names, `waterIntervalDays`, `sprayIntervalDays`, `birthDate`, notes
    - Validate `waterIntervalDays >= 1`; show error if not
    - Fall back to `speciesCommonName` when nickname is cleared
    - On save: call `PlantRepository.updatePlant`, call `NotificationScheduler.rescheduleAll`
    - _Requirements: 3.2, 3.3, 3.4, 3.5, 3.6_

  - [ ] 10.3 Implement delete plant action
    - Show confirmation dialog before deletion
    - On confirm: call `PlantRepository.deletePlant`, call `NotificationScheduler.rescheduleAll`, navigate back to Dashboard
    - _Requirements: 9.1, 9.2, 9.3_

- [ ] 11. Implement TodayScreen (notification deep-link target)
  - Create `TodayScreen` in `src/screens/TodayScreen.tsx`
  - Load plants due today from `PlantRepository.getPlantsNeedingWaterToday()`
  - Render checklist of due plants with individual "Water" checkboxes
  - Implement "Mark All Done" button that calls `PlantRepository.updatePlant` for each due plant and then `NotificationScheduler.rescheduleAll`
  - Configure notification tap deep-link to navigate to TodayScreen
  - _Requirements: 4.5, 6.8_

- [ ] 12. Implement Settings screen and notification permission handling
  - [ ] 12.1 Create `SettingsScreen` in `src/screens/SettingsScreen.tsx`
    - Display notification enabled/disabled toggle
    - Display notification time picker (default 09:00)
    - Persist settings to `AsyncStorage` as `AppSettings`
    - On time change: call `NotificationScheduler.rescheduleAll`
    - Show "Enable Notifications" button when permission is denied; tapping it re-prompts iOS permission dialog
    - _Requirements: 7.2, 12.1, 12.2, 12.3_

  - [ ] 12.2 Implement permission-denied onboarding message
    - After `NotificationScheduler.requestPermissions()` returns `false`, display an inline message explaining the benefit of notifications with a "Remind me later" option
    - App must remain fully functional when notifications are disabled
    - _Requirements: 7.1, 7.3_

- [ ] 13. Implement onboarding and privacy notice
  - Create `OnboardingScreen` in `src/screens/OnboardingScreen.tsx`
  - Display privacy notice: plant photos are sent to Plant.id API for species recognition
  - Show on first launch only (persist seen flag in `AsyncStorage`)
  - Request notification permission from this screen via `NotificationScheduler.requestPermissions()`
  - _Requirements: 6.1, 14.1, 14.2_

- [ ] 14. Checkpoint — Ensure all tests pass
  - Run full test suite. Verify all screens render without errors and all service integrations are wired correctly. Ask the user if any questions arise.

- [ ] 15. Configure EAS Build and environment variables
  - Create `.env` file with `PLANT_ID_API_KEY=` and add `.env` to `.gitignore`
  - Configure `app.config.ts` to expose `PLANT_ID_API_KEY` via `Constants.expoConfig.extra.plantIdApiKey`
  - Create `eas.json` with `production` build profile targeting iOS
  - Verify `eas build --platform ios` configuration is valid (bundle ID, provisioning)
  - _Requirements: 13.1, 13.2, 13.3, 14.3_

- [ ] 16. Final checkpoint — Ensure all tests pass
  - Run full test suite one final time. Confirm EAS build config is valid. Ask the user if any questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation at key milestones
- Property tests (fast-check) validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- The Plant.id API key must never be committed to version control
