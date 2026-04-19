Generating the Data Encryption Config and Key
Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to encrypt cluster data at rest, that is, the data stored within etcd.

In this lab you will generate an encryption key and an encryption config suitable for encrypting Kubernetes Secrets.

## The Encryption Key
Generate an encryption key. This is simply 32 bytes of random data, which we base64 encode:
```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File
Create the encryption-config.yaml encryption config file:
```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Copy the encryption-config.yaml encryption config file to each controller instance:
```bash
for instance in controlplane01 controlplane02; do
  scp encryption-config.yaml ${instance}:~/
done
```

Move encryption-config.yaml encryption config file to appropriate directory:
```bash
for instance in controlplane01 controlplane02; do
  ssh ${instance} sudo mkdir -p /var/lib/kubernetes/
  ssh ${instance} sudo mv encryption-config.yaml /var/lib/kubernetes/
done
```
