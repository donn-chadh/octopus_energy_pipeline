# Octopus Energy Automated ETL Pipeline
This pipeline is an automated ETL that fetches consumption data, unit rates, and standing charges from the Octopus Energy API before processing the data, and loading them into a Neon PostgreSQL database. 
The infrastructure is packaged via Docker, hosted on AWS ECR, executed by AWS Lambda, and scheduled via AWS EventBridge. 

## Pipeline process overview
1. Trigger: AWS EventBridge cron rule wakes up the pipeline daily at 1am.
2. Compute: AWS Lambda pulls and runs the Docker container.
3. Extract: Pythons requests package calls the Octopus Energy REST API.
4. Transform: Added a 'pulled_at' Timestamp to every row to track data freshness.
5. Load: psycop2 pushes record to Neon PostgreSQL with stuctural conflict handling.

## What has changed and why
### 1. Switched from one tariff to another
- The problem: The data being pulled stopped mid-2023. Our electric provider Bulb, was acquired by Octopus and therefore a change of tariffs occured.
- The solution: Sourced the new tariff code from Octopus and created a loop through the list of tariff codes to automatically fetch data for both.

### 2. Hard coding variables
- The problem: I harded coded variables which created a security risk for when I uploaded the code to my github. 
- The solution: Used OS package with .getenv and a requirements.txt file saved locally with my variables, however, this also became problematic as AWS would not be able to read my local files so I updated the information to Enviroment Variables within AWS. 

### 3. Fixed a code cut off bug
- The problem: The new tariff data was complaetly missing. This happened because a return command was placed inside the loop. The code would grab the Bulb data, hit that return command, and immediately shut down before it ever got a chance to look as the new tariff data.
- The solution: I moved that return command to the bottom. Now the code is forced to finish checking both tariff codes in the list before it stops running.

### 4. Fixed Docker
- The Problem: Because of continued code changes, back and forth fixes, Docker saved broken versions of the project. Even when the correct and final code was saved, Docker refused to look at the new file and kept pulling from the dirty memory.
- The solution: I added --no-cache rule to the commmend which completly wiped Dockers dirty memory and forced it to pull the correct file for the AWS upload.

## How to upload updates
Step 1: Log into AWS via terminal
aws ecr get-login-password --region eu-north-1 | docker login --username AWS --password-stdin ************.dkr.ecr.eu-north-1.amazonaws.com

Step 2: Package the code (had to use --no-cache as there were a few attempts made before this worked correctly)
docker build --no-cache --platform linux/amd64 --provenance=false -t my-octopus-repo .

Step 3: Label the package
docker tag my-octopus-repo:latest ************.dkr.ecr.eu-north-1.amazonaws.com/my-octopus-repo:latest

Step 4: Push to AWS ECR
docker push ************.dkr.ecr.eu-north-1.amazonaws.com/my-octopus-repo:latest

Step 5: Make it live
- Go to AWS Lambda console, click the octopus_energy function, go to the image tab, and click deploy new image (feel free to test it after)

## Automatic Run Schedule
- Set CRON to (0 1 * * ? *)
- Runs automatically as 1:00AM UTC every day.

## AWS settings checklist
- Ensure the following variables are listed under Enviroment variables
1. DATABASE_URL
2. API_KEY
3. MPAN
4. SERIAL_NUM
5. ACC_NUM

## Database tables
The pipeline loads data into three distinct tables in Neon PostgreSQL
- Consumption: Stores the history of electicity used (kWH).
- Unit Rates: Tracks the changing cost per kWh for both tariffs.
- Standing Charges: Stores the daily fixed connection fees. (boo)
 
## What I learned
- Things go wrong all the time - expect it and use a diagnostic cell to debug.
- Dirty variables are a thing and you need to reset or force through some changes from time to time.
- Best practise for README files is to write as you go i.e end of day log the changes you made.
