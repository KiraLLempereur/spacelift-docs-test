# Creating a stack

Unless you're defining a stack programmatically using our [Terraform provider](../../vendors/terraform/terraform-provider.md), you will be creating one from the root of your Spacelift account:

![](<../../assets/screenshots/Stacks\_·\_spacelift-io (2).png>)

{% hint style="info" %}
You need to be an admin to create a stack. By default, GitHub account owners and admins are automatically given Spacelift admin privileges, but this can be customized using [login policies](../policy/login-policy.md) and/or [SSO integration](../../integrations/single-sign-on.md).
{% endhint %}

The stack creation process involves four simple steps:

1. [Creating a link between your new stack and an existing Git repository](./#integrate-vcs);
2. [Defining common behavior of the stack](./#define-behavior);
3. [Defining backend-specific behavior](creating-a-stack.md#configure-backend) (different for each supported backend, eg. [Terraform](creating-a-stack.md#terraform), Pulumi)
4. [Naming, describing and labeling](creating-a-stack.md#name-your-stack);

&#x20;Please see below for a step-by-step walkthrough and explanation.

## Integrate VCS

![](<../../assets/screenshots/New\_stack\_·\_spacelift-io (9).png>)

In the first step you will need to tell Spacelift where to look for the Terraform code for the stack - a combination of Git repository and one of its existing branches. The branch that you specify set here is what we called a _tracked_ branch. By default, anything that you push to this branch will be considered for deployment. Anything you push to a different branch will be tested for changes against the current state.

A few things worth noting:

* you can point multiple Spacelift stacks to the same repository, even the same branch;
* the default behavior can be tweaked extensively to work with all sorts of Git and deployment workflows (yes, we like monorepos, too) using [push](../policy/git-push-policy.md) and [trigger](../policy/trigger-policy.md) policies, which are more advanced topics;
* in order to learn what exactly our Git hosting provider integration means, please refer to [GitHub](../../integrations/source-control/github.md) and [GitLab](../../integrations/source-control/gitlab.md) integration documentation;

{% hint style="info" %}
If you're using our default GitHub App integration, we only list the repositories you've given us access to. If some repositories appear to be missing in the selection dropdown, it's likely that you've installed the app on a few selected repositories. That's fine, too, just [whitelist the desired repositories](../../integrations/source-control/github.md) and retry.
{% endhint %}

## Define behavior

Regardless of which of the supported backends (Terraform, Pulumi etc.) you're setting up your stack to use, there are a few common settings that apply to all of them. You'll have a chance to define them in the next step:

![](<../../assets/screenshots/New\_stack\_·\_spacelift-io (10).png>)

The basic settings are:

* whether the stack is [administrative](./#administrative);
* project root (optional), i.e. where inside the repository Spacelift should look for the infra project source code;
* [worker pool](../worker-pools.md) to use, if applicable;

![](<../../assets/screenshots/New\_stack\_·\_spacelift-io (11).png>)

The advanced settings are:

* whether the changes should [automatically deploy](./#autodeploy);
* whether obsolete tests should be [automatically retried](./#autoretry);
* list of commands to run before project initialization;
* Docker image to use to for your job container;

## Configure backend

At this point you'll probably know whether you want to create a [Terraform](creating-a-stack.md#terraform) or a [Pulumi](creating-a-stack.md#pulumi) stack. Each of the supported vendors has some settings that are specific to it, and the backend configuration step is where you can define them.

### Terraform

![](<../../assets/screenshots/New\_stack\_·\_spacelift-io (12).png>)

When selecting **Terraform**, you can choose which **version of Terraform** to start with - we support Terraform 0.12.0 and above. You don't need to dwell on this decision since you can change the version later - Spacelift supports full [Terraform version management](../../vendors/terraform/version-management.md) allowing you to even preview the impact of upgrading to a newer version.

The next two decisions involves your Terraform state. First, whether you want us to provide a Terraform state backend for your state. We do offer that as a convenience feature, though Spacelift works just fine with any remote backend, like S3.

{% hint style="info" %}
If you want to bring your own backend, there's no point in doing additional [state locking](https://www.terraform.io/docs/state/locking.html) - Spacelift itself provides a more sophisticated state access control mechanism than Terraform.
{% endhint %}

If you choose not to use our state backend, feel free to proceed. If you do want us to manage your state, you have an option to import an existing state file from your previous backend. This is only relevant if you're migrating an existing Terraform project to Spacelift. If you have no state yet and Spacelift will be creating resources from scratch, this step is unnecessary.

{% hint style="warning" %}
Remember - this is the only time you can ask Spacelift to be the state backend for a given stack, so choose wisely. You can read more about state management [here](../../vendors/terraform/state-management.md).
{% endhint %}

### Pulumi

![](<../../assets/screenshots/New\_stack\_·\_spacelift-io (13).png>)

When creating a Pulumi stack, you will need to provide two things. First, the login URL to your Pulumi state backend, as currently we don't provide one like we do for Terraform, so you will need to bring your own.

Second, you need to specify the name of the Pulumi stack. This is separate from the name of the Spacelift stack, which you will specify in the [next step](creating-a-stack.md#name-your-stack). That said, nothing prevents you from keeping them in sync.

## Name your stack

![](<../../assets/screenshots/New\_stack\_·\_spacelift-io (14).png>)

We're almost there, but here comes the most difficult step - naming things. Here's where you give your new stack a nice informative [name and an optional description](stack-settings.md#name-and-description) - this one even supports Markdown:

![](<../../assets/screenshots/New\_stack\_·\_spacelift-io (8).png>)

You'll be able to change the name and description later, too - with one caveat. Based on the original _name_, Spacelift generates an immutable slug that serves as a unique identifier of this stack. If the name and the slug diverge significantly, things may become confusing.

Also, this is the opportunity to set a few [labels](stack-settings.md#labels). Labels are useful for searching and grouping things, but also work extremely well with policies.
