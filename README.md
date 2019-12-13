# aws-ec2-docker-swarm-travis-ci-cd-instructions
**DESCRIPTION**
------
    This is a set of steps to setup ci/cd with travis, github, docker-hub and aws.
    Example repo: https://github.com/ArtemAlagizov/instability
------
**PREREQUISITES**
------
    accounts for:
      * github
      * docker-hub
      * travis 
      * aws (free tier is enough)
------
**STEPS**
------
  repo level
------
* **add .travis.yml to your repo** (see example below)
------
  aws level
------
* **start new ec2 instance** (i recommend using this as the image: Amazon Linux 2 AMI 2.0.20190823.1 x86_64 HVM gp2)
    * when launching the instance, uncheck “delete on termination” at **4. Add Storage** to preserve data in case instance restarts
    * the rest leave default
* **attach elastic ip to the ec2 instance** (for the instance to have constant ip address, even after restarts)
    * go to EC2 > NETWORK & SECURITY > Elastic IP addresses
      * choose Allocate Elastic IP address
         * create the elastic ip
      * select your new elastic ip in EC2 > Elastic IP addresses
         * choose Actions => Associate Elastic IP address
         * associate your new elastic ip address with your running ec2 instance
* **allow indound traffic to the ec2 instance** (in case vpc that contains ec2 was created with default values, otherwise we'll need to change inbound traffic rules for vpc and subnet containing the ec2)
    * go to EC2 > NETWORK & SECURITY > Security Groups
      * choose Security Group associated with the ec2
         * in the inbound tab add two rules:
           * All TCP - TCP - 0 - 65535 - 0.0.0.0/0
           * All TCP - TCP - 0 - 65535 - ::/0
* **log into the ec2** (using EC2 Instance Connect: browser-based SSH connection => right click on ec2 instance => connect)
    * **install git**
    ```
    sudo yum install git -y
    ```
    * **install docker**
    ```
    sudo yum update -y
    sudo amazon-linux-extras install docker
    sudo service docker start
    sudo usermod -a -G docker ec2-user
    ```
    * **install ssm agent** (for travis to be able to execute commands and have access to ec2 instance)
    ```
    sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
    ```
    * **start ssm agent** 
    ```
    sudo systemctl enable amazon-ssm-agent
    sudo systemctl start amazon-ssm-agent
    ```
    * **check that the ssm agent is running** 
    ```
    sudo systemctl status amazon-ssm-agent
    ```
* **log out and log in again** (to be able to execute docker commands without sudo)
    * **clone your repo**: 
    ```
    sudo git clone https://github.com/billiamdan/warmeup.git
    ```
    * **pull your docker image**
    ```
    docker pull billiamdan/warmeup:latest # check exact image name in your docker hub
    ```
    * **start docker swarm cluster**
    ```
    docker swarm init --advertise-addr <private ip address of the ec2> # can be found in the Description section of the ec2
    # example: docker swarm init --advertise-addr 172.31.5.67
    ```
    * **go to your checked out repo** (to be in the same folder docker-compose.depl.yml file is)
    ```
    cd warmeup
    ```
    * **deploy your stack of services using docker-compose.depl.yml**
    ```
    docker stack deploy shadowMarathonStackk -c docker-compose.depl.yml
    ```
    * **check that the stack is running alright**
    ```
    watch docker stack ps shadowMarathonStackk 
    ```
* **create User in AWS IAM to be able to execute commands from travis**
    * **create a new policy**
        * go to IAM => policies
        * create new policy (see policy JSON version below)
            * replace **us-east-2:650732200008** with region your ec2 instance is located in and your account id (can be seen in the Owner section of ec2 Description)
    * **create group** 
        * go to IAM => groups
        * create new group
            * when creating a group attach the policy created above to the group
    * **create a user**
        * go to IAM => users
        * add new user
            * check checkbox Programmatic access
            * in permissions step add the user to the group created above
            * skip tags for now
            * at the next step you’ll see Access key ID and Secret access key
                * download them (as csv) as those are needed for travis to access your ec2
------
  travis level
------
* **set up travis env variables**
    * in travis => build => options => settings have env variables set:
    ```
    AWS_ACCESS_KEY => the Access key ID from above 
    AWS_SECRET_KEY => Secret access key from above
    DEPLOYMENT_REGION => region in which the ec2 is deployed, example: us-east-2
    DOCKER_IMAGE_TAG => tag for the docker image to be built, example: latest
    DOCKER_PASSWORD => password for your docker-hub account, example: ••••••••••••••••
    DOCKER_REPOSITORY => name of the docker-hub repo, example: instability_client-app
    DOCKER_SWARM_SERVICE_NAME => docker swarm cluster stack name, example: unstableStackk_client-app
    DOCKER_USERNAME => docker-hub repo image name, example: alagiz
    IMAGE_NAME => docker-hub repo image name, example: instability_client-app
    INSTANCE_ID => get it in the Description section of your ec2, example:  
    ```
------
* **try it out**
    * commit code that would change something and see if it is deployed by mista travis
------

**TRAVIS.YML**
```
language: node_js
node_js:
    - "stable"
cache:
    directories:
        - node_modules
services:
    - docker
script:
    - cd client-app
    - npm i
    - npm test
    - npm run build
    - if [ "$TRAVIS_BRANCH" == "master" ]; then
        docker login -u="$DOCKER_USERNAME" -p="$DOCKER_PASSWORD";
        docker build . -t $IMAGE_NAME;
        docker tag $IMAGE_NAME $DOCKER_USERNAME/$DOCKER_REPOSITORY;
        docker push $DOCKER_USERNAME/$DOCKER_REPOSITORY;
        DOCKER_PULL_COMMAND="docker pull $DOCKER_USERNAME/$DOCKER_REPOSITORY:$DOCKER_IMAGE_TAG"
        DOCKER_SERVICE_UPDATE_COMMAND="docker service update --force $DOCKER_SWARM_SERVICE_NAME --image $DOCKER_USERNAME/$DOCKER_REPOSITORY:$DOCKER_IMAGE_TAG";
        DOCKER_CLEANUP_COMMAND="docker system prune -af && docker volume prune -af";
        RESULTING_COMMAND="$DOCKER_PULL_COMMAND && $DOCKER_SERVICE_UPDATE_COMMAND && $DOCKER_CLEANUP_COMMAND";
        pip install --upgrade --user awscli;
        aws configure set aws_access_key_id $AWS_ACCESS_KEY;
        aws configure set aws_secret_access_key $AWS_SECRET_KEY;
        aws configure set default.region $DEPLOYMENT_REGION;
        aws configure set metadata_service_timeout 1200;
        aws configure set metadata_service_num_attempts 3;
        aws configure list;
        aws ssm describe-instance-information --output text;
        aws ssm send-command --document-name "AWS-RunShellScript" --instance-ids $INSTANCE_ID --parameters '{"commands":['\""$RESULTING_COMMAND"\"'],"executionTimeout":["3600"]}' --timeout-seconds 600 --region $DEPLOYMENT_REGION;
        fi
#TODO: concatenating strings as in --parameters above introduces vulnerability of possible command injection, it needs to change
```
**POLICY:**
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ssm:SendCommand",
                "ssm:DescribeInstanceInformation"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:ec2:us-east-2:650732200008:instance/*",
                "arn:aws:ssm:us-east-2:650732200008:*",
                "arn:aws:ssm:us-east-2::document/AWS-RunShellScript"
            ]
        }
    ]
}
```
