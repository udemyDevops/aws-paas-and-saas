### Steps to set up the paas and saas arch as per the details in [_arch-and-flow.md_](arch-and-flow.md) file

* Fisrt make sure to select the same region for creating the resources

* Create a security group for backend
    - add an inbound rule after the SG is created (to get the SG Id) to allow the traffic within the SG
    - later an inbound rule to allow traffic from beanstalk SG for ec2 after the creation of beanstalk

* Create a key pair used to login to beanstalk ec2 instance for any troubleshooting

* Create the RDS instance --> DB instance
    > Note: In real time there may be a case of changing the parameter of Database but in RDS no login option (like ssh) to change the configuration file. To change any parameter, we can make use of parameter group. We can create a parameter group and select it while creating RDS. This allows to make changes to parameter group settings whenever required which will be reflected in RDS instance.

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