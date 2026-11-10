# Camera preview and final confirmation

## Source API

Based on 0be60ebf164a0861cac642e73f769e884abaa689, matching the integrating app's original dependency.

ZLCustomCamera.saveToAlbum defaults to true for legacy consumers: the PHAsset preview path creates an album asset before preview.
When afterTakePhotoDidPreview is true and saveToAlbum is false, the original ZLPhotoPreviewController instead previews an in-memory image.
It retains editing, original selection, zoom, confirmation and back-to-retake. The result arrives through photoPreviewDoneBlock after
both preview and camera dismiss. Direct-edit camera usage is unaffected.

## HikMoa JSAPI persistence contract

HikFunc_CaptureCamera has a separate optional JSAPI saveToAlbum argument (default true).
The JSAPI now always sets the picker/source option to false to avoid creating a PHAsset merely to preview.
Only its final confirmed-image handler decides whether to save to Photos:

- Before final confirmation: neither true nor false creates an album asset; cancel/retake writes nothing.
- Confirmation: true saves one final edited image; false skips persistence. The current single-selection editor may auto-confirm on Done.
- The original checkbox controls H5 return quality, not which edited image is saved.
- captureCameraMe and the chat's direct-edit path are unchanged.

This avoids the legacy pre-preview save and the separate edited-asset save controlled by shared saveNewImageAfterEdit configuration.
The core editor itself does not save an asset. In-memory preview does not run ZLFetchImageOperation.

## Confirmed dismissal fix

The user confirmed saveToAlbum=false now dismisses normally. Their follow-up trace shows the editor first dismissing,
then a preview dismiss request with no presented editor, then a camera dismiss request with no presented preview;
final camera completion has attached=false and presenter=nil, followed by one manager result.

Both the full editor and crop-only editor wait for their dismissal before returning a synchronous in-memory preview result.
Asset preview keeps its legacy callback ordering. No fixed delay or extra dismiss retry is used.
Temporary lifecycle/ownership logs and their 0.8-second observation timers have now been removed after user confirmation.
Only the integrating JSAPI's small final-save checkpoints remain while the persistence contract is verified.

## Manual verification (pending unless noted)

- Confirmed by user: false mode does not persist and edited-image completion exits normally.
- true/default: take a photo, stay in preview, edit or retake/cancel; no album entry before final confirmation.
- true/default: confirm an edited photo with an obvious mark/crop; Photos contains exactly the final edited result, one image per acceptance.
- false: edit/confirm returns the image with no album asset. Preview controls and original selection remain available.
- Editing cancellation, preview back, camera cancel, and crop-only mode preserve their expected state.
- In the current single-selection flow, editor Done auto-confirms and returns; save=true therefore persists at that acceptance.
- Regress chat camera editing and captureCameraMe. iPad should return/save its final edited image.
- Verify the H5 original flag, cache file and optional base64. Album save failure produces a failure callback once.

Filter the host console by [HikCameraTrace]. build=camconfirm-20261110c identifies the final-confirm JSAPI:
jsapi.present (previewAlbumWrite=0), jsapi.confirmed, then either jsapi.album.skipped or jsapi.album.request/result.
The request must not precede confirmed. Logs contain flags/error code only, no photo data, paths, names or asset identifiers.
System Photos UI visibility is a separate observation from the Photos transaction completion; no unsupported timing guarantee is inferred from either.

## Build

HikMoa now requires iOS 15 under both Xcode 26 and 27. Archive the ZLPhotoBrowser scheme separately for
generic/platform=iOS and generic/platform=iOS Simulator with Release, SKIP_INSTALL=NO,
BUILD_LIBRARY_FOR_DISTRIBUTION=YES, IPHONEOS_DEPLOYMENT_TARGET=15.0 and signing disabled.
Combine both archived frameworks and their dSYMs using xcodebuild -create-xcframework.
Preserve device arm64 and simulator arm64/x86_64. Do not edit the integrating app's Carthage/Checkouts copy.

## Static self-check

1. Bounds: memory mode bypasses PHAsset array access; nil selection and released controllers remain safe.
2. Lifecycle: result consumption still prevents double return; editor → preview → camera dismissals are ordered.
3. Memory/disk: callback lifetime is independent of controller release; no new temporary disk image or cache.
4. Concurrency: UI work stays in the existing main-thread flow; no new locks or delayed transition actions.
5. Crashes: mode access is read-only/optional, local cell registration is retained, no invented PHAsset/force unwrap.
6. Performance: local preview reuses the image; diagnostic timers/logs were removed, and no asset readback is needed by the JSAPI.

Build verification is recorded in the integrating PR; final-confirm album behavior remains pending user validation.
