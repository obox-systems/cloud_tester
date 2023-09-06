# Run 3d tests on GPU remote instance

The CI/CD providers provide base infrastructure for testing and it is enough for the 2d/3d web testing. But some browser WEBGL 3d test cases require GPU devices and you need to setup the environment to run the tests on remote machine.

To run such tests you need two components:
- a vps/cloud provider with GPU instances
- self hosted runners provided by CI/CD service

### Remote instance setup

Steps:
- choose a vps/cloud provider
- create instance/vps
- update software
- install drivers
- setup graphical environment

My experience:
- we decided to use AWS EC2 service and `g4ad.xlarge` instance as the cheapest gpu machine.
- we choose a [Amazon Linux 2 AMI with AMD Radeon Pro Driver](https://aws.amazon.com/marketplace/pp/prodview-h2zpfhnkvdiko?sr=0-2&ref_=beagle&applicationId=AWSMPContessa) and created an instance using the AMI. It allows us to skip step `install drivers` because the AMI has installed drivers.
- I connected to the instance using ssh and updated software.
- as I said above, we skiped the step with drivers. Anyway, to install the proper drivers use official instruction or the other manuals. For example, the AWS [has own tutorials](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/accelerated-computing-instances.html).
- finally I installed Xorg server with the proper extensions. It is important to know the Xorg display server number because the 3d tests should use it. The default value is `0` and I used it in workflow setup.

### Self hosted runner and workflow setup

Steps:
- setup self hosted runner using official tutorial
- setup workflow to use self hosted runner
- optimize workflow if it is required

My experience:
- we use `CircleCI` self hosted runners. To setup it I used [the instruction](https://circleci.com/docs/runner-installation-linux/).
- the instruction contains information how to [setup workflow](https://circleci.com/docs/runner-installation-linux/#machine-runner-configuration-example). The additional settings that we added in the workflow are the graphical settings:
```yaml
      - run:
          name: Setup environments
          command: |
            echo 'export DISPLAY=:0' >> "$BASH_ENV"
```
The command says that the current ssh session between CircleCI server and self hosted runner uses the display `0`. And the graphical applications that will be opened in the session should use display `0`. It is important to use the display with proper graphical drivers.
- we required optimization of the workflow because we pay for the instance in running state and it costs a lot of money. So we use the scheme:
  - start the instance
```yaml
  run_instance:
    executor: aws-cli/default # executor that have aws cli utility
    steps:
      - aws-cli/setup:
          profile_name: default
      - run:
          name: "Setup credentials" # setup AWS credentials to use utility
          command: |
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
      - run:
          name: "Start server" # start instance with no exceptions in 180 sec or fail
          command: |
            echo -ne "delay=180
            delta=8
            while true; do
                aws ec2 start-instances \
                    --instance-ids [id] \ # replace id of instance
                    --region us-east-1    # use valid region
                if [[ \$? == 0 ]]; then
                    break
                fi
                delay=\$((\$delay-\$delta))
                if [[ \$delay == 0 ]] ; then
                    echo Cannot start the instance.
                    exit 1
                fi
                echo Server is not started. Waiting...
                sleep \$delta
            done" | bash
      - run:
          name: "Wait for instance" # wait for running state, it requires for the testing
          command: |
            echo -ne "while true; do
                status=\$(aws ec2 describe-instances --instance-ids [id] --region us-east-1 |\ # replace data
                    jq --raw-output .Reservations[0].Instances[0].State.Name)
                if [[ \$status == 'running' ]]; then
                    break
                fi
                echo Server is \$status. Waiting...
                sleep 8
            done" | bash
      - run:
          name: "Cancel extra workflows" # each branch should have single active run
          command: |
            echo -ne "output=\$(curl -s --request GET --header \"Circle-Token: \$CIRCLECI_TOKEN\" --url https://circleci.com/api/v2/project/github/[owner]/[repo]/pipeline?branch=\$CIRCLE_BRANCH) # replace data

              for i in {0..2}
              do
                  id=\$(echo \$output | jq -r .items[\$i].id)
                  workflows=\$(curl -s --request GET --header \"Circle-Token: \$CIRCLECI_TOKEN\" --url https://circleci.com/api/v2/pipeline/\"\$id\"/workflow)
                  len=\$(echo \$workflows | jq '.items | length')

                  j=0
                  while [[ \$j < \$len ]] ;
                  do
                      name=\$(echo \$workflows | jq -r .items[\$j].name)
                      if [[ \$name == [workflow name] ]] ; then           # replace workflow name
                          id=\$(echo \$workflows | jq -r .items[\$j].id)
                          if [[ \$id != \$CIRCLE_WORKFLOW_ID ]] ; then
                              status=\$(echo \$workflows | jq -r .items[\$j].status)
                              if [[ \$status == 'running' ]] ; then
                                  echo Workflow \"\$id\" in the running state. Cancelling...
                                  curl --request POST \
                                    --url https://circleci.com/api/v2/workflow/\$id/cancel \
                                    --header \"Circle-Token: \$CIRCLECI_TOKEN\"
                              fi
                          fi
                      fi
                      j=\$((\$j+1))
                  done
              done" | bash
```
  - run tests. The standard run on self hosted runner.
  - stop instance
```yaml
      - run:
          name: "Stop server" # Stop the server to decrease cost
          when: always
          command: |
            echo -ne "output=\$(curl -s --request GET --header \"Circle-Token: \$CIRCLECI_TOKEN\" --url 'https://circleci.com/api/v2/project/github/[owner]/[repo]/pipeline') # replace data

              for i in {0..9}
              do
                  id=\$(echo \$output | jq -r .items[\$i].id)
                  workflows=\$(curl -s --request GET --header \"Circle-Token: \$CIRCLECI_TOKEN\" --url https://circleci.com/api/v2/pipeline/\"\$id\"/workflow)
                  len=\$(echo \$workflows | jq '.items | length')

                  j=0
                  while [[ \$j < \$len ]] ;
                  do
                      name=\$(echo \$workflows | jq -r .items[\$j].name)
                      if [[ \$name == [workflow name] ]] ; then           # replace workflow name
                          id=\$(echo \$workflows | jq -r .items[\$j].id)
                          if [[ \$id != \$CIRCLE_WORKFLOW_ID ]] ; then
                              status=\$(echo \$workflows | jq -r .items[\$j].status)
                              if [[ \$status == 'running' ]] ; then
                                  echo Another workflow in the running state. Exiting... # skip stopping if an active waiting run exists
                                  exit 0
                              fi
                          fi
                      fi
                      j=\$((\$j+1))
                  done
              done

              echo Stopping instance...
              aws ec2 stop-instances \
                  --instance-ids [id] \ # replace instance id
                  --region us-east-1" | bash # replace region
```
