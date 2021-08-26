---
title: Using Github Actions
author: Joe Post
date: 2021-08-26 14:10:00 +0800
categories: [Blogging, Tutorial]
tags: [git, github, conda, python, php, actions, yml]
---


## Github Actions

In my first post, I wanted to share a very cool functionality I just recently discovered within github called 'github actions'. These are actions that are triggered by a whole suite of events that you can configure. In general, the whole thing is completely configurable by you. 

Actions are really easy to configure and use, and can help any orgnaization run test scripts based on committed code. I imagine it helps significantly for continuous integration/continuous development (CI/CD) operations within distributed organizations. You can also proudly display a status badge on your repo README. Hopefully it displays 'passing'. 

Today we're going to walk through exactly how github actions are configured and run, and hopefully demonstrate the diversity of tasks you can accomplish using them. 

## Basic of Github Actions

If you head to your favorite repository on github and select the 'Actions' tab, you'll see all the currently configured action workflows. If you're hearing about this for the first time, you likely won't see anything (unless you're inheriting someone else's code). We'll get back to this tab later, for now we just need to know it exists.

In order to let github know that we want it to run an action, we need to add a Yet Another Markup Language (yml) file to your repo. Let's say this file is called 'test.yml'. From the root of your repo, place this file in the .github/workflows/ folder. If these directories don't exist, you will have to create them. 

## YML File Contents

Once your test.yml file is in the .github/workflows/ directory, it needs to contain specific fields for github to know how to execute it. Let's look at the first few lines of our test.yml file.

```yaml
---
# create a unique name for this workflow
name: BasicWorkflow

# indicate what events trigger execution of this workflow.
on:
  # Triggers the workflow on push or pull request events but only for the master branch.
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]
  
  # Allows you to run this workflow manually from the Actions tab. Comment out to disable.
  workflow_dispatch:
---
```

There are numerous options for indicating which events trigger execution of a workflow. The above is slightly more verbose than usual, but I prefer it because it is much more readable in a way.

Now that we've indicated which events trigger workflow execution, let's look at the actual jobs within a workflow. You may specify multiple jobs within a workflow, but github has restrictions on these (I believe as of August 2021 the limit is 100 jobs per workflow).

Github actions executes workflows on a virtual machine called a 'runner'. You can specify the OS of the runner, install software on the runner, and execute custom test scripts on the runner. 

```yaml
---
# A workflow run is made up of one or more jobs that can run sequentially or in parallel.
jobs:
  # This workflow contains a single job called "verify-logs"
  verify-logs:
    # The type of runner that the job will run on.
    runs-on: ubuntu-latest
    # specify that the bash shell is the default shell to open and use for all "run" commands. Unless specified, the shell used is /bin/sh. I prefer /bin/bash.
    defaults:
      run:
        shell: bash -l {0}
---
```

Ok, looks good, we've specified which runner OS to use, and what we want the default shell to be. Now let's look at the steps we want to execute within this job in this workflow. 

I personally use a lot of python in my life, so of course I wanted to figure out how to get conda (a nify python package manager) working on the runner. If you've ever worked with conda, you know it's clunky and finicky. I'll show you exactly how I got it to work for me. 

```yaml
---
    # Steps represent a sequence of tasks that will be executed as part of the job.
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it.
      - uses: actions/checkout@v2
      # use the latest miniconda setup action.
      # this should create and activate a production environment. 
      - uses: conda-incubator/setup-miniconda@v2
        with:
          # updates the 'base' conda environment.
          auto-update-conda: true
          # declare the name of the conda env we want to activate.
          activate-environment: prod-env
          # whether we want to automatically activate the base conda environment. it is generally not advisable to use the base environment. we'll be creating our own environment and activating that, so no need to activate the base env. 
          auto-activate-base: false
          # declare what package channels we want conda to look through for packages.
          channels: conda-forge, pypi
          # whether we want to allow softlinks.
          allow-softlinks: true
          # finally, declare the location of the yml file that defines this conda env.
          environment-file: .github/workflows/environment.yml
---
```

Phew, that was a lot to take in. Now, since I'm defining my own conda environment called 'prod-env', let's look at that environment.yml file:

```yaml
---
# Name this conda environment. Note: this needs to match the name in the test.yml file.
name: prod-env

# what channels we want to look through to find the packages below. these are only searched after conda searches its default channels.
channels:
  - pypi

# finally, declare the dependencies of this environment. Note that you can specify specific version numbers for each package, which is generally advisable. Otherwise, the 'latest' version is pulled.
dependencies:
  - python=3.9
  - numpy
  - pip 
  # I've never been successful with using conda to install the below packages, BUT you can use pip to install these within the prod-env environment.
  - pip:
    - reedsolo
    - prettytable
---
```

So now we know what our environment is defined as, let's look at what steps we can execute. Going back to the test.yml file, we define each step with a 'name' and 'run' option defined. Each 'run' of commands is executed within a new subshell, so keep that in mind. 

```yaml
---
      # OPTIONAL: define name for this step
      - name: List all conda environments 
        # Use the "|" character to execute several steps in a row. 
        run: |
          # display info for current conda environment. 
          # this should indicate the prod-env is active.
          conda info -e
          # list all packages installed in the active env.
          conda list

      # you can install many other software on the runner, including php for example.
      - name: Install and test PHP 
        run: |
          sudo add-apt-repository ppa:ondrej/php
          sudo apt-get update
          sudo apt -y install php8.0
          php -v
---
```

Cool! So now we have a conda environment configured and activated, and php installed. So what? Well, now you can execute any number of tests that you define. Below is an example of how to execute test scripts within your repo. 

Note that for things like folder creation, you will need sudo permissions. This is very unfortunate since executing a command using 'sudo' changes your bash profile and basically resets EVERYTHING you just did in the previous steps. Luckily there is a workaround (that security experts would point out very quickly). When using 'sudo', we can specify the ubuntu PATH and import all current bash profile environment variables into the new sudo subshell:

```yaml
---
      - name : Run and verify log files
        # need to give -E and PATH options to sudo to preserve current env variables. 
        run: |      
          # execute a main script
          sudo -E env PATH=$PATH php main.php
          # now execute a test script
          sudo -E env PATH=$PATH php ./php_utils/test.php
---
```

By returning unique return codes from your scripts, github will determine whether your workflow passed or failed, which is very handy. 

After adding the test.yml file to .github/workflows, simply execute a "git commit -am "commit message" and "git push origin", or however you commit and push to remote, head over to your actions tab in github, and see in "real time" the runner being built, and your test scripts running.

You made it! Hopefully this post, although long, helps someone figure out how to automate github actions will the requisite yml files. Thanks for reading.

## Learn More

For more knowledge about Jekyll posts, visit the [Jekyll Docs: Posts](https://jekyllrb.com/docs/posts/).

