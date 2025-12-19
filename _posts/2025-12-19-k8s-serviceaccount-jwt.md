---
layout: post
title: "Is Kubernetes ServiceAccount a JWT token? And how to verify it?"
tags: [kubernetes]
comments: false
---

_(More of my "notes for self", as I continue reading the thesis paper, [Usable Access Control in Cloud Management Systems](https://github.com/luxas/research/blob/main/msc_thesis.pdf), written by Lucas Käldström.)_

I ran another small experiment today.

Today, I learnt that the Kubernetes Service Account tokens that I use very often to authenticate with the API server (using `Authorization: Bearer <token>` header with the HTTP request) are JWT (JSON Web Token) tokens.

I learnt this as a verbal fact first, so, I wanted to verify it in my mighty Kind cluster.

So, let's first create a Kind cluster.

```bash
❯ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.34.0) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```

Create a test pod and exec inside the container.

```bash
❯ kubectl run jwt-test \
   --image=busybox \
   --restart=Never \
   -- sleep 3600
pod/jwt-test created

❯ kubectl exec -it jwt-test -- sh
/ # 
```

Now, I'm inside our container, let's run some tests.

First, print the contents of the `/var/run/secrets/kubernetes.io/serviceaccount/token` file.

```bash
/ # cat /var/run/secrets/kubernetes.io/serviceaccount/token

eyJhbGciOiJSUzI1NiIsImtpZCI6IncwY3FpcXhvZGt1SFlGelNQa1FwenFMcmpoeEFkVi1McjFYcTZVTEh3X1kifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzk3Njg0MTc3LCJpYXQiOjE3NjYxNDgxNzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiMGQyMTFiNGEtNjVjMi00ODEyLWIwYjEtNGUzY2I2NzI5ZGMzIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoia2luZC1jb250cm9sLXBsYW5lIiwidWlkIjoiNWU0MmM4M2YtMmI2NC00ZjU3LWEyZGMtMjI3M2ZmZjk3ZTBlIn0sInBvZCI6eyJuYW1lIjoiand0LXRlc3QiLCJ1aWQiOiJlNDdmMDVlZi00MWMzLTRmNDctYTdmNC01MDc1ZmIzZGQ2ZDMifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiI4Y2ZhNWYxNS0wOWJhLTRmM2QtODE2Ny02OGFhNjE5ZjRmN2YifSwid2FybmFmdGVyIjoxNzY2MTUxNzg0fSwibmJmIjoxNzY2MTQ4MTc3LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWZhdWx0In0.PA010UKl5ldQCOAk5s-iNRHEsbxkyIscTUsNn1c3hE9TL-uTCTl7_7QnI8-NOmx5Qjj7GPvF2QaHCeynOXlLq-Nt5mcvnOb6IipTfcH0Mfa7OCBufgPo82ggUA7T09kwcs7pmxZoL_lHxBBElFOMl9cMyhYO7I46JZ_AmvmzO4ctD3_ojQ6cyciXx4YZt78IwbM9QdM24e64BjyI_rdCGk3Y8990zodydn447VP9V6UAVQJJV49eleUnWMnQHTc3Z8UGjmawLeSaDQTqxXQ_fr9YTpHwbA_MqmXggFAmVIVQo0hTjfZxtcxuJe-8mM69Lm9krNJ7PsEuQeUB_9WyxA
```

Now `base64` decode the above token I got.

```bash
❯ echo "eyJhbGciOiJSUzI1NiIsImtpZCI6IncwY3FpcXhvZGt1SFlGelNQa1FwenFMcmpoeEFkVi1McjFYcTZVTEh3X1kifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzk3Njg0MTc3LCJpYXQiOjE3NjYxNDgxNzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiMGQyMTFiNGEtNjVjMi00ODEyLWIwYjEtNGUzY2I2NzI5ZGMzIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoia2luZC1jb250cm9sLXBsYW5lIiwidWlkIjoiNWU0MmM4M2YtMmI2NC00ZjU3LWEyZGMtMjI3M2ZmZjk3ZTBlIn0sInBvZCI6eyJuYW1lIjoiand0LXRlc3QiLCJ1aWQiOiJlNDdmMDVlZi00MWMzLTRmNDctYTdmNC01MDc1ZmIzZGQ2ZDMifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiI4Y2ZhNWYxNS0wOWJhLTRmM2QtODE2Ny02OGFhNjE5ZjRmN2YifSwid2FybmFmdGVyIjoxNzY2MTUxNzg0fSwibmJmIjoxNzY2MTQ4MTc3LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWZhdWx0In0.PA010UKl5ldQCOAk5s-iNRHEsbxkyIscTUsNn1c3hE9TL-uTCTl7_7QnI8-NOmx5Qjj7GPvF2QaHCeynOXlLq-Nt5mcvnOb6IipTfcH0Mfa7OCBufgPo82ggUA7T09kwcs7pmxZoL_lHxBBElFOMl9cMyhYO7I46JZ_AmvmzO4ctD3_ojQ6cyciXx4YZt78IwbM9QdM24e64BjyI_rdCGk3Y8990zodydn447VP9V6UAVQJJV49eleUnWMnQHTc3Z8UGjmawLeSaDQTqxXQ_fr9YTpHwbA_MqmXggFAmVIVQo0hTjfZxtcxuJe-8mM69Lm9krNJ7PsEuQeUB_9WyxA" | base64 -d

{"alg":"RS256","kid":"w0cqiqxodkuHYFzSPkQpzqLrjhxAdV-Lr1Xq6ULHw_Y"}
base64: invalid input
```

Ah, I got an invalid input.

But what I learnt is that a JWT token is a three part thing.  
Each part is a Base64 encoded blob, separated (or joined by a dot).  
So, a full token will look something like:

```
<base64url(header)>.<base64url(payload)>.<base64url(signature)>
```

What happened in our first attempt at decoding the ServiceAccount token is -  
it just decoded the first blob and then reached a dot, and failed there.

So, if I divide the above token into 3 parts now, it will be:

Part 1:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IncwY3FpcXhvZGt1SFlGelNQa1FwenFMcmpoeEFkVi1McjFYcTZVTEh3X1kifQ
```

Part 2:

```
eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzk3Njg0MTc3LCJpYXQiOjE3NjYxNDgxNzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiMGQyMTFiNGEtNjVjMi00ODEyLWIwYjEtNGUzY2I2NzI5ZGMzIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoia2luZC1jb250cm9sLXBsYW5lIiwidWlkIjoiNWU0MmM4M2YtMmI2NC00ZjU3LWEyZGMtMjI3M2ZmZjk3ZTBlIn0sInBvZCI6eyJuYW1lIjoiand0LXRlc3QiLCJ1aWQiOiJlNDdmMDVlZi00MWMzLTRmNDctYTdmNC01MDc1ZmIzZGQ2ZDMifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiI4Y2ZhNWYxNS0wOWJhLTRmM2QtODE2Ny02OGFhNjE5ZjRmN2YifSwid2FybmFmdGVyIjoxNzY2MTUxNzg0fSwibmJmIjoxNzY2MTQ4MTc3LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWZhdWx0In0
```

Part 3:

```
PA010UKl5ldQCOAk5s-iNRHEsbxkyIscTUsNn1c3hE9TL-uTCTl7_7QnI8-NOmx5Qjj7GPvF2QaHCeynOXlLq-Nt5mcvnOb6IipTfcH0Mfa7OCBufgPo82ggUA7T09kwcs7pmxZoL_lHxBBElFOMl9cMyhYO7I46JZ_AmvmzO4ctD3_ojQ6cyciXx4YZt78IwbM9QdM24e64BjyI_rdCGk3Y8990zodydn447VP9V6UAVQJJV49eleUnWMnQHTc3Z8UGjmawLeSaDQTqxXQ_fr9YTpHwbA_MqmXggFAmVIVQo0hTjfZxtcxuJe-8mM69Lm9krNJ7PsEuQeUB_9WyxA
```

### Ok, now let's decode them one by one again!

_**Decode Part 1:**_

```bash
❯ echo "eyJhbGciOiJSUzI1NiIsImtpZCI6IncwY3FpcXhvZGt1SFlGelNQa1FwenFMcmpoeEFkVi1McjFYcTZVTEh3X1kifQ" | base64 -d | jq .
{
  "alg": "RS256",
  "kid": "w0cqiqxodkuHYFzSPkQpzqLrjhxAdV-Lr1Xq6ULHw_Y"
}
```

The first key-value pair `"alg": "RS256"` in the output tells us, that this JWT token (which I will get in the next Part 2 decoding) was signed using a RS256 (RSA + SHA-256) algorithm.  
So, if I need to verify the token signature, I know what is the algorithm used to sign it.

And the second key-value pair, the `"kid": "w0cqiqxodkuHYFzSPkQpzqLrjhxAdV-Lr1Xq6ULHw_Y"` part.  
This is a hint to figure out which RS256 (RSA + SHA-256) private/pubic key pair was actually used to sign.  
The value I see in front of `kid` is going to help us to figure out the public key of this pair.  

But where to find the Public Key? That information, I will get in the next part.

---


_**Decode Part 2:**_

```bash
❯ echo "eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzk3Njg0MTc3LCJpYXQiOjE3NjYxNDgxNzcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiMGQyMTFiNGEtNjVjMi00ODEyLWIwYjEtNGUzY2I2NzI5ZGMzIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoia2luZC1jb250cm9sLXBsYW5lIiwidWlkIjoiNWU0MmM4M2YtMmI2NC00ZjU3LWEyZGMtMjI3M2ZmZjk3ZTBlIn0sInBvZCI6eyJuYW1lIjoiand0LXRlc3QiLCJ1aWQiOiJlNDdmMDVlZi00MWMzLTRmNDctYTdmNC01MDc1ZmIzZGQ2ZDMifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiI4Y2ZhNWYxNS0wOWJhLTRmM2QtODE2Ny02OGFhNjE5ZjRmN2YifSwid2FybmFmdGVyIjoxNzY2MTUxNzg0fSwibmJmIjoxNzY2MTQ4MTc3LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWZhdWx0In0" | base64 -d | jq .

{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1797684177,
  "iat": 1766148177,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "jti": "0d211b4a-65c2-4812-b0b1-4e3cb6729dc3",
  "kubernetes.io": {
    "namespace": "default",
    "node": {
      "name": "kind-control-plane",
      "uid": "5e42c83f-2b64-4f57-a2dc-2273fff97e0e"
    },
    "pod": {
      "name": "jwt-test",
      "uid": "e47f05ef-41c3-4f47-a7f4-5075fb3dd6d3"
    },
    "serviceaccount": {
      "name": "default",
      "uid": "8cfa5f15-09ba-4f3d-8167-68aa619f4f7f"
    },
    "warnafter": 1766151784
  },
  "nbf": 1766148177,
  "sub": "system:serviceaccount:default:default"
}
```

Ok, this is the JWT token.

- The `"iss": "https://kubernetes.default.svc.cluster.local"` is what issued this JWT token. So, it's the Issuer.  

  This is what I will use (later) to figure out the location of the public key.
  
- The `"aud": ["https://kubernetes.default.svc.cluster.local"]` tells us that who is the intended audience for this token.

- The `"sub": "system:serviceaccount:default:default"` tells us who is the subject of this token.  

  As in this token represents this service account (called `default` in the `default` namespace).

- The `"iat": 1766148177` part stands for Issued-at, and so the value is a timestamp for when this token was issued (in unix format).

- The `"nbf": 1766148177` part stands for "not before" meaning this token can't be used before this time.

  In this example, it is matching the "Issued-at" time, but I'm assuming it can be configured (I don't know how at this point).

- The `"exp": 1797684177,` part stands for "Expiration", basically again it is a timestamp for when this token will expire.

- The `"jti": "0d211b4a-65c2-4812-b0b1-4e3cb6729dc3"` is a unique ID for this JWT token.

- And then the following part is a Kubernetes-specific entity, not really a standard JWT token field.

  In this case, it's giving Kubernetes some metadata information, for which namespace, and which particular instances of node, pod, etc are tied to this ServiceAccount.

  And the `warnafter` bit tells when to automatically rotate this token.

  ```bash
      "kubernetes.io": {
        "namespace": "default",
        "node": {
          "name": "kind-control-plane",
          "uid": "5e42c83f-2b64-4f57-a2dc-2273fff97e0e"
        },
        "pod": {
          "name": "jwt-test",
          "uid": "e47f05ef-41c3-4f47-a7f4-5075fb3dd6d3"
        },
        "serviceaccount": {
          "name": "default",
          "uid": "8cfa5f15-09ba-4f3d-8167-68aa619f4f7f"
        },
        "warnafter": 1766151784
      },
    ```

---

_**And finally, the Part 3 now:**_

```bash
echo "PA010UKl5ldQCOAk5s-iNRHEsbxkyIscTUsNn1c3hE9TL-uTCTl7_7QnI8-NOmx5Qjj7GPvF2QaHCeynOXlLq-Nt5mcvnOb6IipTfcH0Mfa7OCBufgPo82ggUA7T09kwcs7pmxZoL_lHxBBElFOMl9cMyhYO7I46JZ_AmvmzO4ctD3_ojQ6cyciXx4YZt78IwbM9QdM24e64BjyI_rdCGk3Y8990zodydn447VP9V6UAVQJJV49eleUnWMnQHTc3Z8UGjmawLeSaDQTqxXQ_fr9YTpHwbA_MqmXggFAmVIVQo0hTjfZxtcxuJe-8mM69Lm9krNJ7PsEuQeUB_9WyxA" | base64 -d
5�B��W�$�base64: invalid input
```

I got some some random binary code here.

This I learnt is a raw RSA signature, that is used to sign the first 2 parts (Header and Payload) of the token.

I learnt it's something like this:

```
RSA-SIGN(
  SHA256(
    base64url(header) + "." + base64url(payload)
  )
)
```

---

### where to find the Public key (used to sign the JWT token)?

I saw in Part 2 decoded output this entry about who issued the token.

```
"iss": "https://kubernetes.default.svc.cluster.local",
```

Let's see if I can find out some information from this Issuer url.



But one thing to note, before I make any request.

The Issuer of a JWT token stores the information that I am looking for at a path:

```
https://<url-of-the-issuer>/.well-known/openid-configuration
```

Now, back to our pod container.

```bash
❯ kubectl exec -it jwt-test -- sh
/ # wget https://kubernetes.default.svc.cluster.local
Connecting to kubernetes.default.svc.cluster.local (10.96.0.1:443)
wget: note: TLS certificate validation not implemented
wget: server returned error: HTTP/1.1 403 Forbidden
```

ok, when I tried to hit the `https://kubernetes.default.svc.cluster.local` url, I got `403 Forbidden`.

So, I need some credentials.

You know what, here is what the Service Account token is used for.

I will pass the token as a Autherization header in our request.

Let's try again.

```bash
/ # TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

/ # wget --header="Authorization: Bearer $TOKEN" https://kubernetes.default.svc.cluster.local/.well-known/openid-configuration --no-check-certificate

Connecting to kubernetes.default.svc.cluster.local (10.96.0.1:443)
saving to 'openid-configuration'
openid-configuration 100% |********************************************************************************************|   236  0:00:00 ETA
'openid-configuration' saved

/ # cat openid-configuration | jq .
{
  "issuer": "https://kubernetes.default.svc.cluster.local",
  "jwks_uri": "https://172.20.0.2:6443/openid/v1/jwks",
  "response_types_supported": [
    "id_token"
  ],
  "subject_types_supported": [
    "public"
  ],
  "id_token_signing_alg_values_supported": [
    "RS256"
  ]
}
```

Ok, look at this part - `"jwks_uri": "https://172.20.0.2:6443/openid/v1/jwks"`.  
The "jwks" in the `jwks_url` stands for "JSON Web Key Set".  
This is what holds a collection of cryptographic public keys, used primarily for verifying digital signatures on JWT tokens.

So, I have almost gotten what I need.  
Let's hit this JWKS URL now.

```bash
/ # wget --header="Authorization: Bearer $TOKEN" https://172.20.0.2:6443/openid/v1/jwks --no-check-certificate
Connecting to 172.20.0.2:6443 (172.20.0.2:6443)
saving to 'jwks'
jwks                 100% |********************************************************************************************|   462  0:00:00 ETA
'jwks' saved

/# cat jwks | jq .
{
  "keys": [
    {
      "use": "sig",
      "kty": "RSA",
      "kid": "w0cqiqxodkuHYFzSPkQpzqLrjhxAdV-Lr1Xq6ULHw_Y",
      "alg": "RS256",
      "n": "vKsQjvpHQWbez2dLiTb2aJp36SKpVWvk-egE1pRertMJmtq3eeDPskb8n_msAWY4GKIMx3RnmfKBMbs_WHAkVt681cH0AzF5CR_oUtJ0Unde1rInUls5nxQcQ7_cCjApyKQlY5x5Z_vASyh7fOvMKUWmfLJt7M20hDoEvlM0WF9kUeqAgexBXlFv106qc-3CoO2-HPN6mlOn8WqHd-Ky_jQaj5xm__A0o04H7JEu09n7_Z9Rws9TFqBHaGCXwio3cozh2Bjv6da7rmyZUSp7ztH_4UcfYQgt5iJnxUdsjD7vXnyWFwvefs-6Wn6vlRp4fVmCfNrkzDL7QPWsjJJoWQ",
      "e": "AQAB"
    }
  ]
}
```

Voila, I got it.

See, the `kid` and `alg` matches exactly what I got in the decoded output of Part 1 of the JWT token.

```bash
{
  "alg": "RS256",
  "kid": "w0cqiqxodkuHYFzSPkQpzqLrjhxAdV-Lr1Xq6ULHw_Y"
}
```

So, I think the last part left for us now is to understand how can i use this to verify the signature now on the token?

### let's verify the token!

So, to verify the token. I will need the following bit I got from the `jwks_uri` output.

```bash
"n": "vKsQjvpHQWbez2dLiTb2aJp36SKpVWvk-egE1pRertMJmtq3eeDPskb8n_msAWY4GKIMx3RnmfKBMbs_WHAkVt681cH0AzF5CR_oUtJ0Unde1rInUls5nxQcQ7_cCjApyKQlY5x5Z_vASyh7fOvMKUWmfLJt7M20hDoEvlM0WF9kUeqAgexBXlFv106qc-3CoO2-HPN6mlOn8WqHd-Ky_jQaj5xm__A0o04H7JEu09n7_Z9Rws9TFqBHaGCXwio3cozh2Bjv6da7rmyZUSp7ztH_4UcfYQgt5iJnxUdsjD7vXnyWFwvefs-6Wn6vlRp4fVmCfNrkzDL7QPWsjJJoWQ",
"e": "AQAB"
```

Note: all notes this point onwards is me copy/pasting instructions I got from docs or otherwise tinkering with AI. 

The `n` and `e` are respectively called the "modulus" and "public exponent" which is what I will use to contruct the RSA public key.

Below mathematics is what is used to convert the "n" and "e" to a public key.

What the following process does is 5 things for the "n" and "e" values I got:
- convert them from `base64url` to `base64` to `binary`.
- then interpret them as Integers
- then wrap them into something called `ASN.1` structure
- then do something called DER (Distinguished Encoding Rules) encoding this above `ASN.1` structure
- and finall wrap that `DER` into a PEM.

Once I got the PEM version, at that point, openssl will be able to use it.


I'm doing the below steps on my host machine, because i need `openssl`, `base64`, `xxd`, and `jq`, which are not present in the container.

Ok, first step - we decode `n` and `e` to binary, replacing the following and adding padding (`=`):
- `-` to `+`
- `_` to `/`


```bash
❯ n="vKsQjvpHQWbez2dLiTb2aJp36SKpVWvk-egE1pRertMJmtq3eeDPskb8n_msAWY4GKIMx3RnmfKBMbs_WHAkVt681cH0AzF5CR_oUtJ0Unde1rInUls5nxQcQ7_cCjApyKQlY5x5Z_vASyh7fOvMKUWmfLJt7M20hDoEvlM0WF9kUeqAgexBXlFv106qc-3CoO2-HPN6mlOn8WqHd-Ky_jQaj5xm__A0o04H7JEu09n7_Z9Rws9TFqBHaGCXwio3cozh2Bjv6da7rmyZUSp7ztH_4UcfYQgt5iJnxUdsjD7vXnyWFwvefs-6Wn6vlRp4fVmCfNrkzDL7QPWsjJJoWQ"

❯ echo $n | tr '_-' '/+' | base64 -d > n.bin

❯ e="AQAB"

❯ echo $e | base64 -d > e.bin
```

ok, now do:

```bash
❯ xxd e.bin
00000000: 0100 01                                  ...
```

This represents the number `65537`.

Now, we need to convert these 2 `n.bin` and `e.bin` binaries to Hexadecimal strings.


```bash
❯ xxd -p n.bin | tr -d '\n'
bcab108efa474166decf674b8936f6689a77e922a9556be4f9e804d6945eaed3099adab779e0cfb246fc9ff9ac01663818a20cc7746799f28131bb3f58702456debcd5c1f4033179091fe852d27452775ed6b227525b399f141c43bfdc0a3029c8a425639c7967fbc04b287b7cebcc2945a67cb26deccdb4843a04be5334585f6451ea8081ec415e516fd74eaa73edc2a0edbe1cf37a9a53a7f16a8777e2b2fe341a8f9c66fff034a34e07ec912ed3d9fbfd9f51c2cf5316a047686097c22a37728ce1d818efe9d6bbae6c99512a7bced1ffe1471f61082de62267c5476c8c3eef5e7c96170bde7ecfba5a7eaf951a787d59827cdae4cc32fb40f5ac8c926859

❯ xxd -p e.bin
010001
```


Now, we need to convert these into an `ASN.1` format.

so, we create a file called `rsa.asn1`, with the following contents:

```bash
asn1=SEQUENCE:rsa_key

[rsa_key]
modulus=INTEGER:0x<PASTE_HEX_OF_N_HERE>
publicExponent=INTEGER:0x<PASTE_HEX_OF_E_HERE>
```

so, the final version would look something like:

```bash
❯ cat rsa.asn1 
asn1=SEQUENCE:rsa_key

[rsa_key]
modulus=INTEGER:0xbcab108efa474166decf674b8936f6689a77e922a9556be4f9e804d6945eaed3099adab779e0cfb246fc9ff9ac01663818a20cc7746799f28131bb3f58702456debcd5c1f4033179091fe852d27452775ed6b227525b399f141c43bfdc0a3029c8a425639c7967fbc04b287b7cebcc2945a67cb26deccdb4843a04be5334585f6451ea8081ec415e516fd74eaa73edc2a0edbe1cf37a9a53a7f16a8777e2b2fe341a8f9c66fff034a34e07ec912ed3d9fbfd9f51c2cf5316a047686097c22a37728ce1d818efe9d6bbae6c99512a7bced1ffe1471f61082de62267c5476c8c3eef5e7c96170bde7ecfba5a7eaf951a787d59827cdae4cc32fb40f5ac8c926859
publicExponent=INTEGER:0x010001
```

With this `ASN.1` format available now, we can generate a DER encoded value.

```bash
❯ openssl asn1parse \
   -genconf rsa.asn1 \
   -out rsa_pub.der
   
    0:d=0  hl=4 l= 266 cons: SEQUENCE          
    4:d=1  hl=4 l= 257 prim: INTEGER           :BCAB108EFA474166DECF674B8936F6689A77E922A9556BE4F9E804D6945EAED3099ADAB779E0CFB246FC9FF9AC01663818A20CC7746799F28131BB3F58702456DEBCD5C1F4033179091FE852D27452775ED6B227525B399F141C43BFDC0A3029C8A425639C7967FBC04B287B7CEBCC2945A67CB26DECCDB4843A04BE5334585F6451EA8081EC415E516FD74EAA73EDC2A0EDBE1CF37A9A53A7F16A8777E2B2FE341A8F9C66FFF034A34E07EC912ED3D9FBFD9F51C2CF5316A047686097C22A37728CE1D818EFE9D6BBAE6C99512A7BCED1FFE1471F61082DE62267C5476C8C3EEF5E7C96170BDE7ECFBA5A7EAF951A787D59827CDAE4CC32FB40F5AC8C926859
  265:d=1  hl=2 l=   3 prim: INTEGER           :010001
```

Before we move ahead, I want to show why we are doing all this.

Because, the RSA Public key will be of this structure:

```
RSAPublicKey ::= SEQUENCE {
  modulus           INTEGER (n),
  publicExponent    INTEGER (e)
}
```

so, if you see the output of our DER encoding command, we see somethings that are needed in the above RSA Public Key structure.

In the output, we have a `SEQUENCE` consisting of two prime INTEGERS.  
That's exactly what we need.

Now, we are almost close to the final part, of actually converting the "n" and "e" to a RSA Public key.

```bash
❯ openssl rsa \
   -pubin \
   -inform DER \
   -in rsa_pub.der \
   -outform PEM \
   -out public.pem
writing RSA key

❯ cat public.pem 
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvKsQjvpHQWbez2dLiTb2
aJp36SKpVWvk+egE1pRertMJmtq3eeDPskb8n/msAWY4GKIMx3RnmfKBMbs/WHAk
Vt681cH0AzF5CR/oUtJ0Unde1rInUls5nxQcQ7/cCjApyKQlY5x5Z/vASyh7fOvM
KUWmfLJt7M20hDoEvlM0WF9kUeqAgexBXlFv106qc+3CoO2+HPN6mlOn8WqHd+Ky
/jQaj5xm//A0o04H7JEu09n7/Z9Rws9TFqBHaGCXwio3cozh2Bjv6da7rmyZUSp7
ztH/4UcfYQgt5iJnxUdsjD7vXnyWFwvefs+6Wn6vlRp4fVmCfNrkzDL7QPWsjJJo
WQIDAQAB
-----END PUBLIC KEY-----
```

Hurrah! Hurrah! Hurrah! We have the public key! finally!

ok, but let's verify once

```bash
❯ openssl rsa -pubin -in public.pem -text -noout
Public-Key: (2048 bit)
Modulus:
    00:bc:ab:10:8e:fa:47:41:66:de:cf:67:4b:89:36:
    f6:68:9a:77:e9:22:a9:55:6b:e4:f9:e8:04:d6:94:
    5e:ae:d3:09:9a:da:b7:79:e0:cf:b2:46:fc:9f:f9:
    ac:01:66:38:18:a2:0c:c7:74:67:99:f2:81:31:bb:
    3f:58:70:24:56:de:bc:d5:c1:f4:03:31:79:09:1f:
    e8:52:d2:74:52:77:5e:d6:b2:27:52:5b:39:9f:14:
    1c:43:bf:dc:0a:30:29:c8:a4:25:63:9c:79:67:fb:
    c0:4b:28:7b:7c:eb:cc:29:45:a6:7c:b2:6d:ec:cd:
    b4:84:3a:04:be:53:34:58:5f:64:51:ea:80:81:ec:
    41:5e:51:6f:d7:4e:aa:73:ed:c2:a0:ed:be:1c:f3:
    7a:9a:53:a7:f1:6a:87:77:e2:b2:fe:34:1a:8f:9c:
    66:ff:f0:34:a3:4e:07:ec:91:2e:d3:d9:fb:fd:9f:
    51:c2:cf:53:16:a0:47:68:60:97:c2:2a:37:72:8c:
    e1:d8:18:ef:e9:d6:bb:ae:6c:99:51:2a:7b:ce:d1:
    ff:e1:47:1f:61:08:2d:e6:22:67:c5:47:6c:8c:3e:
    ef:5e:7c:96:17:0b:de:7e:cf:ba:5a:7e:af:95:1a:
    78:7d:59:82:7c:da:e4:cc:32:fb:40:f5:ac:8c:92:
    68:59
Exponent: 65537 (0x10001)
```

### We got the Public Key! Let's verify the JWT token now.

Ok, I'm back to doing things by myself now.

Remember from the Part 1 and Part 2 of the token were actually the "Header" and "Payload". And "Part3", the signature.

What we have to verify is Part 1 and Part 2, which is what is signed by Part 3.

So, let's sort out the the required parts. 

```bash
❯ TOKEN=eyJhbGciOiJSUzI1NiIsImtpZCI6IncwY3FpcXhvZGt1SFlGelNQa1FwenFMcmpoeEFkVi1McjFYcTZVTEh3X1kifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzk3Njg3ODgxLCJpYXQiOjE3NjYxNTE4ODEsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiNmJiNzc0MjMtNmE3Yi00MjBmLTg1MDgtOTUzY2Y0YmE5MWZhIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoia2luZC1jb250cm9sLXBsYW5lIiwidWlkIjoiNWU0MmM4M2YtMmI2NC00ZjU3LWEyZGMtMjI3M2ZmZjk3ZTBlIn0sInBvZCI6eyJuYW1lIjoiand0LXRlc3QiLCJ1aWQiOiI4M2E5YmE4OC0yOTg5LTRlYmItYTRiZS04ZjA0ODY2ZmM4OGEifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiI4Y2ZhNWYxNS0wOWJhLTRmM2QtODE2Ny02OGFhNjE5ZjRmN2YifSwid2FybmFmdGVyIjoxNzY2MTU1NDg4fSwibmJmIjoxNzY2MTUxODgxLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWZhdWx0In0.ByKLRb2376aYeAmOWL1LTHsRmwgjWYp3kklUmoDcAzfMXWOBOcU_R4iXC4UqM5iwpEw_lWhDgEndGghiUg6HFau0rtj5VxFiFWwXkxSfzYNvxzW_nO3uFZlI4R3tWJqIeLMX3hqaWQGb_LvfvjI1My6XNZhkt8UTByUg3nKTmtMWmg-9-XKMylmD078vT4n8f0nSL6YlchJuTFivWc1lGE-FrZmWk6WeiRtH_jTyXp95dhg_Chf566otezUrPE8ern-8sI0rSVDzvLNsF4YvL9IXx2JQn57QR_Pr3otFXpeUTgj5oBUllsCTrA2xpXRmWxUD9qoncjviVkAkcj1fiw

❯ echo -n "$TOKEN" | cut -d. -f1,2 > signed-data.txt

❯ echo "$TOKEN" | cut -d. -f3 | tr '_-' '/+' | base64 -d > signature.bin
```

ok, now, we verify the signed-data.txt with the signature.bin

```bash
❯ openssl dgst -sha256 \
∙   -verify public.pem \
∙   -signature signature.bin \
∙   signed-data.txt
Verification failure
40F7505CB27F0000:error:02000068:rsa routines:ossl_rsa_verify:bad signature:crypto/rsa/rsa_sign.c:442:
40F7505CB27F0000:error:1C880004:Provider routines:rsa_verify_directly:RSA lib:providers/implementations/signature/rsa_sig.c:1043:
```

and I failed. 😂

Goodness, the process was tiring, so, I'm not repeating it right now.

I'll come back to it and see at what place, I made mistakes.

But this effort was to learn how it is done, i.e., how a signature verification happens.

And also, I should not forget it all started with me just trying to understand whether a Kubernetes Service Account is a JWT token.

I got my answer and I learnt so much more.

There's an article that I haven't read just yet, but I was recommended to read (by [Lucas Käldström](https://github.com/luxas)).  
Because, we didn't end with a shiny green "verified" message, I will leave you to read that brilliant article -   
[RSA Signing is Not RSA Decryption](https://www.cs.cornell.edu/courses/cs5430/2015sp/notes/rsa_sign_vs_dec.php)

Thank you, if you followed so far. :)
