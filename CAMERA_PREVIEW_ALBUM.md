# Camera preview without album persistence

Based on `0be60ebf164a0861cac642e73f769e884abaa689`, matching the integrating app's pinned dependency.

`ZLCustomCamera.saveToAlbum` defaults to `true`, preserving the PHAsset preview path.
When `ZLPhotoConfiguration.default().afterTakePhotoDidPreview` is true and `saveToAlbum` is false,
the same `ZLPhotoPreviewController` displays the in-memory photo using the existing local-image cell.
Back retakes the photo; editing, original selection, zoom and confirmation remain available.
The result is delivered through `photoPreviewDoneBlock(image, isOriginal)` after both controllers dismiss.
An empty selection returns a nil image. Direct-edit camera usage is unaffected.

No PHAsset is created, no photo-library request is made, and no temporary disk image is needed in memory mode.
The original image byte count is retained for the original-size label. No camera changes are global configuration.

## Manual verification (pending)

- Compare true/false: identical preview controls; false creates no album entry, including after editing and retaking.
- With album access denied, false still previews, edits and returns a photo.
- Toggle original on/off; confirm isOriginal and image data match the user's choice.
- Edit, cancel editing, edit again, confirm: editor state and final image are retained.
- Deselect and confirm: nil result; tap confirm twice: one completion.
- Back from preview, retake, then cancel camera: no image is persisted and cancellation occurs once.
- Rotate and resize: image zoom and toolbar layout follow the container.
- Regress default/true PHAsset preview and afterTakePhotoDidPreview == false (chat direct editing).

## Build

The integrating HikMoa app now requires iOS 15 on both Xcode 26 and Xcode 27.
The locally built Xcode 27 / iOS 15 xcframework can therefore be used directly.
The source API still defaults to album persistence for existing consumers.

For each destination (`generic/platform=iOS` and `generic/platform=iOS Simulator`), run:

```sh
xcodebuild archive -project ZLPhotoBrowser.xcodeproj -scheme ZLPhotoBrowser \
  -configuration Release -destination 'generic/platform=iOS' \
  -archivePath /tmp/ZLPhotoBrowser-device.xcarchive \
  SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES \
  IPHONEOS_DEPLOYMENT_TARGET=15.0 CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO
```

Use a separate archive path for the simulator. Combine both archived frameworks and their dSYMs with
`xcodebuild -create-xcframework`. The simulator archive preserves arm64 and x86_64 slices.
Do not edit the integrating app's Carthage/Checkouts copy; update its generated Carthage/Build artifact and pin the fork commit.

## Static self-check

1. Bounds/failure: memory mode bypasses PHAsset array access; nil selection is explicit.
2. Lifecycle: confirmation consumes the callback once; dismissals finish before returning the image; back retakes.
3. Memory/disk: callbacks weakly reference controllers, no new file storage; photos release with the camera/preview.
4. Concurrency: existing camera main-thread delivery presents the preview; no new queue or blocking wait.
5. Crashes: local cell is registered before dequeue; no invented PHAsset or model force unwrap.
6. Performance: reuses the image and preview cell; byte size comes from captured data, avoiding a second JPEG encoding.

Build status (2026-11-10): final device and simulator archives both succeeded with Xcode 27 / iOS 15 minimum. Functional checks above remain pending manual verification.
