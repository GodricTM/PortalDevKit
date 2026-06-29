# XDA forum notes

- Status: reported
- Source: [Anyone been able to do anything with a Facebook Portal?](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal.3878505/)
- Scope: forum crumbs that help explain stock Portal behavior, setup flow, and recovery paths

## Summary

The thread is mostly speculation, but a few posts are useful because they describe how stock Portal
actually behaved: setup DNS requirements, the built-in test harness path, the state of voice
features, and the fact that USB-C was treated as a factory/debug interface.

## Useful points

1. `cmd testharness enable` was reported to reset the device while preserving ADB, skipping parts of
   setup, and making it possible to get past the login flow by installing a launcher over ADB.
   Reported around [page 11](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal/page-11).
2. The setup flow was reported to depend on DNS for `www.facebook.com`, `graph.facebook.com`, and
   `graph-portal.facebook.com` during different steps such as software update and Portal name
   provisioning. Reported around [page 14](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal/page-14).
3. Forum posts suggest that browser and Bluetooth still worked on some units, while Alexa and the
   old "Hey Portal" path were already dead or unreliable on newer firmware. Reported around
   [page 14](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal/page-14)
   and [page 15](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal/page-15).
4. The stock browser was reported to be throttled heavily enough that YouTube and Spotify became
   barely usable. Reported around [page 6](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal/page-6).
5. One user noted that Spotify appeared in the app store on their device even when it was missing
   from the music-services list, which is a useful clue that app-store visibility and launcher/service
   integration were separate paths. Reported around
   [page 15](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal/page-15).
6. Multiple later posts reinforce that the rear USB-C port was treated by Facebook as a factory-only
   interface, which matches the dump findings and the AOA/accessory path investigation. Reported
   around [page 18](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal/page-18)
   through [page 21](https://xdaforums.com/t/anyone-been-able-to-do-anything-with-a-facebook-portal/page-21).

## Why keep this

These notes are useful when we need to explain why Portal behaves the way it does today, especially
for setup work, factory-style recovery paths, and the line between stock services and user-installable
apps.

