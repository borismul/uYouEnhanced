# uYouEnhanced Build Fix Documentation

**Date:** January 25, 2026
**Repo:** https://github.com/borismul/uYouEnhanced (fork of arichornlover/uYouEnhanced)
**First working build:** Run #21341355813 — `v21.03.2-3.0.4-(33)`

## What Is This Project?

uYouEnhanced is a jailed (non-jailbroken) YouTube tweak for iOS. It's a meta-project that stitches together ~25 git submodules (tweaks) from different authors, compiles them with [Theos](https://theos.dev), and injects them into a decrypted YouTube IPA.

## Why Was It Broken?

The upstream repo (arichornlover/uYouEnhanced) hasn't had a successful build since **December 14, 2025**. Three categories of problems:

### 1. Missing YouTubeHeader files

The project uses `aricloverEXTRA/YouTubeHeader` as a submodule, but it was missing headers that tweaks needed (e.g. `YTPlayerBarScrubberDotDecorationController.h` needed by YouSlider). PoomSmart's original had some of these but not all.

### 2. Missing `#import <Foundation/Foundation.h>` (SDK 17.5 strictness)

The iOS 17.5 SDK enforces Clang module imports. Code like `NSUserDefaults`, `NSPredicate`, `NSTimer` etc. must have Foundation explicitly imported. Older SDKs were lenient about this. Affected tweaks:
- **Return-YouTube-Dislikes** (`API.x`, `Vote.x`) — missing Foundation import
- **YTUHD** (`Tweak.xm`) — missing Foundation import

### 3. Missing interface declarations

The main `uYouPlus.xm` source code calls selectors that aren't declared in any header:
- `[YTMainAppControlsOverlayView uYou]` — runtime addition from uYou tweak
- `[YTPlayerOverlayManager varispeedSwitchController:didSelectRate:]`
- `watchCollapseButton` property on `YTMainAppControlsOverlayView`
- `YT_NOTIFICATIONS` enum value (missing from PoomSmart's `YTIcon.h`)
- `setTitle:` on `YTIPivotBarItemRenderer` (missing properties in PoomSmart's version)

## What We Fixed

### Submodule URL changes (`.gitmodules`)

| Submodule | Original URL | New URL | Why |
|-----------|-------------|---------|-----|
| YouTubeHeader | `aricloverEXTRA/YouTubeHeader` | **`Borismism/YouTubeHeader`** | Fork of PoomSmart + all aricloverEXTRA headers merged in |
| Return-YouTube-Dislikes | `aricloverEXTRA/Return-YouTube-Dislikes` | **`PoomSmart/Return-YouTube-Dislikes`** | PoomSmart's version has `#import <Foundation/Foundation.h>` |
| YTUHD | `aricloverEXTRA/YTUHD` | **`Borismism/YTUHD`** | Fork with `#import <Foundation/Foundation.h>` added to `Tweak.xm` |
| DontEatMyContent | `aricloverEXTRA/DontEatMyContent` | **kept aricloverEXTRA** | PoomSmart's version uses different header properties (jailbreak-specific) |
| YouTimeStamp | `aricloverEXTRA/YouTimeStamp` | **kept aricloverEXTRA** | No PoomSmart alternative exists |
| YTHoldForSpeed | `aricloverEXTRA/YTHoldForSpeed` | **kept aricloverEXTRA** | No PoomSmart alternative exists |

### Borismism/YouTubeHeader fork

This is a fork of `PoomSmart/YouTubeHeader` with ALL headers from `aricloverEXTRA/YouTubeHeader` merged on top. This was necessary because:

- PoomSmart's headers are the most up-to-date for standard YouTube classes
- aricloverEXTRA added custom headers needed by uYouEnhanced (e.g. `YTICompactListItemRenderer.h`, `YTBrowseViewController.h`, `YTNavigationBarTitleView.h`)
- aricloverEXTRA's `YTIcon.h` has `YT_NOTIFICATIONS = 264` enum value that PoomSmart's doesn't
- aricloverEXTRA's `YTIPivotBarItemRenderer.h` has `title` property that PoomSmart's doesn't
- aricloverEXTRA's `YTMainAppControlsOverlayView.h` has `watchCollapseButton` property

**The fix:** Download ALL `.h` files from aricloverEXTRA and overwrite into the Borismism fork. This ensures we have everything from both sources.

### Borismism/YTUHD fork

Fork of `aricloverEXTRA/YTUHD` with one change: added `#import <Foundation/Foundation.h>` at the top of `Tweak.xm`.

We could NOT use PoomSmart's YTUHD because it requires `libundirect` (a jailbreak library not available in jailed builds).

### Source code changes (`Sources/uYouPlus.h`)

Added missing interface declarations at the bottom of the file:

```objc
// uYou runtime additions
@interface YTMainAppControlsOverlayView (uYou)
- (id)uYou;
@end

@interface YTPlayerOverlayManager (uYouEnhanced)
- (void)varispeedSwitchController:(id)controller didSelectRate:(float)rate;
@end
```

These selectors are added at runtime by the uYou tweak but need compile-time declarations.

## How to Reproduce From Scratch

### 1. Fork the repo

```bash
gh repo fork arichornlover/uYouEnhanced --clone
cd uYouEnhanced
```

### 2. Switch Return-YouTube-Dislikes to PoomSmart

```bash
sed -i '' 's|aricloverEXTRA/Return-YouTube-Dislikes|PoomSmart/Return-YouTube-Dislikes|g' .gitmodules
git submodule sync Tweaks/Return-YouTube-Dislikes
cd Tweaks/Return-YouTube-Dislikes
git remote set-url origin https://github.com/PoomSmart/Return-YouTube-Dislikes.git
git fetch origin && git checkout origin/main
cd ../..
```

### 3. Create YouTubeHeader fork (PoomSmart base + ariclover extras)

```bash
# Fork PoomSmart/YouTubeHeader on GitHub, then:
git clone https://github.com/YOUR_USER/YouTubeHeader.git /tmp/YTH-fix
cd /tmp/YTH-fix

# Download ALL headers from aricloverEXTRA (overwrites PoomSmart's where different)
gh api repos/aricloverEXTRA/YouTubeHeader/contents --jq '.[].name' | grep '\.h$' | while read h; do
  gh api "repos/aricloverEXTRA/YouTubeHeader/contents/$h" --jq '.content' | base64 -d > "$h"
done

# Also get headers from arichornloverALT (has extra headers)
for h in YTICompactListItemRenderer YTICompactListItemThumbnailSupportedRenderers YTIIconThumbnailRenderer YTBrowseViewController YTNavigationBarTitleView YTPivotBarItemView YTIIcon YTICompactLinkRenderer YTIItemSectionSupportedRenderers MLPlayerStickySettings YTHeaderViewController; do
  gh api "repos/arichornloverALT/YouTubeHeader/contents/${h}.h" --jq '.content' | base64 -d > "${h}.h" 2>/dev/null
done

git add . && git commit -m "Merge aricloverEXTRA headers" && git push
```

Then update the submodule in uYouEnhanced:

```bash
cd /path/to/uYouEnhanced
sed -i '' 's|aricloverEXTRA/YouTubeHeader|YOUR_USER/YouTubeHeader|g' .gitmodules
# ... sync and update submodule
```

### 4. Create YTUHD fork (add Foundation import)

```bash
gh repo fork aricloverEXTRA/YTUHD --clone=/tmp/YTUHD-fix
cd /tmp/YTUHD-fix
# Add Foundation import as first line of Tweak.xm
sed -i '' '1i\
#import <Foundation/Foundation.h>
' Tweak.xm
git add . && git commit -m "Add Foundation import for SDK 17.5" && git push
```

Then update the submodule:

```bash
sed -i '' 's|aricloverEXTRA/YTUHD|YOUR_USER/YTUHD|g' .gitmodules
# ... sync and update submodule
```

### 5. Add missing interface declarations

Append to `Sources/uYouPlus.h`:

```objc
// uYou runtime additions
@interface YTMainAppControlsOverlayView (uYou)
- (id)uYou;
@end

@interface YTPlayerOverlayManager (uYouEnhanced)
- (void)varispeedSwitchController:(id)controller didSelectRate:(float)rate;
@end
```

### 6. Commit and trigger build

```bash
git add .gitmodules Sources/uYouPlus.h Tweaks/YouTubeHeader Tweaks/Return-YouTube-Dislikes Tweaks/YTUHD
git commit -m "Fix build: update submodules and headers for SDK 17.5"
git push

gh workflow run "Build and Release uYouEnhanced" \
  -f sdk_version="17.5" \
  -f uyou_version="3.0.4" \
  -f decrypted_youtube_url="YOUR_DECRYPTED_IPA_URL" \
  -f bundle_id="com.google.ios.youtube" \
  -f app_name="YouTube" \
  -f upload_artifact=true \
  -f create_release=true
```

## Installing the IPA

The built IPA is **unsigned**. You CANNOT install it with `xcrun devicectl` or `ideviceinstaller` directly.

Use **Sideloadly** (https://sideloadly.io):
1. Download and install Sideloadly
2. Connect iPhone via USB
3. Drag in the `.ipa` file
4. Enter your Apple ID
5. Click Start — it handles signing + provisioning profile creation

## Key Lessons

1. **aricloverEXTRA forks are patched for jailed builds** — PoomSmart originals often assume jailbroken environment (e.g. `libundirect`)
2. **SDK 17.5 is strict about module imports** — older code needs explicit `#import <Foundation/Foundation.h>`
3. **YouTubeHeader needs BOTH sources** — PoomSmart for base + aricloverEXTRA for uYouEnhanced-specific extras
4. **The upstream project is essentially abandoned** — builds have been broken since Dec 2025, no fixes merged
5. **Header dependency chains** — fixing one missing header often reveals another. Best to download ALL headers at once

## Submodule Architecture

```
uYouEnhanced (main repo)
├── .gitmodules                    ← submodule URL config
├── Sources/uYouPlus.{h,xm}       ← main tweak code (THIS repo, not a submodule)
├── Tweaks/YouTubeHeader           ← Borismism/YouTubeHeader (PoomSmart + ariclover merged)
├── Tweaks/Return-YouTube-Dislikes ← PoomSmart (has Foundation import)
├── Tweaks/YTUHD                   ← Borismism/YTUHD (added Foundation import)
├── Tweaks/DontEatMyContent        ← aricloverEXTRA (jailed-compatible)
├── Tweaks/YouTimeStamp            ← aricloverEXTRA (no alternative)
├── Tweaks/YTHoldForSpeed          ← aricloverEXTRA (no alternative)
├── Tweaks/YouSlider               ← PoomSmart
├── Tweaks/YouPiP                  ← PoomSmart
├── Tweaks/YouSpeed                ← PoomSmart
├── Tweaks/YouQuality              ← PoomSmart
├── Tweaks/YouMute                 ← PoomSmart
├── Tweaks/YouLoop                 ← bhackel
├── Tweaks/YouGroupSettings        ← PoomSmart
├── Tweaks/YTIcons                 ← PoomSmart
├── Tweaks/YTABConfig              ← PoomSmart
├── Tweaks/YTVideoOverlay          ← PoomSmart
├── Tweaks/YTClassicVideoQuality   ← PoomSmart
├── Tweaks/iSponsorBlock           ← Galactic-Dev
├── Tweaks/FLEXing                 ← PoomSmart
├── Tweaks/Alderis                 ← hbang
├── Tweaks/protobuf                ← protocolbuffers
└── Tweaks/uYouLocalization        ← dayanch96
```
