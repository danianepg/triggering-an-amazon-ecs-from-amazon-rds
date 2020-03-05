# Triggering an Amazon ECS from Amazon RDS

![](https://miro.medium.com/max/6643/0*aKFQ9twWf__p_8LY)

*Photo by  [chuttersnap](https://unsplash.com/@chuttersnap?utm_source=medium&utm_medium=referral)  on  [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referr*al)*

With a few lines of code, it’s possible to trigger an  **Amazon Elastic Container Service (ECS)** from an  **Amazon Relational Database Service (RDS)**. This article intends to describe how to do it and the struggles I had while trying to have it working.

For the sake of understanding, I will describe an  **imaginary** system to explain why I want to trigger an ECS task. Let’s pretend user data protection just doesn’t exist.

The company IT Paradise offers a cafeteria to its employees where they can order coffee, snacks, and drinks. Since they are riding the “healthy style” wave, they decided to follow the employee’s consumption habits.

IT Paradise implemented a system that tracks the amount of coffee and snacks an employee takes a day. So now, every time someone goes to the Cafeteria and swipe their card, the Cafeteria’s system “A” inserts the consumed items on a database table, which triggers a lambda to run an ECS task.

The ECS task starts the application “B” that will process the information about the employee intakes and feed the employee’s profile on a system “C”. When everything is done, system “B” stops itself.

![](https://miro.medium.com/max/2000/1*etH9CDD22it4ykM1vcBN8Q.jpeg)
*Imaginary system’s flow*

# The database

The database is an Aurora MySQL and it will need the permissions as described on  [AWS documentation.](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Integrating.Lambda.html)

We create a trigger on the table that holds the purchases of the employee. Check  **line 9**  for the piece of code that **calls the lambda.**

```sql
CREATE TRIGGER tasks_trigger
  AFTER INSERT ON employee_purchases
  FOR EACH ROW
BEGIN
	
	SELECT NEW.taskType INTO @taskType;
	
	IF @taskType = 'BUY' THEN
		CALL mysql.lambda_async('arn:aws:lambda:eu-west-1:<MY_ACCOUNT_ID>:function:triggerECSLambda', 
     					 CONCAT('{"taskType" : "', @taskType, '" }')
     		 ); 
	
	END	IF;
  
END
```
*Database trigger to call the lambda*

# The Lambda Saga

Since I’m a Java enthusiast I naively wrote my lambda in Java 11. However, I faced several problems and gave up in a “`NoClassDefFoundError`” (line 14).

Besides that, my jar ended up with 150mb, which is a lot for what I needed, and the initialization time was also discouraging.

I leave the Java 11  **non-working**  code below just for the records.

```java
import com.amazonaws.services.ecs.AmazonECS;
import com.amazonaws.services.ecs.AmazonECSClientBuilder;
import com.amazonaws.services.ecs.model.AwsVpcConfiguration;
import com.amazonaws.services.ecs.model.LaunchType;
import com.amazonaws.services.ecs.model.NetworkConfiguration;
import com.amazonaws.services.ecs.model.RunTaskRequest;
import com.amazonaws.services.ecs.model.RunTaskResult;
import com.amazonaws.services.lambda.runtime.Context;

public class Application {

  public String handleRequest(final Object request, final Context context) {
    
    final AwsVpcConfiguration awsvpcConfiguration = new AwsVpcConfiguration()
        .withSubnets("subnet-<MY_SUBNET>")
        .withSecurityGroups("sg-<MY_SUBGROUP>");

    final NetworkConfiguration networkConfiguration = new NetworkConfiguration()
        .withAwsvpcConfiguration(awsvpcConfiguration);

    final RunTaskRequest runTaskRequest = new RunTaskRequest()
        .withLaunchType(LaunchType.FARGATE)
        .withCluster("<MY_CLUSTER>")
        .withTaskDefinition("<MY_TASK>")
        .withCount(1)
        .withNetworkConfiguration(networkConfiguration);

    final AmazonECS client = AmazonECSClientBuilder.standard().build();
    final RunTaskResult response = client.runTask(runTaskRequest);

    return String.format("Runned: %s.", response);
  }

}
```

When I finally accepted that it was to much trouble to a simple task, I tested a script in  **Node.js and it worked smoothly and painlessly**. :)

It can be checked below. It has plenty of room to improve, but the important thing to note here is that I always  **get the last version**  of the task definition (line 27) and the  _runTask_ command (line 56).

```javascript
var aws = require('aws-sdk');
var ecs = new aws.ECS();

exports.handler = async (event, context) => {
    
	var taskDefinition = null;

	var CLUSTER = process.env.ENV_CLUSTER;
	var SUBNET = process.env.ENV_SUBNET.split(",");
	var SECURITY_GROUP = process.env.ENV_SECURITY_GROUP;
	var LAUNCH_TYPE = "FARGATE";
	var FAMILY_PREFIX = "process-employee-purchases";
	var CONTAINER_NAME = "process-employee-purchases";
	var PROFILE = "";
		
	if(event.hasOwnProperty('taskType') && event.taskType == 'BUY') {
	  PROFILE = "exportRunner";
	} else {
	  PROFILE = "importRunner";
	}
	
	var taskParams = {
		familyPrefix: FAMILY_PREFIX
	};    

	// Get the last version of the task
	const listTaskDefinitionsResult = await ecs.listTaskDefinitions(taskParams).promise();
    
	if(listTaskDefinitionsResult) {
		taskDefinition = listTaskDefinitionsResult.taskDefinitionArns[listTaskDefinitionsResult.taskDefinitionArns.length-1];
		taskDefinition = taskDefinition.split("/")[1];
	}

	var params = {
        cluster: CLUSTER,
        count: 1, 
        launchType: LAUNCH_TYPE,
        networkConfiguration: {
            "awsvpcConfiguration":  {
              "subnets": SUBNET,
              "securityGroups": [SECURITY_GROUP]
            }
        },
        taskDefinition: taskDefinition,
        overrides: {
          containerOverrides: [{
            name: CONTAINER_NAME,
            environment: [{
              name: "SPRING_PROFILES_ACTIVE",
              value: PROFILE
            }]
          }]
        }
	};
    
	const runTaskResult = await ecs.runTask(params).promise();
  
	if (runTaskResult.failures && runTaskResult.failures.length > 0) {
		console.log("Error!");
		console.log(runTaskResult.failures);
	}

	return runTaskResult;
   
};
```

*Lambda  **working** script (Node.js 12.x)*

Lambda permissions are as below.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "logs:CreateLogGroup",
      "Resource": "arn:aws:logs:eu-west-1:<MY_ACCOUNT_ID>:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:eu-west-1:807210873196:log-group:/aws/lambda/triggerECSLambda:*"
      ]
    }
  ]
}
```
*Basic permission that lambda needs to access  **CloudWatch** logs.*
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:RunTask",
        "ecs:ListTaskDefinitions"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "*"
      ],
      "Condition": {
        "StringLike": {
          "iam:PassedToService": "ecs-tasks.amazonaws.com"
        }
      }
    }
  ]
}
```
*Allows the lambda to interact with  **ECS** and run tasks.*

# Conclusions

As far as I’m aware, it’s not possible to trigger Amazon ECS directly from my Amazon RDS Aurora, thus the need for the lambda.

A lambda can be easily called from the Amazon RDS and then, from the lambda, it is possible to run an Amazon ECS task.

After the tests, I got a little bit disappointed with lambdas in Java on AWS, but it’s incredibly easy to work with lambdas in Node.js.

By taking the approach described, it is possible to have a detached architecture with exclusive containers running only during the necessary amount of time and avoid some “pains” such as schedulers or memory share. Also, it saves money! We don’t have the container always up and running and we will pay only for what we use.

----------

_Special thanks to my DevOps Elson Neto that guided me through the AWS permissions maze and my colleague_ [_Diego Hordi_](https://levelup.gitconnected.com/@diego.hordi) _for suggesting some nice touches._

Originally published on [Medium](https://medium.com/@danianepg/triggering-an-amazon-ecs-from-amazon-rds-5425b807fe82).
