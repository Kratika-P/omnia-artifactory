# Clean Up BuildStreaM Image Groups

## Overview

BuildStreaM provides two methods for cleaning up image groups: manual cleanup
through the GitLab pipeline and automated cleanup based on age-based retention
policies. Both methods remove S3 images and artifacts, and for catalog-mode
images, exact image-group cleanup also removes entries owned by that image group
from the Image Build Manager global dictionary.

BuildStreaM enforces a retention guard on non-`CLEANED` image groups. The
default `IMAGE_RETENTION_LIMIT` in the source is 50. When the number of
non-`CLEANED` image groups exceeds this limit, new job creation is blocked
until cleanup is performed.

### Manual Cleanup

Use the manual cleanup pipeline to select a specific image group and delete the
associated job's S3 images and artifacts when the guard blocks a new job or
when an image group is no longer required.

### Automated Cleanup

The BuildStreaM API includes an automatic cleanup scheduler that evaluates
image artifacts periodically and deletes those that meet retention criteria.
The scheduler runs within the BuildStreaM API container and
evaluates image groups based on configured age-based retention policies.

!!! warning

    Do not cancel a running GitLab pipeline or stage. Cancellation prevents some pipeline steps from executing, which leaves the BuildStreaM job in an intermediate, inconsistent state.

## Prerequisites

- Access to the managed GitLab project.
- The BuildStreaM API server and PostgreSQL service are running.
- At least one image group has a status other than `CLEANED`.

## Configure Automated Cleanup

Automated cleanup is configured in the `retention` group of
`build_stream_config.yml`.

### Configuration Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `auto_cleanup_enabled` | boolean | false | Enable or disable automatic cleanup |
| `retention_age_days` | integer | 90 | Minimum age in days before an image is eligible for cleanup (must be ≥ 1) |
| `min_keep_count` | integer | 5 | Minimum number of image groups to retain per catalog identifier (must be ≥ 0) |
| `evaluation_interval_hours` | integer | 24 | Interval in hours between cleanup evaluation cycles (must be ≥ 1) |

### Example Configuration

```yaml
retention:
  auto_cleanup_enabled: true
  retention_age_days: 90
  min_keep_count: 5
  evaluation_interval_hours: 24
```

### Cleanup Eligibility Criteria

An image group is eligible for automated cleanup when it meets ALL of the
following criteria:

- `deploy_count = 0` (All the image groups except the currently deployed)
- Age exceeds `retention_age_days`

The cleanup cycle runs in two phases:

1. **Phase 1**: Cleans image groups in `FAILED` status
2. **Phase 2**: Applies age-based retention to eligible image groups in nonfailed status

A failure on one image group does not stop the cleanup of other image groups.

### Configuration Behavior

- The first cleanup cycle runs 5 minutes after the BuildStreaM API container starts
- Subsequent cycles run once `evaluation_interval_hours` have elapsed
- Invalid configuration values are rejected with a warning; the last known-good
  settings remain in effect
- An unreadable configuration file does not reset the scheduler to defaults

## Manual Cleanup Procedure

Use the manual cleanup pipeline to select and delete specific image groups.

1. Navigate to the GitLab project URL:

    ```text title="GitLab project URL"
    https://<gitlab_host>:<gitlab_https_port>/root/<gitlab_project_name>
    ```

2. Navigate to **Build** → **Pipelines**.

3. Click **New Pipeline**.

4. In the **Run new pipeline** dialog box, enter the variable name as **PIPELINE_TYPE** and enter the value as **cleanup**.

    ![GitLab Clean Pipeline Variable](../../assets/images/gitlab-clean-pipeline-variable.png)

5. Click **Run Pipeline** to execute the cleanup pipeline.

6. In the pipeline, select the image to be cleaned up from the `select_image` stage.

    ![GitLab Clean Select Image](../../assets/images/gitlab-clean-select-image.png)

7. Click the **Play** button on the cleanup stage to execute the cleanup.

    ![GitLab Clean Run Stage](../../assets/images/gitlab-clean-run-stage.png)

8. Monitor the pipeline progress through the GitLab web interface:

    - Click on the running pipeline to view details.
    - Monitor the cleanup stage as it progresses to completion.

    ![GitLab Clean Monitor Pipeline](../../assets/images/gitlab-clean-monitor-pipeline.png)

9. Review the stage status indicators:

    - **Green checkmark**: Stage completed successfully
    - **Red X**: Stage failed (click for error details)
    - **Blue circle**: Stage currently running

## Verification

### Manual Cleanup Verification

1. Check the GitLab pipeline status to ensure the cleanup stage passed.

2. Confirm that the selected image group is reported as `CLEANED` and is no
   longer offered by the cleanup pipeline as a cleanable image group.

3. Review the cleanup pipeline logs in GitLab for specific details about which Image Groups were removed.

4. For a catalog-mode image group, inspect the dictionary and confirm that no
   entry retains the cleaned composite image-group identifier:

    ```text
    $OMNIA_DATA_PATH/image_build_manager/output/$OMNIA_PROJECT_NAME/image_group_dictionary.json
    ```

    A dictionary-cleanup warning is reported separately and does not change an
    otherwise successful artifact cleanup into a failed cleanup operation.

### Automated Cleanup Verification

1. Check the BuildStreaM API logs for `RETENTION_DELETED` audit events to confirm
   which image groups were deleted by the automated cleanup scheduler.

2. Verify that the deleted image groups meet the configured retention criteria:
   - Age exceeds `retention_age_days`
   - `deploy_count = 0`

4. For catalog-mode image groups, verify that the global dictionary no longer
   contains entries for the cleaned composite image-group identifiers.

## Next steps

- [Retry Pipelines](retry_pipelines.md) -- Retry failed pipeline operations
- [Execute Build Pipeline](../../HowTo/build_stream/execute_build_pipeline.md) -- Create new images after cleanup
- [Update Catalog](update_catalog.md) -- Modify catalog definitions for future builds

## Troubleshooting

### Manual Cleanup Issues

- **No cleanup pipeline starts:** Set `PIPELINE_TYPE` to the exact lowercase
  value `cleanup` when creating the pipeline.
- **No image groups are listed:** The Images API returns only groups whose
  status is not `CLEANED`; there is nothing to remove when the list is empty.
- **Cleanup pipeline failing:** Verify that the BuildStreaM API server and
  PostgreSQL database are running. See
  [BuildStreaM troubleshooting](../../Troubleshooting/build_stream/build_stream.md).
- **A dictionary entry remains:** Confirm that the cleanup selected the exact
  composite image-group identifier. Review Image Build Manager cleanup logs
  for a dictionary warning before editing the dictionary manually.

### Automated Cleanup Issues

- **Automated cleanup not running:** Verify that `retention.auto_cleanup_enabled`
  is set to `true` in `build_stream_config.yml` and that the BuildStreaM API
  container is running.
- **Unexpected image deletions:** Review the `retention_age_days`,
  `min_keep_count`, and `evaluation_interval_hours` settings. Check the
  BuildStreaM API logs for `RETENTION_DELETED` audit events to understand which
  images were deleted and why.
- **Invalid configuration rejected:** Check the BuildStreaM API logs for warnings
  about invalid retention parameter values. Ensure `retention_age_days` ≥ 1,
  `min_keep_count` ≥ 0, and `evaluation_interval_hours` ≥ 1.















