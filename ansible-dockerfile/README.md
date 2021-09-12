## Dockerfile used to build and run an Ansible container

### This Dockerfile installs Ansible and dependencies. It also installs boto for interfacing with AWS.

---

### Useful commands to use:

1. Login to DockerHub
   - `docker login -username <DockerHub-UserName>`
2. Build the dockerfile
   - `docker build -t <DockerHub-UserName>/ansible:latest .`
3. Push image to DockerHub
   - `docker push <DockerHub-UserName>/ansible:latest`
4. Run container with a volume mounted

- Linux
  - `docker run -it --rm --volume $(pwd)":/ansible --workdir /ansible ansible`
    - This will run the container and delete it after exiting.
    - The volume will be the curent directory mapped to the /ansible path inside the container.
    - It will start inside the /ansible directory.
- Windows using Docker Desktop
  - `docker run -it --rm --volume <path-of-directory>:/ansible --workdir /ansible <DockerHub-UserName>/ansible`
    - This command will run differently depending depending on which command line interface you are using
    - This one works for me using the cmd shell

5. Once inside you can run Ansible commands
   - `ansible-playbook <YAML-FILE>`
   - `ansible-galaxy init ec2`
   - `ansible-galaxy init k8s_master`
   - `ansible-galaxy init k8s_worker`
