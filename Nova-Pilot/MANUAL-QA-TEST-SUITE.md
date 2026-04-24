# Nova Manual QA Test Suite

## Release Snapshot
- App version: `0.3.2`
- Stage baseline: `7367959`
- This suite covers the current Nova pilot after the advisory-output refactor, page lifecycle tightening, welcome-screen polish, production shadow gating, score-chart rollout, and replay read-only hardening.

## Scope
- welcome dialog
- English and Spanish selection
- replay and simulation mode
- workspace input flow
- draft autosave
- analysis progress flow
- review and confirmation flow
- advisory results flow
- progressive reveal sections
- advanced detail and shadow compare
- page generation lifecycle
- generated page routes
- shareable link flow
- shared results route
- direct PDF download
- full-suite submit flow
- Supabase save flow
- production vs stage shadow gating
- desktop and mobile behavior

## Environments
- local dev on `localhost`
- stage preview
- production host from `VITE_PUBLIC_APP_URL`
- desktop Chrome or Chromium
- mobile width around `390px`
- incognito or cleared storage for first-run welcome and share/shadow checks

## Test Data
- vendor name
- vendor type
- one valid public page link
- business description with at least `20` words
- `5` to `10` real work photos
- one client conversation
- at least `3` completed jobs
- one saved session in Supabase for replay testing
- one session with generated pages ready for share/PDF checks

## Core Flow Cases

### QA-001 Welcome dialog opens on first visit
- Area: Welcome
- Precondition: Fresh browser with no stored welcome state
- Steps:
  1. Open `/`.
- Expected:
  1. Welcome dialog opens immediately.
  2. `What is Nova?` content is visible.
  3. The language strip appears at the top.
  4. Replay card is visible on the right.

### QA-002 Welcome content is compact and visually highlighted
- Area: Welcome
- Precondition: Welcome dialog open
- Steps:
  1. Review the left-side welcome content.
- Expected:
  1. `Evidence-backed, not guesses` appears as a highlighted badge.
  2. The three outcome rows are title-only with no extra body copy.
  3. The discovery line highlights `show up better across more platforms`.
  4. Platform chips are visible for ChatGPT, Claude, Gemini, Perplexity, Google AI Overviews, and Copilot.
  5. The trademark disclaimer appears below the platform chips.

### QA-003 Language strip works and mobile header stays clean
- Area: Welcome
- Precondition: Welcome dialog open on desktop and mobile
- Steps:
  1. Select `Español`.
  2. Review the title, intro, outcomes, and replay copy.
  3. Switch back to `English`.
  4. Check the top-right close button on mobile width.
- Expected:
  1. Language switches all welcome copy between English and Spanish.
  2. `English` and `Español` stay left-aligned within the strip.
  3. The close button does not overlap the strip on mobile.
  4. The dialog remains scrollable on mobile.

### QA-004 Welcome auto-shows only once and replay stays capped
- Area: Welcome
- Precondition: Welcome dialog has already been dismissed once in the same browser
- Steps:
  1. Return to `/`.
  2. Refresh the app.
  3. Open a shared link and come back to `/`.
  4. Reopen the welcome dialog manually if the UI provides that action.
- Expected:
  1. The welcome dialog does not auto-open again after the first dismissal.
  2. Returning from a shared link lands in the workspace, not back in welcome.
  3. Replay list shows at most `3` saved sessions.
  4. Manual reopening does not reset the current workspace state.

### QA-005 Replay loads a simulation session cleanly
- Area: Replay
- Precondition: At least one saved session exists in Supabase
- Steps:
  1. Open the welcome dialog.
  2. Click `Load` on a replay item.
- Expected:
  1. The saved session loads into the workspace.
  2. Results, confirmed work, and generated pages restore.
  3. A strong `Read only` simulation notice appears near the top.
  4. Replay list still stays capped at `3` items in the welcome dialog.

### QA-006 Simulation mode keeps edit actions locked
- Area: Replay
- Precondition: Replay session loaded
- Steps:
  1. Try editing vendor fields.
  2. Try editing jobs and extracted review cards.
  3. Try upload, remove, or replace photos.
  4. Open advanced detail.
  5. Open a generated page if pages exist.
  6. Click `Vendor Analysis`.
- Expected:
  1. Workspace inputs are locked.
  2. Review edit actions are locked.
  3. Photo actions stay locked and no new files are accepted.
  4. Advanced detail still opens.
  5. Generated page open actions still work.
  6. `Vendor Analysis` exits simulation mode and starts a fresh editable run.

### QA-007 Start CTA opens the editable workspace
- Area: Welcome
- Precondition: Welcome dialog open
- Steps:
  1. Click `Ready for your business?`
- Expected:
  1. Welcome dialog closes.
  2. Editable workspace opens.
  3. Workspace header and `Input summary` row are visible.

### QA-008 Business info unlocks the job-data gate
- Area: Workspace
- Precondition: Fresh editable workspace
- Steps:
  1. Fill vendor name.
  2. Pick vendor type.
  3. Enter one valid public link.
  4. Enter a business description with at least `20` words.
  5. Observe the lower input area before and after those fields are complete.
- Expected:
  1. `Job data flow` does not show initially.
  2. A small `Ready for job data?` chip appears only after business info is complete.
  3. Clicking the chip reveals the `Job data flow` card.
  4. The chip disappears after the card is revealed.

### QA-009 Focus guidance stays smooth and URL validation is stable
- Area: Workspace
- Precondition: Fresh editable workspace on desktop and mobile
- Steps:
  1. Type the first character in `Vendor Name`.
  2. Continue through vendor type, website, description, and work inputs.
  3. Enter only `https://` in the website field.
  4. Replace it with a valid public URL.
- Expected:
  1. Focus does not jump away from `Vendor Name` after the first character on mobile.
  2. Guided focus highlight is visible without extra `Done / Current / Next` chips.
  3. Entering only `https://` does not count as a valid public link.
  4. Analyze remains locked until a real usable URL is present.

### QA-010 Business description rule and draft autosave work
- Area: Workspace
- Precondition: Editable workspace
- Steps:
  1. Enter fewer than `20` words in `Describe business min 20 words`.
  2. Observe the helper text.
  3. Increase the field to `20` or more words.
  4. Wait for the autosave state.
- Expected:
  1. The hint says `Add 20 words about the business.`
  2. Word count shows as `x/20 words`.
  3. The requirement clears at `20` words.
  4. Save status uses `Saving draft to Supabase...` and then `Draft saved.`

### QA-011 Photo upload rules work
- Area: Workspace
- Precondition: Editable workspace
- Steps:
  1. Upload `5` photos.
  2. Review the work-photo area on desktop and mobile.
  3. Upload more photos up to the pilot limit.
- Expected:
  1. `5` photos are enough to satisfy the confidence threshold.
  2. The upload area remains stable on desktop and mobile.
  3. Mobile `Choose files` control stays compact and aligned beside the work-photo area.
  4. Nova uses no more than the first `10` photos in pilot.

### QA-012 Conversation field supports manual paste and `.txt` import
- Area: Workspace
- Precondition: Editable workspace
- Steps:
  1. Paste one client conversation into the text area.
  2. Use the `Add .txt` action with a plain-text file.
- Expected:
  1. The conversation textarea remains usable and readable.
  2. Imported text fills the same conversation field.
  3. The `.txt` action stays visually compact inside the field area.

### QA-013 Jobs entry flow stays editable
- Area: Workspace
- Precondition: Job data flow revealed
- Steps:
  1. Paste multiple completed jobs one per line in the first jobs box.
  2. Add another job field using `+ Job`.
  3. Type into the second job field one character at a time.
- Expected:
  1. The jobs area stays editable after each keystroke.
  2. The second field does not auto-save and blur after the first character.
  3. Minimum `3` completed jobs are required before Analyze can unlock.

### QA-014 Analyze progress, background mode, and cancel work
- Area: Analysis
- Precondition: Analyze is enabled
- Steps:
  1. Click `Analyze`.
  2. Watch the progress overlay.
  3. Use `Background`.
  4. Reopen with `Open results`.
  5. Run another analysis and cancel it.
- Expected:
  1. Progress overlay shows stage-by-stage progress.
  2. Background tracker appears and updates correctly.
  3. `Open results` returns focus to review/output after completion.
  4. Cancel stops the run and unlocks the workspace.
  5. No broken partial output remains after cancel.

### QA-015 Review cards edit, confirm, and save correctly
- Area: Review
- Precondition: Analysis completed
- Steps:
  1. Confirm one extracted event.
  2. Edit another event.
  3. Save the change.
  4. Refresh or replay the session.
  5. On mobile width, review the confirmed-events area.
- Expected:
  1. Review cards update cleanly.
  2. Save state moves through `Saving...` and then `Saved`.
  3. `Total`, `Confirmed`, and `Target` metrics update correctly.
  4. Persisted state restores without drift after refresh or replay.
  5. On mobile, confirmed events can stay collapsed under one compact label row until opened.

### QA-016 Guided tips appear by section and localize
- Area: Review and Results
- Precondition: Analysis completed
- Steps:
  1. Scroll into Review.
  2. Scroll into `Outcome`.
  3. Scroll into `Pages`.
  4. Scroll into `Share`.
  5. Repeat the same flow in Spanish.
- Expected:
  1. Each key section shows one floating guided tip at the right time.
  2. Tips explain the section clearly and disappear on scroll.
  3. Tips do not stack on top of each other.
  4. Spanish tips appear fully localized with no English fallback.

### QA-017 Outcome section follows the advisory flow
- Area: Results
- Precondition: Analysis completed
- Steps:
  1. Open the `Outcome` section.
  2. Review the first visible content before expanding anything else.
- Expected:
  1. `Where you stand` is the top result heading.
  2. The hero reads as a business mirror, not a raw analytics dump.
  3. `What clients see` chips are visible.
  4. Only the three headline scores are shown initially.
  5. Each headline score includes a compact horizontal score bar below the number.
  6. Deeper sections stay collapsed behind reveal rows by default.

### QA-018 Progressive reveal sections open in sequence
- Area: Results
- Precondition: Analysis completed
- Steps:
  1. Open `why this result`.
  2. Open `growth insight`.
  3. Open `risk / gap`.
  4. Open `generated pages`.
  5. Open `the scores behind the result`.
- Expected:
  1. Each hidden section opens only when requested.
  2. The reveal row label matches the section content.
  3. `How Nova calculated this` appears only in the detailed-score section.
  4. Detailed technical scores stay below the main advisory read.
  5. Numeric detailed score cards include compact horizontal bars.
  6. `Top signal` stays text-only with no bar.

### QA-019 Advanced detail toggle works
- Area: Results
- Precondition: Analysis completed
- Steps:
  1. Click `Show detail`.
  2. If pilot feedback gate appears, respond or skip.
  3. Review the expanded detail cards.
  4. Click `Hide detail`.
- Expected:
  1. Full signal detail remains hidden by default.
  2. Pilot feedback gate appears before the deeper detail on eligible runs.
  3. `What is helping and what is missing` appears once advanced detail is open.
  4. `Keyword edge` shows an overall score bar plus ranked keyword bars when keyword signals exist.
  5. Hiding returns the output to the compact advisory state.

### QA-020 Shadow compare works on local or stage when enabled
- Area: Advanced detail
- Precondition: Local dev or non-production host with shadow enabled
- Steps:
  1. Enable shadow for a non-production host.
  2. Run analysis.
  3. Open advanced detail.
- Expected:
  1. `Shadow enabled` badge appears near advanced detail.
  2. `Dev shadow compare` card appears only after enabling.
  3. Card moves through `running`, `done`, or `error`.
  4. Main analysis output remains unchanged by the shadow run.

### QA-021 Shadow compare is locked off on production
- Area: Advanced detail
- Precondition: Production host using the configured public app URL
- Steps:
  1. Open the production host normally.
  2. Open the production host with `?shadow=1`.
  3. If possible, first enable shadow on a non-production host, then revisit production in the same browser.
  4. Run analysis and open advanced detail.
- Expected:
  1. No `Shadow enabled` badge appears on production.
  2. No shadow compare card appears on production.
  3. Existing runtime shadow storage from non-production does not turn shadow on in production.
  4. Advanced detail still works normally without shadow.

### QA-022 Page generation lifecycle is clear and gated
- Area: Pages
- Precondition: At least `3` events confirmed
- Steps:
  1. Click `Generate Pages`.
  2. Watch the lifecycle through preparation and writing.
  3. Try to open pages before completion.
- Expected:
  1. Pages move through `Preparing Pages...` and `Writing Pages...`.
  2. Progress bar and percentage appear while generation is active.
  3. Page cards do not appear until the state becomes `Pages Ready`.
  4. Opening pages is not possible before the ready state.

### QA-023 Pages become stale and require update after result changes
- Area: Pages and Share
- Precondition: Pages already generated and ready
- Steps:
  1. Change a confirmed event or another page-driving field.
  2. Return to the `Pages` section.
  3. Check the `Share` section.
  4. Click `Update Pages`.
- Expected:
  1. Pages remain visible but show `Pages need an update`.
  2. Primary page CTA changes from `Pages Ready` to `Update Pages`.
  3. Share area warns `Update pages before sharing`.
  4. After regeneration, pages return to ready state and share becomes available again.

### QA-024 Page preview cards stay compact
- Area: Pages
- Precondition: Pages ready
- Steps:
  1. Review page cards on desktop.
  2. Review page cards on mobile width.
- Expected:
  1. Desktop cards show one-line title, one-line slug, and one-line lead text.
  2. Mobile cards show one-line title and one-line slug only.
  3. `Open ↗` stays compact and does not dominate the card.
  4. Desktop page layout stays side-by-side and clean.

### QA-025 Generated page route and back navigation feel correct
- Area: Generated pages
- Precondition: Pages ready
- Steps:
  1. Open a generated page.
  2. Check the top of the page on load.
  3. Click `Back to workspace`.
- Expected:
  1. The generated page opens at the top, not mid-page near the CTA.
  2. Hero, content, FAQ, CTA, and service areas render cleanly.
  3. `Back to workspace` returns to the workspace near the generated pages section, not the very top of the app.
  4. Direct refresh of the generated page route still works.

### QA-026 Share link generation reuses the same code for the same session
- Area: Share
- Precondition: Results saved and pages ready
- Steps:
  1. Click `Generate shareable link`.
  2. Note the returned code.
  3. Click the action again.
  4. Refresh the workspace and generate again.
  5. Update pages and generate again for the same session.
- Expected:
  1. The first request creates one share code.
  2. Repeated requests for the same session reuse the same code.
  3. Refresh does not mint a new code.
  4. Updating the session snapshot updates the same share record instead of creating a second code.

### QA-027 Shared results route matches the advisory result layout
- Area: Share
- Precondition: Valid share link exists
- Steps:
  1. Open the share URL.
  2. Review the page top to bottom.
  3. Toggle `How Nova calculated this`.
- Expected:
  1. Shared route loads from the share code.
  2. Advisory sections are visible in this order: identity, headline scores, confirmed work, why this result, growth insight, risk/gap, generated pages.
  3. Headline scores include the same compact score bars shown in the workspace.
  4. Generated pages show title, slug, and short context only.
  5. `Keyword edge` bars appear in shared advanced detail when keyword signals exist.
  6. Full-suite follow-up block appears at the bottom.
  7. Detailed scores remain hidden until the advanced toggle is opened.

### QA-028 PDF downloads directly and matches the shared result
- Area: Share and PDF
- Precondition: Results visible and pages ready
- Steps:
  1. Click `Download PDF`.
  2. Open the downloaded PDF.
  3. Compare it with the shared result route.
- Expected:
  1. Download starts on the first click.
  2. No blank or black tab remains open.
  3. PDF uses the same overall theme, hierarchy, and section order as the shared result.
  4. Share code and share URL appear near the top.
  5. Headline and detailed numeric scores include matching compact score bars.
  6. `Keyword edge` bars appear in the PDF when keyword signals exist.
  7. Generated pages in the PDF show title, slug, and short context only, not full page bodies.

### QA-029 Full-suite submit works for editable runs
- Area: Final CTA
- Precondition: Editable analysis completed with results visible
- Steps:
  1. Scroll to `Ready for the full suite?`
  2. Confirm vendor name is prefilled.
  3. Enter contact if missing.
  4. Click `Submit`.
- Expected:
  1. Submit shows a loading state.
  2. Success state appears only after persistence completes.
  3. Supabase row is written to `nova_vendor_leads`.
  4. Saved payload includes vendor name, contact, language, and `full_suite` source.

### QA-030 Full-suite area stays read-only in replay mode
- Area: Replay and Final CTA
- Precondition: Replay session loaded
- Steps:
  1. Scroll to the final CTA area.
- Expected:
  1. Replay mode does not show a live submit form.
  2. Follow-up state is clearly read-only.
  3. No lead-submit action is available from replay mode.

### QA-031 Hydration restores saved state cleanly
- Area: Persistence
- Precondition: Saved draft or saved results exist
- Steps:
  1. Fill required inputs and wait for `Draft saved.`
  2. Refresh the app.
  3. Repeat after completing analysis.
  4. Repeat after pages are ready.
- Expected:
  1. Draft input state restores correctly.
  2. Analysis results restore correctly.
  3. Ready pages restore in ready state.
  4. Restored state does not break the workspace or results layout.

### QA-032 Crawl document routes work
- Area: Crawl docs
- Steps:
  1. Open `/robots.txt`.
  2. Open `/sitemap.xml`.
  3. Open `/llms.txt`.
- Expected:
  1. Each route loads without shell breakage.
  2. Content is route-specific and readable.

### QA-033 Mobile layout stays polished across key sections
- Area: Mobile
- Precondition: Mobile width around `390px`
- Steps:
  1. Review the workspace header row.
  2. Review the `Input summary` row.
  3. Review the `Review the work` metrics.
  4. Review the page cards and share section.
- Expected:
  1. `?` and `Reset` stay right-aligned in the `Workspace` row.
  2. `Vendor Analysis` stays beside `Input summary`.
  3. `Input help` is a compact `?` icon.
  4. `Total`, `Confirmed`, and `Target` remain horizontal instead of stacking.
  5. Pages and share controls do not overflow or look oversized.

## Corner Cases

### CC-001 Exactly 4 photos
- Area: Workspace
- Expected:
  1. Photo step is not ready.
  2. Analyze remains disabled.

### CC-002 Exactly 5 photos
- Area: Workspace
- Expected:
  1. Photo requirement becomes ready.
  2. Analyze can unlock if the rest of the inputs are complete.

### CC-003 More than 10 photos
- Area: Workspace
- Expected:
  1. Nova keeps only the first `10` for pilot processing.
  2. Layout stays stable with no duplicate preview corruption.

### CC-004 Vendor name with only spaces
- Area: Workspace
- Expected:
  1. Treated as empty.
  2. Analyze stays disabled.

### CC-005 Website field only `https://`
- Area: Workspace
- Expected:
  1. Treated as incomplete.
  2. Analyze stays disabled until a usable public URL is entered.

### CC-006 Invalid website text
- Area: Workspace
- Expected:
  1. Invalid URL text does not silently pass validation.
  2. Focus remains stable while typing.
  3. Analyze stays locked.

### CC-007 Business description below 20 words
- Area: Workspace
- Expected:
  1. Word threshold stays incomplete.
  2. Helper text remains visible.
  3. Analyze stays disabled.

### CC-008 Only 2 completed jobs
- Area: Workspace
- Expected:
  1. Jobs step is not ready.
  2. Analyze stays disabled.

### CC-009 Very long conversation paste
- Area: Workspace
- Expected:
  1. Text area remains usable.
  2. No overlap, focus loss, or freeze occurs.

### CC-010 Duplicate imported photos from link
- Area: Workspace
- Expected:
  1. Duplicate fetched images are not added twice.
  2. Existing uploaded images stay stable.
  3. Imported count remains accurate.

### CC-011 Empty replay list
- Area: Welcome
- Expected:
  1. Replay area shows a clean empty state.
  2. Main welcome flow still works.

### CC-012 Replay with missing optional fields
- Area: Replay
- Expected:
  1. Session restores without a broken layout.
  2. Missing optional values do not crash the output or final CTA.

### CC-013 Analysis canceled mid-run
- Area: Analysis
- Expected:
  1. No broken partial output remains.
  2. Workspace unlocks correctly.

### CC-014 OpenAI temporary failure
- Area: Analysis
- Expected:
  1. Pilot fallback extraction path works.
  2. Review and result flow still appear.
  3. User is informed cleanly without a fatal crash.

### CC-015 Supabase unavailable during draft save
- Area: Persistence
- Expected:
  1. App remains usable.
  2. Save state shows a clear failure.
  3. No fatal crash occurs.

### CC-016 Generate pages with fewer than 3 confirmed events
- Area: Pages
- Expected:
  1. Generation is blocked or clearly not ready.
  2. No partial page package appears.

### CC-017 Page generation error
- Area: Pages
- Expected:
  1. Error state shows `Page generation failed`.
  2. Existing visible pages remain stable if they already existed.
  3. Retry action is available.

### CC-018 Share before a saved session exists
- Area: Share
- Expected:
  1. Share generation is blocked with a save-first message.
  2. No orphan share row is created.

### CC-019 Shared route while Supabase is slow
- Area: Share
- Expected:
  1. Shared route shows a loading state first.
  2. Final layout resolves cleanly after data arrives.
  3. No broken partial section appears.

### CC-020 Shared link not found
- Area: Share
- Steps:
  1. Open an invalid `/v/XXX-YYY` link.
- Expected:
  1. Clean not-found state appears.
  2. Return path to the workspace is visible.

### CC-021 Download PDF while pages are stale
- Area: Share
- Expected:
  1. Share/PDF actions are blocked or clearly gated.
  2. User sees `Update pages before sharing`.
  3. No stale package is downloaded.

### CC-022 Existing share link reused after refresh
- Area: Share
- Expected:
  1. Same session returns the existing share code instantly.
  2. Refresh does not mint a new code.

### CC-023 Production host with `?shadow=1`
- Area: Advanced detail
- Expected:
  1. Production host still hides all shadow UI.
  2. Query param does not enable the badge or compare card.

### CC-024 Production host after prior stage shadow enablement
- Area: Advanced detail
- Expected:
  1. Previously stored non-production shadow state does not leak into production.
  2. Production advanced detail stays shadow-free.

### CC-025 Spanish mode on mobile
- Area: Mobile
- Expected:
  1. No clipping or overflow in welcome, workspace, result, page, or share sections.
  2. Guided tips remain readable in Spanish.
  3. Buttons and chips stay aligned.

### CC-026 Generated page direct refresh
- Area: Generated pages
- Expected:
  1. Direct route refresh still loads the generated page.
  2. Back path remains available afterward.

### CC-027 Download PDF on first click
- Area: Share
- Expected:
  1. First click starts download immediately.
  2. No second click is needed.
  3. No stale popup blocker message appears from the app itself.

### CC-028 Detailed score section stays secondary
- Area: Results
- Expected:
  1. Detailed score cards remain hidden by default.
  2. They appear only under `The scores behind the result`.
  3. Main advisory read stays primary.

### CC-029 Replay write attempts stay blocked
- Area: Replay
- Expected:
  1. Replay mode ignores any attempt to edit business fields, jobs, notes, photos, or review cards.
  2. Replay mode does not start analysis or page generation.
  3. Replay mode does not submit the full-suite form.

### CC-030 Keyword edge with no keyword signals
- Area: Advanced detail and share
- Expected:
  1. No broken chart shell appears when keyword signals are absent.
  2. The surrounding detail layout remains clean and readable.

## Sign-off
- welcome passed
- replay and simulation passed
- workspace input flow passed
- analysis and review passed
- advisory result flow passed
- pages lifecycle passed
- share and PDF passed
- production shadow lock passed
- mobile checks passed
- Spanish checks passed
- persistence checks passed
- corner cases sampled and acceptable
