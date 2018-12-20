# maps
clone of google mapping functions snapToRoads and distancematrix using Open Street Maps data with Djiktra shortest path algorithm.

## SERVICE DNS NAMES
NONPROD internal and external dns for environment ENV will be
- http://maps.ENV.rws
- http://maps.ENV.gocurb.com

PROD internal and external service URLs will be
- http://maps.prod.rws
- http://maps.gocurb.com

### TESTING
Curl by appending the following example path and query string to the host names above.  JSON should be returned with code 200.
```shell
/v1/snapToRoads?key=&path=40.76965%2C-73.95791%7C40.7696%2C-73.95796%7C40.76967%2C-73.95826%7C40.77039%2C-73.95974%7C40.77105%2C-73.9596%7C40.76906%2C-73.95493"
```
```shell
{"snappedPoints":[{"placeId":"","originalIndex":1,"location":{"latitude":40.76947,"longitude":-73.95787}},{"placeId":"","originalIndex":2,"location":{"latitude":40.76957,"longitude":-73.9581}},{"placeId":"","originalIndex":3,"location":{"latitude":40.77042,"longitude":-73.96011}},{"placeId":"","originalIndex":4,"location":{"latitude":40.77105,"longitude":-73.95965}},{"placeId":"","originalIndex":5,"location":{"latitude":40.77011,"longitude":-73.95741}},{"placeId":"","originalIndex":6,"location":{"latitude":40.7691,"longitude":-73.95505}}]}
```

## DEPLOYING
### Prerequisites
You must have awscli installed, and a AWS IAM identity that is able to launch instances, create EBS volumes and snapshots, create ELBs, and manipulate DNS records. See the individual scripts for more details.
### Overview steps to deploy the map server for an environment
0. set up the environment definition file.
   use deploy/config/environments/release.sh as a template for your env
1. create the read-only database (see provision_database.sh)
2. clone the database volume in each target availability zone (see deploy_osm_db_ebs.sh)
3. set up load balancing ELB (see deploy_elb.sh)
4. deploy the application to a new instance (see deploy_instance.sh)
5. add instance(s) to load balancer (see elb_instance.sh)
6. create a standard human friendly DNS alias for the ELB (see elb_dns.sh)
### EXAMPLE DEPLOYMENT
Assuming a previously provisioned database snapshot, to deploy the OSM application to a new environment called "release", with your personal credentials aliased under "gocurb", to the us-east-1d zone: 
```shell
bash deploy_osm_db_ebs.sh -e release -p gocurb -z us-east-1d -v
bash deploy_instance.sh -p gocurb -e release -v
bash deploy_elb.sh -p gocurb -e release -i -r -v (running it with "-r" will delete your ELB!)
bash elb_instance.sh -p gocurb -e release  -v -a "all" -v 
bash elb_dns.sh  -p gocurb -q ridecharge -e release -i -v
```
### Failure Recovery
some of the steps above may be skipped when recovering from an instance failure.  First determine the root problem by following the packets -- DNS -> ELB -> instance -> application -> database.

Check that the application returns data.  The test below can be applied to each instance to test its health.  Replace the address with the IP of the instance, and the port with the backend port (8080):
```shell
[msv@awscurblxjump01]~% curl "http://maps.prod.rws/v1/snapToRoads?key=&path=40.76965%2C-73.95791%7C40.7696%2C-73.95796%7C40.76967%2C-73.95826%7C40.77039%2C-73.95974%7C40.77105%2C-73.9596%7C40.76906%2C-73.95493"
{"snappedPoints":[{"placeId":"","originalIndex":1,"location":{"latitude":40.76947,"longitude":-73.95787}},{"placeId":"","originalIndex":2,"location":{"latitude":40.76957,"longitude":-73.9581}},{"placeId":"","originalIndex":3,"location":{"latitude":40.77042,"longitude":-73.96011}},{"placeId":"","originalIndex":4,"location":{"latitude":40.77105,"longitude":-73.95965}},{"placeId":"","originalIndex":5,"location":{"latitude":40.77011,"longitude":-73.95741}},{"placeId":"","originalIndex":6,"location":{"latitude":40.7691,"longitude":-73.95505}}]}
```

Check that the DNS record exists and resolves to an ELB. For example, to check the internal "prod" environment, where the endpoint should respond at "maps.prod.rws". Note that DNS for an ELB with no healthy instances attached will not resolve.  
```shell
$ elb_dns.sh -p gocurb -q ridecharge -e prod -i -v -l
dns target is "maps.prod.rws."
load balancer name is "maps-prod-internal"
finding internal zone id for environment prod
[
    {
            "Name": "maps.prod.rws.",
            "Type": "A",
            "AliasTarget": {
                "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "internal-maps-prod-internal-1223913944.us-east-1.elb.amazonaws.com.",
                "EvaluateTargetHealth": false
        }
    }
]
```

Check that the ELB instances are healthy:
```shell
$ elb_instance.sh -p gocurb -e prod -l -i -v
available instances:
instance-id		private-IP	state	tags
i-08e23996924bf73ad	10.16.13.191	running	App:OSM, Version:0.0.1, Environment:prod, Name:AWSCURBOSM

available load balancer name and dns:
maps-prod-internal	internal-maps-prod-internal-1223913944.us-east-1.elb.amazonaws.com

registered instances on maps-prod-internal:
i-08e23996924bf73ad
```

Check available instances:
```shell
$ deploy_instance.sh -p gocurb -e prod -lv
discovering az for subnet-f599d9d8
instance-id    private-IP	state	tags
i-08e23996924bf73ad		10.16.13.191	running	App:OSM, Version:0.0.1, Environment:prod, Name:AWSCURBOSM
```

Note that there is only one available instance.  create a new one:
```shell
$ deploy_instance.sh -p gocurb -e prod -v -w
discovering az for subnet-f599d9d8
discovering OSM database volume id in availability zone us-east-1b
preparing SQS queue
preparing user_data
launching instance
instance id i-0bfe63243b0da8fe4 starting. waiting for running state
started i-0bfe63243b0da8fe4 with private IP 10.16.13.174
attaching osm database volume to instance
2018-05-16T13:18:38.649Z      xvdj	i-0bfe63243b0da8fe4	attaching	vol-0bfd54cd287fa3af5
waiting up to 2 minutes for instance to mount volume and post message
not waiting for instance to sync local storage, complete startup and unmount volume. This will take ~25 minutes for i3.xlarge and 2 hrs for m1.medium
completed.
```

Attach newly minted instance to ELB:
```shell
$ elb_instance.sh -p gocurb -e prod -a i-0bfe63243b0da8fe4 -v -i
Looking for ELB name "maps-prod-internal"
INSTANCES   i-0bfe63243b0da8fe4
INSTANCES   i-08e23996924bf73ad
```

Faulty instances should be removed from ELB:
```shell
$ elb_instance.sh -p gocurb -e prod -r i-08e23996924bf73ad -v -i
Looking for ELB name "maps-prod-internal"
INSTANCES   i-0bfe63243b0da8fe4
```

## API ENDPOINTS 
### snap to roads 

returns a list of coordinates represening the highest likelihood path following roads, given a list of coordinates with measurement errors. 
- endpoint is: http://\<SERVICE DNS\>/v1/snapToRoads
- GET request should be urlencoded in the form path=\<latitude\>,\<longitude\>|\<latitude\>,\<longitude\>.....
where \<latitude\> is decimal value indicating degrees north and \<latitude\> is decimal value indicating degrees east
- return value is a JSON array as described here (placeId is non-functional): https://developers.google.com/maps/documentation/roads/snap

### distance matrix
returns estimated time enroute (ete) and driving distance enroute (ede) between a given coordinate and each member of a list of starting coordinates
- endpoint is: http://\<SERVICE DNS\>/v1/distancematrix
- GET request should be urlecoded string of the form json={"loc":{"lat":"\<dlat\>","lon":"\<dlon\>"},"vehicles":[{"id":"34","lat":"\<slat\>","lon":"\<slon\>"},{"id":"56","lat":"\<slat\>","lon":"\<slon\>"} ... ]}

where dlat/dlon is the destination coordinate and slat/slon is starting coordinate in decimal degrees north/east, and vehicle "id" is freeform string, which should be unique for each item. 
- returned value is unencoded JSON of the form {"vehicles":[{"ete":\<integer\>,"ede":\<integer\>,"id":"34"},{"ete":\<integer\>,"ede":\<integer\>,"id":"56"} ... ]}

where ete is in seconds and ede is in meters, and vehicle "id" correspond to those submitted.

- example request: 
prepared json to urlencode:
```
{"loc" : {"lat":"40.793985","lon":"-73.970231"},"vehicles" : [ {"id" : "34", "lat" : "40.755496", "lon" : "-73.983637" },  {"id" : "56", "lat" : "40.766739", "lon" : "-73.969137"} ] }
```
shell request from curb jump server:
```
[msv@awscurblxjump01]~% curl "http://maps.release.rws/v1/distancematrix?json=%7B%22loc%22%20%3A%20%7B%22lat%22%3A%2240.793985%22%2C%22lon%22%3A%22-73.970231%22%7D%2C%22vehicles%22%20%3A%20%5B%20%7B%22id%22%20%3A%20%2234%22%2C%20%22lat%22%20%3A%20%2240.755496%22%2C%20%22lon%22%20%3A%20%22-73.983637%22%20%7D%2C%20%20%7B%22id%22%20%3A%20%2256%22%2C%20%22lat%22%20%3A%20%2240.766739%22%2C%20%22lon%22%20%3A%20%22-73.969137%22%7D%20%5D%20%7D%0A"
```
response:
```
{"vehicles":[{"ete":352,"ede":4983,"id":"34"},{"ete":350,"ede":4077,"id":"56"}]}
```
