---
layout: post
title:  'Basic Chef Policyfile Pipeline with Azure DevOps Pipelines'
tags: [ Chef, Azure ]
excerpt: 'Create a quick and dirty simple Chef policyfile pipeline using Azure DevOps Pipelines'
---

#### Background
Find yourself wanting to get started with policyfiles in Chef but aren't sure where to start with the release pipeline for those policyfiles? If you are using Azure DevOps, here is a quick and basic outline for a policyfile pipeline to get you started.

#### Caveats
This is not intended for use as a production final form pipeline. Before this is production ready it would be best to parameterize some areas as well as **not pass a clear text tokens or store .pem files in version control**. You can pull your ADO PAT token using a variable, a secret variable group, keyvault, environment variable -- there are many options not covered here. A good document on the topic can be found <a href='https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=yaml' target='_blank'>here</a>

Things you will need to store as secrets: 
- your .pem file used to access the chef server
- your Azure DevOps PAT token

#### Setup Steps

- First, sign in to your ADO organization and create a personal access token. Store it somewhere safe
- Decide how you want to store you policyfiles in version control. Will you use a mono-repo? Will each policyfile have it's own repo? I chose to go with the latter so I can easily find/store both my policyfile and my policyfile.lock.json in source control. **There is no right answer here just preference**
- Decide a starting structure for your policyfiles. Will you have one big policyfile with a ton of logic? Will you break that up?
- Create a deliveryuser on your Chef Server to use for upload
- Ensure OC-ID is enabled between your Chef Server and internal Supermarket (if you are using an internal Supermarket)
- Use branches for your policygroups (ie. dev, staging, prod) or something similar
- Set up branch policies on your staging and prod branches. You will want "build validation" enabled.
    - To get to branch policies, from your policyfile repo click branches in the left menu

    ![Branches Button](/assets/images/posts/2020/ado-policyfile-pipeline/branches-button.png)
    
    - In branches, select the three dots to the right of the branch you wish to create a policy for

    ![Branch Policy Menu](/assets/images/posts/2020/ado-policyfile-pipeline/branch-policies-menu.png)

    - Click the plus sign in the Build Validation box to add a build validation policy

    ![Add Build Validation](/assets/images/posts/2020/ado-policyfile-pipeline/add-build-validation.png)

    - Fill in the appropriate settings for your Build Validation step then click save

    ![Branch Policy Button](/assets/images/posts/2020/ado-policyfile-pipeline/branch-policy-button.png)

#### The Policyfile
Below is an example of a *very* standard policyfile. This is what is generated when you run `chef generate policyfile policyfileName`:

<pre><code class="language-ruby"># Policyfile.rb - Describe how you want Chef Infra Client to build your system.
#
# For more information on the Policyfile feature, visit
# https://docs.chef.io/policyfile/
#
# A name that describes what the system you're building with Chef does.
name 'base'

# Where to find external cookbooks:
default_source :supermarket
default_source :supermarket, 'https://your-supermarket-server.com'

# run_list: chef-client will run these recipes in the order specified.
run_list 'your_cookbook_name'

# Cookbook versions
cookbook 'your_cookbook_name', '= 0.1.1'</code></pre>

#### The azure-pipelines.yml File
What does this pipeline actually do? 
- Downloads Chef Workstation
- Fetches Supermarket and Chef Infra Server certificates
- Runs cookstyle
- Checks for a yourpolicy.lock.json file in the working directory. If one is present it will update it. If one is not present it will compile one using chef install
- Pushes the policyfile to the Chef Server using branches as policy groups
- Cleans up the "merge" policygroup (when using branch policies and PRs in ADO you will inevitably end up with a merge branch)
- Creates a pull request in ADO to allow for "human" gates for promotion to staging and production

<pre><code class="language-yaml">
trigger:
  branches:
    include:
    - '*'
steps:
  - bash: |
      wget https://packages.chef.io/files/stable/chef-workstation/20.8.111/ubuntu/20.04/chef-workstation_20.8.111-1_amd64.deb
      sudo dpkg -i chef-workstation_20.8.111-1_amd64.deb
      knife ssl fetch https://your-supermarket-server.com
      knife ssl fetch https://your-chef-server.com
      chmod 400 yourcreds.pem
    displayName: 'Installing Chef Workstation'
  - bash: |
      cookstyle | tee cookstyle.log
      grep "no offenses" cookstyle.log && exit 0 || exit 1
    displayName: 'Linting and Syntax Checks Using Cookstyle'
  - bash: |
      ls -l
      echo $BUILD_SOURCEBRANCHNAME
        if [ -f "yourpolicyname.lock.json" ]
        then
          chef update yourpolicyname.rb --chef-license 'accept'
        else  
          chef install yourpolicyname.rb --chef-license 'accept' 
        fi
        chef push $BUILD_SOURCEBRANCHNAME yourpolicyname.lock.json --chef-license 'accept' -c config.rb
      if [ $BUILD_SOURCEBRANCHNAME == "merge" ]
      then
        chef delete-policy-group merge --chef-license 'accept' -c config.rb
      fi
    displayName: 'Compiling Policy and Push to Chef Server Policy Group'
  - bash: |
      case $BUILD_SOURCEBRANCHNAME in
      'dev')
        curl -X POST -H "Content-Type: application/json" -d @devtostaging.json https://your-ado-user:yourPersonalAccessTokenHere@dev.azure.com/your-ado-organization/your-ado-project/_apis/git/repositories/your-repository-id/pullrequests\?api-version\=6.0
        ;;
      'staging')
       curl -X POST -H "Content-Type: application/json" -d @stagingtoprod.json https://your-ado-user:yourPersonalAccessTokenHere@dev.azure.com/your-ado-organization/your-ado-project/_apis/git/repositories/your-repository-id/pullrequests\?api-version\=6.0
        ;;
      esac
    displayName: 'Create Pull Request'
    condition: or(eq(variables['Build.SourceBranchName'], 'dev'), eq(variables['Build.SourceBranchName'], 'staging'))
</code></pre>

#### What Does the Release Cycle Look Like? 
Developers or policyfile owners will always work in the dev branch when making changes to their policyfiles. Upon push, the pipeline will run making the updates available to anything with the dev policy group applied. There will also be a pull request created automatically to promote from dev to staging. 

Once the developer has ensured everything works in dev and nothing breaks it is then time to promote to staging. Once the automatically created pull request is merged, the pipeline will run and push the policyfile to the staging policy group. Because this is moving into another subprod environment, this can be left to the developer to merge. Another pull request will be created to promote to production. 

The same verification steps should be made to ensure staging remained in a good state before the pull request from staging to production is merged. This one, by way of branch policies, requires approval and completion by someone other than the developer. 

Below is an image of one such PR request: 
![Automated Pull Request Example](/assets/images/posts/2020/ado-policyfile-pipeline/automated-pull-request.png)

#### Summary
This pipeline, as mentioned, is far from done. But should you find yourself trying to learn policyfiles, trying to understand their release process, and curious how to get started on a pipeline in Azure DevOps -- especially if you have never built one on that platform -- this blog is for you as I was all those things. Coming from the environments world, policyfiles seemed unattainable and difficult to learn. Finding resources is not that easy, especially resources on how to actually implement. Generating the policyfile is the easy part. Using them to manage actual servers is another. Hopefully this gives you an understading of policyfiles and a place to get started on your journey using them. 

#### Recommended Reads
- One of the earliest people cheering on Policyfiles in the Chef Community was Michael Hedgepeth. If you want to learn more, definitely check out his blog <a href='http://hedge-ops.com/policyfiles-update/' target='_blank'>here</a>
- The <a href='https://learn.chef.io/courses/course-v1:chef+Infra101+perpetual/about' target='_blank'>Manage Your Fleet with Chef Infra</a> course on <a href='learn.chef.io'>learn.chef.io</a> has some great baseline information on using policyfiles to manage your infrastructure

#### Shout Outs
Being new to policyfiles, huge shout outs to the fine people at Chef (Progress) for their help to understand policyfiles, what needs to happen in a release pipeline, and the whole gambit. In particular, big thanks to Balkar Singh Kang for assisting in the creation of the above basic pipeline framework. 

![Betty White Cheers with Wine Glass](https://media4.giphy.com/media/d3aCGhVlPA6axSAo/giphy.gif)