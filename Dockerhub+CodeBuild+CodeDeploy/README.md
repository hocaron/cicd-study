## πCI / CD μν€νμ³
![image](https://user-images.githubusercontent.com/66551410/135451992-e6d06f81-6a88-4e36-b76b-2df2449a3f95.png)

## πμ€λͺ

### appspec.yml  
- CodeDeploy Agent κ° μνν  μμκ³Ό νκ²½μ μ€μ νλ νμΌ
- source λ CodeBuild λ‘ λΆν° μ λ¬λ°μ artifacts μ νμΌ μμ€ν μ€ κ°μ Έμ€κ³ μ νλ νμΌ
- destination μ λ°°ν¬νκ³ μ νλ νΈμ€νΈ(EC2) μ νμΌ μμ€νμμ source λ‘λΆν° κ°μ Έμ¨ artifacts κ° μμΉν  λλ ν°λ¦¬
```
version: 0.0
os: linux
files:
  - source: /
    destination: /home/ec2-user/app
hooks:
  AfterInstall:
    - location: scripts/pullDocker.sh
      timeout: 300
      runas: ec2-user
  ApplicationStart:
    - location: scripts/runDocker.sh
      timeout: 300
      runas: ec2-user
  ApplicationStop:
    - location: scripts/stopDocker.sh
      timeout: 60
      runas: ec2-user
```

### buildspec.dev.yml 
- CodeBuild κ° μμ€μ½λλ₯Ό λΉλν  λ μνν  μμκ³Ό νκ²½μ μ€μ νλ νμΌ
- phases λ CodeBuild μμ μμ λ¨κ³λ₯Ό λνλ΄λ νΉμν λ¨κ³
- artifacts λ CodeBuild κ° μμμ λ§μΉ λ€ CodeDeploy μκ² μ λ¬ν  νμΌλ€μ λͺμνλ λΆλΆμΌλ‘, μ¬κΈ°μλ appspec.yml κ³Ό docker μ»€λ§¨λκ° ν¬ν¨λ scripts/* , κ·Έλ¦¬κ³  docker-compose.dev.yml νμΌμ μ λ¬
```
version: 0.1
phases:
  pre_build:
    commands:
      - 'echo Logging in to Docker Hub...'
      - 'docker login --username="<DOCKERHUB_USERNAME>" --password="<DOCKERHUB_PASSWORD>"'
  build:
    commands:
      - 'echo Build started on `date`'
      - 'echo Building the Docker image...'
      - 'docker-compose -f docker-compose.yml build'
  post_build:
    commands:
      - 'echo Build completed on `date`'
      - 'echo Pushing the Docker image...'
      - 'docker-compose -f docker-compose.yml push'

artifacts:
  files:
    - 'appspec.yml'
    - 'scripts/*'
    - 'docker-compose.yml'
    - 'Dockerfile'
    - 'dist/*'
    - 'node_modules/*'
    - 'package.json'
    - 'package-lock.json'
    - 'buildspec.dev.yml'
    - 'nest-cli.json'
    - 'tsconfig.build.json'
    - 'tsconfig.json'
    - 'src/*'
```

### Dockerfile
- builder :μμ€μ½λλ₯Ό λ³΅μ¬ν΄μ λΉλλ₯Ό μ§ν
- runtime : λΉλλ νμΌκ³Ό λΈλ λͺ¨λλ§μ κ°μ Έμ¨ λ€ μ»€λ§¨λλ₯Ό μ€ν
```
FROM node:14.16.0-alpine3.11 as builder
WORKDIR /app

COPY package*.json ./
COPY ./tsconfig.json ./

RUN npm install

COPY . .
## compile typescript
RUN npm run build
## remove packages of devDependencies
# RUN npm prune --production
# ===================================================
FROM node:14.16.0-alpine3.11 as runtime
WORKDIR /app

# ENV NODE_ENV="development"
# ENV DOCKER_ENV="development"
# ENV PORT=5000
## Copy the necessary files form builder
COPY --from=builder "/app/dist/" "/app/dist/"
COPY --from=builder "/app/node_modules/" "/app/node_modules/"
COPY --from=builder "/app/package.json" "/app/package.json"

EXPOSE 3000
CMD ["npm", "run", "start"]
```

### docker-compose.yml
```
version: '3'
services:
  app:
    restart: always
    image: <DOCKERHUB_IMAGES>
    build:
      dockerfile: Dockerfile
      context: .
    container_name: 'app'
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - '3000:3000'

volumes:
  node_modules:
```

### stopDocker.sh
- λͺ¨λ  μ»¨νμ΄λλ₯Ό stop μν€λ λͺλ Ή
```
docker stop $(docker ps -a -q)
```

### runDocker.sh
- CodeDeploy κ° EC2 μμμ κ°μ Έμ¨ μ΄λ―Έμ§λ₯Ό μ»¨νμ΄λλ‘ λμ°λ μ μ€ν¬λ¦½νΈ
```
if [ "$DEPLOYMENT_GROUP_NAME" == "dev" ]
then
  pwd
  # Remove any anonymous volumes attached to containers
  docker-compose -f /home/ec2-user/app/docker-compose.yml rm -v 
  # build images and run containers
  docker-compose -f /home/ec2-user/app/docker-compose.yml up --detach --renew-anon-volumes
elif [ "$DEPLOYMENT_GROUP_NAME" == "stage" ]
then
  # Remove any anonymous volumes attached to containers
  docker-compose -f /deploy/docker-compose.stage.yml rm -v 
  # build images and run containers
  docker-compose -f /deploy/docker-compose.stage.yml up --detach --renew-anon-volumes
elif [ "$DEPLOYMENT_GROUP_NAME" == "production" ]
then
  # Remove any anonymous volumes attached to containers
  docker-compose -f /deploy/docker-compose.yml rm -v 
  # build images and run containers
  docker-compose -f /deploy/docker-compose.yml up --detach --renew-anon-volumes
fi
```

### pullDocker.sh
- CodeDeploy κ° EC2 μμμ μ€νν  μ μ€ν¬λ¦½νΈ
```
# docker login
docker login -u <DOCKERHUB_USERNAME> -p <DOCKERHUB_PASSWORD>

# pull docker image
if [ "$DEPLOYMENT_GROUP_NAME" == "dev" ]
then
  pwd
  docker-compose -f /home/ec2-user/app/docker-compose.yml pull
elif [ "$DEPLOYMENT_GROUP_NAME" == "stage" ]
then
  docker-compose -f /deploy/docker-compose.stage.yml pull
elif [ "$DEPLOYMENT_GROUP_NAME" == "production" ]
then
  docker-compose -f /deploy/docker-compose.yml pull
fi
```
