Publish the drift detection and handling POC documentation to Confluence by running each md2confluence command below sequentially. Each command must succeed before proceeding to the next. If any command fails, stop and report the error.

Run these commands in order:

1. `md2confluence docs/projects/universal-control-plane-ucp/pocs/drift-detection-and-handling.md`
2. `md2confluence docs/projects/universal-control-plane-ucp/pocs/drift-detection-and-handling/approach-a-polling-watcher.md`
3. `md2confluence docs/projects/universal-control-plane-ucp/pocs/drift-detection-and-handling/approach-b-watchoperations.md`
4. `md2confluence docs/projects/universal-control-plane-ucp/pocs/drift-detection-and-handling/approach-c-informer-watcher.md`
5. `md2confluence docs/projects/universal-control-plane-ucp/pocs/drift-detection-and-handling/approach-d-temporal-schedule.md`

After all commands succeed, confirm that all 5 documents have been published successfully.
