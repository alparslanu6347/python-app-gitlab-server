# python-app-gitlab-server : Launch a gitlab-server for app and install the programmes

- Go to AWS Management Console, we need a `gitlab-server` and we need to install agent into that server.

- Launch an AWS EC2 instance of Amazon Linux 2023 AMI `t2.micro` with security group allowing ``SSH:22``, ``HTTP:80`` and ``TCP 8080`` ports.

- Make a remote-ssh VSC connection to your instance

```bash
ssh -i .ssh/call-training.pem ec2-user@ec2-3-133-106-98.us-east-2.compute.amazonaws.com
```

- You should write these commands into `.bashrc` file, first go to bottom line and hit enter 2 times then write

```bash (.bashrc)
export PS1="\[\e[1;34m\]\u\[\e[33m\]@\h# \W:\[\e[34m\]\\$\[\e[m\] "  # yellow-blue-yellow
hostnamectl set-hostname "gitlab"
```
```bash (pwd : home/ec2-user)
bash
```

- Install `docker and docker compose`
```bash (pwd : home/ec2-user)
# Update the installed packages and package cache on your instance.
sudo dnf update -y
# Install the most recent Docker Community Edition package.
sudo dnf install docker -y
# Start docker service.
sudo systemctl start docker
# Enable docker service so that docker service can restart automatically after reboots.
sudo systemctl enable docker
# Check if the docker service is up and running.
sudo systemctl status docker
# Add the `ec2-user` to the `docker` group to run docker commands without using `sudo`.
sudo usermod -a -G docker ec2-user    # VEYA     sudo usermod -aG docker $USER
# `newgrp` command can be used activate `docker` group for `ec2-user`
newgrp docker
# Check the docker version without `sudo`.
docker version
# Check the docker info without `sudo`.
docker info

# install docker-compose
sudo curl -SL https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# check the docker-compose version
docker-compose --version

# install git
sudo dnf install git
```

# Create gitlab project/repository, application files
- Go to gitlab and create a new project/repository
    Click `+` --> Select `New project/repository` -->> Click `Create blank project`
    `Project name` : `python-app`
    `Project URL`  : `https://gitlab.com/arrowlevent/python-app`
    `Visibility level` : `Public`
    `Project Configurations` : Put check mark on the `Initialize repository with a README`
    Click `Create project`

    python-app/ `+` ==>> Click `+` Select ``New file` from the list
    `filename` : `app.py`
    copy-paste into this file

```py   (app.py)
from http.server import BaseHTTPRequestHandler, HTTPServer

class MyHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/plain')
        self.end_headers()
        self.wfile.write(b'''
          ##         .
    ## ## ##        ==
 ## ## ## ## ##    ===
/"""""""""""""""""\___/ ===
{                       /  ===-
\______ O           __/
 \    \         __/
  \____\_______/


Hello from Docker!
''')

def run():
    print('Starting server...')
    server_address = ('', 8080)
    httpd = HTTPServer(server_address, MyHandler)
    print('Server started!')
    httpd.serve_forever()

if __name__ == '__main__':
    run()
```

- Click `Commit changes`

- python-app/ `+` ==>> Click `+` Select ``New file` from the list
    `filename` : `Dockerfile`
    copy-paste into this file

```Dockerfile
FROM python:3.8-alpine

RUN mkdir /app

ADD . /app

WORKDIR /app

EXPOSE 8080

CMD ["python3", "app.py"]
```
- Click `Commit changes`


- python-app/ `+` ==>> Click `+` Select ``New file` from the list
    `filename` : `docker-compose.yaml`
    copy-paste into this file

```yaml (docker-compose.yaml)
# $CI_PIPELINE_IID : image'a her build işleminde 1,2,3.. diye versiyon numarası vermek için gitlab variable
services:
  app:
    image: alparslanu6347/python-cicd:v$CI_PIPELINE_IID
    #build: .
    ports:
      - "8080:8080"
    command: python3 app.py
```
- Click `Commit changes`


- python-app/ `+` ==>> Click `+` Select ``New file` from the list
    `filename` : `requirements.txt`
    copy-paste into this file

```txt (requirements.txt)
gunicorn
```
- Click `Commit changes`

# Prepare runner in the gitlab

- Click `Settings` Select `CI/CD` -->> Hit the `Expand` for `Runners`

Terminalde de ``sudo gitlab runner registry`` komutu ile runner oluşturabiliriz `tag`leri manuel yazıyoruz, token'ını da buradan `New project runner` -->> `3 nokta üst üste` `Registration token` alıyoruz ve terminalde giriyoruz

Ama biz burada oluşturacağız:

- Click `New project runner`  ==>> `Platform` - `Operating systems` -> `Linux`
                              ==>> `Tags` : `python-app` ***pipeline içerisinde her bir job altına bu tag'i kullandık ÖNEMLİ, her bir job için ayrı bir tag kullanabilirsin ve böylece her job farklı bir runner içerisinde çalışır***
                              ==>> `Details (optional)` : `*****optional*****`
        ``Create runner``

`Step 1` altındaki komutu `sudo` ekleyerek terminalde koşmam gerek AMA BU KOMUTTAN ÖNCE gitlab runner'ı agent olarak kurmam gerekiyor.

- Install GitLab Runner manually on GNU/Linux
    - https://docs.gitlab.com/runner/install/linux-manually.html
    - copy the true one `Linux x86-64`

```bash
# Simply download one of the binaries for your system:
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64"
# Give it permissions to execute:
sudo chmod +x /usr/local/bin/gitlab-runner
# Create a GitLab CI user:
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
# Install and run as service:
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
# Register a runner  -->> `Step 1` altındaki komutu `sudo` ekleyerek terminalde koşmam gerek
sudo gitlab-runner register  --url https://gitlab.com  --token glrt-sJBzBRhDmENAZzGvgQws

    Enter the GitLab instance URL (for example, https://gitlab.com/):
    [https://gitlab.com]:   write `https://gitlab.com` then hit enter

    Enter a name for the runner. This is stored only in the local config.toml file:
    [gitlab]:               write `python-app-runner` then hit enter        NE YAZDIĞIN FARKETMEZ

    Enter an executor: docker, virtualbox, docker-autoscaler, kubernetes, custom, docker-windows, parallels, shell, ssh, docker+machine, instance:               write `shell` then hit enter     KOMUTLARIMI shell ile yazdığım için

sudo gitlab-runner status
```
- Go to gitlab
- Click `Settings` Select `CI/CD` -->> Hit the `Expand` for `Runners`   ==>> See `Assigned project runners` under `Project runners` -- Go to green circle and see the note `Runner is online; last connect was 10 minutes ago` ==>> Burada gördüğün `#30856663 (sJBzBRhDm)` ifadesi benim ec2-instance içindeki ile eşleşmesi lazım

```bash
sudo gitlab-runner verify
        Verifying runner... is valid                        runner=sJBzBRhDm        # yukarıdaki ile aynı
```

- ec2-user'a docker'a dahil ettim ki docker kullanabiliyor, aynı şekilde gitlab kurarken oluşturduğum `gitlab-runner` kullanıcısını da docker'a dahil etmeliyim ki gitlab gitlab-runner üzerinden docker komutları koşabilsin.

```bash
# Add the `gitlab-runner` to the `docker` group to run docker commands without using `sudo`.
sudo usermod -a -G docker gitlab-runner

### BU KOMUTU GİRMEDEN OLDU
# `newgrp` command can be used activate `docker` group for `gitlab-runner`
newgrp docker
```

- Pipeline hazırlamadan önce pipeline içinde kullandığımız variables tanımlaması yapmalıyız
`$CI_PIPELINE_IID`: predefined variable olduğu için gitlab bunu algılayacak zaten.
 CI_PIPELINE_IID : The project-level IID (internal ID) of the current pipeline. This ID is unique only within the current project.
BU DEĞİŞKENLERİ TANIMLAMAMIZ GEREKİYOR : Docker Hub için user ve password -->> ``$DOCKER_HUB_USER`` ``$DOCKER_HUB_PWD``

   - `Settings` --> `CI/CD` --> `Variables` - `Expand` --> `Add variable` ==> `Key` : `DOCKER_HUB_USER`
                                                                          ==> `Value` : `alparslanu6347`  --> Click `Add variable`
                                                                          ==> `Type` : `Default`
                                                                          ==> `Environment` : `Default`
                                                       --> `Add variable` ==> `Key` : `DOCKER_HUB_PWD`
                                                                          ==> `Value` : `********`  ***write yours***
                                                                          ==> `Type` : `Default`
                                                                          ==> `Environment` : `Default`
                                                                          İLAVE OLARAK put check mark `Mask variable` --> Click `Add variable` 

- Prepare `.gitlab-ci.yml`
    - Her job için farklı runner atayabiliriz ama burada proje küçük olduğu için tek runner kullandık, Best-practice her bir job için ayrı runner atamak, hatta deploy yapacağımız runner'ı service runner'ı yapmamız daha doğru olurdu

    - python-app/ `+` ==>> Click `+` Select ``New file` from the list
    `filename` : `.gitlab-ci.yml`
    copy-paste into this file

```yaml (.gitlab-ci.yml)
stages:
    - test
    - build_stage
    - push_stage
    - deploy_stage

test_server:
    stage: test
    script:
        - pwd
        - ls
        - whoami
        - docker ps
        - docker ps -a
        - docker images
    tags:
        - python-app    


build:
    stage: build_stage
    script:
        - docker --version
        - docker build . -t $DOCKER_HUB_USER/python-cicd:v$CI_PIPELINE_IID
    tags:
        - python-app 

push:
    stage: push_stage
    script:
        - docker login -u $DOCKER_HUB_USER -p $DOCKER_HUB_PWD
        - docker push $DOCKER_HUB_USER/python-cicd:v$CI_PIPELINE_IID
    tags:
        - python-app 


deploy:
    stage: deploy_stage
    script:
        - docker-compose down -v
        - docker-compose up -d
    tags:
        - python-app
```
- Click `Commit changes`

- Pipeline aşamalarını/joblarını buradan gözlemleyebilirsin`Build` -- `Pipelines`

- Uygulamayı görmek için :
    - http://ec2 Public IP:8080/
    - http://44.203.147.157:8080/   



# Resources

- https://gitlab.com/arrowlevent/python-app
- https://docs.gitlab.com/ee/ci/yaml/index.html
- https://docs.gitlab.com/ee/ci/variables/ 
- https://docs.gitlab.com/ee/ci/variables/predefined_variables.html
- https://docs.gitlab.com/ee/ci/variables/where_variables_can_be_used.html
- https://docs.gitlab.com/runner/install/linux-manually.html 