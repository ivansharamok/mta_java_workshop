# Image signing

## Setting up Docker Content Trust (DCT)

This task leverages CI/CD pipeline configured in the scope of `task 8`.

>Because this task uses local volume to persist data, you will need to SSH and run some commands directly on a worker node that runs Gitlab Runner instance.

To sign images, you need to configure Docker Content Trust (DCT). We will use a worker node that runs Gitlab Runner instance to configure DCT.

* List available nodes and pick one that has Gitlab Runner instance

    ```bash
    docker node ls
    ```

* SSH into the node

    ```bash
    runner_node='worker-2'
    # use either RSA key or user/password to SSH into the worker node
    ssh -i ~/path/to/rsa_key node_user@<node_IP>
    ```

* Create local volumes to store DCT related data

```bash
docker volume create -d local dct
docker volume create -d local dtr-cert
```

* Download client bundle into `dct` volume

>This task uses public [nicolaka/netshoot](https://hub.docker.com/r/nicolaka/netshoot) image from [Docker HUB](https://hub.docker.com/). This image contains a number of useful tools to troubleshoot network issues. In the scope of this task we will use only `curl` utility.

```bash
# adjust these variables as needed
UCP_HOST='ucp.example.com'
UCP_USER='admin'
UCP_PASS='password'
# get authentication token
docker run --rm -t nicolaka/netshoot:latest sh -c "apk add -q jq && curl -kLs -d '{\"username\":\"${UCP_USER}\",\"password\":\"${UCP_PASS}\"}' https://${UCP_HOST}/auth/login | jq -r .auth_token"
# copy authentication token
# download client bundle using auth token and extract it
TOKEN='<replace_with_auth_token>'
docker run --rm -it -v dct:/dct nicolaka/netshoot:latest sh -c "curl -k -H 'Authorization: Bearer $TOKEN' https://${UCP_HOST}/api/clientbundle/gitlab -o /dct/client-bundle.zip && mkdir /dct/gitlab_bundle && unzip /dct/client-bundle.zip -o -d /dct/gitlab_bundle && rm /dct/client-bundle.zip"
```

* Download DTR CA cert into `dtr-cert` volume

>because this example uses a container based on `docker:dind` image to run `docker trust` commands, we need to configure DTR CA certificate within a container certificate store. The `dtr-cert` volume will be used to store DTR CA cert and map it into the container.

Download DTC CA certificate into `dtr-cert` volume

```bash
# use your DTR FQDN
DTR_HOST='dtr.example.com'
docker run --rm -t -v dtr-cert:/dtr_cert nicolaka/netshoot:latest sh -c "curl -k https://${DTR_HOST}/ca -o /dtr_cert/dtr_ca.pem"
```

* Initialize DCT folder and upload signer cert for the repository

```bash
# use your DTR FQDN
DTR_HOST='dtr.example.com'
DTR_USER='gitlab'
DTR_PASS='user1234'
DCT_PASS='MyDctPassw0rd!'
docker run --rm -t -v dct:/root/.docker -v dtr-cert:/etc/ssl/certs -v /var/run/docker.sock:/var/run/docker.sock -e DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE=$DCT_PASS -e DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE=$DCT_PASS docker:dind sh -c "docker login -u $DTR_USER -p $DTR_PASS $DTR_HOST && docker trust signer add --key /root/.docker/gitlab_bundle/cert.pem gitlab ${DTR_HOST}/ci/java_app_build"
```

* Move DCT private key into a secure vault

    >Once DCT is initialized, it has a root key stored at `/<user>/.docker/trust/private` path. The root key is the master key to image security management process. It must be removed and stored in a secure location (e.g. some sort of vault repository).

  * Identify DCT root key

    >To identify the root key, we will `exec` into the container and inspect all keys in DCT path.

    ```bash
    # exec into container
    docker run --rm -it -v dct:/dct -v /var/run/docker.sock:/var/run/docker.sock nicolaka/netshoot bash
    # find key that has 'root' keyword
    for i in $(ls -d -1 /dct/trust/private/*);do if grep -q root $i; then ls $i;fi;done
    # copy identified path end exit container
    ```

  * Copy root key out of the container and remove it from the volume

    ```bash
    # set this variable to the path of root key you copied from container
    # path to rook key should look like: /dct/trust/private/<keyid>.key
    ROOT_PATH="/path/to/root.key"
    # copy root key from container into your home dir and remove it from the volume
    docker run --name=dct -v dct:/dct -v /var/run/docker.sock:/var/run/docker.sock nicolaka/netshoot sleep 5;  docker cp dct:$ROOT_PATH ~/; docker start dct && docker exec -t dct rm $ROOT_PATH; docker rm -f dct
    ```

* Import signer's key

    In order to sign an image, a private key from user's client bundle (in this case `gitlab` user) should be imported into DCT

    ```bash
    DCT_PASS='MyDctPassw0rd!'
    docker run --rm -t -v dct:/root/.docker -v dtr-cert:/etc/ssl/certs -v /var/run/docker.sock:/var/run/docker.sock -e DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE=$DCT_PASS -e DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE=$DCT_PASS docker:dind docker trust key load /root/.docker/gitlab_bundle/key.pem
    ```

## Sign the image

* Sign the image and push to DTR

    There are two ways to sign the image and push to DTR. Use either approach to sign the image.

    1. Use `docker trust` command

        ```bash
        # use your DTR FQDN
        DTR_HOST='dtr.example.com'
        DCT_PASS='MyDctPassw0rd!'
        # use the tag that was created at task 8
        TAG="1"
        docker run --rm -t -v dct:/root/.docker -v dtr-cert:/etc/ssl/certs -v /var/run/docker.sock:/var/run/docker.sock -e DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE=$DCT_PASS -e DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE=$DCT_PASS docker:dind sh -c "docker pull ${DTR_HOST}/ci/java_app_build:${TAG} && docker trust sign ${DTR_HOST}/ci/java_app_build:${TAG}"
        ```

    2. Use DCT and `docker push` command

        This approach requires to enable DCT and push the image to the DTR.

        ```bash
        # use your DTR FQDN
        DTR_HOST='dtr.example.com'
        DTR_USER='gitlab'
        DTR_PASS='user1234'
        DCT_PASS='MyDctPassw0rd!'
        # use the tag that was created at task 8
        TAG="1"
        docker run --rm -t -v dct:/root/.docker -v dtr-cert:/etc/ssl/certs -v /var/run/docker.sock:/var/run/docker.sock docker:dind sh -c "docker pull ${DTR_HOST}/ci/java_app_build:${TAG} && docker login -u $DTR_USER -p $DTR_PASS ${DTR_HOST} && export DOCKER_CONTENT_TRUST=1 && docker push ${DTR_HOST}/ci/java_app_build:${TAG}"
        ```

* Verify the result

  * Navigate to `java_app` project in Gitlab and review the output of CI/CD Pipeline tasks.
  * Navigate to `java_app_build` repository in DTR and switch to `Tags` tab. You should be able to see `Signed` column filled in for the image tag you signed.
  ![](images/signed_image.png)

## Configuring automated image signing

Once you have all moving parts of DCT configured, you can incorporate image signing process into CI build process.  

* Navigate to `./task_9_signing` directory and clone `java_app` git repository created in `task 8`

    ```bash
    # adjust git clone URL if needed
    GIT_REPO="http://${UCP_HOST}:9090/root/java_app.git"
    cd ../task_9_signing
    git clone $GIT_REPO
    ```

* Create gitlab build declarative with image signing command

>Use example at `./build/.gitlab-ci.yml` file.

```bash
cp ./build/.gitlab-ci.yml ./java_app/
```

* Open `./java_app/.gitlab-ci.yml` file and set `DTR_PASSWORD` and `DTR_SERVER` variables.

>Use your UCP `admin` user account credentials for DTR user and password variables.

* Commit changes

```bash
cd ./java_app
git add .gitlab-ci.yml
git commit -m 'build and sign image'
git push origin
```

* Verify the result

  * Navigate to `java_app` project in Gitlab and review the output of CI/CD Pipeline tasks.
  * Navigate to `java_app_build` repository in DTR and view its tags. The image tag should match the job build number in Gitlab and `Signed` stamp should be appear in Signed column for your image tag.
