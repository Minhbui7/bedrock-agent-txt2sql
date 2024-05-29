# Setup Amazon Bedrock Agent for Text2SQL Using Amazon Athena with Streamlit

## Introduction
We will setup an Amazon Bedrock agent with an action group that will be able to translate natural language to SQL queries. In this project, we will be querying an Amazon Athena database, but the concept can be applied to most SQL databases.

## Prerequisites
- An active AWS Account.
- Familiarity with AWS services like Amazon Bedrock, Amazon S3, AWS Lambda, Amazon Athena, and Amazon Cloud9.
- Access will need to be granted to the **Anthropic: Claude 3 Haiku** model from the Amazon Bedrock console.


## Diagram

![Diagram](streamlit_app/images/diagram.png)

## Configuration and Setup

### Step 1: Grant Model Access

- We will need to grant access to the models that will be needed for our Bedrock agent. Navigate to the Amazon Bedrock console, then on the left of the screen, scroll down and select **Model access**. On the right, select the orange **Manage model access** button.

![Model access](streamlit_app/images/model_access.png)

- Select the checkbox for the base model column **Anthropic: Claude 3 Haiku**. This will provide you access to the required models. After, scroll down to the bottom right and select **Request model access**.


- After, verify that the Access status of the Models are green with **Access granted**.

![Access granted](streamlit_app/images/access_granted.png)



### Step 2: Creating S3 Buckets
- Make sure that you are in the **us-west-2** region. If another region is required, you will need to update the region in the `InvokeAgent.py` file on line 24 of the code. 
- **Domain Data Bucket**: Create an S3 bucket to store the domain data. For example, call the S3 bucket `athena-datasource-{alias}`. We will use the default settings. 
(Make sure to update **{alias}** with the appropriate value throughout the README instructions.)


![Bucket create 1](streamlit_app/images/bucket_setup.gif)



- Next, we will download .csv files that contain mock data for customers and procedures. Open up a terminal or command prompt, and run the following `curl` commands to download and save these files to the **Documents** folder:

For **Mac**
```linux
curl https://raw.githubusercontent.com/build-on-aws/bedrock-agent-txt2sql/main/S3data/mock-data-customers.csv --output ~/Documents/mock-data-customers.csv

curl https://raw.githubusercontent.com/build-on-aws/bedrock-agent-txt2sql/main/S3data/mock-data-procedures.csv --output ~/Documents/mock-data-procedures.csv
```

For **Windows**

```windows
curl https://raw.githubusercontent.com/build-on-aws/bedrock-agent-txt2sql/main/S3data/mock-data-customers.csv --output %USERPROFILE%\Documents\mock-data-customers.csv

curl https://raw.githubusercontent.com/build-on-aws/bedrock-agent-txt2sql/main/S3data/mock-data-procedures.csv --output %USERPROFILE%\Documents\mock-data-procedures.csv
```

- These files are the datasource for Amazon Athena. Upload these files to S3 bucket `athena-datasource-{alias}`. Once the documents are uploaded, please review them.

![bucket domain data](streamlit_app/images/bucket_domain_data.png)


- **Amazon Athena Bucket**: Create another S3 bucket for the Athena service. Call it `athena-destination-store-{alias}`. You will need to use this S3 bucket when configuring Amazon Athena in the next step. 


### Step 3: Setup  Amazon Athena

- Search for the Amazon Athena service, then navigate to the Athena management console. Validate that the **Query your data with Trino SQL** radio button is selected, then press **Launch query editor**.

![Athena query button](streamlit_app/images/athena_query_edit_btn.png)

- Before you run your first query in Athena, you need to set up a query result location with Amazon S3. Select the **Settings** tab, then the **Manage** button in the **Query result location and ecryption** section. 

![Athena manage button](streamlit_app/images/athena_manage_btn.png)

- Add the S3 prefix below for the query results location, then select the Save button:

```text
s3://athena-destination-store-{alias}
```

![choose athena bucket.png](streamlit_app/images/choose_bucket.png)


- Next, we will create an Athena database. Select the **Editor** tab, then copy/paste the following query in the empty query screen. After, select Run:

```sql
CREATE DATABASE IF NOT EXISTS athena_db;
```

![Create DB query](streamlit_app/images/create_athena_db.png)


- You should see query successful at the bottom. On the left side under **Database**, change the default database to `athena_db`, if not by default.

- We'll need to create the `customers` table. Run the following query in Athena. `(Remember to update the {alias} field)`:

```sql
CREATE EXTERNAL TABLE athena_db.customers (
  `Cust_Id` integer,
  `Customer` string,
  `Balance` integer,
  `Past_Due` integer,
  `Vip` string
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
LINES TERMINATED BY '\n'
STORED AS TEXTFILE
LOCATION 's3://athena-datasource-{alias}/';
```


- Open another query tab and create the `procedures` table by running this query. `(Remember to update the {alias} field)`:

```sql
CREATE EXTERNAL TABLE athena_db.procedures (
  `Procedure_Id` string,
  `Procedure` string,
  `Category` string,
  `Price` integer,
  `Duration` integer,
  `Insurance` string,
  `Customer_Id` integer
)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
LINES TERMINATED BY '\n'
STORED AS TEXTFILE
LOCATION 's3://athena-datasource-{alias}/';
```


- Your tables for Athena within editor should look similar to the following:

![Athena editor env created](streamlit_app/images/env_created.png)

- Now, lets quickly test the queries against the customers and procedures table by running the following two example queries below:

```sql
SELECT *
FROM athena_db.procedures
WHERE insurance = 'yes' OR insurance = 'no';
```

![procedures query](streamlit_app/images/procedure_query.png)


```sql
SELECT * 
FROM athena_db.customers
WHERE balance >= 0;
```

![customers query](streamlit_app/images/customer_query.png)


- If tests were succesful, we can move to the next step.



### Step 4: Lambda Function Configuration
- Create a Lambda function (Python 3.12) for the Bedrock agent's action group. We will call this Lambda function `bedrock-agent-txtsql-action`. 

![Create Function](streamlit_app/images/create_function.png)

![Create Function2](streamlit_app/images/create_function2.png)

- Copy the provided code from [here](https://github.com/build-on-aws/bedrock-agent-txt2sql/blob/main/function/lambda_function.py), or from below into the Lambda function.
  
```python
import boto3
from time import sleep

# Initialize the Athena client
athena_client = boto3.client('athena')

def lambda_handler(event, context):
    print(event)

    def athena_query_handler(event):
        # Fetch parameters for the new fields

        # Extracting the SQL query
        query = event['requestBody']['content']['application/json']['properties'][0]['value']

        print("the received QUERY:",  query)
        
        s3_output = 's3://athena-destination-store-alias'  # Replace with your S3 bucket

        # Execute the query and wait for completion
        execution_id = execute_athena_query(query, s3_output)
        result = get_query_results(execution_id)

        return result

    def execute_athena_query(query, s3_output):
        response = athena_client.start_query_execution(
            QueryString=query,
            ResultConfiguration={'OutputLocation': s3_output}
        )
        return response['QueryExecutionId']

    def check_query_status(execution_id):
        response = athena_client.get_query_execution(QueryExecutionId=execution_id)
        return response['QueryExecution']['Status']['State']

    def get_query_results(execution_id):
        while True:
            status = check_query_status(execution_id)
            if status in ['SUCCEEDED', 'FAILED', 'CANCELLED']:
                break
            sleep(1)  # Polling interval

        if status == 'SUCCEEDED':
            return athena_client.get_query_results(QueryExecutionId=execution_id)
        else:
            raise Exception(f"Query failed with status '{status}'")

    action_group = event.get('actionGroup')
    api_path = event.get('apiPath')

    print("api_path: ", api_path)

    result = ''
    response_code = 200


    if api_path == '/athenaQuery':
        result = athena_query_handler(event)
    else:
        response_code = 404
        result = {"error": f"Unrecognized api path: {action_group}::{api_path}"}

    response_body = {
        'application/json': {
            'body': result
        }
    }

    action_response = {
        'actionGroup': action_group,
        'apiPath': api_path,
        'httpMethod': event.get('httpMethod'),
        'httpStatusCode': response_code,
        'responseBody': response_body
    }

    api_response = {'messageVersion': '1.0', 'response': action_response}
    return api_response
```

- Then, update the **alias** value for the `s3_output` variable in the python code above. After, select **Deploy** under **Code source** in the Lambda console. Review the code provided before moving to the next step.


![Lambda deploy](streamlit_app/images/lambda_deploy.png)

- Now, we need to apply a resource policy to Lambda that grants Bedrock agent access. To do this, we will switch the top tab from **code** to **configuration** and the side tab to **Permissions**. Then, scroll to the **Resource-based policy statements** section and click the **Add permissions** button.

![Permissions config](streamlit_app/images/permissions_config.png)

![Lambda resource policy create](streamlit_app/images/lambda_resource_policy_create.png)

- Enter `arn:aws:bedrock:us-west-2:{aws-account-id}:agent/* `. ***Please note, AWS recommends least privilage so only the allowed agent can invoke this Lambda function***. A `*` at the end of the ARN grants any agent in the account access to invoke this Lambda. Ideally, we would not use this in a production environment. Lastly, for the Action, select `lambda:InvokeAction`, then ***Save***.

![Lambda resource policy](streamlit_app/images/lambda_resource_policy.png)

- We also need to provide this Lambda function permissions to interact with an S3 bucket, and Amazon Athena service. While on the `Configuration` tab -> `Permissions` section, select the Role name:

![Lambda role name 1](streamlit_app/images/lambda_role1.png)

- Select `Add permissions -> Attach policies`. Then, attach the AWS managed policies ***AmazonAthenaFullAccess***,  and ***AmazonS3FullAccess*** by selecting, then adding the permissions. Please note, in a real world environment, it's recommended that you practice least privilage.

![Lambda role name 2](streamlit_app/images/lambda_role2.png)

- The last thing we need to do with the Lambda is update the configurations. Navigate to the `Configuration` tab, then `General Configuration` section on the left. From here select Edit.

![Lambda role name 2](streamlit_app/images/lambda_config1.png)

- Update the memory to 1024 MB, and Timeout to 1 minute. Scroll to the bottom, and save the changes.

![Lambda role name 2](streamlit_app/images/lambda_config2.png)


![Lambda role name 3](streamlit_app/images/lambda_config3.png)


- We are now done setting up the Lambda function



### Step 5: Setup Bedrock agent and action group 
- Navigate to the Bedrock console. Go to the toggle on the left, and under **Orchestration** select ***Agents***, then ***Create Agent***. Provide an agent name, like `athena-agent` then ***Create***.

![agent create](streamlit_app/images/agent_create.png)

- The agent description is optional, and use the default new service role. For the model, select **Anthropic Claude 3 Haiku**. Next, provide the following instruction for the agent:


```instruction
Role: You are a SQL developer creating queries for Amazon Athena.

Objective: Generate SQL queries to return data based on the provided schema and user request. Also, returns SQL query created.

1. Query Decomposition and Understanding:
   - Analyze the user’s request to understand the main objective.
   - Break down reqeusts into sub-queries that can each address a part of the user's request, using the schema provided.

2. SQL Query Creation:
   - For each sub-query, use the relevant tables and fields from the provided schema.
   - Construct SQL queries that are precise and tailored to retrieve the exact data required by the user’s request.

3. Query Execution and Response:
   - Execute the constructed SQL queries against the Amazon Athena database.
   - Return the results exactly as they are fetched from the database, ensuring data integrity and accuracy. Include the query generated and results in the response.
```

It should look similar to the following: 

![agent instruction](streamlit_app/images/agent_instruction.png)

- Scroll to the top, then select ***Save***.

- Keep in mind that these instructions guide the generative AI application in its role as a SQL developer creating efficient and accurate queries for Amazon Athena. The process involves understanding user requests, decomposing them into manageable sub-queries, and executing these to fetch precise data. This structured approach ensures that responses are not only accurate but also relevant to the user's needs, thereby enhancing user interaction and data retrieval efficiency.


- Next, we will add an action group. Scroll down to `Action groups` then select ***Add***.

- Call the action group `query-athena`. We will keep `Action group type` set to ***Define with API schemas***. `Action group invocations` should be set to ***select an existing Lambda function***. For the Lambda function, select `bedrock-agent-txtsql-action`.

- For the `Action group Schema`, we will choose ***Define with in-line OpenAPI schema editor***. Replace the default schema in the **In-line OpenAPI schema** editor with the schema provided below. You can also retrieve the schema from the repo [here](https://github.com/build-on-aws/bedrock-agent-txt2sql/blob/main/schema/athena-schema.json). After, select ***Add***.
`(This API schema is needed so that the bedrock agent knows the format structure and parameters needed for the action group to interact with the Lambda function.)`

```schema
{
  "openapi": "3.0.1",
  "info": {
    "title": "AthenaQuery API",
    "description": "API for querying data from an Athena database",
    "version": "1.0.0"
  },
  "paths": {
    "/athenaQuery": {
      "post": {
        "description": "Execute a query on an Athena database",
        "requestBody": {
          "description": "Athena query details",
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "Procedure ID": {
                    "type": "string",
                    "description": "Unique identifier for the procedure",
                    "nullable": true
                  },
                  "Query": {
                    "type": "string",
                    "description": "SQL Query"
                  }
                }
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Successful response with query results",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "ResultSet": {
                      "type": "array",
                      "items": {
                        "type": "object",
                        "description": "A single row of query results"
                      },
                      "description": "Results returned by the query"
                    }
                  }
                }
              }
            }
          },
          "default": {
            "description": "Error response",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "message": {
                      "type": "string"
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

Your configuration should look like the following:


![ag create gif](streamlit_app/images/action_group_creation.gif)



- Now we will need to modify the **Advanced prompts**. Select the orange **Edit in Agent Builder** button at the top. Scroll to the bottom and in the advanced prompts section, select `Edit`.

![ad prompt btn](streamlit_app/images/advance_prompt_btn.png)


- In the `Advanced prompts`, navigate to the **Orchestration** tab. Enable the `Override orchestration template defaults` option. Also, make sure that `Activate orchestration template` is enabled. 

- In the `Prompt template editor`, go to line 22-23, then copy/paste the following prompt:

```sql
Here are the table schemas for the Amazon Athena database <athena_schemas>. 

<athena_schemas>
  <athena_schema>
  CREATE EXTERNAL TABLE athena_db.customers (
    `Cust_Id` integer,
    `Customer` string,
    `Balance` integer,
    `Past_Due` integer,
    `Vip` string
  )
  ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
  LINES TERMINATED BY '\n'
  STORED AS TEXTFILE
  LOCATION 's3://athena-datasource-{alias}/';  
  </athena_schema>
  
  <athena_schema>
  CREATE EXTERNAL TABLE athena_db.procedures (
    `Procedure_ID` string,
    `Procedure` string,
    `Category` string,
    `Price` integer,
    `Duration` integer,
    `Insurance` string,
    `Customer_Id` integer
  )
  ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
  LINES TERMINATED BY '\n'
  STORED AS TEXTFILE
  LOCATION 's3://athena-datasource-{alias}/';  
  </athena_schema>
</athena_schemas>

Here are examples of Amazon Athena queries <athena_examples>.

<athena_examples>
  <athena_example>
  SELECT * FROM athena_db.procedures WHERE insurance = 'yes' OR insurance = 'no';  
  </athena_example>
  
  <athena_example>
    SELECT * FROM athena_db.customers WHERE balance >= 0;
  </athena_example>
</athena_examples>
```

![adv prompt create gif](streamlit_app/images/adv_prompt_creation.gif)


- This prompt helps provide the agent an example of the table schema(s) used for the database, along with an example of how the Amazon Athena query should be formatted. Additionally, there is an option to use a [custom parser Lambda function](https://docs.aws.amazon.com/bedrock/latest/userguide/lambda-parser.html) for more granular formatting. 

- Scroll to the bottom and select the ***Save and exit***. Then, ***Save and exit*** one more time.



### Step 6: Create an alias
- While query-agent is still selected, scroll down to the Alias secion and select ***Create***. Choose a name of your liking. Make sure to copy and save your **AliasID**. You will need this in step 9.
 
![Create alias](streamlit_app/images/create_alias.png)

- Next, navigate to the **Agent Overview** settings for the agent created by selecting **Agents** under the Orchestration dropdown menu on the left of the screen, then select the agent. Save the **AgentID** because you will also need this in step 9.

![Agent ARN2](streamlit_app/images/agent_arn2.png)


## Step 7: Testing the Setup

### Testing the Bedrock Agent
- While on the Bedrock console, select **Agents** under the *Orchestration* tab, then the agent you created. Select ***Edit in Agent Builder***, and make sure to **Prepare** the agent so that the changes made can update. After, ***Save and exit***. On the right, you will be able to enter prompts in the user interface to test your Bedrock agent.`(You may be prompt to prepare the agent once more before testing the latest chages form the AWS console)` 


![Agent test](streamlit_app/images/agent_test.png)


- Example prompts for Action Groups:

    1. Show me all of the procedures in the imaging category that are insured.

    2. Show me all of the customers that are vip, and have a balance over 200 dollars.
       


## Step 8: Setting Up Cloud9 Environment (IDE)

1.	Navigate in the Cloud9 management console. Then, select **Create Environment**

![create_environment](streamlit_app/images/create_environment.png)

2. Here, you will enter the following values in each field
   - **Name:** Bedrock-Environment (Enter any name)
   - **Instance type:** t3.small
   - **Platform:** Ubuntu Server 22.04 LTS
   - **Timeout:** 1 hour  

![ce2](streamlit_app/images/ce2.png)

   - Once complete, select the **Create** button at the bottom of the screen. The environment will take a couple of minutes to spin up. If you get an error spinning up Cloud9 due to lack of resources, you can also choose t2.micro for the instance type and try again. (The Cloud9 environment has Python 3.10.12 version at the time of this publication)


![ce3](streamlit_app/images/ce3.png)

3. Navigate back to the Cloud9 Environment, then select **open** next to the Cloud9 you just created. Now, you are ready to setup the Streamlit app!

![environment](streamlit_app/images/environment.png)


## Step 9: Setting Up and Running the Streamlit App
1. **Obtain the Streamlit App ZIP File**: Download the zip file of the project [here](https://github.com/build-on-aws/bedrock-agent-txt2sql/archive/refs/heads/main.zip).

2. **Upload to Cloud9**:
   - In your Cloud9 environment, upload the ZIP file.

![Upload file to Cloud9](streamlit_app/images/upload_file_cloud9.png)

3. **Unzip the File**:
   - Use the command `unzip bedrock-agent-txt2sql-main.zip` to extract the contents.
4. **Navigate to streamlit_app Folder**:
   - Change to the directory containing the Streamlit app. Use the command `cd ~/environment/bedrock-agent-txt2sql-main/streamlit_app`
5. **Update Configuration**:
   - Open the `InvokeAgent.py` file.
   - Update the `agentId` and `agentAliasId` variables with the appropriate values, then save it.

![Update Agent ID and alias](streamlit_app/images/update_agentId_and_alias.png)

6. **Install Streamlit** (if not already installed):
   - Run the following command to install all of the dependencies needed:

     ```bash
     pip install streamlit boto3 pandas
     ```
     
7. **Run the Streamlit App**:
   - Execute the command `streamlit run app.py --server.address=0.0.0.0 --server.port=8080`.
   - Streamlit will start the app, and you can view it by selecting "Preview" within the Cloud9 IDE at the top, then **Preview Running Application**
   - 
  ![preview button](streamlit_app/images/preview_btn.png)

   - Once the app is running, please test some of the sample prompts provided. (On 1st try, if you receive an error, try again.)

![Running App ](streamlit_app/images/running_app.png)

Optionally, you can review the trace events in the left toggle of the screen. This data will include the rational tracing, invocation input tracing, and observation tracing.

![Trace events ](streamlit_app/images/trace_events.png)

## Setup an IoT Device as data Source

1. Setup your IoT device to send data to IoT Core. Here we assume that you are using AWS IoT Core service to ingest MQTT Payload from your IoT Device
   A typical JSON payload would be:

```
{
  "tof": 54,
  "t_temp_low": 29.57,
  "t_temp": 31.52,
  "t_temp_high": 33.59,
  "e_temp": 30.9,
  "e_pressure": 1006.9,
  "e_humidity": 70.8,
  "light2": 294.57
}
```
This post does not describe the details of onboarding an IoT Device to the IoT Core. But you can follow the simple steps described in the blog post
There are a lot of resources (AWS and non-AWS) to quikcly get started on connecting to IoT Core, I'm linking a few of them:

- [arduino](https://docs.arduino.cc/tutorials/opta/getting-started-with-aws-iot-core/)

- [raspberry pi link 1](https://docs.aws.amazon.com/iot/latest/developerguide/connecting-to-existing-device.html)
or
- [raspberry pi link 2](https://community.aws/concepts/getting-started-with-pico-w-iot-core)

- [IoT Core getting started](https://docs.aws.amazon.com/iot/latest/developerguide/iot-gs.html)

You can also just create an application running in EC2 or your local laptop to simulate an IoT Device. 

For step by step instructions, refer to:
[AWS IoT Core Workshop](https://catalog.workshops.aws/aws-iot-immersionday-workshop/en-US/aws-iot-core/device-sdk-v2/lab11-gettingstarted2)


2. Create a S3 bucket that will hold the IoT Data
for example:
 s3://athena-datasource-mybucket/
  
4. Configure IoT Core rules that will send the data from IoT Core to the S3 bucket
IoT Core -> routing -> Create rules
Configure your SQL statement, assuming the IoT Device is sending data to 'mytopic' IoT Topic:

```
select * from 'mytopic'
```

Set the S3 prefix at:
```
iotdata/${timestamp}
```

So the data will be sent in an S3 folder:
```
s3://athena-datasource-mybucket/iotdata
```

4. Create a crawler that will discover the schema of the data in the S3 bucket and create the corresponding Athena Table
   For example a table called "athena_datasource_mytable" in the "athena_db" database:
   ```
   athena_db.athena_datasource_mytable
   ```

6. Now you can override with the orchestration template in the Bedrock agent action group

```
   Human:
You are a research assistant AI that has been equipped with one or more functions to help you answer a <question>. Your goal is to answer the user's question to the best of your ability, using the function(s) to gather more information if necessary to better answer the question. If you choose to call a function, the result of the function call will be added to the conversation history in <function_results> tags (if the call succeeded) or <error> tags (if the function failed). $ask_user_missing_parameters$
You were created with these instructions to consider as well:
<auxiliary_instructions>$instruction$</auxiliary_instructions>

Here are the table schemas for the Amazon Athena database <athena_schemas>. 

  <athena_schema>
  CREATE EXTERNAL TABLE athena_db.athena_datasource_mytable (
    `t_temp_high` double,
    `e_temp` double,
    `tof` integer,
    `t_temp_low` double,
    `e_pressure` double,
    `light2` double,
    `e_humidity` double,
    `timestamp` string,
    `t_temp` double,
    `light2` double
  )
  ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
  LINES TERMINATED BY '\n'
  STORED AS TEXTFILE
  LOCATION 's3://athena-datasource-mybucket/iotdata';  
  </athena_schema>

</athena_schemas>

Here are examples of Amazon Athena SQL queries to create when IoT data, sensors data or device data are requested.
The examples includes:
- SQL query to return all the data from the sensors between the dates: 10 march 2024 14:42 and 10 march 2024 14:45 <iot_example1>
- SQL query to return the temperature data from the sensors between the dates: 9 march 2024 22:41 and 9 march 2024 22:45 <iot_example2>
- SQL query to return the last 10 values for iot sensors time of flight distance data or "tof" data <iot_example3>
- SQL query to return the last (or latest) value for pressure sensor reading <iot_example4>
- SQL query to return the average light intensity, or average brightness level from the iot light sensors for the past last 1 hour <iot_example5>
- SQL query to return the maximum light intensity, or brightness level from the iot sensors reading for the past last 3 hour <iot_example6>
- SQL query to return the minimum humidity level from the iot sensors between 9 march 2024 22:43 and 9 march 2024 22:44:48  <iot_example7>
- SQL query to return the dates when the maximum temperature was more than 29.12 degrees <iot_example8>

Double check every query for correct format. Provide the Amazon SQL Athena queries if requested.


  <iot_example1>
  SELECT * FROM "athena_db"."athena_datasource_mytable" where "timestamp" between '2024-03-10 14:42:00' and '2024-03-10 14:45:00';
  </iot_example1>

  <iot_example2>
  SELECT "t_temp", "timestamp" FROM "athena_db"."athena_datasource_mytable" WHERE "timestamp" between '2024-03-09 22:41' and '2024-03-09 22:45' ORDER BY "timestamp" DESC;
  </iot_example2>

  <iot_example3>
  SELECT "tof" AS timeofflight, "timestamp" FROM "athena_db"."athena_datasource_mytable" ORDER BY "timestamp"  DESC LIMIT 10;
  </iot_example3>

  <iot_example4>
  SELECT "e_pressure" AS pressure,"timestamp" FROM "athena_db"."athena_datasource_mytable" ORDER BY "timestamp" DESC LIMIT 1;
  </iot_example4>
  
  <iot_example5>
  SELECT AVG("light2") as average_light
  FROM "athena_db"."athena_datasource_mytable" WHERE cast("timestamp" as timestamp) >= ((current_timestamp + interval '8' hour) - interval '1' hour )
  AND cast("timestamp" as timestamp) <= (current_timestamp + interval '8' hour)
  </iot_example5>

  <iot_example6>
  SELECT MAX("light2") as max_light
  FROM "athena_db"."athena_datasource_mytable" WHERE cast("timestamp" as timestamp) >= ((current_timestamp + interval '8' hour) - interval '3' hour )
  AND cast("timestamp" as timestamp) <= (current_timestamp + interval '8' hour)
  </iot_example6>

  <iot_example7>
  SELECT MIN("e_humidity") AS min_humidity FROM "athena_db"."athena_datasource_mytable" WHERE "timestamp" between '2024-03-09 22:43' and '2024-03-09 22:44:48';
  </iot_example7>

  <iot_example8>
  SELECT "t_temp", "timestamp" FROM "athena_db"."athena_datasource_mytable" WHERE "t_temp" >= 29.12 ORDER BY "timestamp" DESC;
  </iot_example8>

```

6. Now your chatbot should be able to respond to query such as:

   - Provide the most recent 10 dates from the iot sensor data when the maximum temperature was more than 29.12 degrees. Provide the values and SQL Query
   - Provide the average temperature from the iot sensors for the past last 2 hours.
   - Return the latest value for pressure sensor reading. Provide the timestamp and the SQL Query
   - etc

   

## Cleanup

After completing the setup and testing of the Bedrock Agent and Streamlit app, follow these steps to clean up your AWS environment and avoid unnecessary charges:
1. Delete S3 Buckets:
- Navigate to the S3 console.
- Select the buckets "athena-datasource-alias" and "bedrock-agents-athena-output-alias". Make sure that both of these buckets are empty by deleting the files. 
- Choose 'Delete' and confirm by entering the bucket name.

2.	Remove Lambda Function:
- Go to the Lambda console.
- Select the "bedrock-agent-txtsql-action" function.
- Click 'Delete' and confirm the action.

3.	Delete Bedrock Agent:
- In the Bedrock console, navigate to 'Agents'.
- Select the created agent, then choose 'Delete'.

4.	Clean Up Cloud9 Environment:
- Navigate to the Cloud9 management console.
- Select the Cloud9 environment you created, then delete.




## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

