---
title: "Using Github Actions to scaffold repositories"
date: 2024-02-10T21:00:00+01:00
categories:
  - Blog
tags:
  - GithubActions
  - CookieCutter
  - Selfservice
  - Automation
---

Usually, in many projects, we need to create different repositories with a similar folder structure. In this scenario, you can use GitHub Templates, which I will cover in another blog post. For now, in this post, I'd like to discuss how to use GitHub Actions to scaffold monorepo repositories using CookieCutter in a self-service manner.

## CookieCutter
CookieCutter is a tool that help us creating templates using Jinja2 as templating language. Some examples are:

{% highlight jinja %}
apiVersion: v2
name: {% raw %}{{cookiecutter.service}}{% endraw %}
description: The helm chart for {% raw %}{{cookiecutter.service}}{% endraw %}
type: application
version: 0.1.1
appVersion: 0.1.1
dependencies:
  - name: ecs-service-generic
    version: ~1
    repository: "oci://my-ecr-repo/helm-charts"
{% endhighlight %}

In addition, this tool uses an answer file called cookiecutter.json to set up the default values for each variable in the templates. We can take advantage of this file to configure default settings for our environments, such as:

{% highlight json %}
{
    "tenant": "tenantName",
    "service": "serviceName",
    "environments": {
        "dev": {
            "accountId": "11111",
            "vpcId": "11111",
            "albSubnets":"11111",
            "serviceSubnets":"11111",
            "permissionsBoundary": "11111",
            "kmsARN": "11111",
            "albDns": "11111",
            "domain": "11111",
            "zoneId": "11111",
            "securityGroupECS": "11111",
            "certificateArn": "11111"
        }
    }
}
{% endhighlight %}

Later, by running the following command, we can create the template. Please note that we are passing the tenant and service as parameters in the command and using --no-input to avoid prompting for values, relying solely on the responses from the cookiecutter.json file.

{% highlight terminal %}
cookiecutter -s -f --no-input _template tenant=tenant1 service=myservice1
{% endhighlight %}

We are using `-f` to overwrite the content if folder exists, it allow us to creates new files if needed, and `-s` to skip render files again if already exists to don't overwrite if the user has made any change.

You can find more information about it in the [official documentation](https://cookiecutter.readthedocs.io/en/2.5.0/index.html).

## Github Actions

Now, let's use GitHub Actions to establish a self-service process, allowing users to automatically create a pull request with their requirements for new tenants or services.

First, let's create the user input interface, define the parameters, and configure it as a manually triggered pipeline:

{% highlight yaml %}
name: Create new service or tenant
run-name: ${{ github.actor }} is creating new service or tenant 🚀
on:
    workflow_dispatch:      # Manual execution
        inputs:
            tenant:         # Static parameters
                description: 'Tenant'
                type: choice
                options:
                    - tenant1
                    - tenant2
                    - tenant3
                    - tenant4
                required: true
            serviceName:    # Dynamic parameters
                description: 'Service Name'
                type: string
                required: true
{% endhighlight %}

And now, we use these parameters in the workflow execution and automatize the process. 

{% highlight yaml %}
jobs:
    scaffolding:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@v4
        - run: |
            echo "Building {% raw %}${{ inputs.serviceName }}{% endraw %} on {% raw %}${{ inputs.tenant }}{% endraw %}"
            python3 -m pip install --user cookiecutter
            python3 -m cookiecutter -s -f --no-input {% raw %}${{ github.workspace }}{% endraw %}/_template tenant={% raw %}${{ inputs.tenant }}{% endraw %} service={% raw %}${{ inputs.serviceName }}{% endraw %}
        - name: Create Pull Request
          uses: peter-evans/create-pull-request@v6
          with:
            token: {% raw %}${{ secrets.TOKEN }}{% endraw %}
            title: "Adding service {% raw %}${{ inputs.serviceName }}{% endraw %} to {% raw %}${{ inputs.tenant }}{% endraw %}"
            branch: feat/add-{% raw %}${{ inputs.serviceName }}{% endraw %}
{% endhighlight %}

We can create a `CODEOWNERS` file set up the PR's approvers for each tenant. [Here more information.](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

You can find the full code in the [repository](https://github.com/dmaganto/monorepo-scaffolding)