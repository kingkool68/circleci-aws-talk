# CircleCI + AWS
## = Happy Deployments

Tonight we're going to talk about Continuous Integration/Continuous Deployment using CircleCI and Amazon Web Services. A less jargon-filled way to say that is we're going to talk about getting from code Point A to Point B.

We build software.

We need to get the software from our local machine to:
 - Different enviornments: Staging, Test, Production
 - Other developers machines

But we don't want things to break. 

We need an automated way of deploying code in a sane way that we can rely on so things don't break.

And that's what CI/CD is for.

When I joined Spirited Media in 2016 we had a deployment process that had several manual steps in the middle of it. We would have to log into different dashboards and click buttons and verify things. It was tedious and error prone. We would ask everyone in the CMS to to log out to ensure no one would lose any work during a deployment. This was not going to work as we got bigger.

Deploying was scary. The last step in the checklist was a reminder to "breathe". We want to build new features, solve problems for our audience, and get things to production as fast as possible. A deployment process that caused anixety and slowed us down wasn't helping us get things done. 

So when we switched to AWS we decided to figure out how to make our deployment workflow work like a well oiled machine. 

## CircleCI

We ended up using CircleCI. What CircleCI does is it listens to our repo for changes. When changes to a branch are pushed up or a new release is created CircleCI springs into actions and performs actions based on those changes. 

Setting up CircleCI is a matter of logging in with your GitHub or Bitbucket account and granting access to your repo(s). There's a good step-by-step guide for getting setup at https://github.com/dwyl/learn-circleci 

Once CircleCI is listening for changes to your repo you need to specify what happens when changes are detected. We do this with a YAML file (`config.yml`) within a `.circleci` directory in the root of our repo. Here we specify the steps (known as Jobs) that should happen when CircleCI runs our build. 

The jobs can be run sequentially or run in parallel to speed things up. One or more jobs make up a workflow which you can monitor from the CircleCI dashboard. 

Our build process is rather straightforward.

 - Setup the build enviornment
 - Download dependencies
 - Compile static assets (SCSS --> CSS, minifying JavaScript files)
 - Lint files to make sure they conform to coding standards

If any of those steps fail for whatever reason, then the build stops and is marked as failed. No deployment happens until we fix it.

So let's go step by step through our `config.yml` file to show how this works...

```
version: 2
```

Here we tell CircleCI we're using version 2.0 of their service. There was a forced migration from 1.0 to 2.0. It was messy but we came away unscathed. We also got some performance improvements that I'll show you a little later.

```
references:
  # Default container configuration
  docker_config: &docker_config
    docker:
      - image: circleci/php:7.0-node-browsers
```

Here we define our docker image that we'll be using throught our build. You can define different docker images for different jobs. See https://circleci.com/docs/2.0/circleci-images/ for more information on pre-built docker images.

```
jobs:
  build:
    <<: *docker_config
    steps:
      - checkout
```

Now we're going to define a job called `build` using our docker image we defined moments earlier. Kinda weird that I setup the docker image as a variable within YAML since I only use it once. Whatever.

The first step in this job is to checkout the branch

```
      - restore_cache:
          keys:
            - v1-dependency-cache-{{ checksum "yarn.lock" }}
            - v1-dependency-cache
      - restore_cache:
          keys:
            - v1-composer-cache-{{ checksum "composer.lock" }}
            - v1-composer-cache
```

The next step is to restore caches of our dependencies. We get the checksum of lock files for node dependencies and composer dependencies. If the cache already exists then we can skip the step of downloading unnecssary files and save some time. 

```
      - run:
          name: Remove previous Yarn
          command: sudo rm /usr/local/bin/yarn*
      - run:
          name: Install Yarn
          command: sudo npm install -g yarn
      - run:
          name: Install Grunt CLI
          command: sudo npm install -g grunt-cli
```

The docker image comes with an outdated version of Yarn so we remove it and install it ourselves. We also install grunt which we will use to run our build steps.

```
      - run:
          name: Install NPM Dependencies
          command: yarn install
      - run:
          name: Install Composer Dependencies
          command: composer install -o --no-dev
```

Remember earlier how we restored the cache? When we install our dependencies we can look at what was previously installed and do an incremental update instead of downloading and compiling all of our dependencies from scratch. 

```
      - save_cache:
          key: v1-dependency-cache-{{ checksum "yarn.lock" }}
          paths: node_modules
      - save_cache:
          key: v1-composer-cache-{{ checksum "composer.lock" }}
          paths: vendor       
```

And for good measure we save the cache again. 

```
      - run:
          name: Building
          command: |
            if [ "${CIRCLE_BRANCH}" == "staging" ]; then
              grunt
            else
              grunt build
            fi
```

Now we've got everytihng in place so we can actually run our build steps which we do via Grunt. We run a different grunt command if the branch being built is our staging branch. Its a little more forgiving/adds some debugging helpers like sourcemaps for CSS and JavaScript files. Otherwise we run the steps as if it were production which includes our strict linting checks.

```
      - run:
          name: Maybe Deploying?
          command: |
            if [ "${CIRCLE_BRANCH}" == "staging" ]; then
              .circleci/deploy.sh
            fi
            if [ "${CIRCLE_TAG}" ]; then
              .circleci/deploy.sh
            fi
```

If everything was sucessful then maybe we can deploy the build. CircleCI has a bunch of different enviornment variables that get set so you can make decisions in different circumstances. Here if the branch is `staging` or the build is a tagged released then run our `deploy.sh` script See https://circleci.com/docs/2.0/env-vars/#built-in-environment-variables

So that sums up our one job that gets run. The next part is to define when that job should or shouldn't be run.

```
workflows:
  version: 2
  build-and-deploy:
    jobs:
      - build:
          filters:
            branches:
              ignore: master
            tags:
              only: /^v[0-9]+(\.[0-9]+)*/
```

Here we define our workflow called `build-and-deploy` that runs the `build` job. We filter the branches that this job is run on. What this means is we run the build job on every branch unless the branch is `master`. We also only run the build job on tags that match the semver format of v0.0.0. 

Why do we ignore the `master` branch? Sometimes we merge code into `master` but don't want it to be deployed. A deployment to production for us is whenever a new tag is pushed up to the repo.

## Deployment

When it comes to deployment we want to get our code out to the production or staging server as fast as possible. We don't want to download an entire code bundle when only a few files changed. We already use Git which does a great job of downloading the differences that changed between two points in time. So lets just tell our servers to do a git pull to get the latest code changes! For that we need a build-specific version of our code repo.

### deployment.sh

```
#!/bin/bash
echo "Starting Deploy..."
git config --global user.name "CircleCI"
git config --global user.email "<some-random-email-address>@spiritedmedia.com"
```

First things first, we set the Git username and email so we know the future commits are coming from CircleCI. This isn't strictly necessary but it is a good idea.

```
# Make a new README.md with build status details
rm README.md
cat > README.md <<EOF
Build [#$CIRCLE_BUILD_NUM]($CIRCLE_BUILD_URL) by $CIRCLE_USERNAME at $TIMESTAMP
[$CIRCLE_COMPARE_URL]($CIRCLE_COMPARE_URL)
EOF
```

Next we make a new README file that contains details about the build. If the build is a tagged release then we parse the git log and build a change log from the merged pull request titles. See https://gist.github.com/kingkool68/09a201a35c83e43af08fcbacee5c315a for details about how that works.

```
# This should be ignored in .gitignore-build but let's try and remove it just to be safe
rm -rf node_modules/

# Find all .git/ directories and remove them. If we commit directories with .git in them then they are treated like sub-modules and screw that.
find . | grep -w ".git" | xargs rm -rf

# Remove all .gitignore files in the wp-content/ and vendor/ directories
find wp-content/ vendor/ -name ".gitignore" | xargs rm

rm .gitignore
mv .gitignore-build .gitignore
```

Do some cleanup of our repo for the build repo.
 - We don't need the `node_modules/` directory so to be safe we delete it. 
 - Remove all `.git/` subdirectories and `.gitignore` files
 - Remove our original `.gitignore` file and replace it with the `.gitignore-build` file that we keep version controlled in our repo. There are things we don't need in production like dependency lock files and coding standard files.

```
git clone git@github.com:spiritedmedia/spiritedmedia-build.git tmp/
mv tmp/.git .
rm -rf tmp/
```

Clone our build repo into a temp directory and copy the .git directory back up to the root of our built-but-yet-to-be-deployed branch. 

```
# If no branch is set then assume master
if [ ! $CIRCLE_BRANCH  ]; then
  CIRCLE_BRANCH="master"
fi

# Switch branches... maybe?
if [ ! `git branch --list $CIRCLE_BRANCH` ]; then
  # Branch doesn't exist. Create it and check it out. See http://stackoverflow.com/a/21151276
  git checkout -b $CIRCLE_BRANCH
else
  git checkout $CIRCLE_BRANCH
fi
```

Make sure our branch exists in the build repo and switch to it so we deploy our changes to the right place. We can tweak our `.circleci/config.yml` file and not have to worry about updating our deployment script.

```
# Add everything and commit
git add -A
git commit -m "Build #$CIRCLE_BUILD_NUM by $CIRCLE_USERNAME on $TIMESTAMP"
git push origin $CIRCLE_BRANCH --force

echo "Code changes pushed!"
```

Finally add everything, make a commit, and do a force push up to our build specific repo.

🎉 Hooray!

Now we have a versioned history of changes to our tested and built codebase that can be deployed wherever we like and it should just work™. If we need to we can rollback to a specific point in time by checking out a certain commit i.e. `git reset --hard HEAD~1` or `git reset --hard 0d1d7fc32`. 

Having a repo with code that is ready to go is great but if the changes can't make their way to a server then what is the point?

We could SSH into the server and manually run a command. But we want this to be automated. Enter AWS CodePipeline

## AWS CodePipeline
CodePipeline is the glue between our GitHub repo and AWS. It sets up webhooks to monitor changes. When changes happen to a branch we then kick off AWS CodeDeploy that actually handles telling all of our servers to update. We need one pipeline per branch. 

## AWS CodeDeploy
CodePipeline let us know there is a new deployment that needs to happen. You need to install the AWS CodeDeploy Agent on each of your instances (see https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install.html). AWS CodeDeploy lets you specify how that deployment should happen. You can do an in-place deployment which will update each EC2 instance. Or you can do a blue/green deployment where AWS will spin up new EC2 machines and swap out the old ones.

We do an in-place deployment. All of our servers are tagged with what environment they belong in so we can target all of the machines where `Environment` == `Staging` or `Environment` == `Production`. 

You can also configure to run your deployment to all of the machines at once, half at a time, or one at a time. If the deployment should fail you can specify that the machines rollback. 

CodeDeploy is also configured by a YAML file, `appspec.yml`.

```
version: 0.0
os: linux
files:
hooks:
  BeforeInstall:
    - location: bin/codedeploy.sh
      timeout: 600
      runas: root
```

When CodeDeploy runs there are various hooks you can configure events for. In our case we run the `codedeploy.sh` script when the `BeforeInstall` hook is fired on each instance.

```
#!/bin/bash
# The appsec.yml file prohibits calling arbitrary scripts. This stub shell script
# downloads and executes the shell script specified in the EC2 instance User Data field.
# The user data shell script calls the deploy script baked in to the AMI running on the server.

# Fetch the user data script associated with this type of instance and save it into a temporary shell script
sudo curl http://169.254.169.254/latest/user-data > temp.sh

# Execute the shell script to run the update
sudo bash temp.sh

# Clean up
sudo rm temp.sh
```

Since the `appsec.yml` file prohibits calling arbitrary scripts we have to jump through some hoops to make things work. We make an HTTP request from the instance to an endpoint to get metadata about the EC2 instance. When we created our instance we specified a line in the User Data field:

```
#!/bin/bash
source /var/www/spiritedmedia.com/scripts/deploy-production.sh
```

This script simply executes another script already included on the machine baked into the server image used to start the EC2 instance. We tell CodeDeploy to create a new temporary script that calls this script already on our server.
```
#!/bin/bash
# Shell script to update the app level with the latest changes from GitHub (ex. when called form AWS CodeDeploy)
# Should be placed in /var/www/spiritedmedia.com/scripts/ and run as root

cd /var/www/spiritedmedia.com/htdocs/

# Force git pull
git fetch --all
git reset --hard origin/master

# Reset file ownership
chown -R www-data:www-data /var/www/spiritedmedia.com/htdocs/
```
This final deployment script goes to the root directory of our application and does a hard reset of the master branch. Other tasks include resetting file ownership, copying static assets to an S3 bucket and eventually to our CDN, flushing caches, and restarting nginx for good measure. 

And that is how we autmatically deploy our code from start to finish.


## Tips and Tricks

If a build fails in CircleCI you can rebuild it and enable SSH access. When it fails again the container will stay up for 30 minutes and you can SSH into the machine and poke around. Super helpful for debugging. 

Don't tie your access credentials to a single user. Make a separate GitHub user and use that for managing credentials to different servers. That way if someone leaves your organization your entire workflow doesn't break down due to access issues. GitHub calls these "Machine Users" See https://developer.github.com/v3/guides/managing-deploy-keys/#machine-users

If you want to set-up notifications for monitioring your deployment (like to a Slack channel) see https://medium.com/cohealo-engineering/how-set-up-a-slack-channel-to-be-an-aws-sns-subscriber-63b4d57ad3ea 

If you don't need the codepipeline artifacts be sure to setup Lifecycle rules in S3 to occassionally delete the file buildup

## Pricing

CircleCI has a free plan limitied to 1000 build minutes (16.666 hours) per month. It's $50/month per extra container after that for unlimited build minutes. They also give open source projects 4 containers for free.

AWS CodeBuild is $0.005 per build minute for the smallest tier. The first 100 minutes are free.

The break even point is 168.333 hours of build time. If you spend more time doing that then you should go with CircleCI otherwise use AWS CodeBuild.

## CircleCI vs AWS CodeBuild

I setup our build pipeline 2 years ago and haven't had issues since. This was long before AWS introduced their competitior CodeBuild. They work pretty much in the same way; define build steps in a YAML file in your code repo.

Pros of CircleCI
 - Awesome debugging experience
 - Much easier to set-up paralell jobs
 - Pricing if you need 1000 build minutes or less

Cons of CircleCI
 - Expensive once you need to move past the free tier

Pros of AWS CodeBuild
 - Pricing if you need more than 1000 build minutes
 - Run as many concurrent jobs as you need

Cons of AWS CodeBuild
 - More work to set up (you need codepipeline for example)
 - Breaking up one job into multiple parts is difficult