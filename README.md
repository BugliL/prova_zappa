# Zappa Test
This repository is to try to work with Zappa and a dummy Django project, from
installing all requisites to deploy a real project (in python3).

**Sources**
 - Most of this article is copied from the original documentation, cutted for my job and my work ([https://edgarroman.github.io/zappa-django-guide/](https://edgarroman.github.io/zappa-django-guide/))

## Setup Local Account Credentials
When you interact with AWS, you use AWS security credentials to verify who you are and whether you have permission to access the resources that you are requesting. In other words, security credentials are used to authenticate and authorize calls that you make to AWS.
Access keys consist of an access key ID (like AKIAIOSFODNN7EXAMPLE) and a secret access key (like wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY). You use access keys to sign programmatic requests that you make to AWS whether you're using the AWS SDK, REST, or Query APIs.... Access keys are also used with command line interfaces (CLIs).
You can create new access keys for the account by going to the Security Credentials page. In the [Access Keys section](https://console.aws.amazon.com/iam/home?#security_credential), click Create New Access Key.

### Configure the environment
The first thing to do is to set up AWS Credentials. The best way to do that is using a config file. You can store multiple sets of keys for different environments using profiles. In addition, you can provide multiple profiles that provides some isolation between AWS accounts and/or roles.
Regions are the locations of server farms in which you can deploy apps
In my case, the region is Europe
```
[default]
aws_access_key_id = your_access_key_id
aws_secret_access_key = your_secret_access_key

[zappa]
region=eu-west-1
aws_access_key_id = your_access_key_id_specific_to_zappa
aws_secret_access_key = your_secret_access_key_specific_to_zappa
```
to use the profile zappa as default
```bash
$ export AWS_PROFILE=zappa
```

## Setup your Environment
### Docker
The main goal of using Docker is to create an environment that closely matches the AWS lambda environment. The closer it matches, then there will be less difficult-to-debug problems.
The folks from lambci have created github repo called
docker-lambda ([lambci/lambda:build-python3.6](https://hub.docker.com/r/lambci/lambda/tags/)) that accurately reflects the lambda environment can be used for
 - A 'build' image for compilation, package creation, and deployment
 - A 'run' image for testing and execution of your code

#### Inital Setup
 - Install Docker
 - docker pull lambci/lambda:build-python3.6
 - Create a shortcut that allows AWS credentials to pass through to the docker container
   ```bash
   $ alias zappashell3='docker run -ti -e AWS_PROFILE=$AWS_PROFILE -v $(pwd):/var/task -v ~/.aws/:/root/.aws  --rm lambci/lambda:build-python3.6 bash'
   $ alias zappashell3 >> ~/.bash_profile   
   ```
   Example of hardcoding the alias:
   ```bash
   $ alias zappashell3='docker run -ti -e AWS_PROFILE=zappa -v $(pwd):/var/task -v ~/.aws/:/root/.aws  --rm lambci/lambda:build-python3.6 bash'
   ```

#### Create Virtual environment
If everything is gone ok, it will be possible to run the docker bash:
```bash
$ cd /your_zappa_project
$ zappashell3
bash-4.2#
```
Next, create the required virtual environment, activate it, and install needed dependencies
```bash
bash-4.2# virtualenv ve
bash-4.2# source ve/bin/activate
(ve) bash-4.2# pip install -r requirements.txt
(ve) bash-4.2# pip install zappa
.....
Successfully installed PyYAML-3.12 Unidecode-1.0.22 ... zappa-0.46.2
```

Since the virtual environment is contained in the current directory, and the current directory is mapped to your local machine, any changes you make will be persisted between Docker container instances. But if you depend on libraries that are installed in the system (essentially anything out of the current directory and virtual environment), they will be lost when the container exits. The solution for this is to create a custom Dockerfile.

#### Use zappa
All zappa commands can be used to deploy your project:
```bash
$ cd /your_zappa_project
$ zappashell3
bash-4.2#
bash-4.2# source ve/bin/activate
(ve) bash-4.2# zappa status dev
zappa status dev
(python-dateutil 2.7.3 (/var/runtime), Requirement.parse('python-dateutil<2.7.0,>=2.6.1'), {'zappa'})
Oh no! An error occurred! :(
```

### Core Django setup
This section documents setting up a Django project with only core Python functionality responding to HTTP calls. The value of this core walkthrough could be to power an API driven compute engine or a event-driven data processing tool without the need to provide a UI.

#### Setup Zappa
```
(ve) bash-4.2# zappa init

███████╗ █████╗ ██████╗ ██████╗  █████╗
╚══███╔╝██╔══██╗██╔══██╗██╔══██╗██╔══██╗
  ███╔╝ ███████║██████╔╝██████╔╝███████║
 ███╔╝  ██╔══██║██╔═══╝ ██╔═══╝ ██╔══██║
███████╗██║  ██║██║     ██║     ██║  ██║
╚══════╝╚═╝  ╚═╝╚═╝     ╚═╝     ╚═╝  ╚═╝

Welcome to Zappa!

Zappa is a system for running server-less Python web applications on AWS Lambda and AWS API Gateway.
This 'init' command will help you create and configure your new Zappa deployment.
Let's get started!
```

It will ask you some questions about the configuration:
 - Environment name
    ```
    Your Zappa configuration can support multiple production stages, like 'dev', 'staging', and 'production'.
    What do you want to call this environment (default 'dev'): dev
    ```
 - Profile name
    ```
    AWS Lambda and API Gateway are only available in certain regions. Let's check to make sure you have a profile set up in one that will work.
    We found the following profiles: default, and zappa. Which would you like us to use? (default 'default'): zappa
    ```
 - Bucket where to upload source code
    ```
    Your Zappa deployments will need to be uploaded to a private S3 bucket.
    If you don't have a bucket yet, we'll create one for you too.
    What do you want to call your bucket? (default 'zappa-k57qiirom'): zappa-django-test
    ```
 - The settings files
    ```
    It looks like this is a Django application!
    What is the module path to your projects's Django settings?
    We discovered: prova.prova.settings
    Where are your project's settings? (default 'prova.prova.settings'):
    ```
 - The region where do you want to upload and deploy the webserver, i prefer to edit it by hand because there's a bug that is not fixed while I'm writing
    ```
    You can optionally deploy to all available regions in order to provide fast global service.
    If you are using Zappa for the first time, you probably don't want to do this!
    Would you like to deploy this application globally? (default 'n') [y/n/(p)rimary]: n
    ```
 - Confirmation
    ```
    Okay, here's your zappa_settings.json:

    {
        "dev": {
            "aws_region": "eu-west-1",
            "django_settings": "prova.prova.settings",
            "profile_name": "zappa",
            "project_name": "task",
            "runtime": "python3.6",
            "s3_bucket": "zappa-django-test"
        }
    }

    Does this look okay? (default 'y') [y/n]: y

    Done! Now you can deploy your Zappa application by executing:

    	$ zappa deploy dev

    After that, you can update your application code with:

    	$ zappa update dev

    To learn more, check out our project page on GitHub here: https://github.com/Miserlou/Zappa
    and stop by our Slack channel here: https://slack.zappa.io

    Enjoy!,
     ~ Team Zappa!
    ```

#### Deploying zappa
```bash
(ve) bash-4.2# zappa deploy dev
(python-dateutil 2.7.3 (/var/runtime), Requirement.parse('python-dateutil<2.7.0,>=2.6.1'), {'zappa'})
Calling deploy for stage dev..
Creating task1-dev-ZappaLambdaExecutionRole IAM Role..
Creating zappa-permissions policy on task1-dev-ZappaLambdaExecutionRole IAM Role.
Downloading and installing dependencies..
 - psycopg2==2.7.5: Using locally cached manylinux wheel
 - sqlite==python36: Using precompiled lambda package
Packaging project as zip.
Uploading task1-dev-1536243693.zip (16.8MiB)..
100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 17.6M/17.6M [00:09<00:00, 1.70MB/s]
Scheduling..
Scheduled task1-dev-zappa-keep-warm-handler.keep_warm_callback with expression rate(4 minutes)!
Uploading task1-dev-template-1536243724.json (1.6KiB)..
100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1.59K/1.59K [00:00<00:00, 11.2KB/s]
Waiting for stack task1-dev to create (this can take a bit)..
 75%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████                                         | 3/4 [00:06<00:02,  2.72s/res]
Deploying API Gateway..
Deployment complete!: https://tqltu05s5i.execute-api.eu-west-1.amazonaws.com/dev
```

#### Updating the project
```
(ve) bash-4.2# zappa update dev
(python-dateutil 2.7.3 (/var/runtime), Requirement.parse('python-dateutil<2.7.0,>=2.6.1'), {'zappa'})
Calling update for stage dev..
Downloading and installing dependencies..
 - psycopg2==2.7.5: Using locally cached manylinux wheel
 - sqlite==python36: Using precompiled lambda package
Packaging project as zip.
Uploading task1-dev-1536244168.zip (16.8MiB)..
100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 17.6M/17.6M [00:10<00:00, 1.61MB/s]
Updating Lambda function code..
Updating Lambda function configuration..
Uploading task1-dev-template-1536244201.json (1.6KiB)..
100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1.59K/1.59K [00:00<00:00, 12.5KB/s]
Deploying API Gateway..
Scheduling..
Unscheduled task1-dev-zappa-keep-warm-handler.keep_warm_callback.
Scheduled task1-dev-zappa-keep-warm-handler.keep_warm_callback with expression rate(4 minutes)!
Deployment complete!: https://tqltu05s5i.execute-api.eu-west-1.amazonaws.com/dev
```
*at this point, Django Core Deployment, is completed, in next steps i will describe the procedure to add DB and how to serve static files*

## Setup database connection (PostGres SQL)
First of all we need the engine to use postGres SQL
```bash
(ve) bash-4.2# pip install
```

### Create database in AWS
First of all you need to create a DB in AWS, I will use postGres SQL [https://eu-west-1.console.aws.amazon.com/rds](https://eu-west-1.console.aws.amazon.com/rds)
You should record some key information we'll need here:
 - The username and password for the root user
 - The subnets (there should be at least two) in which we can access the database
 - The endpoint (hostname) of the database and the port

All data are provided selecting the database instance in the AWS RDS section
