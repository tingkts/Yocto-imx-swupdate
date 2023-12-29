In default, imx swupdate has already put their default sign key pair in the codebase.

private key :  swupdate-scripts/update_image_build/priv.pem

public key : sources/meta-swupdate-imx/recipes-support/swupdate/swupdate/swu_public.pem

pass phrase："test"

Supported SSL modes are RSA-PKCS-1.5, RSA-PSS and DGST.

---

Here is the demo of to use self created RSA-PKCS-1.5 sign key：

1. Generate key :

```
    Private key: openssl genrsa -aes256 -out priv.pem
    Public key: openssl rsa -in priv.pem -out swu_public.pem -outform PEM -pubout
```

Put this key pair in the path, `swupdate-scripts/update_image_build/priv.pem`, `sources/meta-swupdate-imx/recipes-support/swupdate/swupdate/swu_public.pem`, respectively.



2. rebuild yocto, bitbake imx-image-full && bitbake swupdate-image, it's the same as readme guidance.


3. rebuild swupdate base/update image, it's the same as readme guidance.

----
