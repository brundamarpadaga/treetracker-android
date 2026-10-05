# Task 23 - Address review findings on upload failure logging

## Goal

Fix the problems found while reviewing the upload failure logging PR (issue
#1317), so that failure types are recorded accurately without changing how
uploads behave.

## Problem

The review ran the PR against `master` on an emulator with induced failures:

1. A tree whose photo failed to upload was sent without a photo, marked
   uploaded and had its local photo deleted (`master` kept it pending).
2. Network failures were labelled `server_failure`: the AWS SDK wraps them in
   `AmazonClientException`, which the `IOException` checks never matched.
3. `failure_type` was always empty in Crashlytics: it was cleared right after
   `recordException`, but Crashlytics reads custom keys later, on a background
   thread.
4. Side effects: one failed tree bundle ended the whole sync, cancelling a sync
   left `is_syncing=true`, and each failure was reported to Crashlytics 3 times.

## Changes

- `TreeUploader`: a failed image upload fails the bundle again (trees stay
  pending, photos stay on disk). The failure is not re-recorded because
  `UploadImageUseCase` already did. A failed bundle no longer aborts the sync.
- `ExceptionDataCollector.recordFailure(throwable, message)` now classifies the
  failure itself (`classify`), replacing the per-uploader `when`/catch blocks.
  `AmazonServiceException` is `server_failure`, an `AmazonClientException` caused
  by an `IOException` is `network_failure`.
- `failure_type` is no longer cleared after each failure. It holds the most
  recent upload failure type and is reset when the next sync starts.
- `TreeSyncWorker` resets `is_syncing` in a `finally`.
- `SyncDataUseCase` logs failures that were already recorded at debug level, so
  each failure reaches Crashlytics once.

## Out of scope

- Attaching `failure_type` to a single event with
  `recordException(Throwable, CustomKeysAndValues)` needs Crashlytics 19.x, i.e.
  a Firebase BoM upgrade (currently 32.8.0). Left for a separate change.

## Verification

- `./gradlew :app:detekt :app:ktlintCheck` pass.
- Unit tests for the classifier, `recordFailure`, and the `TreeUploader`
  behaviours above.
