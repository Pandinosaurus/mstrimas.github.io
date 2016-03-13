---
layout: post
title: "RStudio in the Cloud II: Syncing Code & Data with AWS"
published: true
excerpt: >
  Tutorial on transferring and syncing data between an Amazon Web Services (AWS)
  EC2 instance and your local machine, with GitHub and S3.
category: r
tags: r cloud
---

This is part II in a series of posts about using RStudio on Amazon Web Services (AWS). In [part I](http://strimas.com/r/rstudio-cloud-1/), I outlined how to quickly get RStudio Server running on an AWS EC2 instance. This gives you access to RStudio via a web browser with as much (or as little) computing power as you need for any given task.

Once you have RStudio up and running, you'll likely want to transfer or sync some data and/or code between the remote AWS instance and your local machine. This tutorial covers this syncing process.

# Goal

Typically, each project I'm working on gets its own folder, which is also an RStudio project and a git repository. Inside the project directory is an `R/` subdirectory containing R scripts and a `data/` directory that contains the data I'll be analyzing. For a variety of reasons, I usually avoid putting the data directory on GitHub. In this situation, if I want to run an R script in the cloud, I follow these steps:

1. Deploy an AWS EC2 instance with RStudio Server (see [part I](http://strimas.com/r/rstudio-cloud-1/))
2. Bring the RStudio project onto the instance from GitHub
3. Transfer the data to the AWS instance
4. Run the script
5. Transfer any outputs created by the scripts back to my local computer, or store them in the cloud
6. Terminate the instance

For the sake of having a concrete example to work with, I've created a [simple RStudio project](https://github.com/mstrimas/aws-example) for demonstration purposes. In this project, I look at global trends in forest loss using a data set taken from the UN [Food and Agriculture Organization's](http://www.fao.org/home/en/) [Forest Resources Assessment](http://www.fao.org/forest-resources-assessment/explore-data/en/). The R script that performs the analysis is on GitHub in the `R/` subdirectory. The dataset can be found [here](https://s3-us-west-2.amazonaws.com/strimas-bucket/data.zip). Download and unzip this file in preparation.

# Syncing code using GitHub

Syncing code with your EC2 instance is most easily done with **git** and [**GitHub**](https://github.com/). These tools are specifically designed for version control, collaborative coding, and keeping code in sync between different machines. Furthermore, [RStudio](https://www.rstudio.com/) has great integration with git and GitHub. If you're new to these tools, there are many great tutorials online. [Jenny Bryan's notes](http://stat545-ubc.github.io/git00_index.html) for STAT545 at UBC are an excellent place to start. For now I'll just demonstrate how to create an RStudio project on an EC2 instance from a repository on GitHub.

## Creating a project from a GitHub repository

Point your web browser to the public DNS of your EC2 instance, which is available via the *Instances* page in the [*EC2 Console*](https://us-west-2.console.aws.amazon.com/ec2/v2/). Then log in with `rstudio` as the username and the password you set. From the *File* menu, select *New Project...*, and click *Version Control* then *Git* to create a new project based on a git repository. Finally, fill in the correct URL for the example repository I've created (or a repo of your own).

<img src="/img/rstudio-cloud/github-repo.png" style="display: block; margin: auto;" />

## Syncing data using S3

Amazon's **Simple Storage Service (S3)** offers [extremely cheap](https://aws.amazon.com/s3/pricing/) (~$0.03 per TB per month), highly scalable cloud-based storage of objects of almost any size. In S3, data (i.e. files) are bundled together with metadata into [**objects**](https://en.wikipedia.org/wiki/Object_storage), and objects are organized into **buckets**. There are many ways to move data to and from an EC2 instance, but S3 is perhaps the simplest.

## Install the AWS CLI

To use S3, you'll need to install the AWS Command Line Interface (CLI) on your local machine. For Linux and Mac OS X users, run the following commands in the Terminal.

```
curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
unzip awscli-bundle.zip
sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
rm -r awscli-bundle*
```

For Windows users, use the [installer provided by AWS](http://docs.aws.amazon.com/cli/latest/userguide/installing.html#install-msi-on-windows).

## Configure the AWS CLI

Now you'll need to configure the AWS CLI to interact with AWS. This will need to be done on your local machine and the EC2 instance (via SSH). For this, you'll need a pair of security keys, which you can generate on the [*Security Credentials*](https://console.aws.amazon.com/iam/home?#security_credential) page of the AWS Console. Expand the *Access Keys* section, click *Create New Access Key* and a `credentials.csv` file should be downloaded. This file contains two keys that you'll need: an access key and a *secret* access key.

Open a Terminal window, type `aws configure` to start the CLI configuration, and paste in your access keys from the `credentials.csv` file when prompted:

```bash
aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
```

Go through this process on your local machine and the EC2 instance.

## Create a bucket

On S3, all files are stored in buckets, so let's create our first bucket. Open a Terminal window on your local machine and enter:

```bash
aws s3 mb s3://example-bucket
```
This creates a bucket named `example-bucket`. Note that bucket names must be globally unique across all of S3, so make sure you replace `example-bucket` with something more unique.

## Copy files to S3

If you haven't already done so, download and unzip the [data for this tutorial]( https://s3-us-west-2.amazonaws.com/strimas-bucket/data.zip). Now, in the Terminal, change to the `data/` directory you just downloaded, and run the following command to copy `fao-fra.csv` to S3:

```bash
aws s3 cp fao-fra.csv s3://example-bucket
```

## Sync data with EC2

Now that the data we want to work with is on S3, we'll need to bring that data onto our EC2 instance. Note that this is slightly complicated by the fact that we can only SSH into the instance as user `ubuntu`, but we log into RStudio as user `rstudio`.

SSH into your EC2 instance with the following command, making sure to fill in the correct location to your `.pem` file and the public DNS of your instance:

```bash
ssh -i ~/aws.pem ubuntu@ec2-52-36-52-70.us-west-2.compute.amazonaws.com
```

Change to the project directory in the `rstudio` home directory and create and `data/` subdirectory. `sudo -u rstudio` creates the directory as user `rstudio`, which allows you to add files to the directory through RStudio.

```bash
cd ~rstudio/aws-example/
sudo -u rstudio mkdir data
```

Now sync the contents of the S3 bucket with this `data/` directory:

```bash
sudo aws s3 sync s3://example-bucket data/
```

Log on to your cloud-based RStudio instance in the browser and open the RStudio project you created at the start of this tutorial. In the `data/` directory you should see the `fao-fra.csv` file you just synced from S3. Open the `forest-loss.r` script in the `R/` directory and source the file. Running this script will create two files in the `data/` directory: `fao-fra-region.csv` and `forest-change.png`. Let's put these files back on S3 to save them.

```bash
sudo aws s3 sync data/ s3://example-bucket
```

Now that the files are on S3, you can terminate the S3 instance safely. If you want to bring these files onto your local machine, just run the following command:

```bash
aws s3 sync s3://example-bucket .
```

# Further S3 details

To learn more about using S3 through the AWS CLI consult the [AWS CLI Command Reference](http://docs.aws.amazon.com/cli/latest/reference/s3/). Some particularly useful commands are:

- List buckets
```bash
aws s3 ls
```
- List files within a bucket
```bash
aws s3 ls s3://example-bucket
```
- Remove files from a bucket
```bash
aws s3 rm s3://example-bucket/fao-fra.csv
```
- Remove a bucket (must empty bucket first)
```bash
aws s3 rb s3://example-bucket/
```

Finally, to make files publicly available use the `--acl public-read` flag with `cp` or `sync`. For example: 

```bash
aws s3 cp fao-fra.csv s3://example-bucket/ --acl public-read
```

Files made public in this way are available through standard URLs of the form `http://bucket.s3.amazonaws.com/file`. For example, `fao-fra-csv` could be downloaded from `http://example-bucket.s3.amazonaws.com/fao-fra.csv`.
