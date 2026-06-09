# CNPG auto restore operator

Operator that automatically provisions CNPG clusters with `bootstrap` either set to `initdb` or set to `restore`, depending on whether there is a backup present in S3 that can be restored from.