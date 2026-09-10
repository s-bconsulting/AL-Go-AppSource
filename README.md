# AL-Go AppSource App Template

This template repository can be used for managing AppSource Apps for Business Central.

Please go to https://aka.ms/AL-Go to learn more.

## ⚠️ Custom workflow: Update AppSource App For Customer — TEST BEFORE REAL USE ⚠️

**`.github/workflows/UpdateAppSourceAppForCustomer.yaml`** calls the Business Central Admin Center API directly (`useEnvironmentUpdateWindow`) to schedule an AppSource app update on a **customer's** environment. This workflow is **not validated end-to-end yet** (unlike `DeployToSaaS.ps1`, which was tested live).

**Before using it on a real customer:** run it once against a **fake/throwaway production environment** (not a real customer's live environment) to confirm authentication, the `availableUpdates` lookup, and the scheduled update all behave as expected. Only use it against real customers once you've done that.

## Contributing

Please read [this](https://github.com/microsoft/AL-Go/blob/main/Scenarios/Contribute.md) description on how to contribute to AL-Go for GitHub.

We do not accept Pull Requests on the template repository directly.
