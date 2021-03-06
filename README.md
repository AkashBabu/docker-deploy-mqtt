# docker-deploy-mqtt
This library is inspired by [docker-deploy-webhook](https://github.com/iaincollins/docker-deploy-webhook).  
(Please read about MQTT protocol before proceeding with the rest of the document)  

MQTT Hook for automated deployment of images to docker-swarm on any machine(Irrespective of global/static/dynamic/local IP).

<!-- Here is a [blog]() about how to integrate this library with gitlab-ci -->

# Docs

## Environment Variables
**CONFIG** (*Defaults to 'production'*)  
Information about which configuration to pick from ddm_config.json (Sample file described below)

## Docker-Config
**broker_url**  
MQTT Broker Url. Prescribed to use publicly available brokers([List](https://github.com/mqtt/mqtt.github.io/wiki/public_brokers) of public MQTT brokers)

**registry**  
Docker registry name (Ex. registry.gitlab.com)

**deployments**
```JSON
"deployments": {
    "<environment>": {
        "<image-name>:<image-tag>": {
            "service": "<service-name>"
        }
    }
}
```
In here you can specify different environments according to your needs. Each environment can have any number of images configured to be deployed onto the docker-swarm. 

* **\<environment\>**  
    It's used to distinguish between different types of deployment preferences(like production/staging/testing etc). For instance one may choose to deploy only `latest` versions on `production`, while deploy every single image on staging environment.
* **\<image-name\>:\<image-tag\>**  
    `<image-name>` corresponds to the docker image-name to be deployed and `<image-tag>` is the docker image-tag. `<image-tag>` also supports pattern matching (*`<image-tag>` defaults to latest*)  
    Hence all the below patterns are valid
    * `helloworld` *(By default tag is `latest`)*
    * `helloworld:*`
    * `helloworld:test_*`
    * `helloworld:latest`
    * `helloworld:!ignored` *(This corresponds to all images except `ignored` tag)*  
    For more details of supported patterns visit [matcher](https://www.npmjs.com/package/matcher)
* **\<service-name\>**  
    Docker-swarm service name whose image is intended to be updated.

## Docker-Secrets
* **ddm_token**  
    Secret token that will be used to form MQTT Topic during publish and subscribe

* **docker_user**  
    Docker login - username

* **docker_pass**  
    Docker login - password

## MQTT-Topic to publish
docker-deploy-mqtt would subscribe to `/docker-deploy-mqtt/${ddm_token}` topic. So your CI/CD design must include a step to publish on the same MQTT Topic with the message in the following format:
```JSON
{
    "name"  : "<image-name>(Without Tag)",
    "tag"   : "<image-tag>(Defaults to latest)"
}
``` 
*Since a VARIABLE can be resolved only within `"`(Double quotes), for convenience of using VARIABLES in MQTT message support is also added for using `'` instead of `"`, so that MQTT message can look like `"{'name': '$PROJECT_NAME', 'tag': '$CI_COMMIT_REF_NAME'}"`*


# Docker Image
Get the image from [here](https://hub.docker.com/r/akashbabu256/docker-deploy-mqtt/) or build it yourself by cloning this repo and run:
> docker build -t docker-deploy-mqtt .


# Usage
1. First create a ddm_config.json file as below:
    ```JSON
    {
        "broker_url": "mqtt://test.mosquitto.org",
        "registry": "registry.gitlab.com",
        "deployments": {
            "production": {
                "username/helloworld:latest": {
                    "service": "helloworld"
                }
            },
            "staging": {
                "username/helloworld:*": {
                    "service": "helloworld"
                }
            }
        }
    }
    ```
    Then create docker config from the above file:
    > docker config create ddm_config ddm_config.json

2. create a ddm_token.txt (Secret Token for MQTT Hook): 
    ```txt
    secret-token
    ```  
    Then create docker secret from ddm_token.txt file:
    > docker secret create ddm_token ddm_token.txt

3. create a docker_user.txt (Docker registry username):  
    ```text
    docker-user
    ```  
    Then create docker secret from docker_user.txt file:
    > docker secret create docker_user docker_user.txt

4. create a docker_pass.txt (Docker registry password):
    ```text
    docker-password
    ```  
    Then create docker secret from docker_pass.txt file:
    > docker secret create docker_pass docker_pass.txt

5. Last step is to deploy the service in docker-swarm and add the constraint to run it only on a manager-node:
    > docker service create \\  
    --config src=ddm_config,target=/docker-deploy-mqtt/ddm_config.json \\  
    --secret src=ddm_token,target=/docker-deploy-mqtt/ddm_token.txt \\  
    --secret src=docker_user,target=/docker-deploy-mqtt/ddm_user.txt \\  
    --secret src=docker_pass,target=/docker-deploy-mqtt/ddm_pass.txt \\  
    --constraint "node.role==manager" \\  
    --name docker-deploy-mqtt \\  
    -e CONFIG=production \\  
    --mount type=bind,src=/var/run/docker.sock,target=/var/run/docker.sock \\  
    docker-deploy-mqtt:latest



## Caveats
* Do not include (+,#) in ddm_tokens
* Be careful when wrapping CI/CD VARIABLES in `'`(single quotes) as they will be considered as strings. Hence suggested solution is to wrap the entire string in `"`(double quotes) and then use `'`(single quotes) for inner strings.

## Contributions
For contributions, please take up the tasks in Roadmap or If you find any potential improvement to this library, feel free to create a PR or raise an issue for the same. 


# FAQs

* *How to integrate with CD systems?*  
    All that you need to do is to add one line in the end of CD which is typically as below:
    >  docker run --rm aksakalli/mqtt-client pub -h test.mosquitto.org -t
      "/docker-deploy-mqtt/$DDM_TOKEN" -m "{'name': 'helloworld','tag': 'latest'}"

* *How to update config.json into the running docker-deploy-mqtt service?*  
    > docker service update --config-rm ddm_config --config-add ddm_config_updated --force docker-deploy-mqtt
   

# Roadmap
* [ ] Notification support
* [ ] Add hook support after deployment
* [ ] Design a way to check if any images has been updated on every restart