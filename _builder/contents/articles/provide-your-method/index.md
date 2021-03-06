---
title: Provide Your Method
template: article.jade
---

# Introduction

 Time: 60 min

For this Tutorial, the aim is to create a Docker Image that is able to perform Otsu Binarization and then use this image on DIVAServices.
To simplify the process, we provide the jar-file for this method and we will also guide the process step by step.
The jar-file can be downloaded [here](/DIVAServicesweb/articles/provide-your-method/sources/otsubinarization.jar) and a test image to use [here](/DIVAServicesweb/articles/provide-your-method/sources/example.png)

The solutions are available in a zip-file do download [here](/DIVAServicesweb/articles/provide-your-method/solution/solution.zip)


The Process will involve the following steps:
 - Test the method locally
 - Create a shell script that runs the method
 - Create the Docker Image for the method
 - Test the Docker Image
 - Upload the Docker Image to Docker Hub
 - Create the POST request to install the method on DIVAServices
 - Test the method on DIVAServices

The steps performed in this Tutorial are applicable to other methods as well.

# Pre-Requisites

In order to run this tutorial you need to have an installation of the following tools:
- docker (see: [Docker Installation](https://docs.docker.com/install/), for this purposes the Docker Community Edition is enough)
- python
- Java (>= java 8)
- An account on [Docker Hub](http://hub.docker.com), remember your username as we will need it later for tagging your Docker Image.

# Procedure

## 1. Run the method locally

In a first step we want to ensure, that the application we want to deploy is actually working locally.
The application takes two parameters:
 - path to input image
 - path to output folder

This means we can execute the application like this (run in the directory where you downloaded the jar-file in a terminal):
```
java -jar otsubinarization.jar example.png ./
```
This will create a file name `otsuBinaryImage.png` in the same folder.

## 2. Create a shell script that runs the method
As a next step we want to create a shell script that takes the necessary parameters as inputs and executes the method. We will use this shell script again within the Docker image.

Create a file named `script.sh` in the same folder as the jar file with the following contents:

``` bash
#!/bin/sh
inputImage=${1}
outputFolder=${2}

java -jar otsubinarization.jar ${inputImage} ${outputFolder}
```
and make it executable using
``` bash
chmod +x script.sh
```
Now we can test the script with (run in the same folder as the jar file)
``` bash
./script.sh example.png ./
```
Which should generate the same `otsuBinaryImage.png` as before.

To use this script inside the Docker Image we will have to use full paths to reference all files, so we change the content of this file to:
``` bash
#!/bin/sh
inputImage=${1}
outputFolder=${2}

java -jar /input/otsubinarization.jar ${inputImage} ${outputFolder}
```
And remember we will have to put the jar file to `/input/otsubinarization.jar`


## 3. Create the Docker Image for the method
For this Tutorial we will build the Image using a practice similar to Makefiles. For this, create a file with the name `Dockerfile` (no file extension) in the `sources` directory and add the following contents:

``` docker
FROM openjdk:8
LABEL maintainer="your.name@email.com"
COPY otsubinarization.jar /input/otsubinarization.jar
COPY script.sh /input/script.sh
```

This will do the following:
`FROM openjdk:8` Is an instruction to use a base installation of openJDK 8. Meaning all our changes will be built on top of a clean environment that has openjdk available.

There are images available for almost any environments one might want to use and they can be search on the [Docker Store](https://store.docker.com)

`LABEL maintainer="your.name@email.com"` Provides information about who maintains this Docker Image. This should be your email address.

`COPY <src> <dest>` Copies a local file into the Docker Image. This will make the files permanently available inside the image.

The complete reference on Dockerfiles is available [here](https://docs.docker.com/engine/reference/builder/).

We can now build the Docker Image using the command:

``` bash 
docker build -t docker_hub_account_name/test_otsubinarization -f Dockerfile .
```

This will instruct Docker to read the Docker file and execute all the commands.
With `-t` we can provide a name for the Docker Image that is created and that can be used as reference later on.

If this process runs fine you should fine a line like this at the bottom of your console:

``` bash
Successfully tagged docker_hub_account_name/test_2018_otsubinarization:latest
```

## 4. Test the Docker Image
We can now test if the Docker Image is running as expected. For this we can execute the newly created image like this:

``` bash
docker run -it --rm -v /FULL_PATH_TO/example.png:/input/example.png -v /FULL_PATH_TO_OUTPUT_FOLDER/:/output/ docker_hub_account_name/test_otsubinarization sh /input/script.sh /input/example.png /output/
```

This command is built from the following:
 - `docker run` Executes a command in a new container
 - `-it` runs the container in interactive mode and allocates a pseudo TTY (not important)
 - `--rm` removes the container from the system after the execution
 - `-v <src> <dst>` mounts a local file/folder from \<src\> (needs to be full path) into the container at \<dst\>. If \<dst\> does not exist it will be created automatically.
 - `docker_hub_account_name/test_otsubinarization` reference to the image that should be used for running the command
 - `sh /input/script.sh /input/example.png /output/` The command that will be executed 

If this result operates fine, you should find the `otsuBinaryImage.png` in the `<src>` folder provided to the `/output/` directory/.

## 5. Upload image to Docker Hub
The Docker Image can now be uloaded and made available to others with:

``` bash
docker push docker_hub_account_name/test_otsubinarization
```

This will upload the image into the official Docker repository, which is publicly available. 
Discussions about hosting a closed DIVAServices repository are currently ongoing.

## 6. Create the POST request for putting the method on DIVAServices

Now we can install the method on DIVAServices. For the purpose of this Tutorial we will install it on a test installation of DIVAServices.
For live deployment there is also [DIVAServices-Management](http://divaservices.unifr.ch/management), but we need to add support for Docker Image deployment to it.

A method is deployed by executing an HTTP POST request to the `/algorithms` route of DIVAServices.
To deploy the Otsu Binarisation we need to attach a JSON request body that looks as follows:

We provide a python script `solutions/createMethod.py` that can be adapted to execute this request.

``` JSON
{
  "general": 
  {
    "name": "Firstname Otsu Binarization",
    "description": "Otsu Binarization",
    "developer": "YOUR NAME",
    "affiliation": "YOUR AFFILIATION",
    "email": "your.name@email.com",
    "author": "YOUR NAME",
    "DOI": "10.1109/TSMC.1979.4310076",
    "type": "binarization",
    "license": "Other",
    "ownsCopyright": "1"
  },
  "input": [
      {
        "file": {
            "name": "inputImage",
            "description": "The input image to binarize",
            "options": {
                "required": true,
                "mimeTypes": {
                    "allowed": [
                        "image/jpeg",
                        "image/png"
                    ],
                    "default": "image/jpeg"
                }
            }
        }
      },
      {
        "outputFolder": {}
      }
    ],
  "output": [
    {
      "file": {
        "name": "otsuBinaryImage",
          "type": "image",
          "description": "Generated Binary Image",
          "options": {
            "mimeTypes": {
            "allowed": [
                            "image/jpeg",
                            "image/png"
                        ],
                        "default": "image/jpeg"
                    },
                    "colorspace": "binary",
                    "visualization": true
                }
            }
        }
    ],
    "method": {
        "imageType": "docker",
        "imageName": "my_name/test_otsubinarization",
        "testData": "https://dl.getdropbox.com/s/l6mobixty0k2o3i/testData.zip",
        "executableType": "bash",
        "executable_path": "/input/script.sh"
    }
}
```

The `general` part contains information about the nature of the method.

The `input` array contains information about the parameters the method requires to run. They need to be in the order in which the method takes the parameter from the command line. Currently DIVAServices supports the following inputs:
 - **file**: refers to a single input file
 - **folder** refers to a folder that contains the data
 - **text** refers to a string provided to the method
 - **number** refers to a number provided to the method
 - **select** refers to a string provided to the method, but the options are fix

The JSON-Schema for the method creation is available [here](https://raw.githubusercontent.com/lunactic/DIVAServices/development/conf/schemas/createAlgorithmSchema.json)

Copy the python script into the `sources` directory:

```bash

cp solutions/createMethod.py sources/createMethod.py

```
In the copy change the following values of the JSON body (in the `payload` variable)


 - `general:name` set the name of the method in the form of "YOURFIRSTNAME Otsu Binarization"
 - `general:developer` provide your name
 - `general:author` provide your name
 - `general:affiliation` provide your affiliation
 - `general:email` provide your email address
 - `method:imageName` provide the name of your Docker Image as on Docker Hub

After performing the changes execute the script with: `python createMethod.py`

## 7. Testing the method on DIVAServices
You can use the script `executeOnDivaservices.py` in the `solutions` folder to test your method on DIVAServices.
The script can be executed in the form of:
``` bash
python executeOnDivaservices.py URL_TO_METHOD FOLDER_TO_STORE_RESULT
```
, e.g. 
``` bash 
python executeOnDivaservices.py http://divaservices.unifr.ch/api/v2/binarization/otsubinarization/1 OtsuBinarization
```

This will execute the official Otsu Binarization on DIVAServices and the resulting image will be stored in the `OtsuBinarization` folder as `otsuBinaryImage.png`.
To test your method replace the URL with: `http://XXX.XXX.XXX.XXX/binarization/firstnameotsubinarization/1`. 

If this results in the image stored in the output folder, Congratulations! You are now able to deploy a method on DIVAServices.
