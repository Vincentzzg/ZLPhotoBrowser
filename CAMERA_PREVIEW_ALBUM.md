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


## Runtime diagnostic build camdiag-20261110a (pending user logs)

The user confirmed saveToAlbum=false does not persist photos, but editing/finishing can leave ZLCustomCamera visible without controls.
The screenshot identifies the exposed camera controller; it does not prove an iOS 27 compatibility defect.
This revision adds diagnostics only and does not reorder callbacks or dismiss requests.

Filter Xcode console by `[HikCameraTrace]`. Logs contain controller identities/ownership, transition flags,
effective save/preview options, and album transaction events; no images, paths, names or asset identifiers.
The memory-preview entry includes build=camdiag-20261110a to verify the loaded binary.

Reproduce on the same device:
1. false: shoot → preview → confirm without editing (control case).
2. false: shoot → preview → edit something → finish (reported issue); wait two seconds when it sticks.
3. false: edit → cancel editing → confirm; then preview → back → retake.
4. true (remove any local forced-false override): shoot → wait in preview → confirm. Compare album.save.transaction/completion timestamps with preview appearance and confirmation, since the user observes album visibility only after confirming.

Return the filtered log from manager.cameraEntry through the stuck state / final manager result.
Key distinctions: editor.callback.beforePreviewCallback → preview.edit.autoDoneDecision → preview.done.entry;
which controller is presented when camera.previewDismiss.request runs; whether previewDismiss.completion and cameraDismiss.completion fire;
postcheck ownership at 0.8 seconds. The timer only observes; it never drives dismissal.

Diagnostic static self-check: optional/released controllers are logged safely; all new lifecycle overrides call super;
closures use weak controller references; no new file storage, shared mutable state, UI mutation or per-frame logging.
Both framework archives and host build must pass before handing over. Functional resolution remains pending runtime logs.
Temporary logs will be removed or made switchable after the user confirms the fix.
