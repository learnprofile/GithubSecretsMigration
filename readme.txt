

Github Secrets Migration





Introduction
In this article, I will discuss the process of migrating GitHub Secrets from one organization to another. This may be necessary due to mergers, acquisitions, or internal organizational changes that result in modifications to GitHub structures.


Problem:
Recently, I had to migrate my GitHub repositories from one organization to another. While searching over Internet, found very little to none documentation on efficiently moving GitHub secrets.

During the migration, Some of the Initial challenges I encountered were:

GitHub does not provide a direct way to export and import secrets.
Environment and repository secrets must be manually re-added until and unless you are aware of the past secrets else you end up creating new secrets.
Large teams often have multiple secrets, making it difficult to keep track of them.
The migration process can be time-consuming, especially when dealing with multiple environments (e.g., test, prod). Based on downtime and if you have back and forth deployments happening.
To solve these issues, I developed a workaround using GitHub Actions and Python scripts to automate the extraction and deployment of secrets between organizations.

Solution:
Since there is no built-in automated way to migrate secrets, I implemented a workaround.

1. Extracting Secrets Using jq (JQuery)
GitHub does not allow direct export of secrets. However, we can use the jq command-line tool to construct a JSON file containing secrets.

I found jq useful for extracting secrets with specific arguments. Below is an example of how to extract secrets into a JSON file:

jq -n \
              --arg ARMPEMFILE "${{ secrets.ARMPEMFILE }}" \
 '{
                ARMPEMFILE: $ARMPEMFILE,
 }' > secrets_prod.json
This method allows us to generate a structured JSON file containing all relevant secrets for an environment.

2. Automating the Process with GitHub Actions
To streamline the migration, I created a GitHub Actions workflow to extract secrets across environments.

Rather than saving the secrets anywhere, the workflow fetches the secrets dynamically and passes them to the next step, where a Python script processes them.

Here’s an example screenshot of my existing Environment Secrets that I wanted to migrate to the new GitHub organization. These secrets were created by different team members over time.

Similarly, here’s a screenshot of my Repository Secrets before migration.


and the Repository Secrets screenshot.


Current Available environments


name: Extract Secrets

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  fetch_and_deploy_secrets:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [test, prod]  # Run for both environments consider 2 environments

    environment: ${{ matrix.environment }}

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Generate Environment Secrets File
        run: |
          if [ "${{ matrix.environment }}" == "test" ]; then
            jq -n \
              --arg ARMPEMFILE "${{ secrets.ARMPEMFILE }}" \
              --arg AZURE_CREDENTIALS "${{ secrets.AZURE_CREDENTIALS }}" \
              --arg CERT_PASSWORD "${{ secrets.CERT_PASSWORD }}" \
              --arg ARM_PEM_FILE "${{ secrets.ARM_PEM_FILE }}" \
              --arg CLIENTID "${{ secrets.CLIENTID }}" \
              --arg CREDENTIALS "${{ secrets.CREDENTIALS }}" \
              --arg DEV_HUB_NAME "${{ secrets.DEV_HUB_NAME }}" \
              --arg IOT_HUB_NAME "${{ secrets.IOT_HUB_NAME }}" \
              --arg SPNCERT "${{ secrets.SPNCERT }}" \
              --arg SPN_CERT_PASSWORD "${{ secrets.SPN_CERT_PASSWORD }}" \
              --arg SPN_CLIENT_ID "${{ secrets.SPN_CLIENT_ID }}" \
              --arg SPN_SUBSCRIPTION_ID "${{ secrets.SPN_SUBSCRIPTION_ID }}" \
              --arg SPN_TENANT_ID "${{ secrets.SPN_TENANT_ID }}" \
              --arg SUBSCRIPTION_ID "${{ secrets.SUBSCRIPTION_ID }}" \
              --arg TENANT_ID "${{ secrets.TENANT_ID }}" \
              '{
                ARMPEMFILE: $ARMPEMFILE,
                AZURE_CREDENTIALS: $AZURE_CREDENTIALS,
                CERT_PASSWORD: $CERT_PASSWORD,
                ARM_PEM_FILE: $ARM_PEM_FILE,
                CLIENTID: $CLIENTID,
                CREDENTIALS: $CREDENTIALS,
                DEV_HUB_NAME: $DEV_HUB_NAME,
                IOT_HUB_NAME: $IOT_HUB_NAME,
                SPNCERT: $SPNCERT,
                SPN_CERT_PASSWORD: $SPN_CERT_PASSWORD,
                SPN_CLIENT_ID: $SPN_CLIENT_ID,
                SPN_SUBSCRIPTION_ID: $SPN_SUBSCRIPTION_ID,
                SPN_TENANT_ID: $SPN_TENANT_ID,
                SUBSCRIPTION_ID: $SUBSCRIPTION_ID,
                TENANT_ID: $TENANT_ID
              }' > secrets_cert.json
          else
            jq -n \
              --arg ENVIRONMENT_NAME "${{ secrets.ENVIRONMENT_NAME }}" \
              --arg SPN_CERT "${{ secrets.SPN_CERT }}" \
              --arg SPN_CERT_PASSWORD "${{ secrets.SPN_CERT_PASSWORD }}" \
              --arg SPN_CLIENT_ID "${{ secrets.SPN_CLIENT_ID }}" \
              --arg SPN_SUBSCRIPTION_ID "${{ secrets.SPN_SUBSCRIPTION_ID }}" \
              --arg SPN_TENANT_ID "${{ secrets.SPN_TENANT_ID }}" \
              '{
                ENVIRONMENT_NAME: $ENVIRONMENT_NAME,
                SPN_CERT: $SPN_CERT,
                SPN_CERT_PASSWORD: $SPN_CERT_PASSWORD,
                SPN_CLIENT_ID: $SPN_CLIENT_ID,
                SPN_SUBSCRIPTION_ID: $SPN_SUBSCRIPTION_ID,
                SPN_TENANT_ID: $SPN_TENANT_ID
              }' > secrets_prod.json
          fi

      - name: Generate Repository Secrets File
        run: |
          jq -n \
            --arg SONAR_HOST_URL "${{ secrets.SONAR_HOST_URL }}" \
            --arg SONAR_TOKEN "${{ secrets.SONAR_TOKEN }}" \
            '{
              SONAR_HOST_URL: $SONAR_HOST_URL,
              SONAR_TOKEN: $SONAR_TOKEN
            }' > repository.json
Note: If you want to verify the extracted secrets before migration, you can add an artifact upload step to store the JSON file temporarily.

One common question that may arise is: Can we fetch environment secret names dynamically instead of hardcoding them in the YAML file?

Unfortunately, GitHub does not support this.

While the GitHub REST API allows fetching available secret names, there is no direct way to pass these names dynamically to a YAML or jq script. Due to this limitation, I had to explicitly define the secret names in my workflow. 😞


I have raised this question on external GitHub forums, but I have not yet received a solution. Here are the relevant links:

How to retrieve a secret where the secret variable name is stored inside another variable ? ·…
Select Topic Area Not able to derive a Secret value , when the name of the Secret variable is dynamic and not static…
github.com

Trying to fetch secret names and want to add in my github workflow dynamically · community ·…
Select Topic Area Question Body I am trying to fetch my environment secret names using gh api and then wants to pass in…
github.com

3. Uploading Secrets to the New Organization
Once secrets are extracted, they need to be securely transferred to the new organization. The following Python script encrypts secrets and uploads them to the target organization using GitHub API.

Here is the python script which can be used for setting up Environment and Repository secrets.

Breakdown of my Python script logic:

Breakdown of the Python Script for Migrating GitHub Secrets
1. Set Up Environment Variables
The script reads environment variables, including:

GITHUB_TOKEN: Used for authentication with GitHub API.
TARGET_ENV: Specifies whether secrets are for test or prod environments.
TARGET_ORG & TARGET_REPO: Define the destination organization and repository.

2. Fetching Public Keys for Encryption
Since GitHub requires secrets to be encrypted before storing them, the script must first fetch the public encryption keys for:

Environment secrets → /environments/{TARGET_ENV}/secrets/public-key
Repository secrets → /actions/secrets/public-key
Each key is later used to encrypt the corresponding secrets before uploading.

For more details, refer to:

REST API endpoints for GitHub Actions Secrets - GitHub Docs
Use the REST API to interact with secrets in GitHub Actions.
docs.github.com

REST API endpoints for GitHub Actions Secrets - GitHub Docs
Use the REST API to interact with secrets in GitHub Actions.
docs.github.com

3. Reading Secrets from JSON Files
Next, the script reads the previously generated JSON files containing secrets.

For example, if I have two environments (cert and prod), I will push secrets accordingly:

Environment-specific secrets → secrets_cert.json, secrets_prod.json
Repository-level secrets → repository.json
If any JSON file is missing or corrupted, the script will display an error message.

4. Encrypting Secrets Using GitHub’s Public Key
GitHub requires secrets to be encrypted using NaCl (Networking and Cryptography Library) before they can be stored.

The script follows these steps for encryption:
It encrypts the secret value using NaCl’s SealedBox (so only GitHub can decrypt it).
The encrypted secret is then base64 encoded again for transmission.

Encrypting secrets for the REST API - GitHub Docs
In order to create or update a secret with the REST API, you must encrypt the value of the secret.
docs.github.com

5. Uploading Encrypted Secrets to GitHub
Finally the script iterates over both environment and repository secrets and uploads them to GitHub,

It makes PUT requests to GitHub’s API

# for environment secrets:
UPLOAD_SECRET_URL_ENV = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/environments/{TARGET_ENV}/secrets/{secret_name}"
# for repository secrets:
UPLOAD_SECRET_URL_REPO = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/actions/secrets/{secret_name}"
Python Script to Upload secrets under Environment secrets.
Here is a Python script to test locally and push secrets under Environment Secrets from Visual Studio Code.

Prerequisite: Ensure that environments are created beforehand, as my script does not include logic to append new environments automatically.

Usage Instructions
Update the script to point to the correct secrets_environment.json file.
Set TARGET_ENV = “corresponding environment” before execution.


import base64
import json  
import requests
from nacl import encoding, public


GITHUB_TOKEN = "xxxx" # your New target Github environment PAT token.
TARGET_ORG = "org name"
TARGET_REPO = "new target repo name" #make sure it already exists over there
TARGET_ENV = "dev"  # Specify the target environment
SECRETS_FILE_PATH = r'C:\Users\PrashanthKumar\Downloads\appfolder\secrets_dev.json'  # Path to secrets.json file

# Step 1: Fetch the Environment Public Key
GITHUB_API_URL = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/environments/{TARGET_ENV}/secrets/public-key"
headers = {
    "Authorization": f"Bearer {GITHUB_TOKEN}",
    "Accept": "application/vnd.github.v3+json",
    "X-GitHub-Api-Version": "2022-11-28"
}

response = requests.get(GITHUB_API_URL, headers=headers)
if response.status_code != 200:
    print(f"Failed to fetch public key! Response: {response.json()}")
    exit(1)

public_key_data = response.json()
public_key = public_key_data["key"]
public_key_id = public_key_data["key_id"]


# Step 2: Read secrets from secrets.json
try:
    with open(SECRETS_FILE_PATH, 'r') as f:
        secrets_dict = json.load(f)
except json.JSONDecodeError as e:
    print(f"Error decoding JSON: {e}")
    exit(1)
except Exception as e:
    print(f"Error reading secrets file: {e}")
    exit(1)

# Step 3: Encrypt and upload each secret
def encrypt_secret(secret_value, public_key):
    public_key_bytes = base64.b64decode(public_key + "==")  
    public_key_obj = public.PublicKey(public_key_bytes)
    sealed_box = public.SealedBox(public_key_obj)
    encrypted_secret = sealed_box.encrypt(secret_value.encode("utf-8"))
    return base64.b64encode(encrypted_secret).decode("utf-8")

for secret_name, secret_value in secrets_dict.items():
    print(f"🔹 Processing Secret: {secret_name}")

    encrypted_secret_value = encrypt_secret(secret_value, public_key)


    # Step 4: Upload the Encrypted Secret to GitHub
    UPLOAD_SECRET_URL = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/environments/{TARGET_ENV}/secrets/{secret_name}"
    payload = json.dumps({
        "encrypted_value": encrypted_secret_value,
        "key_id": public_key_id
    })

    upload_response = requests.put(UPLOAD_SECRET_URL, headers=headers, data=payload)

    if upload_response.status_code == 201:
        print(f"Secret '{secret_name}' uploaded successfully!")
    elif upload_response.status_code == 204:
        print(f"Secret '{secret_name}' updated successfully!")
    else:
        print(f"Error storing secret '{secret_name}' in GitHub! Response: {upload_response.json()}")

print("All secrets processed successfully!")
and the only change in this above script is to point it to right secrets_environment.json file and change TARGET_ENV = “corresponding environment”.

Python Script to Upload Secrets under Repository secrets.
The logic remains the same for repository secrets, except for a change in the GitHub API endpoint URL.

Simply modify the API request to point to repository secrets instead of environment secrets.

##version 3 from line 212 until line 286 works. and it loads keys under repo secret. this is the actual working script.
import base64
import json
import requests
from nacl import encoding, public


GITHUB_TOKEN = "xxxx" # your New target Github environment PAT token.
TARGET_ORG = "org name"
TARGET_REPO = "new target repo name" #make sure it already exists over there
SECRETS_FILE_PATH = r'C:\Users\PrashanthKumar\Downloads\appfolder\repository.json'  # Path to secrets.json file


GITHUB_API_URL = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/actions/secrets/public-key"
headers = {
    "Authorization": f"Bearer {GITHUB_TOKEN}",
    "Accept": "application/vnd.github+json",
    "X-GitHub-Api-Version": "2022-11-28"
}

response = requests.get(GITHUB_API_URL, headers=headers)
if response.status_code != 200:
    print(f"❌ Failed to fetch public key! Response: {response.json()}")
    exit(1)

public_key_data = response.json()
public_key = public_key_data["key"]
public_key_id = public_key_data["key_id"]



# Step 2: Read secrets from secrets.json
try:
    with open(SECRETS_FILE_PATH, 'r') as f:
        secrets_dict = json.load(f)
except json.JSONDecodeError as e:
    print(f"❌ Error decoding JSON: {e}")
    exit(1)
except Exception as e:
    print(f"❌ Error reading secrets file: {e}")
    exit(1)

# Step 3: Encrypt and upload each secret
def encrypt_secret(secret_value, public_key):
    public_key_bytes = base64.b64decode(public_key + "==")  # Ensure proper padding for base64 decoding
    public_key_obj = public.PublicKey(public_key_bytes)
    sealed_box = public.SealedBox(public_key_obj)
    encrypted_secret = sealed_box.encrypt(secret_value.encode("utf-8"))
    return base64.b64encode(encrypted_secret).decode("utf-8")

for secret_name, secret_value in secrets_dict.items():
    print(f"🔹 Processing Secret: {secret_name}")

    encrypted_secret_value = encrypt_secret(secret_value, public_key)


    # Step 4: Upload the Encrypted Secret to GitHub
    UPLOAD_SECRET_URL = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/actions/secrets/{secret_name}"
    payload = json.dumps({
        "encrypted_value": encrypted_secret_value,
        "key_id": public_key_id
    })

    upload_response = requests.put(UPLOAD_SECRET_URL, headers=headers, data=payload)

    if upload_response.status_code == 201:
        print(f"Secret '{secret_name}' uploaded successfully!")
    elif upload_response.status_code == 204:
        print(f"Secret '{secret_name}' updated successfully!")
    else:
        print(f"Error storing secret '{secret_name}' in GitHub! Response: {upload_response.json()}")

print("All secrets processed successfully!")


###version 3 until line 286 works. - 8.09 pm.
Full YAML file to extract secrets and pushing it to new Target Github repository.
name: Extract and Deploy Secrets

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  fetch_and_deploy_secrets:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [test, prod]  # Run for both environments

    environment: ${{ matrix.environment }}

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Generate Environment Secrets File
        run: |
          if [ "${{ matrix.environment }}" == "test" ]; then
            jq -n \
              --arg ARMPEMFILE "${{ secrets.ARMPEMFILE }}" \
              --arg AZURE_CREDENTIALS "${{ secrets.AZURE_CREDENTIALS }}" \
              --arg CERT_PASSWORD "${{ secrets.CERT_PASSWORD }}" \
              --arg ARM_PEM_FILE "${{ secrets.ARM_PEM_FILE }}" \
              --arg CLIENTID "${{ secrets.CLIENTID }}" \
              --arg CREDENTIALS "${{ secrets.CREDENTIALS }}" \
              --arg DEV_HUB_NAME "${{ secrets.DEV_HUB_NAME }}" \
              --arg IOT_HUB_NAME "${{ secrets.IOT_HUB_NAME }}" \
              --arg SPNCERT "${{ secrets.SPNCERT }}" \
              --arg SPN_CERT_PASSWORD "${{ secrets.SPN_CERT_PASSWORD }}" \
              --arg SPN_CLIENT_ID "${{ secrets.SPN_CLIENT_ID }}" \
              --arg SPN_SUBSCRIPTION_ID "${{ secrets.SPN_SUBSCRIPTION_ID }}" \
              --arg SPN_TENANT_ID "${{ secrets.SPN_TENANT_ID }}" \
              --arg SUBSCRIPTION_ID "${{ secrets.SUBSCRIPTION_ID }}" \
              --arg TENANT_ID "${{ secrets.TENANT_ID }}" \
              '{
                ARMPEMFILE: $ARMPEMFILE,
                AZURE_CREDENTIALS: $AZURE_CREDENTIALS,
                CERT_PASSWORD: $CERT_PASSWORD,
                ARM_PEM_FILE: $ARM_PEM_FILE,
                CLIENTID: $CLIENTID,
                CREDENTIALS: $CREDENTIALS,
                DEV_HUB_NAME: $DEV_HUB_NAME,
                IOT_HUB_NAME: $IOT_HUB_NAME,
                SPNCERT: $SPNCERT,
                SPN_CERT_PASSWORD: $SPN_CERT_PASSWORD,
                SPN_CLIENT_ID: $SPN_CLIENT_ID,
                SPN_SUBSCRIPTION_ID: $SPN_SUBSCRIPTION_ID,
                SPN_TENANT_ID: $SPN_TENANT_ID,
                SUBSCRIPTION_ID: $SUBSCRIPTION_ID,
                TENANT_ID: $TENANT_ID
              }' > secrets_cert.json
          else
            jq -n \
              --arg ENVIRONMENT_NAME "${{ secrets.ENVIRONMENT_NAME }}" \
              --arg SPN_CERT "${{ secrets.SPN_CERT }}" \
              --arg SPN_CERT_PASSWORD "${{ secrets.SPN_CERT_PASSWORD }}" \
              --arg SPN_CLIENT_ID "${{ secrets.SPN_CLIENT_ID }}" \
              --arg SPN_SUBSCRIPTION_ID "${{ secrets.SPN_SUBSCRIPTION_ID }}" \
              --arg SPN_TENANT_ID "${{ secrets.SPN_TENANT_ID }}" \
              '{
                ENVIRONMENT_NAME: $ENVIRONMENT_NAME,
                SPN_CERT: $SPN_CERT,
                SPN_CERT_PASSWORD: $SPN_CERT_PASSWORD,
                SPN_CLIENT_ID: $SPN_CLIENT_ID,
                SPN_SUBSCRIPTION_ID: $SPN_SUBSCRIPTION_ID,
                SPN_TENANT_ID: $SPN_TENANT_ID
              }' > secrets_prod.json
          fi

      - name: Generate Repository Secrets File
        run: |
          jq -n \
            --arg SONAR_HOST_URL "${{ secrets.SONAR_HOST_URL }}" \
            --arg SONAR_TOKEN "${{ secrets.SONAR_TOKEN }}" \
            '{
              SONAR_HOST_URL: $SONAR_HOST_URL,
              SONAR_TOKEN: $SONAR_TOKEN
            }' > repository.json

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'
      
      - name: Install dependencies
        run: |
          pip install pynacl requests
      
      - name: Verify requests installation
        run: |
          python -m pip show requests

      - name: Run Python Script Inline
        run: |
          python <<EOF
          import base64
          import json  
          import requests
          from nacl import encoding, public
          import os

          TARGET_ENV = os.getenv("TARGET_ENV")
          print(f"Using Target Environment: {TARGET_ENV}")

          GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
          TARGET_ENV = os.getenv("TARGET_ENV")
          TARGET_ORG = "sede-x"
          TARGET_REPO = "qgciotcameraconnectivity-test"
          
          secrets_file = "secrets_cert.json" if TARGET_ENV == "test" else "secrets_prod.json"
          repo_secrets_file = "repository.json"

          print(f"Using Secrets File: {secrets_file}")
          print(f"Using Repository Secrets File: {repo_secrets_file}")

          # Construct GitHub API URL for environment secrets
          GITHUB_API_URL_ENV = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/environments/{TARGET_ENV}/secrets/public-key"
          print(f"Fetching Public Key for Environment from: {GITHUB_API_URL_ENV}")
          
          # Construct GitHub API URL for repository secrets
          GITHUB_API_URL_REPO = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/actions/secrets/public-key"
          print(f"Fetching Public Key for Repository from: {GITHUB_API_URL_REPO}")
          
          headers = {
              "Authorization": f"Bearer {GITHUB_TOKEN}",
              "Accept": "application/vnd.github.v3+json",
              "X-GitHub-Api-Version": "2022-11-28"
          }

          # Fetch the public key for environment secrets
          response_env = requests.get(GITHUB_API_URL_ENV, headers=headers)
          if response_env.status_code != 200:
              print(f"Failed to fetch environment public key! Response: {response_env.json()}")
              exit(1)

          public_key_data_env = response_env.json()
          public_key_env = public_key_data_env["key"]
          public_key_id_env = public_key_data_env["key_id"]

          print(f"Environment Public Key Retrieved (Base64): {public_key_env}")
          print(f"Environment Public Key ID: {public_key_id_env}")

          # Fetch the public key for repository secrets
          response_repo = requests.get(GITHUB_API_URL_REPO, headers=headers)
          if response_repo.status_code != 200:
              print(f"Failed to fetch repository public key! Response: {response_repo.json()}")
              exit(1)

          public_key_data_repo = response_repo.json()
          public_key_repo = public_key_data_repo["key"]
          public_key_id_repo = public_key_data_repo["key_id"]

          print(f"Repository Public Key Retrieved (Base64): {public_key_repo}")
          print(f"Repository Public Key ID: {public_key_id_repo}")

          # Read environment secrets from the selected JSON file
          try:
              with open(secrets_file, 'r') as f:
                  secrets_dict = json.load(f)
          except json.JSONDecodeError as e:
              print(f"Error decoding JSON: {e}")
              exit(1)
          except Exception as e:
              print(f"Error reading secrets file: {e}")
              exit(1)

          # Read repository secrets from the repository JSON file
          try:
              with open(repo_secrets_file, 'r') as f:
                  repo_secrets_dict = json.load(f)
          except json.JSONDecodeError as e:
              print(f"Error decoding repository JSON: {e}")
              exit(1)
          except Exception as e:
              print(f"Error reading repository secrets file: {e}")
              exit(1)

          # Encrypt and upload each secret
          def encrypt_secret(secret_value, public_key):
              public_key_bytes = base64.b64decode(public_key + "==")  
              public_key_obj = public.PublicKey(public_key_bytes)
              sealed_box = public.SealedBox(public_key_obj)
              encrypted_secret = sealed_box.encrypt(secret_value.encode("utf-8"))
              return base64.b64encode(encrypted_secret).decode("utf-8")

          # Upload environment secrets
          for secret_name, secret_value in secrets_dict.items():
              print(f"🔹 Processing Secret (Environment): {secret_name}")
              encrypted_secret_value = encrypt_secret(secret_value, public_key_env)
              print(f"🔹 Encrypted Secret (Base64): {encrypted_secret_value}")

              # Correct the API URL format for environment secrets
              UPLOAD_SECRET_URL_ENV = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/environments/{TARGET_ENV}/secrets/{secret_name}"
              payload = json.dumps({
                  "encrypted_value": encrypted_secret_value,
                  "key_id": public_key_id_env
              })

              upload_response_env = requests.put(UPLOAD_SECRET_URL_ENV, headers=headers, data=payload)

              if upload_response_env.status_code == 201:
                  print(f"Secret (Environment) '{secret_name}' uploaded successfully!")
              elif upload_response_env.status_code == 204:
                  print(f"Secret (Environment) '{secret_name}' updated successfully!")
              else:
                  print(f"Error storing secret (Environment) '{secret_name}' in GitHub! Response: {upload_response_env.json()}")

          # Upload repository secrets
          for secret_name, secret_value in repo_secrets_dict.items():
              print(f"🔹 Processing Secret (Repository): {secret_name}")
              encrypted_secret_value = encrypt_secret(secret_value, public_key_repo)
              print(f"🔹 Encrypted Secret (Base64): {encrypted_secret_value}")

              # Correct the API URL format for repository secrets
              UPLOAD_SECRET_URL_REPO = f"https://api.github.com/repos/{TARGET_ORG}/{TARGET_REPO}/actions/secrets/{secret_name}"
              payload = json.dumps({
                  "encrypted_value": encrypted_secret_value,
                  "key_id": public_key_id_repo
              })

              upload_response_repo = requests.put(UPLOAD_SECRET_URL_REPO, headers=headers, data=payload)

              if upload_response_repo.status_code == 201:
                  print(f"Secret (Repository) '{secret_name}' uploaded successfully!")
              elif upload_response_repo.status_code == 204:
                  print(f"Secret (Repository) '{secret_name}' updated successfully!")
              else:
                  print(f"Error storing secret (Repository) '{secret_name}' in GitHub! Response: {upload_response_repo.json()}")
          
          print("All secrets processed successfully!")
          EOF
        env:
          GITHUB_TOKEN: ${{ secrets.TARGET_PAT }}
          TARGET_ENV: ${{ matrix.environment }}



here is the sample Jq script if you have more than 2 environments

     jobs:
  fetch_and_deploy_secrets:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [UAT, test, Development]

    environment: ${{ matrix.environment }}

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Generate Environment Secrets File
        run: |
          if [ "${{ matrix.environment }}" == "test" ]; then
            jq -n \
              --arg SPN_CERT "${{ secrets.SPN_CERT }}" \
              --arg SPN_CLIENTID "${{ secrets.SPN_CLIENTID }}" \
              --arg SPN_CLIENT_PASSWORD "${{ secrets.SPN_CLIENT_PASSWORD }}" \
              --arg SPN_SUBSCRIPTION_ID "${{ secrets.SPN_SUBSCRIPTION_ID }}" \
              --arg SPN_TENANT_ID "${{ secrets.SPN_TENANT_ID }}" \
              '{
                SPN_CERT: $SPN_CERT,
                SPN_CLIENTID: $SPN_CLIENTID,
                SPN_CLIENT_PASSWORD: $SPN_CLIENT_PASSWORD,
                SPN_SUBSCRIPTION_ID: $SPN_SUBSCRIPTION_ID,
                SPN_TENANT_ID: $SPN_TENANT_ID
              }' > secrets_test.json
          elif [ "${{ matrix.environment }}" == "Development" ]; then
            jq -n \
              --arg SPN_CERT "${{ secrets.SPN_CERT }}" \
              --arg SPN_CERT_PASSWORD "${{ secrets.SPN_CERT_PASSWORD }}" \
              --arg SPN_CERT_PFX "${{ secrets.SPN_CERT_PFX }}" \
              --arg SPN_CLIENT_ID "${{ secrets.SPN_CLIENT_ID }}" \
              --arg SPN_SUBSCRIPTION_ID "${{ secrets.SPN_SUBSCRIPTION_ID }}" \
              --arg SPN_TENANT_ID "${{ secrets.SPN_TENANT_ID }}" \
              --arg TF_BACKEND_STORAGE_KEY "${{ secrets.TF_BACKEND_STORAGE_KEY }}" \
              '{
                SPN_CERT: $SPN_CERT,
                SPN_CERT_PASSWORD: $SPN_CERT_PASSWORD,
                SPN_CERT_PFX: $SPN_CERT_PFX,
                SPN_CLIENT_ID: $SPN_CLIENT_ID,
                SPN_SUBSCRIPTION_ID: $SPN_SUBSCRIPTION_ID,
                SPN_TENANT_ID: $SPN_TENANT_ID,
                TF_BACKEND_STORAGE_KEY: $TF_BACKEND_STORAGE_KEY
              }' > secrets_dev.json
          else
            jq -n \
              --arg RESOURCE_GROUP "${{ secrets.RESOURCE_GROUP }}" \
              '{
                RESOURCE_GROUP: $RESOURCE_GROUP
              }' > secrets_uat.json
          fi
Here is the quick snippet after migrating secrets from one org to another.


Reference Links:
These are some of the reference articles which helped me creating this entire solution.

Encrypting secrets for the REST API - GitHub Docs
In order to create or update a secret with the REST API, you must encrypt the value of the secret.
docs.github.com

How to encrypt repository secret for Github Action Secrets API
To use the Github Action Secrets API for creating repository secret, we have to encrypt the secret key using sodium…
stackoverflow.com

GitHub API secret encryption with libsodium in Node.js: UnhandledPromiseRejectionWarning: Error…
I want to set a repository secret via the GitHub REST API. I use the example from the docs: const sodium =…
stackoverflow.com

GitHub - PSModule/Sodium: A PowerShell module for handling Sodium encrypted secrets.
A PowerShell module for handling Sodium encrypted secrets. - PSModule/Sodium
github.com

REST API endpoints for GitHub Actions Secrets - GitHub Docs
Use the REST API to interact with secrets in GitHub Actions.
docs.github.com

REST API endpoints for GitHub Actions Secrets - GitHub Docs
Use the REST API to interact with secrets in GitHub Actions.
docs.github.com

How to retrieve a secret where the secret variable name is stored inside another variable ? ·…
Select Topic Area Not able to derive a Secret value , when the name of the Secret variable is dynamic and not static…
github.com

Trying to fetch secret names and want to add in my github workflow dynamically · community ·…
Select Topic Area Question Body I am trying to fetch my environment secret names using gh api and then wants to pass in…
github.com

jq: error: test1/0 is not defined at , line 1
I have below JSON file and facing error when trying to add values to array dynamically in the shell. Below is a…
stackoverflow.com

1





Prashanth Kumar
Written by Prashanth Kumar
164 Followers
·
3 Following
IT professional with 20+ years experience, feel free to contact me at: Prashanth.kumar.ms@outlook.com

Edit profile
No responses yet

Prashanth Kumar
Prashanth Kumar
﻿

Cancel
Respond

Also publish to my profile

More from Prashanth Kumar
Azure AppInsights- Extracting Application Insight data using PowerShell & REST API
Prashanth Kumar
Prashanth Kumar

Azure AppInsights- Extracting Application Insight data using PowerShell & REST API
Introduction:
Feb 12, 2024
50


Azure Databricks — Multiple Asset Bundle Deployment and Runs
Prashanth Kumar
Prashanth Kumar

Azure Databricks — Multiple Asset Bundle Deployment and Runs
Introduction to Azure Databricks Asset Bundles
Jul 10, 2024
9


Azure- How to move/transfer files from Azure Files Shares to Azure Blob Storage
Prashanth Kumar
Prashanth Kumar

Azure- How to move/transfer files from Azure Files Shares to Azure Blob Storage
Problem statement:
Aug 13, 2023
16


Azure Databricks — Different ways to achieve Platform level monitoring
Prashanth Kumar
Prashanth Kumar

Azure Databricks — Different ways to achieve Platform level monitoring
Here I will be talking about my experiences with Azure Databricks Monitoring. One of the common element for any backend support teams is…
Jan 13, 2024
12
1


See all from Prashanth Kumar
Recommended from Medium
End-to-End DevSecOps CI/CD Pipeline: Automating Kubernetes Deployment with GitHub Actions & ArgoCD
Neamul Kabir Emon
Neamul Kabir Emon

End-to-End DevSecOps CI/CD Pipeline: Automating Kubernetes Deployment with GitHub Actions & ArgoCD
Introduction
Mar 18
7
2


Visualizing Infrastructure as Code with Diagrams by Mingrammer
T3CH
In

T3CH

by

Maciej

Visualizing Infrastructure as Code with Diagrams by Mingrammer
Infrastructure as Code (IaC) has become a standard practice in cloud and DevOps environments. However, maintaining clear and up-to-date…

Mar 15
71


Mastering EKS Scaling with Karpenter: A Practical Guide
Diatom Labs
In

Diatom Labs

by

Altin Ukshini

Mastering EKS Scaling with Karpenter: A Practical Guide
Karpenter empowers you to scale faster, cut costs, and simplify node management, a must-learn tool for Kubernetes administrators
6d ago
138
2


SSH Like a Boss: Why Remember Hosts When Your Terminal Can Do It for You?
devsecops-community
In

devsecops-community

by

Karthick Dkk

SSH Like a Boss: Why Remember Hosts When Your Terminal Can Do It for You?
Stop Typing, Start Tab-Completing! Your SSH Life Just Got Easier!

Mar 17
261
7


Advanced Kubernetes Pod Concepts Every DevOps Engineer Should Know
Harold Finch
Harold Finch

Advanced Kubernetes Pod Concepts Every DevOps Engineer Should Know
Kubernetes Pods are the fundamental execution unit in Kubernetes. While basic Pod usage is straightforward, advanced concepts allow DevOps…
5d ago
68


Building a DevSecOps Project: Automating Secure Software Delivery with Observability & Automation
@Harsh
@Harsh

Building a DevSecOps Project: Automating Secure Software Delivery with Observability & Automation
In this blog, we’ll take a deep dive into implementing a DevSecOps pipeline integrated with GitOps principles, enhanced security scanning…
Mar 3
29


See more recommendations
Help

Status

About

Careers

Press

Blog

Privacy

Terms

Text to speech

Teams
