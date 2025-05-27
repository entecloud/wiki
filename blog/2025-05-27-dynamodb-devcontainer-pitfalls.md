---
title: Using AWS DynamoDB for local development  
description: A quick gude to set up your own DevContainer having DynamoDB locally to help the development a bit faster.
slug: dynamodb-devcontainer
authors: [giri]
tags: [aws, dynamodb, devcontainer]
---

Having a local DynamoDB service will help to speed up the development process  

<!-- truncate -->

## Getting Ready with DevContainers

Before DevContainers, I was using [Nix package manager](https://nixos.org/download/) combined with [direnv](https://direnv.net/) for the development dependency management. I still love the Nix philosophy, but the steeper learning curve is a bit of a trade-off to be honest. I've spent more time configuring the perfect reproducible dev environment rather than contributing to the actual project. Never tired of it, but then, sometimes it's a pain. 

That's when I switched to [DevContainers](https://containers.dev/). Looks fun, no need to learn any extra language, and this too guarantee the reproducibility. On top of that, customized VSCode environment is also a plus. I think DevContainer will be my first preference to set up dev env. So make sure you've [VSCode](https://code.visualstudio.com/download) + [DevContainers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) installed before moving further

## DevContainers Configuration  

To configure a devcontainers in VSCode all you need is a `.devcontainer/devcontainer.json` file. You can find the detailed spec of the JSON [here](https://containers.dev/implementors/json_reference/). 

Since we need to spin multiple services, one for the DynamoDB and the other for your code (API, script or whatever), we'll be going forward with the `docker-compose.yaml` route. 

The compose file would look like this:  

File: `.devcontainer/docker-compose.yaml`  

```yaml
services:
  code:
    image: python:3.12-bullseye
    tty: true
    volumes:
      - ../:/workspace:cached
    depends_on:
      - dynamodb
  dynamodb:
    command: "-jar DynamoDBLocal.jar -sharedDb -dbPath ./data"
    image: "amazon/dynamodb-local:latest"
    ports:
      - "8000:8000"
    volumes:
      - "./data/dynamodb/:/home/dynamodblocal/data"
    working_dir: /home/dynamodblocal
```

Pretty much self explanatory. It exposes two `services`.  
- `code` (you can replace this with your own source container, this is just for example)
  - Which is built on `python 3.12 bullseye` image
  - Kept the `tty: true` to not to get killed. You can also use `command: sleep infinity`, which essentially does the same
  - Mounts one directory UP to the `/workspace` folder in the container. This is highly opinionated, I like to keep the app code under `/workspace`.
  - Depends on the `dynamodb` service.
- `dynamodb`
  - Built on official AWS DynamoDB image
  - Runs the JAR file on startup, which uses the `data` folder as the DB path
  - Exposes the port 8000 to host's 8000
  - Mounts the volume `./data/dynamodb` (this is inside the `.devcontainer` folder) to the the `/home/dynamodb/data` (The same `data` folder we pointed as the DB Path)
  - Sets the working directory to `/home/dynamodblocal`

File: `.devcontainer/devcontainer.json`

```json
{
  "name": "api",
  "dockerComposeFile": "docker-compose.yaml",
  "service": "code",
  "workspaceFolder": "/workspace"
}
```

This is a bery basic devcontainer. There's a LOT you can configure. But to keep this doc minimum, we're going with the bare minimum. As per the spec we've defined above, it:  

- Initializes a new DevContainer with the name of `api`
- Uses `docker-compose.yaml` to spin the environment
- The main service is `code` that is, it's the main container that VSCode will connect to.
- Sets the `/workspace` as the worksapce volume (required when we use docker compose)

That's it!. Now when you open this folder in VSCode, you'll see a small pop up at the bottom right asking to Re-Open the workspace in DevContainers. Click Open, and wait for few minutes to get the containers up and running.  

:::warning[READ THIS BEFORE BOOTING DEVCONTAINER]

Since the `docker` runs on `root`, you've to **make sure the mounting volume path (here, it's the `.devcontainer/data/dynamodb`) is created first before we boot the containers**. Otherwise it'll be created under `root` and since the DynamoDB container runs on a normal user `dynamodblocal` it won't have RW access to that path causing unexpected errors. I learnt this in the hard way, almost spent two days to figure out what was going on.

You can leverage the `initializeCommand` option in `devcontainer.json` to create the folder. So that no one forgets.

:::

## Connecting to Local Instance  

Now that you've logged into the `code` container which has python installed, let's try connecting via a python script. 

Make sure to install `boto3` before running the following script, inside the container.  

Also, don't forget to export AWS creds to the environment. You can use some garbage values here since it's not connecting to real AWS instance.  

```env
export AWS_ACCESS_KEY_ID=fake
export AWS_SECRET_ACCESS_KEY=fake
export AWS_DEFAULT_REGION=us-west-2
```

```python
import boto3

ddbclient = boto3.client("dynamodb", endpoint_url="http://dynamodb:8000")
tables = ddbclient.list_tables()

print(f"Found {len(tables)} tables in DynamoDB")
```

