# hiremefordevops

GOAL: install a basic Wordpress site on AWS EC2.
LINK TO PROJECT: www.hiremefordevops.com

          TABLE OF CONTENTS:
1. Needed tech
2. Steps
3. Expansion and growing for bigger load and audience
4. Scalling the project
5. Listing all VPC and Subnets
6. Things that can be improved
7. Ideas for real-life project creation




          NEEDED TECH:
- LAMP stack
- Wordpress (or a single static HTML will do too?)
- Coffee





          STEPS:
We could go with Vagrant and load up some local VM's, but the AWS cloud ones seem to get our job and setup done quicker for now.
1. LAMP stack installation: Amazon has a great guide to install it from the link in step 2:
2. https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-lamp-amazon-linux-2.html
5. Domain registration and purchase is optional or only for fun.
6. How_Not_to_Waste_time_like_Me: If you want to buy a domain for the website, do it now and register it with their ELASTIC DOMAIN NOW BEFORE CONTINUING from link in step 5: 
7. https://aws.amazon.com/getting-started/hands-on/get-a-domain/ (Otherwise you may face hours of troubleshooting and pointless search as when Wordpress gets installed it statically assigns the IP somewhere in its wp_config DB table of the machine before switching to the Elastic IP and your domain will not work at all). Test if the domain works with the Apache/NginX test pages or the phpinfo.php one. The first simptoms are slow DNS resolving and page even not loading until it reaches the TTL you set for the domain in AWS.
8. Wordpress install and configuration link in step 9:
9. https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/hosting-wordpress.html
10. systemctl enable httpd, mariadb to make sure everything boots up after a reboot.
11. The site should be up and running. Test with your domain.





          Expansion and growing for bigger load and audience
Use NginX instead of Apache. There are some AWS options for advanced scalling, but be aware of the payments they might add up to your monthly bill. #to-check-further. 





          Scaling
We could utilize the AWS Lambda functions to trigger expansions of resources on the machine or other trigger (serverless) needed actions.




          Listing all VPC and Subnets
I went with BASH script route for this one as Lambda function seemed to overcomplicated this.
Extract the data with a oneliner:

          ifconfig | grep inet | grep -v 127.0.0.1 | grep -v inet6 | awk '{print $2 echo"\n" $4}' > network_info.txt

Here we get an output like this one:
          inet 172.31.92.41 
          netmast 255.255.240.0 
          boardcast 172.31.95.255
We just need to get this to fill in a seperate file for the netmask and the IP and netmask details from the "awk" filter are needed so we don't need all of them.

                    ifconfig | grep inet | grep -v 127.0.0.1 | grep -v inet6 | awk '{print $2}' > network_ip.txt
                    
                    ifconfig | grep inet | grep -v 127.0.0.1 | grep -v inet6 | awk '{print $4}' > network_mask.txt

Also, the IP for the VPC website we can simply ping and filter it with this:

          ping -c 1 www.hiremefordevops.com | awk '{gsub( /\(|\)/ , ""); print $3}' | grep -v ping | grep -v transmitted > website_ip.txt
          
Now we need to get this stored into a Database. From MariaDB KB we can see here how to import .txt files data into it:
https://mariadb.com/kb/en/importing-data-into-mariadb/

Now we can continue by creating a utilizing all the oneliners who extracted the needed info in the outputs and try to make it work.
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

But I was unable to handle the data from the bash script to input it into the database like that,and frankly I tried for a long time to convert the data from the file to the INET_ATON input in the DB with no success, so we will simply implement a simple Char(15) use case to fit in all the IP symbols, slower but for this use case will be fine for now. #LazyButWorks

First we enter MariaDB and create a DB. Then we create a table with the columns and configuration for them.
1. DB creation

                    Create database network_info;
3. Table creation
                    CREATE TABLE `networks` (
                    `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
                    `netmask` Char(15) DEFAULT '',
                    `name` varchar(255) DEFAULT '',
                    `ip_vpc` Char(15) DEFAULT '',
                    `ip_web` Char(15) DEFAULT '',
                    PRIMARY KEY (`id`),
                    KEY `netmask` (`netmask`),
                    KEY `ip_vpc` (`ip_vpc`),
                    KEY `ip_web` (`ip_web`)
                    )

3. Loading data into table

                    load data local infile '/home/ec2-user/network_ip.txt' into table networks fields terminated by '|' (ip_vpc);
                    load data local infile '/home/ec2-user/website_ip.txt' into table networks fields terminated by '|' (ip_web);
                    load data local infile '/home/ec2-user/network_mask.txt' into table networks fields terminated by '|' (netmask);


These lines load data from .txt file into DB columns we need. Well, not the best solution but I really have no idea how to make it only in one DB table otherwise without junk/filler data.

Check if all data is imported correctly:

                              Select * from networks;

Now we have a running website with a DB behind it, for now with Apache, later for more traffic we could utilize NginX. Improve the DB handling for the IP INET_ATON conversions and add to cron jobs the oneliners for periodically extracting the network info. Maybe a monitoring with NAgios server to ping if something is missing or not responding?

I really had a lot of fun doing this project, I hope I am not overdoing the text here and that you find the domain name funny for this as I did!

I am looking forward to be working with the ZetaHosting DevOps team and trully eager to learn new things, ones I get a docking to not be working only on my laptop screen as I currently have made this lol.

                              Things that can be improved
SSL ceritficates and HTTPS would be great in these days to make users more comfortable. 


                              Ideas for real-life project creation:
As my wife's little brother recently won a mental arithmentics competition in Dubai which was real fun as much as the expenses for the trip too. The SmartyKids center in Vratsa would like to seperate itself from the brand name which they get a 20% increase in the payment they make for it every year, which for the low standard in Vratsa, kinda kills the business side and also lower the chance of focusing resources into trully motivated teachers to lead some of the online classes.
So some different approaches to make a math calculator which flashesh different colored numbers in the screen, depending on the color you add, substract, multiply, etc. which seems to be great mental exercise for all ages as this was from a test done for the aviation tests for mental toughness.

A database for the users will be needed. A Database for the scores and challenges which are per day, per week, etc.
Some beatuful automation with Python to webscrape the web for recent news of any math competitions around the world, although they might need to be translated, but not a must as most kids are learning outside of Bulgaria and are in UK so they will be fine.

Any ideas on that too would be appreciated or if we get to be working together, we could discuss these kinds of things to create an online free internships to attract people and get a free marketing viewerships.
