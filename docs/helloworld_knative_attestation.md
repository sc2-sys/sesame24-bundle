# Knative + CoCo + Attestation

In this demo, we set up CoCo to attest the guest VM that is bootstrapped.
In order to run a functional attested system we need a service that, given
a launch digest, returns a secret (i.e. a relying party). We use CoCo's
[simple KBS](https://github.com/confidential-containers/simple-kbs).

For demonstration purposes, the simple KBS runs as a process in the same
node (outside Knative, and Kubernetes for that same reason). To start it,
just run:

```bash
inv kbs.start
```

Depending on what features do you want to measure/attest, browse through the
following subsections (in increasing order of security):
* [Firmware Digest](#firmware-digest) - Attest that the VM runs in an SEV node,
with the right firmware, kernel, initrd, and Kata Agent configuration.
* [Signed Container Images](#signed-container-images) - Attest that the images
used have been signed with a well-known private key.
* [Encrypted Container Images](#encrypted-container-images) - In addition to
signatures, consume encrypted container images.

After that, you may jump to [running the application](#run-the-application).

> Note that, if you are running different attestation types one after the other
> you may want to clear the contents of the KBS between runs using:
> `inv kbs.clear-db`

## Firmware Digest

Before running the application, we need to generate the expected launch digest
for our confidential VM. This launch digest will be generated by reading some
of the fields in the Kata config file, so we must update it before generating
the digest.

First, we need to set `guest_pre_attestation` so that the Kata Shim opens a
secure channel between the PSP and the KBS at boot time. We will later use this
channel to validate firmware and software measurements from inside the guest:

```bash
inv coco.guest-attestation --mode on
```

note that this method also sets the right KBS URI, as the default one
(`localhost`) is not reachable from inside the guest.

Now, we must also enable signature verification. Signature verification is the
only method available in `v0.7.0` to check the launch digest against a user-
provided digest:

```bash
inv coco.signature-verification --mode on
```

In order to validate the launch digest, we need to pre-provision the launch
digest. To do so you may run:

```bash
inv kbs.provision-launch-digest --signature-policy none
```

this method will generate launch digests from the data in the Kata configuration
files (i.e. `/opt/confidential-containers/share/defaults/kata-containers/configuration-*.toml`
if installed by the operator), and include them in the KBS.

If you don't want to sign and/or encrypt your container images, you may jump
straight to [running the application](#run-the-application).

## Signed Container Images

In this section we configure the system to, in addition to the HW launch
digest, validate that all container images pulled inside the confidential
VM have been signed by a well-known private key.

To this extent, we will sign the container images used, and then update the
KBS signature-verification policy to validate image signatures (and will
provide the associated public key to do so). We use [`cosign`](
https://github.com/sigstore/cosign) to sign the Docker images, so you will
have to install it first:

```bash
inv cosign.install
```

> [!WARNING]
> We can only sign (and encrypt) images published in registries that we have
> `push` access to. As a consequence, we need to re-tag the side-car image used
> by default by Knative (as we don't have push access to Knative's repo) and
> push it to a registry that we control.

```bash
inv knative.repalce-sidecar
```

Now we are ready to sign both images that are going to live inside the cVM.
As [recommended](https://github.com/sigstore/cosign#sign-a-container-and-store-the-signature-in-the-registry),
we sign images based on their digest. Note that we configure `cosign` to use
a private key with an empty passphrase.

```bash
inv cosign.sign-container-image "docker.io/csegarragonz/coco-helloworld-py@sha256:af0fec55e9aed9a259e8da9dcaa28ab3fc1277dc8db4b8883265f98272cef11d"
inv cosign.sign-container-image "docker.io/csegarragonz/coco-knative-sidecar@sha256:79d5f6031f308cee209c4c32eeab9113b29a1ed4096c5d657504096734ca3b1d"
```

Finally, (re-)provision the KBS with the right FW measurements and the
adequate signature-policy to validate the image signatures:

```bash
inv kbs.provision-launch-digest --signature-policy verify --clean
```

Now, you may proceed to [running the application](#run-the-application).

## Encrypted Container Images

In this last section, we will configure the system to consume encrypted
container images from public registries.

To encrypt docker images, we use [`skopeo`](https://github.com/containers/skopeo).
`skopeo` can be used together with image signatures, just make sure to pass the
`--sign` flag:

```bash
inv skopeo.encrypt-container-image "docker.io/csegarragonz/coco-helloworld-py:unencrypted" --sign
```

> To check that the image is actually encrypted, you may try to run it:
> `docker run docker.io/csegarragonz/coco-helloworld-py:encrypted`

Then, as in the previous section, update the KBS to set the right signature
verification policy:

```bash
# IMPORTANT: do not use the `--clean` flag here, as the encrypt-container-image
# command will provision a secret to the KBS
inv kbs.provision-launch-digest --signature-policy verify
```

Finally, you may proceed to [running the application](#run-the-application).

## Run the application

Once the KBS has been populated with the right measurements and secrets, we can
deploy the workload just like with the `Hello world! (Knative)` app:

```bash
kubectl apply -f ./apps/helloworld-knative
```

note that if you are using encrypted images you will have to do:

```bash
kubectl apply -f ./apps/helloworld-knative-encrypted
```