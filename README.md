# aws-ec2-docker-swarm-instructions

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
    * **install ssm agent** (for travis to be able to execute commands)
    ```
    sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
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
* **set up travis env variables**
    * in travis => warmeup build => options => settings you should have some env variables set. these are the ones we'll need to change (remove them and add again):
      * INSTANCE_ID => get it in the Description section of your ec2
      * AWS_ACCESS_KEY => the Access key ID from above
      * AWS_SECRET_KEY => Secret access key from above
* **try it out**
    * commit code that would change something and see if it is deployed by mista travis


```
POLICY:
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
