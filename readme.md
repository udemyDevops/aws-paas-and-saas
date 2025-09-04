### Steps to set up the paas and saas arch as per the details in [_arch-and-flow.md_](arch-and-flow.md) file

* Fisrt make sure to select the same region for creating the resources

* Create a security group for backend
    - add an inbound rule after the SG is created (to get the SG Id) to allow the traffic within the SG
    - later an inbound rule to allow traffic from beanstalk SG for ec2 after the creation of beanstalk

* Create a key pair used to login to beanstalk ec2 instance for any troubleshooting

* Create the RDS instance --> DB instance
    > Note: In real time there may be a case of changing the parameter of Database but in RDS no login option (like ssh) to change the configuration file. To change any parameter, we can make use of parameter group. Create a parameter group and select it while creating RDS, this allows to make changes to parameter group settings whenever required which will be reflected in RDS instance.

    - Create a parameter group 
    1. select the engine type as per the DB, for vprfile --> MySql community
    2. parameter group family --> mysql8.0 or higher
    3. type --> DB parameter group

    - create a subnet group (required for RDS), if not RDS will create one for itself

    - create DB instance
    1. select the engine as mysql and version as 8.0.x
    2. credential settings 
        - admin (default username) 
        - password --> select auto generate (which can be retrieved when the instance is getting created by clicking 'view credentials')
    3. connectivity --> select the subnet group --> select the security group created for backend
    4. Additional configuration 
        - give the DB name 'accounts' (used for vprofile) 
        - select the parameter group 
        - uncheck the automated backup (only for this practice)
        - log exports --> leave unchecked (only for this practice)

* Create ElastiCache --> for Memcached
    - create a paramter group --> select family type (memcached1.6 for vprofile)
    - create a subnet group (required for ElastiCache)
    - Go to ElastiCache dashboard --> create Memcached
        >If the deployment option is 'serverless' --> click on create. Otherwise, for 'design your own cache' follow below steps
    1. select 'Standard create'
    2. Engine version --> 1.6.x (for vprofile)
    3. port --> 11211 (default, if it different change it to 11211 for vprofile)
    4. select the parameter group created for Memcached
    5. Node type --> select the smallest one (for vprofile --> cache.t2.micro)
    6. Subnet group settings --> select the subnet group
    7. Availability zone --> no preference (default for vprofile)
    8. Next --> security group --> manage --> select the Sg created for Backend
    9. Maintenance --> no preference

* Create Amazon MQ --> for RabbitMQ
    - Search for Amazon MQ --> Get started
    - Broker engine type --> RabbitMQ --> Next
    - Deployment Mode --> single instance broker (for vprofile) --> Next
    - Broker instance type --> select the smallest one (mq.t3.micro for vprofile)
    - RabbitMQ access --> provide username and password (save it as they have to be updated in the application.properties file)
    - Advanced settings
    1. Broker engine version --> 3.13
    2. Network and security --> private access and select the Backend security group
    - Next and Create Broker

* Initialize the Database schema in RDS intance
    - As the RDS is a private instance, create an ec2 instance in the same vpc and region as of RDS
    - Launch an ec2 instnace of Ubuntu and install 'mysql' client
    1. create a new security group for ec2 and add a rule to allow ssh from your IP
    2. ssh to ec2
    3. Use the below command to insatll mysql client to connect with RDS and git to clone the repo for DB schema

    ```
    sudo -i
    ```

    ```
    apt update && apt install mysql-client git -y
    ```
    4. Add an inbound rule in Backend SG to allow port 3306 (mysql) from ec2 SG
    5. Test the Login to MySQL
    ```
    mysql -h <rds endpoint> -u <username> -p<password>
    ```
    to login to database
    ```
    mysql -h <rds endpoint> -u <username> -p<password> <database name>
    ```
    > Note: To make sure the password is not visible in the terminal, use the below command which will prompt to enter the password as securestring
    ```
    mysql -h <rds endpoint> -u <username> -p
    ```
    once the login has no issues
    ```
    exit
    ```
    6. Clone the source code

    ```
    git clone <project url>
    ```
    > eg: 
    ```
    git clone https://github.com/hkhcoder/vprofile-project.git
    ```
    - change the branch
    ```
    git checkout awsrefactor
    ```

    or use the below repo

    ```
    git clone https://github.com/udemyDevops/aws-paas-and-saas.git
    ```
    7. change the working directory to the project folder
    ```
    cd <project dir>
    ```
    8. deploy the schema [_db_backup.sql_](src/main/resources/db_backup.sql)
    
    ```
    mysql -h <rds endpoint> -u <username> -p<password> <database name> < <path of sql file>
    ```
    > eg: mysql -h <rds endpoint> -u <username> -p<password> accounts < src/main/resources/db_backup.sql

    9. login to database to validate the DB initialization is done.
    ```
    mysql -h <rds endpoint> -u <username> -p<password> accounts
    ```
    ```
    show tables;
    ```
    ```
    exit
    ```

* Delete the ec2 instance

* Create Beanstalk
    - First we need to create IAM roles for Beanstalk
    > Beanstalk creates a role whic we can select but it lack certain permissions for this project, so create an IAM role and select it
    * IAM --> roles
    1. create role --> trusted entity type as 'AWS service'
    2. service or use case --> ec2 --> Next
    3. Permission policies --> search for bean and select the below policies
        - AdministratorAccess-AWSElasticBeanstalk
        - AWSElasticBeanstalkCustomPlatformforEC2Role
        - AWSElasticBeanstalkRoleSNS
        - AWSElasticBeanstalkWebTier
    4. Next --> give a name to the role --> create role
    
    - Search for Amazon Elastic Beanstalk --> create application
    1. environment tier --> web server environment
    2. give application name (eg: vprofile-beanapp)
    3. give environment name (eg: vprofile-beanapp-prd)
    > Note: One Beanstalk environment can have applications for different environments (dev, test, stg, prd..) and also have other services like autoscaling group, ec2 instances, ELBs, s3 bucket..., in one Beanstalk application we can have multiple environments.
    4. give a domain name (eg: vprorearch12n3, unique across aws)
    5. Patform type --> Managed 
        - platform --> Tomcat
        - platform branch --> tomcat10 with corretto21 (or corretto17 --> java17)
        - version --> recommended
    6. Application --> sample application for initial creation and later the artifact can be uploaded
    7. Presets --> custom configuration --> Next
    8. Server access
        - server role --> default role (if not there, click on Create role besdie to the option)
        - ec2 instance profile --> select the role creates earlier
        - select the key pair --> Next
    9. select the vpc
    10. instance settings --> tick the 'Activated' box for public IP address
    11. instance subnets --> select all the subnets
    12. Database --> do not select anything as we already created RDS instance
        > Note: Beanstalk has the option to create RDS
    13. Next --> Instances
        - root volume --> change from container default to 'General Purpose 3(SSD)'
        - ec2 security group --> Beanstalk create a SG by itself, later we can add the rules.
    14. Capacity
        - Autoscaling Group --> environment type --> load balanced --> 2 as min and 4 as max instances
        - instance type --> select the smallest one (t2.micro)
        - scaling triggers --> NetworkOut and other values (go with default)
    15. Load balancer network settings
        - visibility --> public --> make sure the subnets are selected
    16. load balancer type --> ALB and Dedicated
    17. listeners --> leave the default port 80
        > Note: vprofile when hosted on ec2 (tomcat listens on port 8080) but in Beanstalk it is port 80
    18. processes --> edit the 'default' --> Sessions --> Session stickiness -> tick the box 'Enabled'
    19. Leave the other options with default --> Next
    20. Monitoring
        - Health reporting --> System --> Enhanced
    21. Rolling updates and deployments
        - Deployment policy --> rolling (for vprofile)
        - Batch size type --> percentage
        - Deployment batch size --> 50(%)
        > Refer the AWS documentation for deployment policies 
        ```
        https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.rolling-version-deploy.html
        ```
    > Platform Software --> Environment properties --> In some case we may need to set the environment variables for the application like DB name, port number, or any other configuration..... can make use of this
    22. Next --> review the Beanstalk configuration --> Submit to create
    > ec2 instances and autoscaling group will be created with respective SGs part of Beanstalk

* Add an inbound rule in the Backend SG to allow all traffic from SG attached to Beanstalk ec2 instances

* Build and deploy artifact to Beanstalk
    - first get the endpoints and port numbers of backend services
        1. RDS endpoint (also username and password)
        2. Amazon MQ --> Connections --> endpoints (exclude amqps://). It will also has the port number which needs to be updated in the application configuration file (also username and password).
        3. Memecached --> configuration endpoint (also has port number).
    - Clone the git repository 
        * 'https://github.com/hkhcoder/vprofile-project' and change the branch to 'awsrefactor'
        >or
        * 'https://github.com/udemyDevops/aws-paas-and-saas'
    - update the backend services details in [_application.properties_](src/main/resources/application.properties)
    - In VS code --> 'ctrl+shift+P' --> type 'terminal: select default profile' --> git bash
    - make sure the working directory is pointed to the project folder, where the pom.xml is located along with 'src/main/resources' (default path which maven looks for application.properties)
    
    ```
    mvn -version
    ```
    > maven should be 3.9.x and java be 17 or higher. If a different version then unistall and install the required version

    ```
    mvn install
    ```
    > Once the build is completed --> 'target' folder with artifact (.war file) will be created in the project folder 

    - Go to Beanstalk in AWS --> upload and deploy --> upload the .war file and give sme version for it
        * deployment preferences -- keep the default unless required to change
            1. deployment policy --> rolling
            2. batch size type --> percentage or fixed (depending on the number of instances)
        * click on 'deploy'
        * can check the events under the beanstalk
* Add ACM certificate to make it _https_
    - under beanstalk --> go to _configuration_ --> instance traffic and scaling --> edit --> load balancer listeners --> add listener --> port '443', protocl 'https', select the ssl certificate --> save --> apply
    - now add the CNAME record for the beanstalk domain endpoint in r53 or domain provider




    


