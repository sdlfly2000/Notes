# Docker Private Registry

## List All Images
```
curl https://registry.activator.com/v2/_catalog
```

## List All Tags of Image
```
curl https://registry.activator.com/v2/authservice/authservice/tags/list
```

## Get digest of image:tag
```
curl -sS -H "Accept: application/vnd.docker.distribution.manifest.v2+json" -I https://registry.activator.com/v2/authservice/authservice/manifests/20251104-000457
```
## Delete Image
```
curl -H "Accept: application/vnd.docker.distribution.manifest.v2+json" -X DELETE https://registry.activator.com/v2/authservice/authservice/manifests/sha256:33758ba5b2ae635436fd452032801554c2fdbdcff07168bd24b1975f19fd4c6e
```

## Clean up in docker
```
bin/registry garbage-collect /etc/distribution/config.yml 
```