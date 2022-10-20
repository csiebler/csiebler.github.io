---
title: "Enforcing Init Scripts on Azure Machine Learning Compute Instances"
date: 2021-07-19
---
## Introduction

This post is a quick tip, showing how you can automatically enforce an Init Script during Azure Machine Learning Compute Instance provisioning. This ensures that even when the user did not specify a script, a default script is always being executed.

For more details on the init script capabilities, have a look at the [documentation](https://web.archive.org/web/20220123155113/https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-manage-compute-instance?tabs=python#setup-script).

## Instructions

Firstly, create your desired init script – in our example, we'll just use a simple example:

```bash
#!/bin/bash

echo "Hello World"
```

Then base64-encode the script and replace the value in line 26 with your encoded script:

```json
{
    "mode": "All",
    "policyRule": {
        "if": {
            "allOf": [
                {
                    "field": "type",
                    "equals": "Microsoft.MachineLearningServices/workspaces/computes"
                },
                {
                    "field": "Microsoft.MachineLearningServices/workspaces/computes/computeType",
                    "in": [
                        "ComputeInstance"
                    ]
                }
            ]
        },
        "then": {
            "effect": "append",
            "details": [{
                    "field": "Microsoft.MachineLearningServices/workspaces/computes/setupScripts.scripts.creationScript.scriptSource",
                    "value": "inline"
                },
                {
                    "field": "Microsoft.MachineLearningServices/workspaces/computes/setupScripts.scripts.creationScript.scriptData",
                    "value": "IyEvYmluL2Jhc2gKCmVjaG8gIkhlbGxvIFdvcmxkIg=="
                }
            ]
        }
    }
}
```

Next, create a new `Policy definition` in Azure Policy:

![New Policy definition](/images/define_new_policy.png "Create a new Policy definition")

You'll need to define in which subscription the definition should live in, give it a name and a description and then finally paste the policy JSON into the Policy Rule. Then hit Save.

![Our Policy definition](/images/create_policy.png "Our Policy definition")

Now that we have a Policy, we can create an assignment. This will apply the policy to Azure resources. Inside Azure Policy, navigate to `Assignments` and select `Assign policy`.

![Creating our assignment](/images/create_new_assigment.png "Creating our assignment")

Then define the scope of the assignment (which subscriptions it should affect), select our Policy definition and hit Create.

![Policy assignment](/images/create_assignment.png "Asssigning our policy")

That's it, our policy is live!

## Making sure it works

Lastly, go to Azure Machine Learning Studio and provision a new Compute Instance without selecting an init script. Once the instance has been provisioned, open the instance details and you should see the `stdout` under the Logs tab.

![Our init script successfully executed](/images/init_script_executed.png "Our init script successfully executed")

Hope this helps!

Happy instance creating!