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

    or

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

