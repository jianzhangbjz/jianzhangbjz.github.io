1. Create the user/password file, as follows:
```console
[cloud-user@preserve-olm-env jian]$ docker run --entrypoint htpasswd registry:2.7.0 -Bbn <username> <password> > /data/jian/auth/htpasswd
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Resolved "registry" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/registry:2.7.0...
Getting image source signatures
Copying blob cd784148e348 done  
Copying blob adee6f546269 done  
Copying blob 918b3ddb9613 done  
Copying blob 0ecb9b11388e done  
Copying blob 5aa847785533 done  
Copying config 33fbbf4a24 done  
Writing manifest to image destination
Storing signatures
```

2. Create a self-signed cerfication.
```console
[cloud-user@preserve-olm-env certs]$ openssl req -newkey rsa:4096 -nodes -sha256 -keyout /data/jian/certs/domain.key -x509 -days 3650 -out /data/jian/certs/domain.crt
Generating a RSA private key
.................................++++
...................................++++
writing new private key to '/data/jian/certs/domain.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Hebei
Locality Name (eg, city) [Default City]:Shijiazhuang
Organization Name (eg, company) [Default Company Ltd]:RedHat
Organizational Unit Name (eg, section) []:QE
Common Name (eg, your name or your server's hostname) []:localhost
Email Address []:jiazha@redhat.com

[cloud-user@preserve-olm-env certs]$ ls
domain.crt  domain.key
```

3. Create the `localhost:5000` folder under the `/etc/docker/certs.d/`(if it doesn't exist, create it) and Copy this `domain.crt` to it.
```console
[cloud-user@preserve-olm-env opm]$ tree /etc/docker/certs.d/
/etc/docker/certs.d/
`-- localhost:5000
    `-- ca.crt

[cloud-user@preserve-olm-env certs]$ cd /etc/docker/certs.d/localhost\:5000/

[cloud-user@preserve-olm-env localhost:5000]$ sudo cp /data/jian/certs/domain.crt ca.crt
[cloud-user@preserve-olm-env localhost:5000]$ ls -l
total 4
-rw-r--r--. 1 root root 2122 Aug  5 02:41 ca.crt
```

4. Create a TLS registry container
```console
[cloud-user@preserve-olm-env jian]$ sudo docker run -d --name registry_native_auth -p 5000:5000 -v /data/jian/auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v /data/jian/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key --privileged registry:2.7.0
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
6d7e880e4b2a056cf75ec01f3bfdbce6079127a50c5ab2ddff2f387ff4ed0d4c
[cloud-user@preserve-olm-env jian]$ sudo docker ps
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
CONTAINER ID  IMAGE                             COMMAND               CREATED        STATUS            PORTS                   NAMES
6d7e880e4b2a  docker.io/library/registry:2.7.0  /etc/docker/regis...  3 seconds ago  Up 4 seconds ago  0.0.0.0:5000->5000/tcp  registry_native_auth
```

5. Login this local registry.
```console
[cloud-user@preserve-olm-env jian]$ docker login localhost:5000
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Authenticating with existing credentials...
Existing credentials are invalid, please enter valid username and password
Username (test): 
Password: 
Error: error authenticating creds for "localhost:5000": error pinging docker registry localhost:5000: Get "https://localhost:5000/v2/": x509: certificate relies on legacy Common Name field, use SANs or temporarily enable Common Name matching with GODEBUG=x509ignoreCN=0
```

6. Export `GODEBUG=x509ignoreCN=0` and relogin.
```console
[cloud-user@preserve-olm-env jian]$ export  GODEBUG=x509ignoreCN=0
[cloud-user@preserve-olm-env jian]$ docker login localhost:5000
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Authenticating with existing credentials...
Existing credentials are valid. Already logged in to localhost:5000

[cloud-user@preserve-olm-env jian]$ curl -k --user <username>:<password> https://localhost:5000/v2/_catalog
{"repositories":[]}
```

7. verifiy this private registry wether works or not.
```console
[cloud-user@preserve-olm-env opm]$ docker tag quay.io/olmqe/nginx-bundle:v4.9 localhost:5000/olmqe/nginx-bundle:v4.9
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.

[cloud-user@preserve-olm-env opm]$ docker push localhost:5000/olmqe/nginx-bundle:v4.9
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Getting image source signatures
Copying blob 076a37ef5f7a done  
Copying blob d02c4c202a5a done  
Copying blob 74536fa4f090 done  
Copying config ba4276c693 done  
Writing manifest to image destination
Storing signatures

[cloud-user@preserve-olm-env opm]$ curl -k --user <username>:<password> https://localhost:5000/v2/_catalog
{"repositories":["olmqe/nginx-bundle"]}
```

8. Mirror an image via `oc`, but get the `x509: certificate signed by unknown authority` error.
```console
[cloud-user@preserve-olm-env opm]$ oc adm catalog mirror quay.io/olmqe/etcd-index:dc-new localhost:5000
src image has index label for declarative configs path: /configs/
using index path mapping: /configs/:/tmp/154039752
wrote declarative configs to /tmp/154039752
using declarative configs at: /tmp/154039752

error: unable to connect to localhost:5000/olmqe/etcd-bundle: Get "https://localhost:5000/v2/": x509: certificate signed by unknown authority
error: unable to connect to localhost:5000/coreos/etcd-operator: Get "https://localhost:5000/v2/": x509: certificate signed by unknown authority
error: unable to connect to localhost:5000/olmqe/etcd-index: Get "https://localhost:5000/v2/": x509: certificate signed by unknown authority

info: Planning completed in 800ms
```

9. Update the truse CA.
```console
[cloud-user@preserve-olm-env opm]$ sudo cp /etc/docker/certs.d/localhost\:5000/ca.crt /etc/pki/ca-trust/source/anchors/
[cloud-user@preserve-olm-env opm]$ 
[cloud-user@preserve-olm-env opm]$ sudo update-ca-trust 
[cloud-user@preserve-olm-env opm]$ 
```

10. Rerun the step 8, it works well.
```console
[cloud-user@preserve-olm-env opm]$ oc adm catalog mirror quay.io/olmqe/etcd-index:dc-new localhost:5000
src image has index label for declarative configs path: /configs/
using index path mapping: /configs/:/tmp/631206284
wrote declarative configs to /tmp/631206284
using declarative configs at: /tmp/631206284
localhost:5000/
  coreos/etcd-operator
    blobs:
      quay.io/coreos/etcd-operator sha2
      ....
...
I0805 03:20:47.645111 3847811 manifest.go:501] warning: Digests are not preserved with schema version 1 images. Support for schema version 1 images will be removed in a future release
sha256:db9eac85a5ca921e38713d0bbf8372b1d354538887b952dd547db0c86a82d2da localhost:5000/coreos/etcd-operator:acb71f74
info: Mirroring completed in 3.28s (23.51MB/s)
no digest mapping available for quay.io/olmqe/etcd-bundle:dc, skip writing to ImageContentSourcePolicy
no digest mapping available for quay.io/olmqe/etcd-index:dc-new, skip writing to ImageContentSourcePolicy
wrote mirroring manifests to manifests-etcd-index-1628148042

[cloud-user@preserve-olm-env opm]$ curl -k --user <username>:<password> https://localhost:5000/v2/_catalog
{"repositories":["coreos/etcd-operator","olmqe/etcd-bundle","olmqe/etcd-index","olmqe/nginx-bundle"]}
```