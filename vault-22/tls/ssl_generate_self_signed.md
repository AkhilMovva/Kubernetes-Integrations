# Use CFSSL to generate certificates

More about [CFSSL here]("https://github.com/cloudflare/cfssl")

```bash

cd hashicorp\vault-2022\tls

docker run -it --rm -v ${PWD}:/work -w /work debian bash

apt-get update && apt-get install -y curl &&
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64 -o /usr/local/bin/cfssl && \
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64 -o /usr/local/bin/cfssljson && \
chmod +x /usr/local/bin/cfssl && \
chmod +x /usr/local/bin/cfssljson

#generate ca in /tmp
cfssl gencert -initca ca-csr.json | cfssljson -bare /tmp/ca

#generate certificate in /tmp
# 1) Kubernetes <servicename>
# 2) <servicename>.<namespace>
# 3) <servicename>.<namespace>.svc
cfssl gencert \
  -ca=/tmp/ca.pem \
  -ca-key=/tmp/ca-key.pem \
  -config=ca-config.json \
  -hostname="vault,vault.vault.svc.cluster.local,vault.vault.svc,localhost,127.0.0.1" \
  -profile=default \
  ca-csr.json | cfssljson -bare /tmp/vault
```

view the files:

```bash
ls -l /tmp
```

access the files:

```bash
mv /tmp/* .
```

```bash
# for the nginx https server
apt-get update && apt-get install -y curl &&
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64 -o /usr/local/bin/cfssl && \
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64 -o /usr/local/bin/cfssljson && \
chmod +x /usr/local/bin/cfssl && \
chmod +x /usr/local/bin/cfssljson

vi ca-csr.json
vi ca-config.json

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

openssl x509 -in ca.pem -text -noout

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=devops akhilmovva-csr.json | cfssljson -bare akhilmovva

openssl x509 -in akhilmovva.pem -text -noout
```

<!-- ca-config.json -->
<!-- {
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "devops": {
        "usages": ["signing", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
} -->
<!-- example-csr.json -->
<!-- {
  "CN": "DevOps by Example",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CA",
      "L": "Toronto",
      "O": "DevOps by Example",
      "OU": "Dev",
      "ST": "ON"
    }
  ]
} -->
