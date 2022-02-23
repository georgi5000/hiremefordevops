# hiremefordevops

GOAL: install a basic Wordpress site on AWS EC2.
LINK TO PROJECT: www.hiremefordevops.com

          TABLE OF CONTENTS:
1. Needed tech
2. Steps
3. Expansion and growing for bigger load and audience
4. Scalling the project
5. Listing all VPC and Subnets




          NEEDED TECH:
- LAMP stack
- Wordpress (or a single static HTML will do too?)
- Coffee





          STEPS:
1. LAMP stack installation: Amazon has a great guide to install it from here:
          https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-lamp-amazon-linux-2.html
          
3. Domain registration and purchase is optional or only for fun.
4. How_Not_to_Waste_time_like_Me: If you want to buy a domain for the website, do it now and register it with their ELASTIC DOMAIN NOW BEFORE CONTINUING: https://aws.amazon.com/getting-started/hands-on/get-a-domain/ (Otherwise you may face hours of troubleshooting and pointless search as when Wordpress gets installed it statically assigns the IP somewhere in its wp_config DB table of the machine before switching to the Elastic IP and your domain will not work at all). Test if the domain works with the Apache/NginX test pages or the phpinfo.php one. The first simptoms are slow DNS resolving and page even not loading until it reaches the TTL you set for the domain in AWS.
5. Wordpress install and configuration https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/hosting-wordpress.html
6. systemctl enable httpd, mariadb to make sure everything boots up after a reboot.

The site should be up and running. Test with your domain.





          Expansion and growing for bigger load and audience
Use NginX instead of Apache. There are some AWS options for advanced scalling, but be aware of the payments they might add up to your monthly bill. #to-check-further. 





          Scaling
We could utilize the AWS Lambda functions to trigger expansions of resources on the machine or other trigger (serverless) needed actions.




          Listing all VPC and Subnets
I went with BASH script route for this one as Lambda function seemed to overcomplicated this.
Extract the data with a oneliner:
          ifconfig | grep inet | grep -v 127.0.0.1 | grep -v inte6 | awk '{print $1 echo" " $2 echo"\n" $3 echo" " $4 echo"\n" $5 echo" " $6}' > network_info.txt
Here we get an output like this one:
          inet 172.31.92.41 
          netmast 255.255.240.0 
          boardcast 172.31.95.255
We just need to get this to fill in a seperate file for the netmask and the IP and netmask details from the "awk" filter are needed so we don't need all of them.

Now we need to get this stored into a Database. From MariaDB KB we can see here how to import .txt files data into it:
https://mariadb.com/kb/en/importing-data-into-mariadb/

Now we can continue by creating a network_info.txt file from the redirect mentioned above with the bash script output and try to make it work.
Say you have an IP address, 192.168.0.10 and want to store that in a database table. You could of course store it in a CHAR(15) and that is in fact what many people do. But you probably want to search on this field and therefore want it indexed also. So can we do better than using a 15 byte character field? We sure can.

MySQL has two built-in functions: INET_ATON() and INET_NTOA(). They are actually based on the equivalent inet_aton() and inet_ntoa() which are C library functions present on pretty much every TCP/IP capable system. Why? These two functions are used allover the place in any TCP/IP stack implementation or even application.
The INET_ATON() function converts Internet addresses from the numbers-and-dots notation into a 32-bit unsigned integer, and INET_NTOA() does the opposite. Isn’t that handy!

Let’s put it to the test:

mysql> SELECT INET_ATON('192.168.0.10') AS ipn;
+------------+
| ipn        |
+------------+
| 3232235530 |
+------------+

mysql> SELECT INET_NTOA(3232235530) AS ipa;
+--------------+
| ipa          |
+--------------+
| 192.168.0.10 |
+--------------+

But as it requires extra attention as to how we handle the data from the bash script to input it into the database,and frankly it I tried for a long time to convert the data from the file to the INET_ATON input in the DB with no success, we will simply implement a simple Char(15) use case to fit in all the IP symbols, slower but for this use case will be fine for now. #LazyButWorks

Load data from .txt file into DB.
load data local infile '/home/ec2-user/network_info.txt' into table networks fields terminated by '|' (name, ip) set name=@name+' ';

