---
layout: post
title: Using AWS Lookout for Vision for finding product anomalies
---

[Introduction](#introduction)\\
[Dataset](#dataset)\\
[Development environment setup](#development-environment-setup)\\
[Data gathering and preprocessing](#data-gathering-and-preprocessing)\\
[Create Amazon Lookout for Vision project](#create-amazon-lookout-for-vision-project)\\
[Create train and test datasets](#create-train-and-test-datasets)\\
[Create the model](#create-the-model)\\
[Evaluate the model](#evaluate-the-model)\\
[Conclusion](#conclusion)\\
[References](#references)

## Introduction

In this article I will demonstrate using Amazon Lookout for Vision service for detecting anomalies in product images.
Amazon Lookout For Vision is just a one of many AI services that Amazon offers. Amazon has plethora of AI/ML services ranging from health and industrial to code and devops. Lookout For Vision is relatively new service, offered by Amazon to general public in the Februrary 2021.

As stated by Amazon: **“Amazon Lookout for Vision uses AWS-trained computer vision models on images and video streams to find anomalies and flaws in products or production processes.”**

It employs a machine learning technique called “few-shot learning” and is able to train a model for a customer using as few as 30 baseline images. Applying computer vision and automating the QA process can reduce costs and increase revenue because of minimizing or complete avoiding production line shutdowns due to missed defects or quality inconsistencies. This is especially important in manufacturing industry where any delay in manufacturing process can prolong time to deliver products to customers and drastically increase costs.

Because building computer vision model from scratch requires large amounts of correctly labeled images it can be very tedious, time consuming and error prone. Beside that activities like defining computer vision project scope, gathering and pre-processing data, building computer vision model, deploying it and monitoring its performance in production has to be done by a team of experienced professionals including, data scientists, data engineers, machine learning engineers, devops engineers etc. Forming such a team of skilled professionals requires time and incurs additional costs.

**The main benefits of Amazon Lookout for Vision service are**:
* high accuracy;
* low cost;
* does not require machine learning experience;
* can process thousand of images per hour in real time.

Amazon Lookout for Vision workflow will be demonstrated in the following sections.

## Dataset

I will use images provides by DAGM (Deutsche Arbeitsgemeinschaft für Mustererkennung e.V., the German chapter of the International Association for Pattern Recognition).\\
Link to the web site where images can be downloaded is provided in the [References](#references) section. Images are artificially generated but similar to real world products.\\
Each image represents a background texture. Images are divided into two datasets, one with defects and the other without them.\\
For this demonstration we will use Class3 and Class3_def datasets where Class3 represents a dataset with normal images and Class3_def a dataset with images containing some form of defects.

## Development environment setup

For the job of data gathering and preprocessing an AWS instance will be created in free tier or to speed things up we can use Cloud Shell service which is a browser based shell that provides CLI to AWS resources in your account and it comes with additional tools pre-installed.

For this tutorial I will use Cloud Shell service.
Before starting we need to install or update AWS CLI if it is already installed.
If you already have AWS CLI installed (if using Cloud Shell service) check its version using:

> aws –version

In my Cloud Shell session the command prints:

> aws-cli/2.1.16 Python/3.7.3 Linux/4.14.209-160.339.amzn2.x86_64 exec-env/CloudShell exe/x86_64.amzn.2 prompt/off

We will download new AWS CLI version using curl:

> curl -O “https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip”

## Data gathering and preprocessing

Now we are ready to proceed with downloading and preparing our dataset. All AWS resources for this demonstration will be created in the us-east-1 region. First we will download archives to our AWS instance:

> curl -O -L -k https://conferences.mpi-inf.mpg.de/dagm/2007/Class3.zip
> curl -O -L -k https://conferences.mpi-inf.mpg.de/dagm/2007/Class3_def.zip

Unzip the archives to current folder:

> unzip Class3.zip
> unzip Class3_def.zip

There are two bash scripts acompanying this demonstration.\\
The first is **make_images_subset.bash** which creates datasets folders for training and test images and copies subset of images in each of the folders.\\
The second one is **create_bucket_manifest.bash** which creates appropriate manifest files for training and test images. Because the model training time and cost depends on the number of images used, for this demonstration we will use 40% of total number of images and that means 400 normal and 60 defects images.

    Amazon Lookout for Vision service requires that the test dataset has at least 10 images labeled as Normal and 10 images labeled as Anomaly.

The images will further be split into training and test images where 80% of images will belong to the training and the remaining 20% to test images.

In order to create a subset of total images we will run the make_images_subset.bash bash script.
You can download the script [here](https://raw.githubusercontent.com/ranzer/aws_lookoutvision_demo/main/make_images_subset.bash).

> curl -O https://raw.githubusercontent.com/ranzer/\
> aws_lookoutvision_demo/main/make_images_subset.bash

Because the BASH scripts contain Windows line endings it is necessary to remove carriage return characters before running them. Remove Windows line endings:

> sed -i ‘s/\r$//’ make_images_subset.bash

The script accepts 4 arguments:
1. A folder where source images will be copied (it will be created if it does not exist).
2. Percentage of total number of source images to copy (0-100).
3. Normal images folder path.
4. Defects images folder path.

Let us create variable for the project name:

> $PROJECT_NAME=dagm-product-images-anomalies

Before running the script let us make it executable and run it:

> chmod u+x make_images_subset.bash
> ./make_images_subset.bash $PROJECT_NAME 10 Class3 Class3_def

Now it is time to create new S3 bucket in the us-east-1 region.

> BUCKET_NAME=$PROJECT_NAME\ aws s3 mb s3://$BUCKET_NAME --region us-east-1

Let us upload local images to S3 bucket:

> aws s3 sync ./$PROJECT_NAME s3://$BUCKET_NAME

After images are successfully copied to the bucket we can proceed with creating manifest files.

In order to create a test or training dataset in AWS Lookout for Vision service appropriate training and test manifest files have to be created.
Manifest file contains information about the images and image labels that you can use to train and test a model. For this task we will use the create_manifest.bash script. The script can be download [here](https://raw.githubusercontent.com/ranzer/aws_lookoutvision_demo/main/create_manifests.bash).

The create_manifest.bash script accepts 5 arguments:
1. Directory containing images what will be included in the manifest.
2. S3 bucket name where images are stored.
3. 1 or 0 label depending on whether an image is normal or with defects.
4. String that represents image class name (normal or anomaly in this case).
5. Prefix of the created manifest file. The script creates train and test manifest files if they do not exist or appends data to existing ones.

First we will create manifest for normal images:

> ./create_manifests.bash ./$PROJECT_NAME/normal $BUCKET_NAME anomaly 0 manifest

Repeat previous command for defects images:

> ./create_manifests.bash ./$PROJECT_NAME/anomaly $BUCKET_NAME normal 1 manifest

Upload manifest files to the S3 bucket:

> aws s3 cp manifest_train.txt s3://$BUCKET_NAME
> aws s3 cp manifest_test.txt s3://$BUCKET_NAME

Now that we have data prepared and uploaded it is time to create Amazon Lookout for Vision project. The general Amazon Lookout for Vision workflow is described here. All steps in the workflow will be run using AWS CLI.

## Create Amazon Lookout for Vision project

First step in the workflow is to create new project.

> aws lookoutvision create-project --project-name $PROJECT_NAME

If there are multiple Amazon Lookout for Vision projects they can be listed using following command:

> aws lookoutvision list-projects

To check whether the project is created from the AWS Management Console open the web page. If this is the first time you accessing Amazon Lookout for Vision page click **“Get started”** button.
On the popup windows with title **“First time setup”** click **“Create S3 bucket”** which will create S3 bucket where files for all our Amazon Lookout for Vision projects are stored. After the bucket is created you can navigate all Amazon Lookout for Vision projects using the side menu option **“Projects”**.

## Create train and test datasets

In order to create Amazon Lookout For Vision dataset we need to provided following information:
* project name to which datasets belong;
* type of the dataset, train or test;
* dataset source describing location of manifest file that Amazon Lookout for Vision uses to create dataset.

The dataset source can be described using shorthand syntax:

> GroundTruthManifest={S3Object={Bucket=string,Key=string,VersionId=string}}

or JSON syntax:

```json
{
    "GroundTruthManifest": {
        "S3Object": {
            "Bucket": "string",
            "Key": "string",
             "VersionId": "string"
        }
    }
}
```

For simplicity we will use shorthand syntax and because using JSON syntax requires escaping all quote characters except the first and the last one.

> aws lookoutvision create-dataset --project $PROJECT_NAME --dataset-type train --dataset-source “GroundTruthManifest={S3Object={Bucket=$BUCKET_NAME,Key=manifest_train.txt}}”

Equivalent command using JSON syntax would be:

> aws lookoutvision create-dataset --project $PROJECT_NAME --dataset-type train --dataset-source “{"GroundTruthManifest": {“\S3Object":{"Bucket":"$BUCKET_NAME","Key":"manifest_train.txt"}}}”

If the command is run succesfully you should receive response similiar to this:

```json
{
    "DatasetMetadata": {
        "DatasetType": "train",
        "CreationTimestamp": "2021-04-07T21:44:22.948000+00:00",
        "Status": "CREATE_IN_PROGRESS",
        "StatusMessage": "The dataset is creating."
    }
}
```

We need to repeat the previous command for the test dataset:

> aws lookoutvision create-dataset --project $PROJECT_NAME --dataset-type test --dataset-source “GroundTruthManifest={S3Object={Bucket=$BUCKET_NAME,Key=manifest_test.txt}}”

To check whether training and test datasets are succesfully created run command that outputs description about each dataset:

> aws lookoutvision describe-dataset --project $PROJECT_NAME --dataset-type train

The previous command will output JSON containing dataset description. Check the **“StatusMessage”** property, if it has value **“The dataset was created successfully.”** then dataset is created without an error. Run the same check for the test dataset:

> aws lookoutvision describe-dataset --project $PROJECT_NAME --dataset-type test

## Create the model

The model can be created using the following command:

> aws lookoutvision create-model –project-name $PROJECT_NAME –output-config “S3Location={Bucket=$BUCKET_NAME,Prefix=DAGM_Model}”

The **--output-config** option specifies the location where Amazon Lookout for Vision saves the training results. This training results file contains information like model description, its training status and the most important the model’s performance metrics like F1 score, recall and precision. The output of the previous command should be similar to:

```json
{
    "ModelMetadata": {
        "CreationTimestamp": "2021-04-08T00:00:43.232000+00:00",
        "ModelVersion": "1",
        "ModelArn": "arn:aws:lookoutvision:us-east-1:177231263820:model/dagm-product-images-anomalies/1",
        "Status": "TRAINING",
        "StatusMessage": "The model is being trained."
    }
}
```

The previous command will create the model and start model training. In order to check model’s status we need to run the command:

> aws lookoutvision describe-model --project $PROJECT_NAME --model-version 1

Model’s status can also be checked from AWS Management Console. Go to the AWS Lookout for Vision main page and from the left sidebar menu open the project dropdown list. Click the **“Models”** button and check the **“Status”** column.
Training the model can take a while so be patient.
After the model training finishes you can check which images had the lowest confidence prediction by opening the model’s page and sorting images by their confidence score.

<img src="/assets/images/aws_lookout_vision_images_confidence_small.png" />

The confidence score is expressed in percentage that an image belongs to a specific class. You cannot assume that all results with a very high confidence score are very good or very bad in case of low confidence score. However, you can generally assume that results with a higher confidence score are more likely to be better.

With a provided insight about confidence prediction for each image in a dataset you can route low confidence predictions from Amazon Lookout for Vision to human reviewers (process engineers, quality managers, or operators) for analysis and manually labeling of images if necessary.

## Evaluate the model

We will evalute the model by running anomaly detection on a randomly selected image. For this purpose we will create “evalute” folder in the “dagm-product-images-anomalies” bucket and put in it a couple of normal and images with anomalies. First we need to find normal and images with defects which are not used in training and test datasets. You can run the following command to find normal and defect images that are not included in the training and test datasets:

For normal images:

> diff -q Class3 $PROJECT_NAME/normal/ \| grep -i class3 \| cut -d: -f2 \| tr -d ' '

For defect images:

> diff -q Class3_def $PROJECT_NAME/anomaly \| grep -i class3 \| cut -d: -f2 \| tr -d ' '

We need to start the model in order to perform anomaly detection:

> aws lookoutvision start-model --project-name $PROJECT_NAME --model-version 1 --min-inference-units 1

The **--min-inference-units** option indicates how many hours of processing we need and can support up to 5 Transaction Pers Second (TPS). In production environment because of the real-time processing of images it makes sense to increase the inference units number to more then 1.

The command will output message that the model is running (“HOSTED” state) if it is executed successfully:

```json
{
    "Status": "STARTING_HOSTING"
}
```

Hosting can take a while depending on the model complexity. To check whether the model hosting is completed run the following command:

> aws lookoutvision describe-model --project-name $PROJECT_NAME --model-version 1

The **“StatusMessage”** property should have **“The model is running.”** value.

After the model is hosted we can run anomaly detection. For evaluation we will use images 100.png and 117.png from the Class3 and Class3_def folders.

> aws lookoutvision detect-anomalies --project-name $PROJECT_NAME --model-version 1 --content-type image/png --body ./Class3/100.png
> aws lookoutvision detect-anomalies --project-name $PROJECT_NAME --model-version 1 --content-type image/png --body ./Class3/117.png
> aws lookoutvision detect-anomalies --project-name $PROJECT_NAME --model-version 1 --content-type image/png --body ./Class3_def/100.png
> aws lookoutvision detect-anomalies --project-name $PROJECT_NAME --model-version 1 --content-type image/png --body ./Class3_def/117.png

Running the first command gives following output:

```json
{
    "DetectAnomalyResult": {
        "Source": {
            "Type": "direct"
        },
        "IsAnomalous": false,
        "Confidence": 0.5925385355949402
    }
}
```

We see from the output that model accurately predicted that the image is normal but will lower confidence score of about 0.59.

Running the second command produces similar output but with much greater confidence prediction:

```json
{
    "DetectAnomalyResult": {
        "Source": {
            "Type": "direct"
        },
        "IsAnomalous": false,
        "Confidence": 0.9584138989448547
    }
}
```

Let us now evaluate images with defects. Running the third command produces the following output:

```json
{
    "DetectAnomalyResult": {
        "Source": {
            "Type": "direct"
        },
        "IsAnomalous": false,
        "Confidence": 0.5542072057723999
    }
}
```

Here the model wrongly predicted that the image is normal.

Running the last command produces the output with correct prediction and high confidence score:

```json
{
    "DetectAnomalyResult": {
        "Source": {
            "Type": "direct"
        },
        "IsAnomalous": true,
        "Confidence": 0.9586485624313354
    }
}
```

After you finish the evaluation stop the model by running:

> aws lookoutvision stop-model –project-name $PROJECT_NAME –model-version 1

## Conclusion

During evaluation our model made prediction error on one of the images with defects. This does not indicate that the model is not useful. It indicates that the dataset using in training is not large enough. This prediction error can be minimized if we use all images during training rather then just a subset.\\
Amazon Lookout for Vision service provides easy to use and intuitive API for developers as well as simple web interface for administrating datasets, models and running trial detections on images stored in S3 buckets or in a local computer.\\
From this simple demo we showed that Amazon Lookout for Vision service provides quick and ease access to high performance and high quality computer vision machine learning model for anomalies detection.

## References

1. https://aws.amazon.com/machine-learning/
2. https://press.aboutamazon.com/news-releases/news-release-details/aws-announces-general-availability-amazon-lookout-vision
3. https://conferences.mpi-inf.mpg.de/dagm/2007/prizes.html
4. https://docs.aws.amazon.com/lookout-for-vision/latest/developer-guide/create-dataset-ground-truth.html
5. https://docs.aws.amazon.com/lookout-for-vision/latest/developer-guide/getting-started.html
6. https://dev.to/jwm4/how-to-select-a-threshold-for-acting-using-confidence-scores-34cj
