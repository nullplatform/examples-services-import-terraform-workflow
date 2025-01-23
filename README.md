### Service Import Terraform Workflow
This repository contains an example on how you can adapt your GitHub Actions workflows to be imported as a nullplatform service.

The process is:

### Workflow import configuration

1. Copy the .github/workflows/provisioning.yml to your project. You need to replace the `uses` property in the `worker` stage with your workflow name.
2. Add a `workflow_call` trigger to thee workflow to import. Take a look at (worker example)[.github/workflows/worker.yml] on how to do this.
3. Create your service spec with **this api call**
4. Create a channel notification with **this api call**. Replace $repo for you repo name, $service-spec-slug with the slug created in step 3.

### Workflow notification
The `provisioning` workflow will receive the nullplatform service notification and extract all relevant parameters (ex: function_arn or dynamo_table name) and dimensions (ex: country and environment) and export them as parameters inside a `notification` input.

The process to extract value follows these rules:

1. Parameters are extracted from 3 paths in the notification json:
	a. Highest priority is `.notification.parameters`. These values will always will be present.
	b. Second priority is `.notification.link.attributes`.
	c. Last priority is `.notification.service.attributes`.
2. All values in `.notification.service.dimensions` are added to the parameters as context information.

This is an example of the nullplatform service notification:

```json
{
  "id": 1234,
  "source": "service",
  "event": "service:action:create",
  "created_at": "2025-01-22T17:31:26.807Z",
  "notification": {
    "action": "service:action:update",
    "id": "some-uuid",
    "name": "Action Name",
    "slug": "action-name",
    "status": "pending",
    "created_at": "2025-01-22T17:31:21.670Z",
    "updated_at": "2025-01-22T17:31:23.530Z",
    "parameters": {
      "parameter1": 1234,
      "parameter2": "some-string",
      "parameter3": true
    },
    "results": {},
    "type": "create",
    "specification": {
      "id": "some-uuid",
      "slug": "action-slug"
    },
    "service": {
      "id": "some-uuid",
      "slug": "service-name",
      "attributes": {
        "parameter1": 4321,
        "parameter4": "some-service-value",
        "parameter5": "other-service-value"
      },
      "specification": {
        "id": "some-uuid",
        "slug": "service-spec-slug"
      },
      "dimensions": {
        "country": "arg",
        "environment": "dev"
      }
    },
    "link": {
      "id": "some-uuid",
      "slug": "link-slug",
      "attributes": {
        "parameter1": 111,
        "parameter4": "some-link-value",
        "parameter6": "other-link-value"
      },
      "specification": {
        "id": "some-uuid",
        "slug": "link-slug"
      },
      "dimensions": {
        "environment": "dev",
        "country": "arg"
      }
    },
    "tags": {
	  "organization_id": "1",
      "organization": "organization-slug",
      "account_id": "2",
      "account": "account-slug",
      "namespace_id": "3",
      "namespace": "namespace-slug",
      "application_id": "4",
      "application": "application-slug",
      "scope_id": "5",
      "scope": "scope-slug"
    },
    "entity_nrn": "organization=1:account=2:namespace=3:application=4:scope=5"
  }
}
```

The notification for this will be:

```json
{
  "notification": {
    "parameter1": 1234, // The value in .notification.parameters has the highest priority
    "parameter2": "some-string",
    "parameter3": true,
    "parameter4": "some-link-value", // The value in .notification.link.attributes has a higher priority than the one in .notification.service.attributes
    "parameter5": "other-service-value", // Values in .notification.service.attributes have the least priority
    "parameter6": "other-link-value",
    "environment": "dev", // Dimensions are added for context from .notification.service.dimensions
    "country": "arg"
  }
}
```

### Adapt your workflow to work with the provisioning.
You need to add a `workflow_call` trigger to your workflow like this:

```yml
on:
  workflow_call:
    inputs:
      notification:
        type: string
        description: some input
        required: true
```

This trigger allows other workflows to import and execute your workflow as a job. In particular, it allows the importing workflow to retrieve the result of your workflow and handle error situations.

If you need to access each value inside notification as separate env variables in your workflow you can add this:

```yml
jobs:
  process-input:
    runs-on: ubuntu-latest
    steps:
      - name: Print Received Input
        env:
          INPUT_PAYLOAD: ${{ inputs.notification }}
        run: |
           # parses the INPUT_PAYLOAD json and exports each property to the GITHUB_ENV variables. Further steps will be able to access properties as env variables
           echo "$INPUT_PAYLOAD" | jq -r '.notification | to_entries | .[] | "\(.key)=\(.value | @sh)"' | while read -r line; do echo "$line" >> $GITHUB_ENV
           done

```