# hashicorp_vault-jenkin-example
how to connect between hashicorp vault to jenkins


# Prerequisite
 - Operating system: `Windows 11 Home Single Language/23H2`
 
 - Suggestion using Postman for API call. Install url: https://www.postman.com/downloads/
 
 - Install jenkin in Docker:  https://www.jenkins.io/doc/book/installing/docker/
    useful command:  
	```
	 docker run -d --name=dev-vault  -e 'SKIP_SETCAP=true' -e 'VAULT_DEV_ROOT_TOKEN_ID=linhpham' -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' -p 8200:8200 -v //f/docker/containers-data/hashicorp-vault/:/vault hashicorp/vault 
    ```
	
 - Install hashicorp vault in Docker:  https://hub.docker.com/r/hashicorp/vault
                                       https://developer.hashicorp.com/vault/docs/configuration
	useful command:	
    ```
	docker run --name jenkins-blueocean --restart=on-failure --detach ^
	  --network jenkins --env DOCKER_HOST=tcp://docker:2376 ^
	  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 ^
	  --volume //f/docker/containers-data/jenkin-local/jenkin-data:/var/jenkins_home ^
	  --volume //f/docker/containers-data/jenkin-local/jenkin-cert:/certs/client:ro ^
	  --publish 8080:8080 --publish 50000:50000 myjenkins-blueocean:2.452.2-1
    ```	
 
 - Add jekin and vault to be the same network:
	```
	 docker network create jenkin-vault
	 docker network connect jenkin-vault container-name-jenkin
	 docker network connect jenkin-vault container-name-hashicorp-vault
	```
	inspect network to check the detail
	 ```
	 docker inspect jenkin-vault
	 ```
 
# Setup

 ### I) Setup approle for jenkin in hashicorp vault
	Follow these links: 
        https://developer.hashicorp.com/vault/docs/auth/approle 
						
	    https://developer.hashicorp.com/vault/api-docs/auth/approle 
	
	    https://developer.hashicorp.com/vault/api-docs 
						
	<br /> 					
						
	this is the summary after reading follow step by step: <br /> 
 
   1. Login http://localhost:8200/ui/vault/ with root token in cmd
   
   2. Go go `Secrets Engines` then click `Enable new engine` button, choose `KV` type and in `Path` input choose `jenkin-vault` label.
   
   3. In `Secrets Engines`  choose `jenkin-vault` Click button `Create secret` button then select `Path`, for example in my demo is `credentials`.
   
   4. In `Secrets Data` choose secret key and secret value. In my demo i create <br/> 
		secret key:   `GIT_URL` and  <br/> 
		secret value: `https://github.com/NguyenNgan14020323/demo-spring-boot`
		
   5. Go back to vault home then select `Authentication Methods` tab then click button `Enable new method`, in this case we choose `AppRole`
   
   6. In `Enable an Authentication Method` page content, choose `Path` input, in my case, i left it as default `approle` then click `Enable method`.<br /> 
     note: you can enable through CLI by this command: ` vault auth enable approle`
		
   7. After these step, you can check in postman by call this url 
       - PATH: http://localhost:8200/v1/`<secrets engine name>`/data/`<secret path>` \ 
	 Method: GET  \
	 Add header value: X-Vault-Token=`<your root token>` \ 
		
		
	  - In my example : <br /> 
		    GET / http://localhost:8200/v1/jenkin-vault/data/credentials <br /> 
		    X-Vault-Token=`linhpham` <br /> 
			
			The response example should be like this: <br /> 
			```
			{
				"request_id": "b82f4309-89bc-f39e-b822-8a5748eac1d2",
				"lease_id": "",
				"renewable": false,
				"lease_duration": 0,
				"data": {
					"data": {
						"GIT_URL": "https://github.com/NguyenNgan14020323/demo-spring-boot"
					},
					"metadata": {
						"created_time": "2024-07-16T14:29:53.455876326Z",
						"custom_metadata": null,
						"deletion_time": "",
						"destroyed": false,
						"version": 1
					}
				},
				"wrap_info": null,
				"warnings": null,
				"auth": null,
				"mount_type": "kv"
			}
			```
		
   8. After create approle, Create a named role in the CLI (in home page select square next to user body icon): <br /> 
		Create a name role `my-role` <br /> 
		```
		vault write auth/approle/role/my-role \
		token_type=batch \
		secret_id_ttl=10000m \
		token_num_uses=10 \
		token_ttl=20000m \
		token_max_ttl=30000m \
		secret_id_num_uses=4000
		```
	   the result in CLI should be: `Success! Data written to: auth/approle/role/my-role` <br /> 
	   
   9. Fetch the RoleID of the AppRole
      - Using postman:
		
		PATH: http://localhost:8200/v1/auth/`<app role path>`/role/`<your role name>`/role-id \
		method: GET \ 
		add header value: X-Vault-Token=`<your root token>` \
	  
	    Example<br /> 
		GET / http://127.0.0.1:8200/v1/auth/approle/role/my-role/role-id <br /> 
	      X-Vault-Token=`linhpham` <br /> 
		  
		The response should be: <br /> 
		```
		{
			"request_id": "80d3c00e-2e37-9cc1-c5a8-f2a0710c9f33",
			"lease_id": "",
			"renewable": false,
			"lease_duration": 0,
			"data": {
				"role_id": "bbbffb85-b98e-3835-95d6-29487a02bee5"
			},
			"wrap_info": null,
			"warnings": null,
			"auth": null,
			"mount_type": "approle"
		}
		```
		  
		  
	  - Using CLI:
	     `vault read auth/approle/role/my-role/role-id` <br /> 
		Get this rolet_id. Example: the result should be <br /> 
		 ```
		 Key     Value                               
         role_id bbbffb85-b98e-3835-95d6-29487a02bee5
		 ```
		
   10. Get a SecretID issued against the AppRole:
	  - Using postman:
		
		PATH: http://localhost:8200/v1/auth/`<app role path>`/role/`<your role name>`/secret-id <br /> 
		method: POST<br /> 
		add header value: X-Vault-Token=`<your root token>` <br /> 
	  
	    Example<br /> 
		POST / http://127.0.0.1:8200/v1/auth/approle/role/my-role/secret-id <br /> 
	      X-Vault-Token=`linhpham` <br /> 
		  
		The response should be:
		```
		{
			"request_id": "7dda123d-e2ed-898e-cdda-85867bcde6be",
			"lease_id": "",
			"renewable": false,
			"lease_duration": 0,
			"data": {
				"secret_id": "6dd1873a-1c5c-0d6f-2b99-be40eb2cc18b",
				"secret_id_accessor": "4a9e7bac-228d-7303-e867-09b053355baf",
				"secret_id_num_uses": 4000,
				"secret_id_ttl": 600000
			},
			"wrap_info": null,
			"warnings": null,
			"auth": null,
			"mount_type": "approle"
		}
		```
		  
		  
	  - Using CLI:
	     `vault write -f auth/approle/role/my-role/secret-id` 
		Get this secret_id. Example: the result should be <br /> 
		 ```
		 Key                Value                               
		secret_id          e88a7b86-e36a-4632-d6d0-485ae79f844e
		secret_id_accessor 122b260f-5d36-167e-7160-1e28837fd6ba
		secret_id_num_uses 4000                                
		secret_id_ttl      600000  
		 ```
		 
	  - Note: each time you call this api or execute in CLI, the respone always change(not fixed 'role_id' in previous step 10)
	  
   11. After get role_id and secret_id from step 9 and 10. Next, we check the approle is success login. By using this api:
		PATH: http://localhost:8200/v1/auth/`<app role path>`/login <br /> 
		method: POST <br /> 
		request body (json): <br /> 
		```
		{
			"role_id":<your role id>,
			"secret_id":<your secret id>
		}
		```
		
		Example<br /> 
		POST / http://localhost:8200/v1/auth/approle/login <br /> 
	    request body (json): <br /> 
		```
		{
			"role_id":"bbbffb85-b98e-3835-95d6-29487a02bee5",
			"secret_id":"6dd1873a-1c5c-0d6f-2b99-be40eb2cc18b"
    
		}
		```
		  
		The response should be: <br /> 
		```
		{
			"request_id": "586c72ef-47e5-497a-c71a-34a4d1fe7acf",
			"lease_id": "",
			"renewable": false,
			"lease_duration": 0,
			"data": null,
			"wrap_info": null,
			"warnings": null,
			"auth": {
				"client_token": "hvb.AAAAAQL9ah4y-5HClqHFUSlT9iBgjhfWYDXvELqj08AnWSG6Oc5gGVWdR2XAUltpuV1dO-xwjGeSb1n3L6vnCs91czpxIZeDISQKggRStK3CYDzJ-NKOEjJbJl5_pbhZDVpOVu-wrQWAb4LzH4MAD4YppWM1Bh54baQ73mbJ91x4cpFumAsKaQcV_m4dwrr0YXzTkOKjIoqJqI6X0PmLkh_HiHaZqHu-Tj4",
				"accessor": "",
				"policies": [
					"default",
					"jenkin-vault"
				],
				"token_policies": [
					"default",
					"jenkin-vault"
				],
				"metadata": {
					"role_name": "my-role"
				},
				"lease_duration": 1200000,
				"renewable": false,
				"entity_id": "9e352406-4539-426f-b171-d139683e8eeb",
				"token_type": "batch",
				"orphan": true,
				"mfa_requirement": null,
				"num_uses": 0
			},
			"mount_type": ""
		}
		```
		
		Note: this api will generate `client_token` and use for get secret password in plugins or dependencies.
		
   12. Create policies for `Secrets Engines`. Go to home then select `Policies` tab (or at 'http://localhost:8200/ui/vault/policies/acl'). <br /> 
		Then click button `Create ACL policy +` => select name, ex: `jenkin-vaults`. In policy input, add policy for your paths. <br /> 
		In my example, it should be:
		```
			path "jenkin-vault/*" {
				capabilities = ["create","read", "update", "delete", "list"]
				min_wrapping_ttl="100000s"
				max_wrapping_ttl="900000s"
			}

			path "jenkin-vault/data/credentials" {
				capabilities = ["read", "list"]
			}
		```
   
   13. After create your policies in step 12. Next, assign this policy to your approle using postman
	 PATH: http://localhost:8200/v1/auth/`<app role path>`/`<your role name>` <br /> 
		method: POST <br /> 
		request body (json): <br /> 
		```
		{"policies": "<your policy>", "token_type": "<approle token type>"}
		```
		
		Example<br /> 
		POST / http://localhost:8200/v1/auth/approle/login <br /> 
	    request body (json):  <br /> 
		```
		{"policies": "jenkin-vault", "token_type": "batch"}
		```
		  
		The response should be:<br /> 
		```
			204 No Content
		```
		
   14. You can check the polices is success add or not by check it in step 11.
    The response should contains this attributes: <br /> 
	```
		"policies": [
			"default",
			"jenkin-vault"
		]
	```
   
   15. Check api list in: http://localhost:8200/ui/vault/tools/api-explorer
   
   16. Check my postman apis in my example in : `hashicorp-vault.postman_collection.json`
   
 ### II) Setup jekin with vault
 
   1. Install hashicorp vault plugin: https://plugins.jenkins.io/hashicorp-vault-plugin/
   
      - Using the GUI: From your Jenkins dashboard navigate to `Manage Jenkins > Manage Plugins` and select the `Available` tab.<br />  
      Locate this plugin by searching for 'hashicorp-vault-plugin'.
	  
	  - Using the CLI tool:
		`jenkins-plugin-cli --plugins hashicorp-vault-plugin:368.v48134f694db_f`
		
   2. Go to `Dashboard > Manage Jenkins > System` scroll to tab `Vault Plugin`
     - Specific `Vault URL`. Ex: http://dev-vault:8200
	 
     - Add `Vault Credential`, Under `Kind` input select `Vault App Role Credential`
	 
	 - Add `Role ID` in vault. Ex: `bbbffb85-b98e-3835-95d6-29487a02bee5`
	 
	 - Add `Secret ID` in vault. Ex: `6dd1873a-1c5c-0d6f-2b99-be40eb2cc18b`
	 
	 - Add `Path`, `<your app role path>`. Ex: `approle` or keep it as default in my example
	 
	 - In `ID`, choose an ID. Ex: `linhvan20202`
	 
	 - For more details, check this link: https://blog.nashtechglobal.com/integrating-jenkins-hashicorp-vault/
	 
   3. Create Jenkin pipeline with name `Jenkin-vault`
   
   4. Choose config and then add this pipeline script:
   
	```
		def configuration = [vaultUrl: 'http://dev-vault:8200', 
							 vaultCredentialId: 'linhvan20202', 
							 engineVersion: 2]

		def secrets = [
		  [  path: 'jenkin-vault/credentials', engineVersion: 2, 
			 secretValues: [[envVar: 'GIT_URL', vaultKey: 'GIT_URL']]
		  ]
		]


		pipeline {
			agent any

			tools {
				// Install the Maven version configured as "M3" and add it to the path.
				maven "mavn"
			}

			stages {
				
				stage('Vault') {
					steps {
					  withVault([configuration:configuration, vaultSecrets: secrets]) {
						sh "echo ${env.GIT_URL}"
						sh "GIT=${env.GIT_URL}"
					  }
					}  
			  }
				
				stage('Build') {
					steps {
						// Get some code from a GitHub repository
						withVault([configuration:configuration, vaultSecrets: secrets]) {
						   //  sh "echo ${env.GIT_URL}"
						  //  git 'GIT'
							sh "git pull ${env.GIT_URL}"
						//  checkout([$class: 'GitSCM', branches: [[name: '*/master']],
						//            userRemoteConfigs: [[url: '${env.GIT_URL}']]])
						}

						// Run Maven on a Unix agent.
						sh "mvn -Dmaven.test.failure.ignore=true clean package"

						// To run Maven on a Windows agent, use
						// bat "mvn -Dmaven.test.failure.ignore=true clean package"
					}

					post {
						// If Maven was able to run the tests, even if some of the test
						// failed, record the test results and archive the jar file.
						success {
							junit '**/target/surefire-reports/TEST-*.xml'
							archiveArtifacts 'target/*.jar'
						}
					}
				}
			}
		}
	```
   
   6. Build and check the result. Here is the example after run in my demo
   ```
	Started by user phamlinh1995
	[Pipeline] Start of Pipeline
	[Pipeline] node
	Running on Jenkins in /var/jenkins_home/workspace/jenkin-vault
	[Pipeline] {
	[Pipeline] stage
	[Pipeline] { (Declarative: Tool Install)
	[Pipeline] tool
	[Pipeline] envVarsForTool
	[Pipeline] }
	[Pipeline] // stage
	[Pipeline] withEnv
	[Pipeline] {
	[Pipeline] stage
	[Pipeline] { (Vault)
	[Pipeline] tool
	[Pipeline] envVarsForTool
	[Pipeline] withEnv
	[Pipeline] {
	[Pipeline] withVault
	Retrieving secret: jenkin-vault/credentials
	[Pipeline] {
	[Pipeline] sh
	Warning: A secret was passed to "sh" using Groovy String interpolation, which is insecure.
			 Affected argument(s) used the following variable(s): [GIT_URL]
			 See https://jenkins.io/redirect/groovy-string-interpolation for details.
	+ echo ****
	****
	[Pipeline] sh
	Warning: A secret was passed to "sh" using Groovy String interpolation, which is insecure.
			 Affected argument(s) used the following variable(s): [GIT_URL]
			 See https://jenkins.io/redirect/groovy-string-interpolation for details.
	+ GIT=****
	[Pipeline] }
	[Pipeline] // withVault
	[Pipeline] }
	[Pipeline] // withEnv
	[Pipeline] }
	[Pipeline] // stage
	[Pipeline] stage
	[Pipeline] { (Build)
	[Pipeline] tool
	[Pipeline] envVarsForTool
	[Pipeline] withEnv
	[Pipeline] {
	[Pipeline] withVault
	Retrieving secret: jenkin-vault/credentials
	[Pipeline] {
	[Pipeline] sh
	Warning: A secret was passed to "sh" using Groovy String interpolation, which is insecure.
			 Affected argument(s) used the following variable(s): [GIT_URL]
			 See https://jenkins.io/redirect/groovy-string-interpolation for details.
	+ git pull ****
	From ****
	 * branch            HEAD       -> FETCH_HEAD
	Already up to date.
	[Pipeline] }
	[Pipeline] // withVault
	[Pipeline] sh
	+ mvn -Dmaven.test.failure.ignore=true clean package
	[INFO] Scanning for projects...
	[INFO] 
	[INFO] --------------------------< com.example:demo >--------------------------
	[INFO] Building demo 0.0.1-SNAPSHOT
	[INFO]   from pom.xml
	[INFO] --------------------------------[ jar ]---------------------------------
	[WARNING] The artifact mysql:mysql-connector-java:jar:8.0.31 has been relocated to com.mysql:mysql-connector-j:jar:8.0.31: MySQL Connector/J artifacts moved to reverse-DNS compliant Maven 2+ coordinates.
	[INFO] 
	...
   ```
   
 
# Note: please check the demo and check with above steps, you can easy config your example.

# Documents reference

 - https://hub.docker.com/r/hashicorp/vault
 
 - https://www.jenkins.io/doc/book/installing/docker/
 
 - https://octopus.com/integrations/hashicorp-vault/hashicorp-vault-key-value-v2-retrieve-secrets
 
 - https://www.ru-rocker.com/2019/09/27/how-to-isolate-database-credentials-in-spring-boot-using-vault/
 
 - https://blog.nashtechglobal.com/integrating-jenkins-hashicorp-vault/
 
 - https://stackoverflow.com/questions/50721424/how-to-add-containers-to-same-network-in-docker
 
 - https://github.com/BetterCloud/vault-java-driver/blob/master/src/main/java/com/bettercloud/vault/api/Auth.java#L504 => key: loginByAppRole
 
 - https://github.com/jenkinsci/hashicorp-vault-plugin/blob/master/src/main/java/com/datapipe/jenkins/vault/VaultAccessor.java => key: com.datapipe.jenkins.vault.exception.VaultPluginException: Access denied to Vault path 'jenkin-vault/credentials'
 
 - https://www.jenkins.io/doc/pipeline/steps/git/
 
 - https://superuser.com/questions/1732641/how-to-use-git-pull-with-jenkins-sh-steps
 
 - https://plugins.jenkins.io/hashicorp-vault-plugin/
 
 - https://developer.hashicorp.com/vault/api-docs/auth/approle
 
 - https://developer.hashicorp.com/vault/api-docs
 
 - https://stackoverflow.com/questions/40213524/using-absolute-path-with-docker-run-command-not-working
 
