# Small Docker 101 Exercise


_This exercise assumes you've got an Ubuntu VM up and running with docker installed. I used 16.04 and followed the guide here: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04 _ 

## 1. Build a Docker image

- `sudo mkdir helloworld`
- `cd helloworld; sudo touch Dockerfile`
- `sudo nano Dockerfile`

### helloworld Dockerfile

```
## Specify base image, in this case an Ubuntu container (latest version-- notice latest tag)
FROM ubuntu:latest
## Execute command on run/start
CMD echo hello world!
```

- `docker build -t helloworld:latest .`

- `docker run helloworld`

We should see a terminal output

## 2. Containerize a simple Flask application

- `git clone <this repo>`
- `cd Dockerpresentation/`

inspect few files in local repo, try to understand this new Dockerfile

Notice we are installing python, flask, blinker, and ddtrace into this container (and we've instrumented the tracing client in our code)

Note our flask app has a single route, is broadcasting to the localhost's network over port 5000. We need to publish that port to the host network explicitly in our docker run command to share networking between the container/host.

- `docker build -t flask_app:latest .`
- `docker run --name my-app -p 5000:5000 -e DD_AGENT_SERVICE_HOST=172.17.0.1 -e DD_AGENT_SERVICE_PORT=8126 flask_app`

Open another terminal session (note if you are on a VM, you will have to ssh back into it-- I recommend using a handy tool like Spectacle to ease multiple window use/placement rather than manually resizing)

Lets curl our app to make sure it is reporting
- `curl localhost:5000/`
Should return 'Flask Dockerized' to the console
At this point we should be seeing errors in our Flask container logs about Connection refused to port 8126 (the agent isn't running yet so that's ok!)

## 3. Spin up the Datadog Agent to collect traces and logs

- ```docker run -d --name dd-agent -v /var/run/docker.sock:/var/run/docker.sock:ro -v /proc/:/host/proc/:ro -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro -v /opt/datadog-agent/conf.d/etc.d/:/etc/datadog-agent/conf.d/etcd.d/:ro -e DD_e DD_LOGS_ENABLED=true -e DD_APM_ENABLED=true -e DD_APM_NON_LOCAL_TRAFFIC=true -e DD_API_KEY=<DD_API_KEY> -p 8126:8126 datadog/agent:latest  ```
curl the Flask endpoint again a few times (watch flask logs mention that traces are being sent to the agent)

navigate to Datadog APM UI and notice traces now reporting!

run our helloworld sample container a few times. the message prints to stdout/stderr so we should now be able to see Agent AND helloworld logs in the Log Management UI. Navigate to UI to confirm

## 4. Explore the agent's autodiscovery capabilities
- `docker pull redis:latest`
- `docker images`
- `docker run --name my-redis -d redis`

Navigate to your Datadog account and check the Infrastructure list to see Redis container reporting and metrics coming in.


## 5. Explore the Docker CLI

- `docker ps`
- `docker ps -a`
- `docker exec -ti dd-agent /bin/bash`

you are now inside the agent container! you can play around as if the agent were on the host. try running `agent status`

- `docker exec -ti dd-agent agent status`
- `docker stop dd-agent`
- `docker start dd-agent`
