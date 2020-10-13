#### Reset the default password


Step 1 : Use the py oneliner script to generate the bcrypt hash


```bash

echo "argocd-pass" | python -c "import bcrypt; import sys; salt = bcrypt.gensalt();  print(bcrypt.hashpw(sys.stdin.read(),salt))"
$2b$12$VTG10ibffUzK4e3CEFv0NOUfD5RIN.WGpjHsAXqp.hK6hd2MsTOrq

# Patch the secret

kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2b$12$VTG10ibffUzK4e3CEFv0NOUfD5RIN.WGpjHsAXqp.hK6hd2MsTOrq",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'

``` 

Access using admin/argocd-pass