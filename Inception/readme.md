
In this post, we're going to walk through the core concepts of AWS Virtual Private Clouds (VPCs) in the context of an analogy. 
My main Objectives are:
- **Explore each core component a part of the analogy**
- **Relate each component to the overall setup**
- **Create a Cloudformation to create the setup in AWS**

When we start using AWS we probably don't want all of our servers, services, etc just thrown into a big melting pot. In this type of ecosystem:
- **everything shares the same network**
- **everything can step on everything else's toes**
- **resource management becomes an army of naming conventions**

Now, this can work if you're just using an S3 bucket, a random EC2 instance or experimenting. But when we go to build a serious cloud infrastructure for an enterprise setup, this Hill Billy ecosystem isn't going to cut it.
An Enterprise set up among other things asks for:
- **Control over the organization of resources.**
- **Control of security.**
- **Control of traffic between our services.**
- **Control to keep differing architectures completely separate from each other.**

These controls can be related to the analogy of creating a new city. If you ever played games like SimCity you will know that we can not throw things around randomly. The City should be organized. Airports, schools, Houses, Roads need to be created such that things can be controlled and organized. In this post, we will draw from that analogy and build a city like VPC. Call it Inception City.
Before we dive into the specifics, the overall city will look as such.

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/rkj1b8uz7hj9kp5eurhm.png)

## The VPC: Inception City
In the analogy of the city, we will draw a piece of land to make the city. The land must be vast enough to accommodate various zonal divide. East, West, and Central. Now to validate the city to the rest of the world, we need an address / Zipcode. This is the CIDR in the computer world.
In the resource section of our cloudformation(City Plan) , we define the VPC as such
```
Resource:
VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock:  !Ref VpcCIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
      - Key: Name
        Value: Inception-VPC
      - Key: "Year"
        Value: "2020"
```
The !Ref is getting that as an input during creation from the parameters.
```
Parameters:
VpcCIDR:
    Type: String
    Default: 192.168.0.0/16
    Description: The CIDR range for the VPC. This should be a valid    private (RFC 1918) CIDR range.
```
## Availability Zones: City zones
Like every good city plan inception will have different isolated zones. Typically the Suburbs, Downtown, and the Industrial zones. Each represents a physically different location in our analogy and very much so in the cloud architecture. This arrangement helps us to isolate any physical failure that might affect our application. The Cloud region you choose comes with the zones and we don't need to create it. But we have to actively use it to build our city.

## Subnets: Neighbourhoods
Inception has a vast resource of land/postal codes (CIDR addresses), but we don't want to build things randomly. Cities plan this out. Build hospitals closer to houses. Airports are closer to industries. Stadiums and museums in downtown. Again we don't have all those buildings(applications) open to anyone in the world. Only a City member can enjoy the library. Only a city member uses a municipal office. But a Museum is open to everyone. So it is an airport. To plan these out, we divide the entire city into divisions called Subnets(Neighbourhoods). Some are public(open to outside world) and some are private(the outside world can not directly go in). We define them as such:

```
Resources:
PublicSubnet01:
    Type: AWS::EC2::Subnet
    Metadata:
      Comment: Public Subnet 01
    Properties:
      AvailabilityZone:
        Fn::Select:
        - '0'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: PublicSubnet01CIDR
      VpcId:
        Ref: VPC
      Tags:
      - Key: Name
        Value: Inception-Public-Subnet-01
PrivateSubnet01:
    Type: AWS::EC2::Subnet
    Metadata:
      Comment: Private Subnet 01
    Properties:
      AvailabilityZone:
        Fn::Select:
        - '0'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: PrivateSubnet01CIDR
      VpcId:
        Ref: VPC
      Tags:
      - Key: Name
        Value: Inception-Private-Subnet-01
```
This snapshot creates a public subnet and a private subnet. CIDR coming from the parameter section as input. Now we have to make sure that the CIDR(postal code) of each subnet is a subset of the city(VPC) postal code(CIDR). In our template it will be as such:
```
Parameters:
PublicSubnet01CIDR:
    Type: String
    Default: 192.168.0.0/19
    Description: CidrBlock for subnet 01 within the VPC.
PrivateSubnet01CIDR:
    Type: String
    Default: 192.168.128.0/19
    Description: CidrBlock for subnet 01 within the VPC.
```
## Route Tables: Roads
We've defined our separate postal codes within our city. Everything is split into geographical regions. Now we need to give "traffic" a way to move in and out of these different areas. What do we do? We build roads. So a postal code has a series of roads that allow traffic to move in and out of it.
In VPCs, even though we have these different subnets, we need to allow traffic to flow through them. We do this with Route Tables. **A Route Table is just a list of CIDR blocks (IP ranges) that our traffic can leave and come from.** By default, newly created Route Tables will have the CIDR of our VPC defined. This means that traffic from anywhere within our VPC is allowed.
In addition to a list of IP ranges that our Route Table connect traffic between, it also has Subnet Associations. Simply put, these are "which subnets use this route table." For our city analogy, it'd be "which postal codes are connected to these roads." A Route Table can have many subnets, but a subnet can belong to only one Route Table.
To sum up- **A subnet is associated with a Route Table and the Route Table dictates what traffic can enter and leave the subnet.**

```
Resource:
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
      - Key: Name
        Value: Inception-Public-RouteTable
      - Key: Network
        Value: Public
      - Key: "Year"
        Value: "2020"
Route:
    DependsOn: VPCGatewayAttachment
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
      - Key: Name
        Value: Inception-Private-RouteTable
      - Key: Network
        Value: Private
      - Key: "Year"
        Value: "2020"
PrivateRoute1:            
      Type: AWS::EC2::Route
      Properties:
        RouteTableId: !Ref PrivateRouteTable
        DestinationCidrBlock: 0.0.0.0/0
        # Route traffic through the NAT Gateway:
        NatGatewayId: !Ref NATGateway01
```

## Internet Gateway: The Highway connection to Inception city
Since the default Route Table starts with all subnets only allowed to route to traffic within the VPC, it's known as a private subnet. How do we make it public? Well first, what's public even mean? It just means it can connect to the internet. How do we do that? We just tell our Route Table it's allowed to do so by attaching an Internet Gateway.

**An Internet Gateway is a portal to the internet. In terms of an analogy, think of it like the highway.**

Imagine that our Government and Private Warehousing postal codes both share a series of roads that only navigate within our city. This makes them private. Now, imagine that one of our commercial codes uses a series of roads that also connect to both the rest of the city AND the highway. That makes it public.

Similarly, **a subnet that's associated with a Route Table that's connected to an internet gateway is public. A subnet with a Route Table that's not connected to an internet gateway is private.**

```
Resources:
InternetGateway:
    Type: "AWS::EC2::InternetGateway"
    Properties:
      Tags:
      - Key: Name
        Value: Inception-IGW
      - Key: "Year"
        Value: "2020"
VPCGatewayAttachment:
    Type: "AWS::EC2::VPCGatewayAttachment"
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC
```

## NAT Gateway: Private Society Gate
In our city, traffic in the Government and Private Warehouse postal codes can't get to the highway. Traffic can only move about within the city. What if it needs to leave? Instead of building more highway on-ramps, we'd probably just tell the traffic to (a) navigate to the government postal code and (b) get on the highway from there.
That's essentially what a NAT Gateway does. **When our Subnets connected to the Private Route Table need access to the internet, we set up a NAT Gateway in the public Subnet.** We then add a rule to our Private Route Table saying that all traffic looking to go to the internet should point to the NAT Gateway.

```
Resource:
ElasticIPAddress01:
    Type: AWS::EC2::EIP
    Properties:
      Domain: VPC
NATGateway01:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt ElasticIPAddress01.AllocationId
      SubnetId: !Ref PublicSubnet01
      Tags:
      - Key: Name
        Value: Inception-NATGateway01
```
## Network ACLs: City Border and Immigration Office
It turns out we've built our city in a very dark time where not everyone can be allowed to just come and go as they please. We need security gates and perimeters outside of each of our postal code areas to ensure only the right traffic is entering and leaving. That means we'll check traffic as it comes in to see if it's permitted AND we'll check as it leaves to make sure it can exit.
In our VPC, these are the **Network ACLs. They dictate what traffic is allowed to enter and leave the subnets they're associated with.**
NACLs are very stateful. They control what comes in and goes out explicitly. We will see a very similar resource going forward called "Security Groups". Security groups are more stateless. Whatever is allowed to go in are also allowed to go out implicitly. Hence they are stateless.

## Servers and Services: The Buildings
Our city has all sorts of surrounding infrastructure and logic, but we're missing buildings! When we create these buildings, given that we now have logical, gated and accessible areas, we can group them better. When we create a building it also will receive its own address.
This is the simplest comparison of all. **Servers and Services launched into our VPC are the buildings of our city.** They receive a "private IP address" when created. We can also set up our subnets to assign "public IP addresses" as well. These are required if we'd like our server/service launched to be able to communicate with the internet via Internet Gateway.

## Security Groups: Building Security
It turns out that our city isn't just in a dark time, it's practically organized chaos. Because of this, we have security guards outside of every building. These guards concern themselves with what traffic is allowed to enter and leave the building. They're not concerned with what traffic is DENIED to enter or leave.
In our VPC, these are our **Security Groups. They protect our servers/services at the resource level instead of at a subnet level.**

**Unlike Network ACLs, Security Groups only care about whether or not traffic is allowed to enter or leave.** I'm reiterating this because of how subtle and yet how different this is.
With ACLs, we can explicitly deny traffic coming from a specific IP CIDR block. With Security Groups we cannot. Instead, we can only allow traffic from specific IP CIDR blocks.
Therefore if we wanted to block traffic from some known idiot IP, with Network ACLs, we could just slap a deny to inbound traffic from that IP. With Security Groups, we'd have to go through and allow everything EXCEPT that IP.
Additionally, any responses to outbound traffic, such as a request that service from within our VPC initiates, are allowed back in. For example, an instance (server) of ours makes an API call, the response from that API call is allowed back in EVEN if we do not allow traffic from that IP range.
Now, this analogy needs one more addition to make it really complete. **Our Security Groups are like a group of security guard employees that work for a security company.
Instead of picking individual security guards, our buildings pick a security company to guard them. The benefit is that buildings that belong to the same security company share the same set of rules.**
This means we can define a set of security rules on one Security Group, and have it used on multiple servers and services. You can also tell Security Groups to allow traffic from other Security Groups, which in our analogy would be like saying, "any buildings that belong to Security Corp BRAVO can enter buildings that Belong to Security Corp ALFA".
Finally, unlike ACLs and Route Tables, we can attach multiple security groups to servers/services. The resulting security is the sum of the security group rules, i.e. if one allows HTTP traffic and the other allows SSH traffic, the result is allowed traffic from both.

## The Analogy all together:
Please fork this analogy from my GitHub and run it on your AWS account to spin up your own city.
https://github.com/anupam-ncsu/AWS-CloudResources/tree/master/Inception

NOTE: the charges associated with the spinup is due to the Elastic IP attached to the NAT gateway, but it is in cents. However, be responsible and delete the city once the job is done.