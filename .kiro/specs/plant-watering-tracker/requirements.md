# Requirements Document

## Introduction

The Plant Watering Tracker is a React Native (Expo) iOS application that helps users manage the care of their houseplants. Users photograph their plants, which are automatically identified via the Plant.id API v3. The app derives a watering schedule from botanical data, sends daily push notifications at 9 AM reminding users which plants need attention, and presents a dashboard with at-a-glance health status for every plant. The MVP stores all data locally on-device (no backend), works offline except for species recognition, and is distributed via TestFlight using Expo EAS Build.

---

## Glossary

- **App**: The Plant Watering Tracker React Native iOS application.
- **PlantRepository**: The local SQLite data-access component responsible for all plant CRUD operations.
- **SpeciesRecognitionService**: The component that communicates with the Plant.id API v3 to identify plant species and derive watering preferences.
- **NotificationScheduler**: The component that manages scheduling and cancellation of local iOS push notifications.
- **DashboardScreen**: The main home screen displaying the plant list and summary statistics.
- **Plant**: A persisted record representing a single houseplant owned by the user.
- **waterIntervalDays**: The number of days between successive waterings for a given plant.
- **nextWaterDate**: The derived date on which a plant is next due to be watered, stored for fast querying.
- **lastWateredAt**: The ISO 8601 timestamp of the most recent watering action for a plant.
- **lastSprayedAt**: The ISO 8601 timestamp of the most recent spraying action for a plant.
- **birthDate**: An optional user-supplied ISO 8601 date representing when the plant was acquired or sprouted, used to compute display age.
- **EAS Build**: Expo Application Services cloud build service used to produce iOS `.ipa` files without a local Mac.
- **TestFlight**: Apple's beta distribution platform used to deliver the app to the recipient.

---

## Requirements

### Requirement 1: Add a Plant via Photo

**User Story:** As a user, I want to photograph or upload a picture of my plant so that the app can identify the species and set up a watering schedule automatically.

#### Acceptance Criteria

1. WHEN a user taps "Add Plant", THE App SHALL open the device camera or photo picker.
2. WHEN a photo is selected or captured, THE SpeciesRecognitionService SHALL compress the image to ≤ 1 MB before sending it to the Plant.id API v3.
3. WHEN the Plant.id API v3 returns a successful response, THE SpeciesRecognitionService SHALL return a ranked list of species suggestions each containing a common name, scientific name, confidence score, and a derived `waterIntervalDays`.
4. WHEN the Plant.id API v3 `watering.max` value is 1, THE SpeciesRecognitionService SHALL set `waterIntervalDays` to 14.
5. WHEN the Plant.id API v3 `watering.max` value is 2, THE SpeciesRecognitionService SHALL set `waterIntervalDays` to 7.
6. WHEN the Plant.id API v3 `watering.max` value is 3, THE SpeciesRecognitionService SHALL set `waterIntervalDays` to 3.
7. WHEN the recognition result is displayed, THE App SHALL allow the user to confirm or override the species name, nickname, watering interval, spray interval, and birth date before saving.
8. WHEN a user saves a new plant, THE PlantRepository SHALL insert a plant record with a UUID v4 `id`, the current timestamp as `addedAt`, and `nextWaterDate` computed as `addedAt + waterIntervalDays`.
9. WHEN a plant is saved, THE NotificationScheduler SHALL reschedule all watering notifications to reflect the new plant.
10. WHEN a plant is saved, THE App SHALL display the new plant on the dashboard.

---

### Requirement 2: Species Recognition Failure Handling

**User Story:** As a user, I want the app to remain usable even when plant identification fails, so that I can still add my plant manually.

#### Acceptance Criteria

1. IF the Plant.id API v3 returns an error or times out, THEN THE SpeciesRecognitionService SHALL return an empty suggestion list without throwing an exception.
2. IF the Plant.id API v3 response does not include a `watering` field, THEN THE SpeciesRecognitionService SHALL return an empty suggestion list without throwing an exception.
3. WHEN the suggestion list is empty, THE App SHALL display an inline message prompting the user to enter the species manually and SHALL pre-fill `waterIntervalDays` with a default value of 7.
4. WHEN the suggestion list is empty, THE App SHALL remain fully functional for manual plant entry.

---

### Requirement 3: Edit and Override Plant Details

**User Story:** As a user, I want to edit my plant's species name, watering interval, and other details after saving, so that I can correct mistakes or update information over time.

#### Acceptance Criteria

1. WHEN a user opens a plant's detail screen, THE App SHALL display the current nickname, species common name, species scientific name, birth date, watering interval, spray interval, last watered date, last sprayed date, and next water date.
2. WHEN a user modifies and saves changes on the plant detail screen, THE PlantRepository SHALL update the plant record with the new values.
3. WHEN `waterIntervalDays` is updated, THE PlantRepository SHALL recompute and store `nextWaterDate` as `lastWateredAt ?? addedAt + new waterIntervalDays`.
4. WHEN changes are saved, THE NotificationScheduler SHALL reschedule all notifications to reflect the updated schedule.
5. IF a user sets `waterIntervalDays` to a value less than 1, THEN THE App SHALL reject the input and display a validation error.
6. IF a user clears the nickname field, THEN THE App SHALL fall back to `speciesCommonName` as the display name.

---

### Requirement 4: Mark Plant as Watered or Sprayed

**User Story:** As a user, I want to mark a plant as watered or sprayed from the dashboard so that the app can track when it next needs attention.

#### Acceptance Criteria

1. WHEN a user taps "Water" on a plant card, THE PlantRepository SHALL update `lastWateredAt` to the current timestamp and recompute `nextWaterDate` as `now + waterIntervalDays`.
2. WHEN a user taps "Spray" on a plant card, THE PlantRepository SHALL update `lastSprayedAt` to the current timestamp.
3. WHEN a mark-watered or mark-sprayed action completes, THE App SHALL update the plant card immediately to reflect the new dates.
4. WHEN a mark-watered action completes, THE NotificationScheduler SHALL reschedule all notifications.
5. WHEN a user taps "Mark All Done" from the notification deep-link screen, THE PlantRepository SHALL update `lastWateredAt` for every plant listed as due today.

---

### Requirement 5: Dashboard Overview

**User Story:** As a user, I want a dashboard that shows all my plants and their watering status at a glance, so that I can quickly see what needs attention.

#### Acceptance Criteria

1. THE DashboardScreen SHALL display a summary row showing the total number of plants, the count of plants due today, and the count of overdue plants.
2. THE DashboardScreen SHALL render a plant card for each plant showing the plant's nickname, last watered date expressed as a relative duration (e.g. "7 days ago"), plant age derived from `birthDate`, and next water date.
3. THE DashboardScreen SHALL sort plant cards in order: overdue plants first, then plants due today, then upcoming plants sorted by `nextWaterDate` ascending.
4. WHEN all plants have been watered and none are overdue or due today, THE DashboardScreen SHALL display a Lottie celebration animation.
5. THE DashboardScreen SHALL provide an "Add Plant" button that navigates to the Add Plant flow.

---

### Requirement 6: Daily Push Notifications

**User Story:** As a user, I want to receive a daily push notification at 9 AM reminding me which plants need watering, so that I never forget to care for them.

#### Acceptance Criteria

1. WHEN the App is first launched, THE NotificationScheduler SHALL request iOS notification permission from the user.
2. WHEN notification permission is granted, THE NotificationScheduler SHALL schedule local notifications for each unique `nextWaterDate` within the next 7 days.
3. THE NotificationScheduler SHALL schedule at most one notification per calendar day.
4. WHEN a notification is scheduled, THE NotificationScheduler SHALL set the trigger time to 9:00 AM local time on the target date.
5. WHEN exactly one plant is due on a given day, THE NotificationScheduler SHALL use the message body "Your {nickname} needs water today".
6. WHEN more than one plant is due on a given day, THE NotificationScheduler SHALL use the message body "{n} plants need water today".
7. WHEN plant data changes (add, update, delete, or mark watered), THE NotificationScheduler SHALL cancel all existing scheduled notifications and recreate them from the current plant list.
8. WHEN a user taps a notification, THE App SHALL deep-link to the dashboard with due plants highlighted.

---

### Requirement 7: Notification Permission Denied Handling

**User Story:** As a user, I want the app to explain why notifications are useful and let me enable them later, so that I don't lose the feature if I initially decline.

#### Acceptance Criteria

1. IF the user denies iOS notification permission, THEN THE App SHALL display an onboarding message explaining the benefit of notifications and offering a "Remind me later" option.
2. WHERE notifications are disabled, THE App SHALL display an "Enable Notifications" button in the settings screen that re-prompts the iOS permission dialog.
3. WHERE notifications are disabled, THE App SHALL remain fully functional for manual plant management.

---

### Requirement 8: Plant Age Display

**User Story:** As a user, I want to see how old each of my plants is, so that I can appreciate their growth over time.

#### Acceptance Criteria

1. WHEN `birthDate` is set on a plant, THE App SHALL display the plant's age as a human-readable string (e.g. "2 years, 3 months").
2. WHEN `birthDate` is null, THE App SHALL display "Unknown" for the plant's age.
3. THE `computePlantAge` function SHALL never throw an exception for any string or null input.

---

### Requirement 9: Delete a Plant

**User Story:** As a user, I want to delete a plant from the app so that I can remove plants I no longer own.

#### Acceptance Criteria

1. WHEN a user confirms deletion on the plant detail screen, THE PlantRepository SHALL remove the plant record from local storage.
2. WHEN a plant is deleted, THE NotificationScheduler SHALL cancel any scheduled notifications for that plant and reschedule remaining plants.
3. WHEN a plant is deleted, THE DashboardScreen SHALL no longer display that plant.

---

### Requirement 10: Local Data Persistence

**User Story:** As a user, I want my plant data to be saved on my device so that it is available offline and persists across app restarts.

#### Acceptance Criteria

1. THE PlantRepository SHALL store all plant records in a local Expo SQLite database.
2. THE PlantRepository SHALL index the `nextWaterDate` column to support efficient querying of due plants.
3. THE PlantRepository SHALL manage SQLite schema migrations to preserve existing data across app updates.
4. IF a SQLite write operation fails, THEN THE App SHALL display an error message "Couldn't save plant — storage may be full" and SHALL NOT silently discard data.
5. THE App SHALL function fully offline for all features except species recognition.

---

### Requirement 11: Image Handling

**User Story:** As a user, I want my plant photos to be stored locally and displayed reliably, so that I can visually identify my plants in the app.

#### Acceptance Criteria

1. WHEN a plant photo is saved, THE App SHALL store the image as a local file using Expo FileSystem and record the local URI in the plant record.
2. WHEN a photo URI is no longer valid (e.g. the file has been deleted), THE App SHALL display a placeholder image and show a toast message "Photo unavailable".
3. THE SpeciesRecognitionService SHALL cache recognition results keyed by image hash to avoid duplicate API calls for the same photo.
4. WHEN `compressImage` is called, THE App SHALL return a JPEG URI of size ≤ `maxSizeKB` without modifying the original file.

---

### Requirement 12: App Settings

**User Story:** As a user, I want to configure notification preferences so that the app fits my daily routine.

#### Acceptance Criteria

1. THE App SHALL persist notification settings (enabled/disabled and notification time) in local storage.
2. WHERE the user has not customised the notification time, THE NotificationScheduler SHALL default to 09:00 local time.
3. WHEN the user changes the notification time, THE NotificationScheduler SHALL reschedule all existing notifications to the new time.

---

### Requirement 13: Deployment via TestFlight

**User Story:** As a developer, I want to distribute the app via TestFlight using Expo EAS Build so that the recipient can install it without requiring a Mac or Xcode on my side.

#### Acceptance Criteria

1. THE App SHALL be buildable using `eas build --platform ios` without requiring a local Mac or Xcode installation.
2. THE App SHALL be distributable to testers via Apple TestFlight by uploading the `.ipa` produced by EAS Build to App Store Connect.
3. THE App SHALL store the Plant.id API key in an `.env` file excluded from version control and expose it via `Constants.expoConfig.extra`.

---

### Requirement 14: Privacy and Security

**User Story:** As a user, I want to know that my data is handled responsibly, so that I can trust the app with my photos.

#### Acceptance Criteria

1. THE App SHALL display a privacy notice during onboarding informing the user that plant photos are sent to the Plant.id API for species recognition.
2. THE App SHALL NOT transmit any personally identifiable information other than plant photos to external services.
3. THE App SHALL store the Plant.id API key outside of version control.
