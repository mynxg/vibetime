# SignPath code signing

This project can use SignPath in two different ways for Windows code signing:

- GitHub Actions connector signing
- Direct PowerShell/API signing

Both approaches submit Windows release artifacts to SignPath and download signed
artifacts. They differ mainly in how SignPath verifies the build origin.

## Current recommendation

For the current SignPath free trial setup, use direct PowerShell/API signing.

The GitHub Actions connector depends on SignPath Trusted Build System
verification. This feature is not included in the free trial subscription, so
the connector flow can fail before a signing request is created.

The workflow currently used for this project is:

- `.github/workflows/release-signpath.yml`

It builds macOS and Windows packages, signs the Windows `.exe` artifacts through
the SignPath PowerShell module, and creates a GitHub Release.

## SignPath concepts

### Organization ID

The SignPath organization identifier. It is a GUID, for example:

```text
50865fbe-bb06-420d-b3b4-be28ea95d27b
```

Store it in GitHub Actions secrets as:

```text
SIGNPATH_ORGANIZATION_ID
```

### API token

Create the token from the SignPath user details/profile page. The token is shown
only once when it is generated.

The token user must be a submitter for the selected signing policy.

Store it in GitHub Actions secrets as:

```text
SIGNPATH_API_TOKEN
```

### Project slug and signing policy slug

For the current development setup:

```text
Project slug: vibetime-dev
Signing policy slug: vibetime-dev
```

The API token user should have at least the `Submitter` role for this signing
policy. If manual approvals are required, an approver must approve the request
before the workflow can download the signed artifact.

## Artifact configuration

GitHub release artifacts for Windows are `.exe` files, for example:

```text
VibeTime-2026.5.1-1.1-x64.exe
VibeTime-Setup-2026.5.1-1.1-x64.exe
```

When these files are zipped before submitting to SignPath, the artifact
configuration should use `zip-file` as the root and sign PE files inside it:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<artifact-configuration xmlns="http://signpath.io/artifact-configuration/v1">
  <zip-file>
    <pe-file-set>
      <include path="*.exe" min-matches="1" max-matches="unbounded" />
      <for-each>
        <authenticode-sign />
      </for-each>
    </pe-file-set>
  </zip-file>
</artifact-configuration>
```

Do not leave the template value `sample.exe` in the configuration unless the
artifact actually contains a file with that name.

## Method 1: GitHub Actions connector

This method uses:

```yaml
uses: signpath/github-action-submit-signing-request@v2
```

### Flow

1. Build Windows artifacts.
2. Upload unsigned artifacts with `actions/upload-artifact`.
3. Pass the uploaded artifact ID to the SignPath GitHub action.
4. SignPath downloads the GitHub artifact, verifies the trusted build system,
   signs the matching files, and returns signed artifacts.

### Required GitHub secrets

```text
SIGNPATH_API_TOKEN
SIGNPATH_ORGANIZATION_ID
SIGNPATH_SIGNING_POLICY_SLUG
SIGNPATH_ARTIFACT_CONFIGURATION_SLUG
```

`github-token: ${{ github.token }}` is provided automatically by GitHub Actions.
It does not need to be configured as a repository secret.

### Required workflow permissions

The job that calls the SignPath action needs:

```yaml
permissions:
  actions: read
  contents: read
```

### Example

```yaml
- name: Upload unsigned Windows artifacts
  id: upload-unsigned-windows
  uses: actions/upload-artifact@v4
  with:
    name: vibetime-windows-x64-unsigned
    path: packages/desktop/release/*.exe
    if-no-files-found: error

- name: Sign Windows artifacts with SignPath
  uses: signpath/github-action-submit-signing-request@v2
  with:
    api-token: ${{ secrets.SIGNPATH_API_TOKEN }}
    organization-id: ${{ secrets.SIGNPATH_ORGANIZATION_ID }}
    project-slug: vibetime-dev
    signing-policy-slug: ${{ secrets.SIGNPATH_SIGNING_POLICY_SLUG }}
    artifact-configuration-slug: ${{ secrets.SIGNPATH_ARTIFACT_CONFIGURATION_SLUG }}
    github-artifact-id: ${{ steps.upload-unsigned-windows.outputs.artifact-id }}
    wait-for-completion: true
    output-artifact-directory: packages/desktop/release-signed
    github-token: ${{ github.token }}
```

### Pros

- Clean CI/CD integration.
- Uses GitHub artifact identity and SignPath trusted build verification.
- Good long-term option for production release pipelines.

### Cons

- Requires SignPath Trusted Build System verification.
- This feature is not included in the free trial subscription.
- A missing or unlinked GitHub trusted build system can block signing.

## Method 2: Direct PowerShell/API signing

This method uses the SignPath PowerShell module and `Submit-SigningRequest`.

It does not use the SignPath GitHub Actions connector and does not depend on
Trusted Build System verification.

### Flow

1. Build Windows artifacts.
2. Zip the unsigned `.exe` files.
3. Submit the ZIP to SignPath with `Submit-SigningRequest`.
4. Wait for the signing request to finish.
5. Download the signed ZIP.
6. Extract the signed `.exe` files.
7. Upload signed artifacts or attach them to a GitHub Release.

### Required GitHub secrets

```text
SIGNPATH_API_TOKEN
SIGNPATH_ORGANIZATION_ID
```

If the signing policy does not provide a default artifact configuration, also
configure and pass:

```text
SIGNPATH_ARTIFACT_CONFIGURATION_SLUG
```

The current workflow does not pass `ArtifactConfigurationSlug`; it relies on the
default configuration for the `vibetime-dev` signing policy.

### Example

```powershell
Submit-SigningRequest `
  -InputArtifactPath $unsignedZip `
  -ProjectSlug 'vibetime-dev' `
  -SigningPolicySlug 'vibetime-dev' `
  -WaitForCompletion `
  -WaitForCompletionTimeoutInSeconds 1800 `
  -Force `
  -OutputArtifactPath $signedZip `
  -OrganizationId $env:SIGNPATH_ORGANIZATION_ID `
  -ApiToken $env:SIGNPATH_API_TOKEN
```

If an explicit artifact configuration is needed:

```powershell
Submit-SigningRequest `
  -InputArtifactPath $unsignedZip `
  -ProjectSlug 'vibetime-dev' `
  -SigningPolicySlug 'vibetime-dev' `
  -ArtifactConfigurationSlug $env:SIGNPATH_ARTIFACT_CONFIGURATION_SLUG `
  -WaitForCompletion `
  -OutputArtifactPath $signedZip `
  -OrganizationId $env:SIGNPATH_ORGANIZATION_ID `
  -ApiToken $env:SIGNPATH_API_TOKEN
```

### Pros

- Works without SignPath Trusted Build System verification.
- Better fit for the free trial subscription.
- Easier to debug because API validation errors are returned directly.

### Cons

- Less integrated with GitHub artifact provenance.
- The workflow has to package and extract artifacts itself.
- For a stricter production pipeline, the connector flow is preferable after the
  SignPath subscription supports Trusted Build System verification.

## Common errors

### 502 from the GitHub connector

Example:

```text
SignPath REST API is temporarily unavailable (server responded with 502).
```

Possible causes:

- SignPath service or connector issue.
- GitHub Trusted Build System verification is unavailable or not enabled.
- The project is not linked to the GitHub.com trusted build system.

For free trial organizations, the most likely cause is that Trusted Build System
verification is not included.

### Artifact configuration slug does not exist

Example:

```text
There is no artifact configuration with slug '...' in the project with slug '...'.
```

The `SIGNPATH_ARTIFACT_CONFIGURATION_SLUG` secret does not match an artifact
configuration in the selected project. Fix the slug in SignPath/GitHub secrets,
or omit `ArtifactConfigurationSlug` and use the signing policy default.

### Template file name left in artifact configuration

Example configuration problem:

```xml
<include path="sample.exe" />
```

The actual release files are named `VibeTime-*.exe`, so SignPath will not find
`sample.exe`. Use:

```xml
<include path="*.exe" min-matches="1" max-matches="unbounded" />
```

## Release trigger behavior

`release-signpath.yml` runs for tags that match:

```yaml
push:
  tags:
    - 'v*'
```

For example:

```sh
git tag v2026.5.12.1
git push origin v2026.5.12.1
```

If another workflow also contains the same tag trigger, both workflows will run.
Make sure only one release workflow creates the GitHub Release for a given tag.
