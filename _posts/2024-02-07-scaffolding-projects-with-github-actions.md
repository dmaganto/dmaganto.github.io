---
title: "Using Github Actions to scaffolding repositories"
date: 2024-02-10T21:00:00+01:00
categories:
  - Blog
tags:
  - GithubActions
  - CookieCutter
  - Selfservice
  - Automation
---

Usually in many projects we need to create different respositories with similar folder structure. In this case you can use Github Templates that I will dedicate another post in the blog. In this post I'd like to talk about how to use Github Actions to scaffolding monorepo repositories using CookieCutter in a selfservice way.

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

In addition this tool use answer file called `cookiecutter.json` to setup the default value for each value in the templates. We can take advantage of this file to setup default configuration for our environments like:

{% highlight jinja %}
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

Later, running the following command we can create the template. Notice that we are passing the `tenant` and `service` in the command and the `--no-input` to don't prompt for values and only use the `cookiecutter.json` responses.

{% highlight terminal %}
cookiecutter -s -f --no-input _template tenant=tenant1 service=myservice1
{% endhighlight %}

We are using `-f` to overwrite the content if folder exists, it allow us to creates new files if needed, and `-s` to skip render files again if already exists to don't overwrite if the user has made any change.

You can find more information about it in the [official documentation](https://cookiecutter.readthedocs.io/en/2.5.0/index.html).

## Github Actions

Now let's use Github Actions to create a selfservice process in which the users can create automatically a PR with their requirements for new tenant or services.

First let create the user input interface, the parameters and configure as manual triggered pipeline:

{% endhighlight yaml %}
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

{% endhighlight yaml %}
jobs:
    scaffolding:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@v4
        - run: |
            echo "Building ${{ inputs.serviceName }} on ${{ inputs.tenant }}"
            python3 -m pip install --user cookiecutter
            python3 -m cookiecutter -s -f --no-input ${{ github.workspace }}/_template tenant=${{ inputs.tenant }} service=${{ inputs.serviceName }}
        - name: Create Pull Request
          uses: peter-evans/create-pull-request@v6
          with:
            token: ${{ secrets.TOKEN }}
            title: "Adding service ${{ inputs.serviceName }} to ${{ inputs.tenant }}"
            branch: feat/add-${{ inputs.serviceName }}
{% endhighlight %}

We can create a `CODEOWNERS` file set up the PR's approvers for each tenant. [Here more information.](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

You can find the full code in the [repository](https://github.com/dmaganto/monorepo-scaffolding)