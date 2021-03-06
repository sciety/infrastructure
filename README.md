# infrastructure
Infrastructure as Code for Sciety

## Environments

- `franklin`: Kubernetes cluster for Sciety.

## Scope

Resources belonging to a Libero AWS account are managed here:

### Kubernetes

Kubernetes clusters to deploy vanilla versions of Libero products and modules are housed here.

They run in their own VPCs in `us-east-1`.

### Servers

EC2 instances are used to demonstrate working versions of Libero's projects.

They run inside the default VPC for `us-east-1` and are based on a minimal Ubuntu OS running Docker and `docker-compose`. Software is deployed in the form of container images.

### DNS

DNS entries for `libero.pub` and similar domains are managed under this AWS account, hence here.

## Setup

This repository is public, hence secrets are encrypted by default. `git-crypt` and optionally `gpg` are used to unlock the contents to be able to use the repository contents.

`git-crypt` encrypts data with a symmetric key, and places encrypted versions of that key in `.git-crypt`; one value for each public key of the users that should be able to access that data. It is easy to add new users, more complex to remove them.

Once your public key has been added, you can unlock the repository (see [Admin](#admin) section below for how), you can decrypt the repository:

```bash
git-crypt unlock
```

## Terraform usage

You will need to set up AWS credentials in your ~/.aws/credentials file:

```
[libero]
aws_access_key_id=<access key id>
aws_secret_access_key=<secret access key>
```

Then set your current AWS profile:
```bash
$ export AWS_PROFILE=libero
```

### New environment
Create a new environment (here named `unstable`):

```
cd tf/envs/
mkdir $ENVIRONMENT_NAME
echo "env = $ENVIRONMENT_NAME" > terraform.tfvars
ln -s ../main.tf main.tf
ln -s ../variables.tf variables.tf
../../terraform init --backend-config="key=$ENVIRONMENT_NAME/terraform.tfstate"
../../terraform plan -out=my.plan
../../terraform apply my.plan
```

### Fetch current environment state
To get the state of an existing environment (here named `unstable`):

```bash
cd tf/envs/$ENVIRONMENT_NAME
rm -rf .terraform
../../terraform init --backend-config="key=$ENVIRONMENT_NAME/terraform.tfstate"
```

### Update an environment

Update the environment:

```bash
../../terraform plan -out=my.plan
../../terraform apply my.plan
```

Note: normally `apply` should not be executed, as it's done via CI on merge.

### Output resource information

The libero-admin.key and libero-admin.pub are a keypair of PGP keys that are used by Terraform to encrypt secrets generated during infrastructure provisioning, such as AWS credentials for applications. Import the libero admin key:

```bash
gpg --import libero-admin.key
```

You can then use the `./terraform output` commmand to extract resource information. For example, to dump the HTTPS certificate:

```bash
# legacy
./terraform output https_certificate_pem
# current (empty at the moment)
../../terraform output
```

An environment named `test` is reserved for destruction, creation and any other exploration.

### SSH into instance (legacy)
SSH into an instance:

```bash
ssh -i "tf/single-node--${ENVIRONMENT_NAME}.key" ubuntu@$(./terraform output single_node_ip)
```

### Kubectl access

Assuming you have AWS credentials setup to access the Libero account, the `aws` cli can generate your `kubectl` authentication configuration for the cluster:

```
aws eks update-kubeconfig libero-eks--franklin
kubectl version
kubectl get pods
```

### Helm access

Helm uses the same authentication as `kubectl`. Use the Helm version provided by this repository.

```
scripts/download-helm.sh
helm ls
```

## Admin

### Adding a user

If you don't already have a key pair, you can generate one as such:

```bash
gpg --gen-key
```

Reasonable defaults: `RSA and RSA`, `2048` bits, no expiration, your work email address.

Export your new key with

```bash
gpg --armor --output myname.gpg --export myname@example.com
```

Now someone with an unlocked repository copy can use your `myname.gpg` public key:

```bash
gpg --import myname.gpg
git-crypt add-gpg-user myname@example.com  # creates a new commit
git push
```

### Removing a user

You can remove a user's key from `.git-crypt/keys/default/0/` but this will not prevent them from using their own copy to continue decrypting. What is needed is to [reinitialize `git-crypt` and re-add all the other keys](https://gist.github.com/developerinlondon/6a853fe175178d4aacb0aa55a4cb09a1). Secrets up to the point of change will still be available to that user, this is no different than for a private repository.


### Creating and rotating Admin Key

The admin key is created with these commands:
```
gpg --gen-key
gpg --armor --export-secret-keys libero-admin@elifesciences.org | tee libero-admin.key
gpg --export libero-admin@elifesciences.org | base64 | tee libero-admin.pub
```

All `.key` files are managed by `git-crypt`, including this one.
