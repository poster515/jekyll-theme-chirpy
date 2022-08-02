---
title: Intro to AWS and Dynamo DB
author: Joe Post
date: 2022-08-01 21:10:00 +0800
categories: [Blogging, Tutorial]
tags: [AWS, dynamodb, python, bash, conda]
---

## Purpose

I really wanted to figure out how to publish data to AWS DynamoDB. This desire came out of a side project where I wanted to create a distributed sensor network that had cloud-based data collection. It turns out that this task is at its core very simple. For this short tutorial you'll need an AWS with admin privileges (check with your organization if you're unsure). I have an account that I use for personal projects so this was simple. 

## Motivation

AWS DynamoDB is, per the documentation: "a fully managed NoSQL database service that provides fast and predictable performance with seamless scalability." NoSQL means there is no defined _schema_, basically you can upload any unstructured data that you want, assuming that it meets certain type requirements. You don't need to define the key names for your table in advance, although it can significantly speed up access times later, should you plan ahead.

## Architecture

For this project, the only goal was to publish data to a DynamoDB. Amazon has a [tremendous amount of documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) on this topic; however, it can be rather confusing for newcomers and I wanted to take the chance to show others what I did for this seemingly simple task. 

My setup consists of a Chromebook running Linux, and that's really it! In theory, this setup can be achieved running on a Raspberry Pi or otherwise embedded Linux, which was the ultimate goal of this project until other life events got in the way. If you want to run on a Raspberry Pi, you'd do everything the exact same as this tutorial (as an added bonus via SSH).

## Creating Credentials

In order to interface your device with AWS, you need credentials. For this I downloaded the AWS command line interface (CLI) using the following:

```bash
sudo apt install aws-shell
```

Before running anything with the AWS CLI though, you need to create an account access ID and secret key via your AWS console online. This will allow you to create a secret key that you'll have visibility on once, and never again via the console. To do this, go to Users, and then Create New User, selecting the 'Access Key' option. Once you generate your account access ID and secret key, it is recommended you stay on that page, with the key accessible, or otherwise copy the key to a temporary location for deletion later.

Once installed, run the following command:

```bash
aws configure
```

There will be an intuitive walk-through of the various pieces of information needed to configure your device for use with AWS. You will need the account ID and secret key from the console from earlier. 

## (Optional) Create a Python Environment

I use conda as my defacto python package manager, for Windows. However, conda sometimes doesn't play well with Linux, so I use a bash script to create an environment as follows:

```bash
#!/bin/bash

if ! command -v conda &> /dev/null
then
    echo "conda program could not be found, using python venv..."
	# first, make sure a bunch of pre-reqs are installed.
	# these are optional but are used by a myriad of python packages.
	sudo apt-get update; sudo apt-get install make build-essential libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm \
  libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev

	# create a new virtual package
	python3 -m venv new_env_name
	source ./new_env_name/bin/activate

    # get the latest version of pip
	python3 -m pip install --upgrade pip

	for PACKAGE in numpy matplotlib opencv-python setuptools scipy boto3
	do
		python3 -m pip install $PACKAGE
	done

	echo \n===========================================\n

	pip list --local
	
else
	echo "found conda software, creating virtual environment..."
	conda env create
fi
```

If you have conda, you'll need an environment.yml file similar to the following for the above script to work:

```yml
name: new_env_name
channels:
  - conda-forge
dependencies:
  - python=3.8.0
  - numpy
  - matplotlib
  - scipy
  - boto3
  - pip
  - pip:
    - mplcursors
```

Once your python environment is created simply activate it using 'source <path_to_env>/bin/activate'. The above scripts install several useful python packages, although the only real package we need is [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html#using-boto3).

## Creating a Table

For this portion, we'll be creating a table using a python script. Each of these attributes is described in exhaustive detail in the above link thankfully. The below will create a very simple table with a few intuitive keys.

```python
import boto3

dynamodb = boto3.resource('dynamodb')

# how to create a new table
table = dynamodb.create_table(
    AttributeDefinitions=[
		{
            'AttributeName': 'sensor_ID',
            'AttributeType': 'S'
        },
        {
            'AttributeName': 'sensor_iso_time',
            'AttributeType': 'S'
        }
    ],
    TableName='environment_data',
    # create partition key using the sensor ID, and use the iso time as a sort key.
    KeySchema=[
        {
            'AttributeName': 'sensor_ID',
            'KeyType': 'HASH'
        },
        {
            'AttributeName': 'sensor_iso_time',
            'KeyType': 'RANGE'
        },
    ],
    BillingMode='PAY_PER_REQUEST',
    Tags=[
        {
            'Key': 'version',
            'Value': '1.0.0'
        },
    ],
    TableClass='STANDARD_INFREQUENT_ACCESS'
)

# Wait until the table exists.
table.wait_until_exists()

# Print out some data about the table.
print(table.item_count)
```

You should observe a valid print out of the number of items in the table (in this case 0). The reason the above works is because of the boto3.resouce('dynamodb) line. This method looks in the users home folder for aws credentials, which it should find after you configure AWS via the CLI from earlier.

## Publishing Data


The below script demonstrates how to publish data to your new table using the boto3 package:

```python
import boto3
import numpy as np
import datetime
from time import sleep

# create resource and obtain relevant table
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('environment_data')

# create a couple test entries in table
with table.batch_writer() as batch:
    for i in range(10):
        batch.put_item(Item={
                            'sensor_ID': str(123),
                            'sensor_iso_time': datetime.datetime.now().isoformat(),
                            'pressure': str(960 + np.random.random() * 10),
                            'temperature': str(25 + np.random.random() * 2),
                            'humidity': str(70 + np.random.random() * 25)
                        })
        sleep(1)

# Print out some data about the table.
print(table.item_count)
```

## Conclusion

I hope this brief tutorial has been helpful. AWS can be an intimidating cloud service provider for beginners, but we've demonstrated how to:

1. Create a new AWS User and configure AWS credentials on the target device
2. Create a new table using python, from the target device.
3. Publish data to your new table using python, from the target device.

Thanks for reading! Any feedback is appreciated.

For more knowledge about Jekyll posts, visit the [Jekyll Docs: Posts](https://jekyllrb.com/docs/posts/).