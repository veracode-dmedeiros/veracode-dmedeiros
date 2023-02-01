# Real World CI/CD integration with Veracode SCA Agent

## Land of confusion

When working with customers to integrate SCA AGent, there is a lot of confusion around how SCA Agent fits into their CI/CD process. The fact that the SCA Agent requires a registration process for it to be used obscures just how it should fit in the CI/CD pipeline.

With most CI/CD systems, An engine will create a job instance that spins up and then destroyed. The necessity to perform a registration under those condition just doesn't fit.

Registration forces the SCA Agent to be either installed to an existing job image that will be reused or the other view of registering the SCA Agent on each creation of an instance is not possible. As each activation key can only be utilized once. This is where the confusion persists from and is understandable.

### Here is the broad strokes

Yet, there is a way to integrate SCA Agent into the CI/CD without the need to perform registration each time. In this writeup I'll indicate how this can be accomplished.

To allow SCA Agent to be used ideally within most CI/CD systems, we need to make it think that it is already an existing registered SCA Agent. Before performing this we fist need to decide what type of agent we want to install. Followed by how we plan on installing the SCA Agent into the job instance for our CI/CD. Finally, how to use the SCA Agent and report results back to the appropriate workspace.

## Selecting the type of SCA Agent to configure

First, you need to decide what type of SCA Agent will need to be configured.

SCA Agent can be registered to work either as a Global Agent or Workspace Agent bound to a specific workspace. The most common configuration written about in Veracode Help Center documentation is set up for a Workspace Agent. This does not work well when integrating with CI/CD systems as it binds results to a single workspace. But this could be what you want for your situation and its simple to implement. When binding the SCA Agent, all you have to do is select the workspace you want and perform the create SCA Agent steps as indicated in the documentation.

To configure the SCA Agent to be a Global Agent which is what I prefer to use for my CI/CD systems. It means that the SCA Agent is reusable and not be bound to a specific workspace. This is accomplished selecting the ***Agent-Based Scan Settings*** button in the upper right of the Software Composition Analysis landing page. Here you select **Agents** in the right side menu and select the **Actions** button to create a new agent.

The difference between these two setups is that with the Global Agent you are provided a new drop down to select a workspace. Upon selection, the workspace slug is changed to represent your selection and shown passed into the srcclr agent scan arguments.

It is this feature that allows us to dynamically select which workspace the SCA Agent will associate scan results upon scan completion.

The CI/CD scripts that I created utilize this functionality. But it requires the agent to be one that is Global and it takes in the workspace(ws) slug as a value.

I provide the script for customer use within these repositories:

- [Azure DevOps](https://github.com/dmedeiros-veracode/devops-scripts-azure-devops)
- [Github Actions](https://github.com/dmedeiros-veracode/github-actions-devops-workflows)

These script perform all the necessary installation process and launching of the scan for both Azure and Github Actions.

## Capturing the Authorization Token

The next task is to capture the ***Agent Authorization Token***. This token is different than the ***Registration Token*** that is provided to the command `srcclr activate`. It is the token that is generated within *agent.yml* by the activation process.

This can also be accessed by setting up the SCA Agent in a non-CI/CD environment such as a docker container, or locally and then copying out the contents of the agent.yml from the `.srccrl` directory located in the user's home directory that performed the install.

Another means of accessing this specific token is to select one of the *Integration Options* listed and select the *Create Agent & Generate Token* button. This will generate a ***Agent Authorization Token*** that you can copy and use.

In either case that is just the first step in accomplishing our goal. The second is to set this value as an environmental variable or passed in to the agent during runtime.

The solution I prefer and utilize is the environmental variable. As it provides flexibility and can be configured for both Global and Workspace agents to be passed during the job instance runtime.

The environmental variable is **SRCCLR_API_TOKEN** and needs to be set to the value of the ***Agent Authorization Token***. That's simple enough to accomplish in most CI/CD systems. If your not familiar with how to accomplish this I recommend you look to your CI/CD system's documentation. As I will not be providing a write up for every CI/CD system and permutation to accomplish this task. I'd recommend storing it as a configurable value to the CI/CD that is controlled or if you must hard code as a value to pass to the job instance.

## Continuous installation of SCA Agent

The next task is to have the SCA Agent install itself into a job instance when created.

You need to ensure that the proper system requirements are meet for SCA Agent's installation. You can find the requirements information within [Veracode Help Center](https://docs.veracode.com/r/Setting_Up_Agent_Based_Scans).

The Veracode Set Up Scanner documentation speaks to how to install and register for each environment. In our case we are going to follow the instructions but up to a point. We are not going to register the SCA Agent and perform `scrclr activation` as we should have already obtained the ***Agent Authorization Token*** by one of the processes stated above.

Instead we will use the environmental variable of **SRCCLR_API_TOKEN** to provide the Authorization Token as if we've already registered the agent to allow for execution when we call `scrclr scan`.

Once you have the **SRCCLR_API_TOKEN** environmental variable setup and  configured within your CI/CD. Follow each step in the SetUp documentation for installation of the SCA Agent. Just don't perform the activation step as it is no longer required.

### Workspace Slug

If you created a Global Agent then all that is left to do is pass in the workspace slug to the agent when performing a scan.

> Example: `scrclr scan --ws=<workspace_slug>

This also can be configured as an environmental variable or argument that you pass in depending on your preference and CI/CD system.

If you did not capture the workspace slug value during the creation of the Global Agent. Then go back to that agent and select it. The workspace slug value can be found by setting the **Target workspace** and copying it.

#### Best use is Scripts

One of the most common ways to perform the installation and execution for SCA Agent is from the shell script using a simple call that installs and runs the SCA Agent.

This can be perform in a Linux environment or using PowerShell for Windows. Each type of script performs the exact same process for installation and requires the SCRCLR_API_TOKEN as an environmental variable to work. Each script will download, install and launch a command prompt.

> For Linux Shell: ``curl -sSL  https://download.sourceclear.com/ci.sh | sh``

> For PowerShell: ``iex ((New-Object System.Net.WebClient).DownloadString('https://download.srcclr.com/ci.ps1'));``

You can easily extend each of these scripts to perform the SCA Agent actions that you need.

##### Shell Script

This example will download, install and run a scan based on the provided URL location and not upload the results to the workspace.

> `curl -sSL  https://download.sourceclear.com/ci.sh | sh -s -- scan --url https://github.com/srcclr/example-ruby --no-upload`

This example will download, install and run a scan based on the provided directory location and upload the results to the workspace indicated by the workspace slug.

> `curl -sSL  https://download.sourceclear.com/ci.sh | sh -s -- scan <source_directory> --ws=123`

##### Powershell

This example will download, install and run a scan based on the provided directory location and upload the results to the workspace indicated by the workspace slug.

> `iex ((New-Object System.Net.WebClient).DownloadString('https://download.srcclr.com/ci.ps1')); srcclr scan <source_directory> --ws=123`

## That's all folks

That is all that is needed to make the SCA Agent dynamically load and run while not having to continuously register the agent.
