### Ansible Roles and Setup files

1. Change any configurations in ansible.cfg

- May need to change the following depending:
  ```
  private_key_file= ./ansible.pem
  remote_user=ubuntu
  ```

2. Change any variables used in `role/vars/main.yml` (ie: `ec2/vars/main.yml`)
3. Add a `cred.yml` file with your AWS credentials
   ```
   access_key: XXXXXXXXXXXXXXXXXXXX
   secret_key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
   ```
4. Run the setup.yml
   ```
   ansible-playbook setup.yml
   ```
