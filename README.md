# project4 group4 bash script
#!/bin/bash


# Group4: Write Bash script that creates:
# 1) 3 Instances in Ohio region with security group allowing ports 22, 80, 443 for inbound connections
# 2) Apache server should be running. 
# 3) "Hello $Instance_ID_of_the_VM" message should be printed when accessing VMs on a browser.
# 4) Create a Load Balancer that balances load between above Instances.
# 5) Optional: If you have Route53 domain, create a Route53 record to the load balancer named "group4.yourdomain.com"

#variables
region="us-east-2"
ami_id="ami-01107263728f3bef4"
instance_type="t2.micro"
key_name="project4.key-pair"
i_name="project4"
sg_id="sg-0176c50eb02f48acf"
subnet_id="subnet-0ddc7b197d7815c66"


# Create EC2 instances

for ((i=1; i<=3; i++))
do
    instance_id=$(aws ec2 run-instances \
                     --region $region \
                     --image-id $ami_id \
                     --instance-type $instance_type \
                     --key-name $key_name \
                     --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$i_name}]" \
                     --security-group-ids $sg_id \
                     --subnet-id $subnet_id \
                     --query 'Instances[0].InstanceId' \
                     --output text)


 # Check if instance creation was successful
 if [ $? -eq 0 ]; then
  echo "EC2 instance $i created successfully with ID: $instance_id"
 else
  echo "Failed to create EC2 instance $i"
 fi
done


# Authorize inbound rules
aws ec2 authorize-security-group-ingress \
  --region $region \
  --group-id $sg_id \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --region $region \
  --group-id $sg_id \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --region $region \
  --group-id $sg_id \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0


# Wait for instance to be running
echo "Waiting for instance $instance_id to be running..."
aws ec2 wait instance-running --region $region --instance-ids $instance_id

# Getting instance's public IP address
public_ip=$(aws ec2 describe-instances \
    --region $region \
    --instance-ids $instance_id \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text
)


# SSH into the instance and install Apache
echo "Installing Apache on instance $instance_id..."
ssh -i "project4.key-pair" ec2-user@$public_ip \
            "sudo yum update && \
            sudo yum install httpd -y && \
            sudo systemctl start httpd && \
            sudo systemctl status httpd"



#browser output
# ssh -i "project4.key-pair" ec2-user@$public_ip \
# echo "<h1>Hello $public_ip from $(hostname -f)</h1>" > /var/www/html/index.html
            

# ssh -i "project4.key-pair" ec2-user@$public_ip 
# sudo touch /var/www/html/index.html 
# echo "<h1>Hello $public_ip from $(hostname -f)</h1>" > /var/www/html/index.html

    # echo "<!DOCTYPE html>"> /var/www/html/index.html
    # echo "<html lang="en">">> /var/www/html/index.html 
    # echo "<h1>Hello "$instance_id"</h1>" >> /var/www/html/index.html 
    # echo "</body>" >> /var/www/html/index.html 
    # echo "</html>" >> /var/www/html/index.html

#create a LB 
echo "Creating load balancer..."
load_balancer_name="my-load-balancer"
load_balancer_arn=$(aws elbv2 create-load-balancer \
                  --name "$load_balancer_name" \
                  --subnets "subnet-07378ba9014ba8ba6","subnet-013f9dc62473d738e" \
                  --security-groups "$sg_id" \
                  --region "us-east-1" \
                  --output text \
                  --query 'LoadBalancers[0].LoadBalancerArn')
