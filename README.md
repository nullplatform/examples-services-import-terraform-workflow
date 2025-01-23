### Service Import Terraform Workflow
This repository contains an example on how you can adapt your GitHub Actions workflows to be imported as a nullplatform service.

The process is:

1. Copy the .github/workflows/provisioning.yml to your project
2. Set the `uses` property in the `Trigger workflow` stage with your original workflow name.
3. Create your service spec with **this api call**
4. Create a channel notification with **this api call**. Replace $repo for you repo name, $service-spec-slug with the slug created in step 3.
