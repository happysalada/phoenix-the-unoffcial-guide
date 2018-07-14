# Continuous integration

add a `.circleci/config.yml` file to your project with the following content

```
version: 2
jobs:
  build:
    parallelism: 1
    docker:
      - image: circleci/elixir:latest-node-browsers
        environment:
          MIX_ENV: test
      - image: circleci/postgres:10-alpine-ram
        environment:
          POSTGRES_USER: postgres
          POSTGRES_DB: app_test
          POSTGRES_PASSWORD:

    working_directory: ~/app

    steps:
      - checkout

      - run: mix local.hex --force
      - run: mix local.rebar --force

      - restore_cache:
          keys:
            - v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v1-mix-cache-{{ .Branch }}
            - v1-mix-cache
      - restore_cache:
          keys:
            - v1-build-cache-{{ .Branch }}
            - v1-build-cache
      - run: mix do deps.get --force
      - run: mix do deps.compile --force
      - save_cache:
          key: v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
          paths: "deps"
      - save_cache:
          key: v1-mix-cache-{{ .Branch }}
          paths: "deps"
      - save_cache:
          key: v1-mix-cache
          paths: "deps"
      - save_cache:
          key: v1-build-cache-{{ .Branch }}
          paths: "_build"
      - save_cache:
          key: v1-build-cache
          paths: "_build"

      - run:
          name: Wait for DB
          command: dockerize -wait tcp://localhost:5432 -timeout 1m

      - run: mix test

      - store_test_results:
          path: _build/test/junit
```

push to master, then go to circleci.com to create an account.

Each build takes ~ 4mins (with tests that run in <10s), that gives you 375 commits for free.

# Continuous delivery

to set up continuous delivery you need add an SSH key to circleci and add that key on the deployment machine

- [add ssh key to cci](https://circleci.com/docs/2.0/add-ssh-key/)
- add ssh key to your deployment machine
  `cat ~/.ssh/id_rsa.pub | ssh root@<YOUR_IP> 'cat - >> ~/.ssh/authorized_keys'`

now add the following at the bottom of your config.yml file

```
deploy:
    docker:
      - image: docker:18.05.0-ce-git
    steps:
      - checkout
      - setup_remote_docker
      - run:
          name: Build and push Docker image
          command: |
            if [[ $CIRCLE_TAG ]]; then
              TAG=$CIRCLE_TAG
              APP_VERSION=$TAG
            else
              TAG="${CIRCLE_BRANCH}-${CIRCLE_BUILD_NUM}-${CIRCLE_SHA1}"
              APP_VERSION="0.0.0-${CIRCLE_BRANCH}.${CIRCLE_BUILD_NUM}+${CIRCLE_SHA1}"
            fi
            docker build -t your_docker_hub_id/your_app_name:$TAG -t your_docker_hub_id/your_app_name:latest --build-arg APP_VERSION=$APP_VERSION .
            docker login -u $DOCKER_LOGIN -p $DOCKER_PASSWORD
            docker push your_docker_hub_id/your_app_name:$TAG
            docker push your_docker_hub_id/your_app_name:latest

      - run:
          name: Deploy to built image to server
          command: |
            ssh -v -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no $DROPLET_USER@$DROPLET_IP "cd /etc/opt/your_app_name; ./circle_deployer.sh"

workflows:
  version: 2
  build-and-deploy:
    jobs:
      - build
      - deploy:
          requires:
            - build
          filters:
            branches:
              only: staging
```

replacing your_docker_hub_id with your docker hub id and your_app_name with your app name

you need to add environment variables to circleci, your docker login password, your deployment_machine IP. Basically everything starting with a $ sign, needs to be added to circle ci.

the app is placed in /etc/opt/your_app_name on the deployment machine
/etc is used for placing configuration files usually
/opt is used for optional (as in not necessary to the boot of the machine)
but you are free to place it anywhere!

with this config

- the image will only be built on the staging branches
- Other than that, images are tagged and pushed to docker hub with `latest` tag

note that my phoenix app is named

One last thing, you need to add a circle_deployer.sh on your deployment machine.
it's basically a file to stop and start the system.
If you are just using docker-compose it's as simple as

```
docker-compose down
docker-compose pull
docker-compose up
```

if you are using the systemctl primitive like advised in the docker deployment part then

```
systemctl stop docker-compose-app
docker-compose pull
systemctl start docker-compose-app
```

an example for a config file can be found [here](/deployment/config.yml)
