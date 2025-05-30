---
description: This article shows how Spacelift can help you manage Terraform providers
---

# Provider registry

{% if is_saas() %}
!!! info
    This feature is only available to Business plan and above. Please check out our [pricing page](https://spacelift.io/pricing){: rel="nofollow"} for more information.
{% endif %}

## Intro

While the Terraform ecosystem is vast and growing, sometimes there is no official provider for your use case, especially if you need to interface with internal or niche tooling. This is where the Spacelift provider registry comes in. It's a place where you can publish your own providers. These providers can be used both inside, and outside of Spacelift.

## Publishing a provider

### Assumptions

We will not be covering the process of writing a Terraform provider in this article. If you are interested in that, please refer to the [official documentation](https://www.terraform.io/docs/extend/writing-custom-providers.html){: rel="nofollow"}. In this article, we will assume that you **already have a provider** that you want to publish.

We will also focus on providing step-by-step instructions for [GitHub Actions](https://github.com/features/actions){: rel="nofollow"} users. If you are using a different CI/CD tool, you will need to adapt the steps accordingly based on its documentation. Note that you don't need to use a CI/CD tool to publish a provider - you could do it from your laptop. However, we recommend using a CI/CD tool to automate the process and provide an audit trail of what's been done, when, and by whom.

Last but not least, we assume you're going to use GoReleaser to build your provider. This is by far the most common way of managing Terraform providers which need to be available for different operating systems and architectures. If you're not familiar with GoReleaser, please refer to the [official documentation](https://goreleaser.com/){: rel="nofollow"}. You can also check out Terraform's official [`terraform-provider-scaffolding` template repository](https://github.com/hashicorp/terraform-provider-scaffolding){: rel="nofollow"} for an example of using GoReleaser with Terraform providers.

### Creating a provider

To create a provider in Spacelift, you have three options:

#### Use our Terraform provider (preferable)

This is the easiest way to do it, as it will let you manage your provider declaratively in the future:

```terraform title="provider.tf"
resource "spacelift_terraform_provider" "provider" {
  # This is the type of your provider.
  type        = "myinternaltool"
  space_id    = "root"
  description = "Explain what this is for"
}
```

!!! warning
    The `type` attribute in the `spacelift_terraform_provider` resource refers to the unique name of the provider within Spacelift account. It must consist of lowercase letters only.

It is possible to mark the provider as public, which will make it available to everyone. This is generally not recommended, as it will make it easy for others to use your provider without your knowledge. At the same time, this is the only way of sharing a provider between Spacelift accounts. If you're doing that, make sure there is nothing sensitive in your provider. In order to mark the provider as public, you need to set its [`public`](https://registry.terraform.io/providers/spacelift-io/spacelift/latest/docs/resources/terraform_provider#public){: rel="nofollow"} attribute to `true`.

#### Use the API

To create a Terraform Provider using the GraphQL API, you can use the `terraformProviderCreate` mutation. This mutation allows you to create a new provider with the specified inputs.

After successfully creating the Terraform Provider, you will receive an output of type `TerraformProvider`, which contains various fields providing information about the provider. These fields include the ID, creation timestamp, description, labels, latest version number, public accessibility, associated space details, update timestamp, specific version details, and a list of all versions of the provider.

For more detailed information about the GraphQL API and its integration, please refer to the [API documentation](../../integrations/api.md).

#### Create the provider manually in the UI

In order to create a provider in the UI, navigate to the _Terraform registry_ section of the account view, switch to the _Providers_ tab and click the _Create provider_ button:

![](../../assets/screenshots/Screenshot 2023-06-02 at 16.12.49.png)

In the _Create Terraform provider_ drawer, you will need to provide the type, space and optionally a description and/or labels:

![](../../assets/screenshots/Screenshot 2023-06-02 at 16.16.47.png)

!!! info
    Note that we've put the provider in the `root` [space](../../concepts/spaces/README.md). This is because we want to give everyone access to it. If you want to make it available only to a specific team, you can put it in a team-specific different space. In general, unless providers are made public, they can be accessed by all users and stacks belonging to the same space and its children (assuming they're set to inherit resources).

### Register a GPG key

!!! warning
    Only Spacelift root admins can manage account GPG keys. If you're not a root admin, you will need to ask one to do it for you.

Terraform uses GPG keys to verify the authenticity of providers. Before you can publish a provider version, you need to register a GPG key with Spacelift. Similarly to creating a provider, you have three options to register a GPG key:

#### Use our CLI tool called `spacectl`

One reason we do not want to do it declaratively through the Terraform provider is that it would inevitably lead to the private key being stored in some Terraform state, which is not ideal. [`spacectl`](https://github.com/spacelift-io/spacectl){: rel="nofollow"} will let you register a GPG key without storing the private key anywhere outside of your system.

If you have an existing GPG key that you want to use, you can use `spacectl` to register it:

```bash
spacectl provider add-gpg-key \
    --import \
    --name="My first GPG key" \
    --path="Path to the ASCII-armored private key"
```

Alternatively, `spacectl` can generate a new key for you. Note that `spacectl` generates GPG keys without a passphrase:

```bash
spacectl provider add-gpg-key \
    --generate \
    --name="My first GPG key" \
    --email="Your email address" \
    --path="Path to save the ASCII-armored private key to"
```

You can list all your GPG keys using `spacectl`:

```bash
spacectl provider list-gpg-keys
```

#### Use the API

To register a GPG Key using the GraphQL API, you can utilize the `gpgKeyCreate` mutation. This mutation allows you to register a GPG key with the specified inputs.

After successfully registering the GPG Key, you will receive an output of type `GpgKey`, which contains various fields providing information about the key. These fields include the creation timestamp, the user who created the key, the optional description, the ID, the name of the key, the revocation timestamp (if the key has been revoked), the user who revoked the key (if applicable), and the timestamp of the last update to the key.

For more detailed information about the GraphQL API and its integration, please refer to the [API documentation](../../integrations/api.md).

#### Register the GPG key manually in the UI

In order to register a GPG key in the UI, navigate to the _Terraform registry_ section of the account view, switch to the _GPG keys_ tab and click the _Register GPG key_ button:

![](../../assets/screenshots/Screenshot 2023-06-02 at 16.28.27.png)

In the _Register GPG key_ drawer, you will need to provide the name, the ASCII-armored private key and optionally a description:

![](../../assets/screenshots/Screenshot 2023-06-02 at 16.31.37.png)

### CI/CD setup

Now that you have a provider and a GPG key, you can set up your CI/CD tool to publish provider versions. First, let's set up the GoReleaser config file in your provider repository:

{% raw %}

```yaml title=".goreleaser.yml"
builds:
  - env: ['CGO_ENABLED=0']
    flags: ['-trimpath']
    ldflags: ['-s -w -X main.version={{ .Version }} -X main.commit={{ .Commit }}']
    goos: [darwin, linux]
    goarch: [amd64, arm64]
    binary: '{{ .ProjectName }}_v{{ .Version }}'

archives:
  - format: zip
    name_template: '{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}'

checksum:
  name_template: '{{ .ProjectName }}_{{ .Version }}_SHA256SUMS'
  algorithm: sha256

signs:
  - artifacts: checksum
    args:
      - "--batch"
      - "--local-user"
      - "{{ .Env.GPG_FINGERPRINT }}"
      - "--output"
      - "${signature}"
      - "--detach-sign"
      - "${artifact}"

release:
  disable: true
```

{% endraw %}

This setup assumes that the name of your project (repository) is `terraform-provider-$name`. If it is not (maybe you're using a monorepo?) then you will need to change the config accordingly, presumably by hardcoding the project name.

Next, let's make sure you have an API key for the Spacelift account you want to publish the provider to. You can refer to the [API key management](../../integrations/api.md#spacelift-api-key--token) section of the API documentation for more information on how to do that. Note that the key you're generating **must** have admin access to the space that the provider lives in. If the provider is in the `root` space, then the key must have root admin access.

We can now add a GitHub Actions workflow definition to our repository:

{% raw %}

```yaml title=".github/workflows/release.yml"
name: release
on:
  push:

permissions:
  contents: write

jobs:
  goreleaser:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Unshallow
        run: git fetch --prune --unshallow

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version-file: 'go.mod'
          cache: true

      - name: Import GPG key
        uses: crazy-max/ghaction-import-gpg@v5
        id: import_gpg
        with:
          # The private key must be stored in an environment variable registered
          # with GitHub. The expected format is ASCII-armored.
          #
          # If you need to use a passphrase, you can populate it in this
          # section, too.
          gpg_private_key: ${{ secrets.GPG_PRIVATE_KEY }}

      - name: Install spacectl
        uses: spacelift-io/setup-spacectl@main

      - name: Run GoReleaser
        # We will only run GoReleaser when a tag is pushed. Semantic versioning
        # is required, but build metadata is not supported.
        if: startsWith(github.ref, 'refs/tags/')
        uses: goreleaser/goreleaser-action@v5
        with:
          version: latest
          args: release --clean
        env:
          GPG_FINGERPRINT: ${{ steps.import_gpg.outputs.fingerprint }}

      - name: Release new version
        if: startsWith(github.ref, 'refs/tags/')
        env:
          GPG_KEY_ID: ${{ steps.import_gpg.outputs.keyid }}

          # This is the URL of the Spacelift account hosting the provider.
          SPACELIFT_API_KEY_ENDPOINT: https://youraccount.app.spacelift.io

          # This is the ID of the API key you generated earlier.
          SPACELIFT_API_KEY_ID: ${{ secrets.SPACELIFT_API_KEY_ID }}

          # This is the secret of the API key you generated earlier.
          SPACELIFT_API_KEY_SECRET: ${{ secrets.SPACELIFT_API_KEY_SECRET }}
        run: # Don't forget to change the provider type!
          spacectl provider create-version --type=TYPE-change-me!!!
```

{% endraw %}

If everything is fine, pushing a tag like `v0.1.0` should create a new **draft** version of the provider in Spacelift. You can list the versions of your provider using `spacectl`:

```bash
spacectl provider list-versions --type=$YOUR_PROVIDER_TYPE
```

!!! warning
    The `type` parameter in the `spacectl provider list-versions` command refers to the unique name of the provider within Spacelift account. It must consist of lowercase letters only.

Note the version status. Versions start their life as drafts, and you can publish them by grabbing their ID (first column) and using `spacectl`:

```bash
spacectl provider publish-version --version=$YOUR_VERSION_ID
```

!!! warning
    The `version` parameter in the `spacectl provider publish-versions` command refers to the unique ID of the provider version within Spacelift account.

Once published, your version is ready to use. See the next section for more information.

## Using providers

Terraform providers hosted by Spacelift can be used the same way as providers hosted by the Terraform Registry. The only difference is that you need to specify the Spacelift registry URL in your Terraform configuration.

```terraform title="main.tf"
terraform {
  required_providers {
    yourprovider = {
      source  = "{% if is_saas() %}spacelift.io{% else %}your-hostname{% endif %}/your-org/yourprovider"
    }
  }
}
```

The above example does not refer to a specific version, meaning that you are going to always use the latest available (published) version of your provider. That said, you can use any versioning syntax supported by the Terraform Registry - learn more about it [here](https://developer.hashicorp.com/terraform/language/expressions/version-constraints){: rel="nofollow"}.

### Using providers inside Spacelift

All runs in any of the stacks belonging to the provider's space or one of its children will be able to read the provider from the Spacelift registry, same as with [modules](../terraform/module-registry.md) This is because the runs are automatically authenticated to the spacelift.io registry while the run is in progress.

### Using providers outside of Spacelift

If you need to use the provider outside of Spacelift, you will need to authenticate to the registry first, either interactively (eg. on your machine) or using an API key (eg. in automation). This process is the same as with [modules](./module-registry.md#using-modules-outside-of-spacelift), so please refer to that section for more information.

Your access to the required provider will be determined by your access to the space that the provider lives in. If you have read access to the space, you will be able to use the provider.

## Other management tasks

Beyond creating and publishing new versions, there are a few other tasks you can perform on your provider. In this section we will cover the most common ones.

### Revoking GPG keys

If you lose control over your GPG key, you will want to revoke it. Revoking a key has no automatic impact on provider versions already published, but it will prevent you from publishing new versions signed with that key. You can revoke a key using `spacectl`:

```bash
spacectl provider revoke-gpg-key --key=$ID_OF_YOUR_KEY
```

... or by using the UI. In order to do that, click the _Revoke_ button inside the three dots menu next to the key you want to revoke:

![](../../assets/screenshots/Screenshot 2023-06-02 at 16.39.36.png)

If you want to also revoke any of the provider versions signed with this key, refer to the section on [revoking provider versions](#revoking-provider-versions).

### Deleting provider versions

You can permanently delete any **draft** provider version using `spacectl`:

```bash
spacectl provider delete-version --version=$ID_OF_YOUR_VERSION
```

You can also do that in the UI, just navigate to the provider page, find the version you want to delete, and click the _Delete_ button inside the three dots menu next to the version you want to delete:

![](../../assets/screenshots/Screenshot 2023-06-02 at 16.38.09.png)

A deleted version disappears from the list of versions, and you can reuse its number in the future.

You cannot delete published versions. If you want to disable a published version, you will need to [revoke it](#revoking-provider-versions) instead.

### Revoking provider versions

You can revoke any published provider version using `spacectl`:

```bash
spacectl provider revoke-version --version=$ID_OF_YOUR_VERSION
```

Similarly to deleting a draft provider version, you can delete a published one in the UI by clicking the _Revoke_ button inside the three dots menu next to the version you want to revoke:

![](../../assets/screenshots/Screenshot 2023-06-02 at 16.40.36.png)

This will prevent anyone from using the version in the future, but it will not delete it. The version will remain on the list of versions, and will never be available again. You will not be able to reuse the version number of a revoked version.
