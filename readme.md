1. List the resources currently managed by Terraform
>terraform state list

2. 
   There is a VPC, two subnets, and two instances. The subnets are private, i.e. they don't have a route to the internet. The instances are running the Apache web server and a default web page is being served.

3. Remove the current instance configuration in the main.tf file:
>sed -i '/.*aws_instance.*/,$d' main.tf

This command deletes all the lines from the file starting from the line matching aws_instance. The instance configuration is the last block in the file so all the other configuration is preserved.

4. 
cat >> main.tf <<'EOF'
resource "aws_instance" "web" {
  count         = "${var.instance_count}"
  # lookup returns a map value for a given key
  ami           = "${lookup(var.ami_ids, "us-west-2")}"
  instance_type = "t2.micro"
  # Use the subnet ids as an array and evenly distribute instances
  subnet_id     = "${element(aws_subnet.web_subnet.*.id, count.index % length(aws_subnet.web_subnet.*.id))}"
  
  # Use instance user_data to serve the custom website
  user_data     = "${file("user_data.sh")}"
  
  tags {
    Name = "Web Server ${count.index + 1}"
  }
}

EOF

5. Create the user_data.sh script that will create and serve the custom website:
   
cat >> user_data.sh <<'EOF'
#!/bin/bash
cat > /var/www/html/index.php <<'END'
<?php
$instance_id = file_get_contents("http://instance-data/latest/meta-data/instance-id");
echo "You've reached instance ", $instance_id, "\n";
?>
END
EOF

6. 
You now have two instances serving your custom website. The instances are running in private subnets in different availability zones and are assigned to the default security group for the VPC that allows all traffic. There are a few resources that need to be created to allow for an Elastic Load Balancer (ELB) to securely distribute traffic between the instances:

Public subnets for each availability zone must be created so the load balancer can be accessed from the internet
This requires additional resources such as an internet gateway to connect to the internet, and route tables that route to the internet
A security group to allow traffic from the internet to the public subnets that will house the ELB on port 80 (HTTP)
A security group to allow traffic from the ELB in the public subnets to the instances in the private subnets on port 80 (HTTP)

7. Create the security groups that will secure traffic into the public and private subnets in a configuration file called security.tf.
For simplicity, the egress rules for both security groups allow outbound traffic to anywhere.

8. Remove the current instance configuration in the main.tf file so you can modify the configuration to attach the web server security group to them:
>sed -i '/.*aws_instance.*/,$d' main.tf

9. 
cat >> main.tf <<'EOF'
resource "aws_instance" "web" {
  count                  = "${var.instance_count}"
  # lookup returns a map value for a given key
  ami                    = "${lookup(var.ami_ids, "us-west-2")}"
  instance_type          = "t2.micro"
  # Use the subnet ids as an array and evenly distribute instances
  subnet_id              = "${element(aws_subnet.web_subnet.*.id, count.index % length(aws_subnet.web_subnet.*.id))}"
  
  # Use instance user_data to serve the custom website
  user_data              = "${file("user_data.sh")}"
  
  # Attach the web server security group
  vpc_security_group_ids = ["${aws_security_group.web_sg.id}"]
  tags { 
    Name = "Web Server ${count.index + 1}" 
  }
}
EOF

10. Store the site_address output in a shell variable:
>site_address=$(terraform output site_address)

11. Send an HTTP request to the ELB every two seconds using the watch and curl command:
>watch curl -s $site_address

12. Delete the ELB security group using the destroy command and the target option, and enter yes to accept the plan:

>terraform destroy -target=aws_security_group.elb_s

13.
>terraform destroy

